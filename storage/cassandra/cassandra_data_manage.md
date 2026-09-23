# Cassandra 데이터 조회·관리와 성능 특성

## 1. 핵심 요약

Cassandra는 LSM(Log-Structured Merge) 계열의 분산 데이터베이스다. CQL 관점에서는 `INSERT`와 `UPDATE`가 현재 데이터를 갱신하는 upsert처럼 보이지만, 디스크의 기존 파일을 직접 수정하지 않는다. 새 변경 사항을 메모리에 기록한 뒤 새 SSTable로 저장하며, 한 logical row의 과거 값·최신 값·삭제 마커가 잠시 여러 SSTable에 공존할 수 있다.

- **Read**: Memtable과 여러 SSTable의 버전을 읽어 timestamp와 tombstone 규칙으로 reconcile(병합)하여 현재 상태를 반환한다.
- **Write**: 기존 디스크 파일을 찾아 수정하지 않고 append 중심으로 처리하므로 대량 분산 write에 유리하다.
- **Compaction**: 여러 SSTable을 병합하고, 안전한 과거 버전과 tombstone을 제거해 read 비용과 디스크 사용량을 관리한다.

이는 이벤트 소싱(event sourcing)은 아니다. 변경 이력이 영구 보존되는 것이 아니라, compaction 후 안전한 과거 버전은 제거된다.

## 2. 데이터 모델: partition key와 clustering key

예시 테이블:

```sql
CREATE TABLE events_by_user (
  user_id text,
  event_time timestamp,
  event_id timeuuid,
  payload text,
  PRIMARY KEY ((user_id), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

| 구성 | 예시 | 역할 |
|---|---|---|
| Partition key | `user_id` | 어느 token/replica node에 저장할지를 결정 |
| Clustering key | `event_time`, `event_id` | 한 partition 안에서 row의 정렬 순서와 범위 조회 방식을 결정 |

동일한 `user_id`의 row는 하나의 partition에 속한다. 그 partition 내부의 row는 clustering key 순서로 정렬된다. 위 예시에서는 최신 이벤트가 앞에 위치한다.

```text
partition: user_id = 'u-42'

2026-09-20 10:03 | event 103
2026-09-20 10:02 | event 102
2026-09-20 10:01 | event 101
```

partition key는 “어느 node에서 읽을지”, clustering key는 “그 node에서 어느 구간을 읽을지”를 담당한다. Cassandra에서 좋은 성능의 출발점은 조회가 partition key를 정확히 지정하고, 필요한 clustering-key 범위만 읽도록 모델링하는 것이다.

## 3. 요청이 replica를 찾는 과정

클라이언트는 어느 Cassandra node에든 요청할 수 있다. 요청을 받은 node는 해당 요청의 **coordinator**가 된다.

```text
partition key
  → hash하여 token 계산
  → token ring과 replication strategy로 replica node 결정
  → replica에 read/write 요청 전달
  → Consistency Level을 만족하는 응답을 수집
  → 클라이언트에 결과 반환
```

예를 들어 RF=3이라면 해당 token 범위의 데이터는 3개 replica에 존재한다. Cassandra는 모든 node를 검색하는 방식이 아니라, partition key로 계산한 token에 해당하는 replica만 대상으로 삼는다.

## 4. Write path와 Memtable

일반적인 write 흐름은 다음과 같다.

```text
Mutation
  → Commit Log에 순차 append (내구성)
  → Memtable에 반영 (메모리)
  → Memtable flush
  → 새 SSTable 생성
```

Memtable은 메모리의 정렬된 자료구조다. 따라서 아직 SSTable로 flush되지 않은 데이터도 read 결과에 포함된다.

```text
Read
  → 활성 Memtable 조회
  → flush 중인 immutable Memtable 조회
  → live SSTable 조회
  → 모든 결과를 reconcile하여 반환
```

flush 중인 Memtable도 새 SSTable이 read 대상으로 전환될 때까지 조회 대상에 남는다. 이 때문에 flush 전후에 데이터가 query 결과에서 사라지는 구간은 없다.

## 5. SSTable과 불변 저장 구조

SSTable(Sorted String Table)은 flush 또는 compaction 결과로 생성되는 불변 디스크 파일 집합이다.

```text
write → Memtable → flush → SSTable A
                    flush → SSTable B
                    flush → SSTable C

SSTable A + B + C → compaction → SSTable D
```

SSTable은 한 번 생성되면 수정되지 않는다. 동일 partition에 대한 insert/update/delete가 반복되면, 한 partition의 여러 버전이 Memtable 및 여러 SSTable에 분산될 수 있다.

```text
SSTable A: u-42.email = old@example.com, write timestamp=100
SSTable B: u-42.email = new@example.com, write timestamp=200
```

read 시에는 최신 write timestamp를 가진 값이 결과가 된다. row의 물리적 정렬은 clustering key 기준이고, **어떤 cell/row 버전이 최신인지의 판단은 write timestamp 기준**이라는 점을 구분해야 한다.

## 6. SSTable에서 partition을 찾는 방법

SSTable에는 실제 데이터 외에 partition 위치를 빠르게 찾기 위한 컴포넌트가 함께 생성된다. 최신 Cassandra SSTable 포맷은 Trie 기반 partition index를 사용할 수 있으며, 이전 포맷은 `Index.db`와 `Summary.db`를 사용했다. 구현은 버전에 따라 달라도, 불변 SSTable별로 lookup metadata를 갖는다는 원리는 같다.

```text
SSTable
├─ Data.db        실제 partition/row 데이터
├─ Filter.db      partition key Bloom filter
├─ Partitions.db  partition key prefix → 데이터 위치 (신규 포맷)
├─ Rows.db        wide partition 내부 row-block 탐색용
├─ Summary.db     partition index 탐색 보조 정보
└─ Statistics.db  timestamp, tombstone, TTL 등 통계
```

SSTable 내부 partition은 token 순서로 정렬되고, partition 내부 row는 clustering key 순서로 정렬된다. 상세 구성은 [Apache Cassandra Storage Engine 문서](https://cassandra.apache.org/doc/stable/cassandra/architecture/storage-engine.html)를 참고한다.

### `live SSTable`의 의미

**live SSTable**은 Cassandra가 현재 테이블의 유효한 데이터 파일로 채택하여 read path에 포함하는 SSTable이다. Compaction은 새 SSTable을 완성한 뒤 read 대상 목록을 원자적으로 교체한다.

```text
Compaction 전: A, B = live
A + B → C 생성
전환 후: C = live, A/B = obsolete
```

obsolete SSTable은 기존 read가 끝날 때까지, snapshot/backup이 참조하는 동안, 또는 파일 삭제가 완료되기 전까지 디스크에 잠시 남아 있을 수 있다. 일반 read는 obsolete SSTable이 아니라 현재의 live SSTable 집합만 대상으로 한다.

### SSTable 조회 절차

partition key 기반 read에서 replica는 현재 live SSTable **각각**에 대해 먼저 Bloom filter를 확인한다. 이는 `Data.db` 전체를 scan하는 작업이 아니다.

```text
각 live SSTable
  → Bloom filter 확인
      ├─ definitely not present: 제외
      └─ might be present
           → partition index 탐색
               ├─ 실제 partition 없음: false positive, 제외
               └─ 실제 partition 있음: Data.db의 해당 partition 범위만 read
```

따라서 다음은 구분해야 한다.

- 모든 live SSTable의 Bloom filter는 확인한다.
- partition index는 Bloom filter를 통과한 SSTable에만 확인한다.
- `Data.db`는 partition이 실제 존재하는 SSTable만 대상으로 필요한 partition/range만 읽는다.

같은 partition이 여러 SSTable에 실제 존재한다면 각 파일의 데이터를 읽고 병합해야 한다. SSTable 수가 많으면 Bloom filter 확인 자체는 가볍더라도 반복 비용, metadata cache 효율, 실제 병합 대상 수가 증가해 read latency에 불리할 수 있다.

### Bloom filter와 false positive

Bloom filter는 false negative가 없고 false positive만 가능하다.

```text
Bloom filter: "u-42가 있을 수도 있음"
partition index: "실제로는 없음"
→ false positive; Data.db의 partition data는 읽지 않고 제외
```

false positive는 정합성 오류가 아니라 추가 metadata lookup 비용이다. index가 page cache에 없다면 I/O가 발생할 수 있다.

Bloom filter는 개별 key를 삭제할 수 없다. 그러나 Cassandra는 하나의 Bloom filter를 계속 수정하는 구조가 아니다. SSTable flush 시 새 Bloom filter를 만들고, compaction 시 입력 SSTable을 폐기한 뒤 출력 SSTable에 대해 새 Bloom filter를 생성한다. tombstone과 과거 데이터가 purge되어 partition 자체가 출력 SSTable에서 사라지면 새 Bloom filter에도 그 key가 없다. 목표 false-positive 확률은 테이블의 `bloom_filter_fp_chance`로 조절하며, 더 낮은 값은 더 많은 메모리를 사용한다.

## 7. Delete와 tombstone

Cassandra의 `DELETE`는 기존 SSTable의 데이터를 즉시 지우지 않는다. 더 최신 timestamp를 가진 **tombstone**을 쓴다.

즉 tombstone은 기존 data에 붙이는 단순 marking이 아니라, storage engine이 삭제로 해석하는 **별도의 삭제 mutation(data entry)** 이다. `DELETE`도 insert/update와 마찬가지로 새로운 write로 처리된다.

```text
DELETE
  → Commit Log에 tombstone mutation append
  → Memtable에 tombstone 기록
  → flush 시 새 SSTable에 tombstone 저장
  → 이후 read/compaction에서 이전 data와 reconcile
```

tombstone은 애플리케이션이 일반 row로 조회하는 값이 아니라, Cassandra가 “이 timestamp 이전의 값은 반환하지 말라”고 해석하는 특수한 삭제 기록이다.

```text
SSTable A: (u-42, 10:02) = old event, timestamp=100
Memtable:  (u-42, 10:02) = tombstone, timestamp=200

read 결과: row 없음
```

read는 tombstone이 더 최신이라는 것을 확인하면 오래된 값을 결과에서 제외한다. tombstone은 replica 중 일부가 일시적으로 다운되어도 삭제 사실을 전파하여, 오래된 데이터가 repair 과정에서 다시 나타나는 zombie data를 막는다.

tombstone은 다음 상황에서 만들어질 수 있다.

- `DELETE` 문
- 특정 컬럼 삭제
- collection의 원소 삭제
- TTL 만료
- `null`로 기록되는 컬럼 삭제

## 8. Tombstone purge와 `gc_grace_seconds`

삭제 데이터와 tombstone은 삭제 직후 물리적으로 사라지지 않는다.

```text
1. SSTable A: old value
2. DELETE: SSTable B에 tombstone 기록
3. grace 기간 중 compaction: old value는 가려지지만 tombstone 유지
4. grace 기간 경과 후 안전한 compaction: old value와 tombstone을 함께 제거 가능
```

`gc_grace_seconds`는 tombstone이 제거 대상이 되기 전의 안전 보존 기간이다. 기본값은 864,000초(10일)다. 이 시간이 지나도 즉시 삭제되는 것은 아니다. tombstone이 가리는 오래된 데이터를 가진 SSTable까지 함께 확인·병합할 수 있는 compaction이 실행되어야 실제 disk 공간이 회수된다.

`gc_grace_seconds`를 단순히 낮추면 disk 공간은 빨리 회수될 수 있지만, 장기간 다운된 replica가 삭제 사실을 받지 못한 경우 오래된 값이 복구되어 zombie data가 생길 위험이 있다. 해당 값을 낮추기 전에는 repair 주기, 노드 장애 복구 시간, hinted handoff 및 데이터 정합성 운영 정책을 함께 검증해야 한다. [공식 tombstone 문서](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/tombstones.html)를 참고한다.

## 9. Tombstone scan과 read latency

**tombstone scan**은 query 결과를 만들기 위해 Cassandra가 조회 범위 내 tombstone을 대량으로 읽고 병합하지만, 그 tombstone들은 결과 row로 반환되지 않는 상황이다.

```text
최신 방향
event 1000 ~ 101: tombstone 900개
event 100  ~ 1:   live row

LIMIT 100
→ live row 100개를 찾기 위해 tombstone 900개를 건너뜀
```

`LIMIT 100`은 디스크 항목 100개를 읽으라는 뜻이 아니라, **live result 100개를 반환하라**는 뜻이다. 따라서 결과 row 수가 적거나 0개여도 query 범위에 tombstone이 많으면 비용이 클 수 있다.

다만 과거 tombstone이 많다고 해서 최근 100건 조회가 항상 느린 것은 아니다. clustering order와 query 방향이 맞아 최근의 live row부터 바로 찾을 수 있고 query 범위가 그 tombstone 위치까지 닿지 않는다면 직접 영향은 작다. 위험은 **쿼리가 실제로 스캔하는 범위**에 tombstone이 많이 포함될 때다.

tombstone scan의 비용은 다음과 같다.

- 더 많은 SSTable/데이터 블록 read
- SSTable 간 version 및 tombstone 병합 CPU 비용
- heap 사용과 GC 압력
- replica 응답 지연에 따른 coordinator latency 증가
- p95/p99 read latency 악화, 심하면 tombstone scan 임계치 초과로 read 실패

직접적인 판단 기준은 테이블 전체 tombstone 수가 아니라 **개별 query가 스캔하는 tombstone 수**다.

## 10. 대량 delete와 성능 저하 조건

### Partition별 tombstone이 적은 경우

delete 총량이 매우 커도 tombstone이 많은 partition에 균등하게 분산되고, query가 partition key와 좁은 clustering range를 사용한다면 tombstone scan 위험은 낮다.

```text
전체 삭제량: 큼
partition cardinality: 높음
각 partition의 tombstone: 적음
각 query의 scan 범위: 좁음
→ 직접적인 read-path tombstone 문제는 상대적으로 작음
```

그러나 전체 delete rate가 높으면 write/compaction path의 비용은 남는다.

### Range scan이 아닌 경우에도 생기는 비용

partition-key 기반 point/narrow read만 한다면 대량 delete의 주요 영향은 tombstone scan보다 다음 경로로 나타날 수 있다.

```text
대량 DELETE
→ tombstone write, replication, commit log, memtable 부담 증가
→ flush와 SSTable 생성 증가
→ compaction I/O·CPU·임시 disk 공간 사용 증가
→ foreground read/write와 resource 경쟁
→ p95/p99 latency 악화 가능
```

즉 range scan이 없다고 tombstone 문제가 완전히 사라지는 것은 아니다. 문제의 중심이 read-path tombstone scan에서 compaction·disk·write-path 부담으로 옮겨간다고 이해하는 것이 정확하다.

## 11. Compaction의 역할과 주의점

Compaction은 여러 SSTable의 동일 row/cell 버전을 병합해 최신 상태를 새 SSTable에 쓰고, 제거 가능한 과거 버전과 tombstone을 정리한다.

```text
SSTable A: old value
SSTable B: new value
SSTable C: tombstone
→ compaction
→ 최신 값 또는 안전하게 유지/제거된 tombstone만 담은 새 SSTable
```

Compaction의 이점:

- read 시 병합해야 할 SSTable 수 감소
- 오래된 version 및 purge 가능한 tombstone 제거
- disk 공간 회수

Compaction의 비용:

- 입력 SSTable read + 출력 SSTable write에 따른 높은 disk I/O
- CPU 사용
- compaction 중 원본과 결과 파일이 공존하는 임시 disk 공간
- foreground read/write와 자원 경쟁

따라서 compaction은 필수이지만, delete/TTL 생성량과 compaction 처리량의 균형이 무너지면 성능 저하가 발생한다. [공식 compaction 개요](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/compaction/overview.html)를 참고한다.

## 12. 운영에서 볼 지표와 해석

| 관측 항목 | 의미 및 경고 신호 |
|---|---|
| Live SSTable 수 | 많을수록 Bloom filter 검사 반복과 version 병합 후보가 증가. 같은 partition이 여러 SSTable에 있으면 실제 read amplification 증가 |
| SSTable tombstone 비율 | query 범위가 그 위치를 지날 때 버려야 할 데이터의 잠재량. 전체 평균만으로 query 영향은 단정 불가 |
| Query별 scanned tombstone warning/failure | 실제 tombstone scan 문제를 직접적으로 보여 주는 신호 |
| Read p95/p99 | 특정 partition 또는 시간 범위 query와 correlation 확인 |
| Compaction pending task/throughput | compaction이 write/delete/TTL 생성량을 따라가는지 확인 |
| Disk 사용률·I/O wait | compaction 경쟁과 공간 압박 여부 확인 |
| Partition별 크기·row 수·tombstone 수 | wide/hot partition 및 tombstone 집중 여부 확인 |

해석의 핵심은 다음이다.

> SSTable 수는 “확인·병합해야 할 후보 파일 수”의 위험 신호이고, tombstone 비율은 “그 파일을 읽을 때 버려야 할 데이터가 많을 가능성”의 위험 신호다. 실제 장애 가능성은 query별 partition 및 clustering-key 범위와 함께 판단해야 한다.

## 13. 실무 설계 체크리스트

- 모든 주요 query가 partition key를 지정하는가?
- partition이 과도하게 넓어지지 않는가? 필요하다면 시간 버킷 등으로 partition을 분할하는가?
- 최근 조회 방향과 clustering order가 일치하는가?
- `LIMIT` query가 live row를 채우기 위해 대량 tombstone을 건너뛰지 않는가?
- TTL/DELETE가 높은 테이블에 시간 특성에 맞는 compaction strategy를 사용하는가?
- repair가 `gc_grace_seconds` 내에 신뢰성 있게 완료되는가?
- `gc_grace_seconds`를 disk 회수 목적만으로 낮추지 않았는가?
- compaction backlog, disk headroom, read/write p99를 함께 모니터링하는가?

## 14. 최종 정리

> Cassandra는 기존 데이터를 제자리에서 갱신하지 않고, 새로운 mutation을 불변 SSTable 구조에 추가한다. Read는 Memtable과 live SSTable의 버전을 reconcile해 현재 상태를 제공하며, compaction은 안전한 과거 버전과 tombstone을 뒤늦게 정리한다.

이 구조는 write throughput과 수평 확장에 강점을 주지만, 과도한 SSTable 수, 넓은 partition, 대량 tombstone을 스캔하는 query, compaction backlog는 read/write latency를 악화시킬 수 있다.

## 15. Replica 불일치와 Repair

Cassandra는 일부 replica가 일시적으로 다운된 상황에도 write를 계속 수락할 수 있다. 따라서 특정 replica가 write 또는 delete를 놓쳐 replica 간 데이터가 잠시 달라질 수 있다.

```text
RF=3

정상:          Node A = V1 | Node B = V1 | Node C = V1
Node C 다운
새 write:      Node A = V2 | Node B = V2 | Node C = V1
Repair 후:     Node A = V2 | Node B = V2 | Node C = V2
```

**Repair(anti-entropy repair)** 는 같은 token range를 보유한 replica들의 데이터를 비교하고 차이를 동기화하는 운영 작업이다. Hinted handoff는 다운된 node에 놓친 mutation을 전달하는 best-effort 기능일 뿐 모든 누락을 보장하지 않으므로, 정기 Repair를 대체하지 못한다.

### Repair가 tombstone에 중요한 이유

삭제 시 tombstone이 모든 replica로 전파되기 전에 `gc_grace_seconds`가 지나 tombstone이 purge되면, 다운되어 삭제를 받지 못한 replica의 오래된 값이 나중에 다시 전파될 수 있다. 이를 zombie data라고 한다.

```text
초기:          A | A | A
한 replica 다운 후 DELETE: T | T | A
Repair 없이 tombstone purge: - | - | A
이후 동기화: 오래된 A가 되살아날 위험
```

따라서 모든 replica token range는 `gc_grace_seconds`가 만료되기 전에 repair되어야 한다. 기본 grace period는 10일이지만, 실제 repair 주기는 데이터량, delete/TTL 비율, 노드 복구 목표 시간, 운영 여유 자원을 기준으로 정해야 한다.

### Repair의 동작: Merkle tree와 streaming

Repair는 모든 row를 직접 대조하지 않는다. replica들은 대상 token range에 대해 Merkle tree(계층형 hash tree)를 생성하고 이를 비교한다.

```text
공통 token range
  → 각 replica가 Merkle tree 생성
  → root/branch hash 비교
  → 불일치 sub-range 식별
  → 해당 범위의 차이 데이터만 streaming하여 동기화
```

이 방식은 비교 전송량을 줄이지만, Merkle tree를 만들기 위한 SSTable read와 불일치 data streaming은 disk I/O, network I/O, CPU를 많이 사용할 수 있다.

### Full repair와 Incremental repair

| 유형 | 대상 | 장점 | 비용/주의점 |
|---|---|---|
| Full repair | 대상 token range의 모든 데이터 | 이미 repaired 상태인 데이터도 재검증하므로 광범위한 불일치 탐지에 유리 | I/O와 network 비용이 큼 |
| Incremental repair | 이전 incremental repair 후 생성된 unrepaired 데이터 | 정기 실행 시 대상이 작아져 일상 비용 감소 | repaired 상태가 된 과거 데이터는 다시 검증하지 않음 |

최근 Cassandra에서 `nodetool repair`는 incremental repair가 기본이며, `nodetool repair --full`로 full repair를 실행할 수 있다. Incremental repair만으로는 이미 repaired로 분류된 과거 데이터의 corruption이나 운영상 손실을 다시 검증하지 못하므로, 주기적인 full repair도 필요하다.

Incremental repair가 완료되면 Cassandra는 repaired data와 unrepaired data를 분리하는 **anticompaction**을 수행한다. 이는 SSTable rewrite를 동반하므로 disk I/O와 일시적인 disk 공간을 사용한다. 특히 큰 SSTable, 많은 SSTable overlap, 기존 대형 클러스터에서 incremental repair를 처음 도입하는 경우 부하가 클 수 있다.

### 실행 범위

한 노드에서 실행한 `nodetool repair`는 그 노드가 보유한 token range를 repair한다. 전체 클러스터를 빠짐없이 repair하려면 각 DC의 모든 node에서 해당 범위를 스케줄링해야 한다. 중복 작업을 줄이기 위해 일반적으로 각 node의 primary range만 대상으로 하는 `-pr` 옵션을 사용한다.

```bash
nodetool repair -pr
```

Repair는 foreground read/write, compaction과 disk·network 자원을 공유한다. 운영 중에는 repair 동시 실행 수, streaming throughput, compaction backlog, disk headroom, read/write p95·p99를 함께 관찰해야 한다.

참고: [Apache Cassandra Repair 문서](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/repair.html), [Auto Repair 문서](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/auto_repair.html)

## 16. Read repair

**Read repair**는 Cassandra의 공식 용어다. 일반 read 중 coordinator가 replica 응답의 불일치를 발견했을 때, 해당 query 범위의 뒤처진 replica에 최신 mutation을 보내 보정하는 read-path 작업이다.

```text
RF=3, quorum read

Replica A: V2
Replica B: V2
Replica C: V1

coordinator가 응답을 reconcile
→ 클라이언트에는 V2 반환
→ 필요 시 C에 V2를 전송하여 보정
```

| 구분 | Read repair | Anti-entropy repair |
|---|---|---|
| 실행 계기 | 일반 read 중 불일치 발견 | 운영자가 정기적으로 실행 |
| 범위 | 해당 `SELECT`가 읽은 범위 | 선택한 token range 전체 |
| 역할 | 조회 중 발견된 불일치의 빠른 보정 | 전체 replica 정합성 유지 |
| 보장 | Best-effort | 정기적으로 완료하면 eventual consistency 유지 |

따라서 read repair는 `nodetool repair`의 대체물이 아니다. read되지 않는 partition의 불일치는 read repair로 발견되지 않으며, tombstone 안전성도 정기 anti-entropy repair에 의존한다.

### `read_repair` table option

최근 Cassandra의 table option은 다음 두 값을 지원한다.

```sql
ALTER TABLE ks.events WITH read_repair = 'BLOCKING';
```

| 값 | 의미 | 일관성 특성 |
|---|---|---|
| `BLOCKING` (기본) | read repair가 발생하면 필요한 repair write가 CL을 만족할 때까지 read가 대기 | Monotonic quorum reads 제공 |
| `NONE` | 응답 차이는 reconcile하지만 뒤처진 replica에 read-path repair write를 보내지 않음 | Partition-level write atomicity 보존에 유리 |

`BLOCKING`은 연속된 quorum read가 더 오래된 상태로 되돌아가는 현상을 막지만, 불일치가 발견된 read의 latency를 높일 수 있다. `NONE`을 선택하면 일반 read의 repair write 비용은 줄지만 정기 repair의 중요성이 더욱 커진다.

참고: [Cassandra Read Repair 문서](https://cassandra.apache.org/doc/4.0/cassandra/operating/read_repair.html)

## 17. Consistency Level

Consistency Level(CL)은 read 또는 write가 성공으로 응답되기 전에 coordinator가 몇 개 replica의 응답을 기다릴지를 요청별로 정한다. write는 모든 replica로 전송을 시도하지만, CL은 클라이언트 성공 응답에 필요한 acknowledgement 수를 결정한다.

```text
write at QUORUM
  → 모든 replica에 write 전송 시도
  → 전체 replica의 과반 acknowledgement를 받으면 성공 응답

read at QUORUM
  → 전체 replica의 과반 응답을 받고 reconcile하여 반환
```

일반적으로 성공한 write를 이후 read에서 보려면 다음 조건을 사용한다.

```text
R + W > RF
```

`R`은 read에서 요구하는 replica 수, `W`는 write에서 요구하는 replica 수, `RF`는 replication factor다. 예를 들어 RF=3에서 `QUORUM` write(2)와 `QUORUM` read(2)는 최소 한 replica가 겹치므로 성공한 write가 이후 quorum read에서 보장된다.

### 주요 consistency level

| CL | read/write 응답 요구 |
|---|---|
| `ONE` | 임의 replica 1개 |
| `LOCAL_ONE` | local DC replica 1개 |
| `QUORUM` | 전체 RF의 과반 |
| `LOCAL_QUORUM` | coordinator가 있는 local DC RF의 과반 |
| `EACH_QUORUM` | 모든 DC에서 각각 과반 |
| `ALL` | 모든 replica |
| `ANY` | write 전용; replica acknowledgement 또는 hint 저장 성공 |

### `LOCAL_QUORUM`과 `QUORUM`

단일 DC에서는 `LOCAL_QUORUM`과 `QUORUM`이 사실상 동일하다. 다중 DC에서는 차이가 있다.

```text
DC1 RF=3, DC2 RF=3

LOCAL_QUORUM: coordinator의 local DC에서 2개 응답
QUORUM:       전체 6개 replica 중 4개 응답
EACH_QUORUM:  DC1에서 2개 + DC2에서 2개 응답
```

일반적인 멀티 DC 서비스에서, 사용자가 특정 DC에 고정되고 해당 DC에서 read-your-writes가 보장되면 `LOCAL_QUORUM` write + `LOCAL_QUORUM` read가 흔한 선택이다. 이는 local DC 안에서 quorum intersection을 보장하면서 cross-DC latency와 원격 DC 장애의 영향을 낮춘다.

반대로 어느 DC에서 읽어도 즉시 최신 상태가 필요하다면 `QUORUM` 또는 `EACH_QUORUM`을 검토할 수 있다. 다만 원격 DC network latency와 장애가 요청의 latency 및 availability에 직접 영향을 준다.

CL은 최신 write의 가시성과 replica acknowledgement에 관한 선택이다. 여러 partition의 트랜잭션 직렬화나 조건부 동시성 제어를 제공하지는 않는다. 그런 요구에는 LWT가 필요하다.

참고: [Cassandra Tunable Consistency 문서](https://cassandra.apache.org/doc/stable/cassandra/architecture/dynamo)

## 18. 트랜잭션, Batch, LWT

Cassandra는 여러 row·여러 partition·여러 table을 묶어 `BEGIN`/`COMMIT`/`ROLLBACK`하는 범용 RDBMS ACID transaction을 제공하지 않는다. 대신 partition 범위의 atomicity, logged batch, LWT를 필요한 수준에 맞춰 제공한다.

### 일반 mutation의 범위

동일 partition key의 update/delete는 atomic하고 isolated하게 적용될 수 있다. 하지만 일반 mutation은 serializable isolation이나 cross-partition transaction을 보장하지 않는다.

| ACID 관점 | Cassandra 일반 동작 |
|---|---|
| Atomicity | 같은 partition의 mutation에 대해 제공 가능 |
| Consistency | 요청별 CL로 선택 |
| Isolation | 일반 write에는 serializable isolation 없음 |
| Durability | Commit Log와 write CL acknowledgement로 보장 |

### Logged batch

```sql
BEGIN BATCH
  INSERT INTO table_a (...) VALUES (...); -- partition A
  INSERT INTO table_a (...) VALUES (...); -- partition B
APPLY BATCH;
```

기본 batch는 **logged batch**이며 batchlog를 사용한다. coordinator 장애 또는 일부 mutation 전달 실패 시, batchlog replay가 누락 mutation을 재시도한다.

```text
A 적용 완료 → B 전달 실패
→ batchlog replay
→ B 재시도
→ 최종적으로 A와 B가 모두 적용되도록 보장
```

따라서 logged batch는 mutation 하나만 영구 반영되고 다른 mutation이 영구 누락되는 상태를 방지하는 목적에 적합하다. 그러나 여러 partition 변경에 대해 RDBMS식 rollback·즉시 원자적 가시성·read isolation을 제공하지는 않는다.

```text
다른 client가 관찰할 수 있는 중간 상태:
partition A의 변경은 보임
partition B의 변경은 아직 보이지 않음
```

또한 client가 `WriteTimeout`을 받더라도 일부 또는 전부의 mutation이 이미 적용됐거나, 이후 batchlog replay로 완료될 수 있다. timeout은 rollback을 의미하지 않는다. 재시도는 같은 primary key/값/timestamp를 활용해 idempotent하게 설계하는 것이 안전하다.

`UNLOGGED BATCH`는 batchlog를 쓰지 않으므로 장애 시 일부 mutation만 남을 수 있다. Cross-partition logged batch는 batchlog, coordinator, network에 부하를 주므로 대량 write 묶음 용도로 사용하지 않는다. 동일 partition의 관련 mutation을 묶는 경우가 일반적으로 적절하다.

### LWT(Lightweight Transaction)

현재 상태를 조건으로 변경해야 할 경우에는 `IF` 조건을 사용한 LWT를 쓴다.

```sql
INSERT INTO users (user_id, email)
VALUES ('u-42', 'a@example.com')
IF NOT EXISTS;

UPDATE inventory
SET quantity = 9
WHERE product_id = 'p-1'
IF quantity = 10;
```

LWT는 Paxos를 사용해 단일 partition 범위의 compare-and-set을 수행하고 linearizable consistency를 제공한다. 합의 단계의 consistency level은 `SERIAL` 또는 `LOCAL_SERIAL`이다. 일반 write보다 replica 간 round-trip과 합의 비용이 크므로 고경합 partition이나 모든 write에 사용하면 latency와 throughput이 크게 악화될 수 있다.

정리하면 다음과 같다.

> Cassandra는 rollback 기반의 범용 ACID transaction 대신, tunable consistency와 partition-local atomicity를 기본으로 제공한다. 조건부 단일-partition 변경에는 LWT를, 실패 후 되돌림이 필요한 비즈니스 흐름에는 애플리케이션 수준의 보상 transaction을 설계한다.

참고: [Cassandra Guarantees 문서](https://cassandra.apache.org/doc/stable/cassandra/architecture/guarantees.html), [Cassandra DML 문서](https://cassandra.apache.org/doc/stable/cassandra/developing/cql/dml.html)
