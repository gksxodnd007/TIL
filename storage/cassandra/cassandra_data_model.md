# Cassandra 데이터 모델과 조회 성능

## 핵심 관점

Apache Cassandra는 CQL에서 `table`, `row`, `column`이라는 SQL과 비슷한 용어를 사용하지만, 데이터 모델은 관계형 모델이 아니라 **partitioned wide-column 모델**이다.

- `table`은 CQL로 정의하는 논리적 스키마와 조회 단위다.
- `partition key`는 데이터가 어느 replica 노드 집합에 배치될지를 결정한다.
- `clustering column`은 같은 partition 안의 row 식별과 정렬 순서를 결정한다.
- Cassandra의 테이블은 엔터티를 정규화해 표현하기보다, 특정 조회 패턴을 빠르게 처리하도록 설계한다.

따라서 Cassandra 모델링은 보통 다음 질문에서 시작한다.

> 이 데이터를 어떤 조건과 정렬 순서로 조회해야 하는가?

관계형 데이터베이스는 엔터티와 관계를 먼저 정규화하고 `JOIN`으로 조합하는 경우가 많다. Cassandra는 일반적으로 join을 사용하지 않으므로, 하나의 조회 또는 비슷한 조회 묶음에 맞춘 테이블을 만들고 필요한 데이터를 중복 저장한다.

## Primary key의 구성

다음 테이블을 예로 든다.

```sql
CREATE TABLE orders_by_user (
  user_id uuid,
  ordered_at timestamp,
  order_id uuid,
  status text,
  total decimal,
  PRIMARY KEY ((user_id), ordered_at, order_id)
) WITH CLUSTERING ORDER BY (
  ordered_at DESC,
  order_id ASC
);
```

`PRIMARY KEY ((user_id), ordered_at, order_id)`는 다음과 같이 해석한다.

| 구성 요소 | column | 역할 |
| --- | --- | --- |
| Partition key | `user_id` | 같은 사용자의 주문을 같은 partition에 묶고, 해당 partition의 replica 위치를 결정한다. |
| 첫 번째 clustering column | `ordered_at` | partition 안에서 주문을 시간순으로 정렬하고 row 식별에 참여한다. |
| 두 번째 clustering column | `order_id` | 같은 주문 시각의 row를 구분하고, 두 번째 정렬 기준이 된다. |

`CLUSTERING ORDER BY`는 clustering column을 선언하는 구문이 아니다. primary key에서 partition key 뒤에 정의된 모든 column이 clustering column이며, 이 옵션은 각 clustering column의 저장 및 기본 조회 정렬 방향을 정한다.

위 테이블에서 한 사용자의 partition 내부 순서는 개념적으로 다음과 같다.

```text
user_id = A
  2026-09-24 10:00, order-1
  2026-09-24 10:00, order-2
  2026-09-24 09:55, order-3
```

## 조회 패턴별 테이블 설계

예를 들어 다음 조회 요구사항이 있다고 하자.

```text
Q1. 특정 사용자의 최근 주문 조회
Q2. 특정 주문 ID로 주문 한 건 조회
Q3. 특정 날짜의 전체 주문 조회
```

각 조회가 다른 partition key와 정렬 방식을 요구한다면, 보통 별도 테이블을 둔다.

```text
orders_by_user      -- Q1
order_by_id         -- Q2
orders_by_date      -- Q3
```

이는 SQL 문장 하나마다 테이블 하나를 만든다는 뜻은 아니다. 같은 partition key와 clustering order로 효율적으로 처리할 수 있는 **조회 패턴 묶음**을 하나의 테이블이 담당한다는 뜻이다.

예를 들어 `orders_by_user`는 다음 조회에 적합하다.

```sql
SELECT *
FROM orders_by_user
WHERE user_id = ?
LIMIT 20;
```

반면 `status`만으로 전체 주문을 찾는 요구사항은 이 테이블의 primary key 접근 경로에 맞지 않는다. 그 조회가 중요하다면 `orders_by_status` 같은 별도 조회 모델이나 적절한 인덱스 전략을 검토한다.

## Hot partition

같은 partition key를 가진 row는 같은 replica 노드 집합에 저장된다. 따라서 특정 partition에 데이터량 또는 트래픽이 집중되면 그 replica들이 병목이 된다.

```sql
PRIMARY KEY ((user_id), occurred_at, event_id)
```

이 모델에서 특정 헤비 유저의 이벤트가 매우 많거나, 한 사용자의 이벤트 조회 및 쓰기가 매우 빈번하면 `user_id` 하나의 partition이 hot partition이 될 수 있다.

hot partition은 두 종류로 나누어 생각하는 것이 좋다.

| 유형 | 원인 | 증상 |
| --- | --- | --- |
| 큰 partition | 하나의 key에 row, cell, tombstone이 장기간 누적 | 읽기와 compaction 비용 증가, 디스크 사용량 증가 |
| 뜨거운 partition | 작은 partition이라도 한 key에 읽기 또는 쓰기 요청이 집중 | 특정 replica의 CPU, 네트워크, latency가 악화 |

### 시간 버킷

시계열 데이터에서는 partition key에 시간 버킷을 추가해 partition의 성장을 제한한다.

```sql
CREATE TABLE events_by_user_day (
  user_id uuid,
  event_day date,
  occurred_at timestamp,
  event_id timeuuid,
  event_type text,
  payload text,
  PRIMARY KEY ((user_id, event_day), occurred_at, event_id)
) WITH CLUSTERING ORDER BY (
  occurred_at DESC,
  event_id ASC
);
```

```text
기존: (user_id=A) 전체 이력
변경: (user_id=A, event_day=2026-09-24) 하루 이력
```

버킷을 시간, 일, 주, 월 중 무엇으로 할지는 가장 바쁜 key의 데이터량과 요청량을 기준으로 정한다. 최근 7일처럼 여러 버킷을 읽는 조회는 애플리케이션이 필요한 partition들을 병렬 조회하고 결과를 병합한다.

### Write shard

시간 버킷만으로도 특정 partition의 쓰기량이 과도하다면 shard를 partition key에 추가할 수 있다.

```sql
CREATE TABLE messages_by_streamer_day_shard (
  streamer_id uuid,
  message_day date,
  shard tinyint,
  occurred_at timestamp,
  message_id timeuuid,
  body text,
  PRIMARY KEY ((streamer_id, message_day, shard), occurred_at, message_id)
);
```

쓰기 시에는 변화하는 식별자로 shard를 결정한다.

```text
shard = hash(message_id) % 16
```

그러면 같은 스트리머의 하루 메시지가 16개 partition으로 나뉜다. 목록 조회는 16개 shard를 읽어 시간순으로 merge해야 하므로, write 분산과 read fan-out 사이의 비용 교환이 생긴다.

```text
쓰기: 한 shard partition에 기록
최근 목록 읽기: 모든 shard를 병렬 조회 → 시간순 merge → 필요한 N건 반환
```

shard 수는 애플리케이션이 알고 있어야 한다. 보통 table 또는 애플리케이션 설정에 고정된 작은 값을 두고, `0..N-1` 전체를 조회한다. shard 수를 사용자나 테넌트별로 다르게 한다면 `shard_count`, `shard_version`을 담은 메타데이터를 별도로 관리해야 한다.

### Shard로 해결되지 않는 단건 hot key

단건 조회의 partition key가 `user_id` 자체이고 특정 사용자 A의 프로필 조회가 매우 빈번한 경우를 생각해 보자.

```text
shard = hash(user_id) % 16
user_id=A → 항상 같은 shard
```

`user_id`만 hash하면 A는 항상 한 partition으로 간다. 따라서 이는 한 사용자의 hot read를 분산하지 못한다.

한 논리적 값을 여러 partition으로 분산하려면 다음 중 하나가 필요하다.

- 읽기 cache를 둔다.
- 여러 read copy에 같은 값을 복제하고, 요청별로 copy를 선택한다. 갱신은 모든 copy에 기록해야 한다.
- 이벤트, follower 목록처럼 여러 항목으로 구성된 데이터라면 시간 또는 hash shard로 데이터를 실제로 나눈다.

## Update의 동작 방식

Cassandra의 일반 `UPDATE`는 기존 SSTable 파일의 값을 제자리에서 덮어쓰지 않는다. 각 write는 timestamp가 있는 mutation으로 기록된다.

```sql
UPDATE users
SET name = 'Kim'
WHERE user_id = 'u1';
```

개념적으로 다음 값을 새로 쓴다.

```text
(user_id=u1, name='Kim', timestamp=새 timestamp)
```

예를 들어 기존 상태와 새 write가 다음과 같다고 하자.

```text
SSTable 1
  name='Lee'                 @100
  email='lee@example.com'    @100

Memtable 또는 새 SSTable
  name='Kim'                 @200
```

읽을 때 Cassandra는 필요한 memtable, SSTable, replica의 version을 reconcile한다.

```text
name  → 'Kim'              @200 선택
email → 'lee@example.com'  @100 유지
```

일반적으로 version은 row 전체가 아니라 **column cell 단위**로 관리된다. `name`만 update하면 `email`은 변경되지 않는다.

`INSERT`와 `UPDATE`는 모두 upsert 성격을 가진다.

- row가 없으면 새 row가 만들어진다.
- row가 있으면 지정한 column에 새 mutation이 기록된다.
- 지정하지 않은 column은 유지된다.

명시적인 timestamp가 없으면 client 또는 coordinator가 write timestamp를 제공한다. `USING TIMESTAMP`로 직접 지정할 수도 있다.

```sql
UPDATE users
USING TIMESTAMP 1758700000000000
SET name = 'Kim'
WHERE user_id = 'u1';
```

같은 cell에 충돌하는 일반 write는 기본적으로 timestamp가 더 큰 값이 승리하는 last-write-wins 방식으로 resolve된다. 따라서 애플리케이션이 임의 timestamp를 넣으면 실제 요청 순서와 결과가 달라질 수 있다.

## Memtable, SSTable, compaction과 reconciliation

write는 우선 memtable에 반영되고, memtable이 flush되면 immutable SSTable이 생성된다. SSTable은 수정할 수 없으므로 update 뒤에도 이전 version이 기존 SSTable에 남을 수 있다.

```text
UPDATE 실행
  → 새 mutation을 memtable에 기록
  → flush 시 새 SSTable 생성
  → read 시 최신 timestamp version 선택
  → background compaction 시 오래된 version을 정리
```

따라서 최신 값을 조회하기 위해 compaction이 끝날 때까지 기다릴 필요는 없다. read path가 관련 version을 reconcile하여 논리적으로 최신 값을 반환한다.

compaction은 여러 SSTable을 합쳐 각 column의 최신 version을 포함한 새 SSTable을 만들고, 더 이상 필요 없는 이전 version을 제거한다. 많은 SSTable에 version이 흩어져 있으면 read amplification이 증가할 수 있으며, compaction은 이를 줄이는 역할을 한다.

`DELETE`는 즉시 물리 삭제하는 대신 tombstone이라는 timestamp가 있는 삭제 marker를 기록한다.

```text
name='Kim'     @200
DELETE name    @300  → tombstone @300
name='Park'    @400  → 최신 write가 tombstone을 이김
```

tombstone은 replica 간 삭제 사실이 전파될 시간을 고려해 유지되며, 안전 조건을 충족하면 compaction에서 제거된다.

## 조회 column에 따른 reconciliation 비용

다음과 같이 `follower_count`만 매우 자주 갱신되는 table을 생각해 보자.

```text
SSTable 1
  name='Kim'       @100
  email='a@x.com'  @100
  follower_count=10 @100

SSTable 2
  follower_count=11 @200

SSTable 3
  follower_count=12 @300
```

### `name`, `email`만 조회

```sql
SELECT name, email
FROM users
WHERE user_id = ?;
```

이 query는 `name`, `email`의 최신 version만 reconcile한다. `follower_count`는 결과에 필요하지 않으므로 여러 version을 가져와 timestamp 비교 대상으로 만들지 않는다.

```text
reconciliation 대상
  name:  'Kim' @100
  email: 'a@x.com' @100

reconciliation 대상이 아님
  follower_count: 10 @100, 11 @200, 12 @300
```

그러나 같은 partition이 여러 SSTable에 존재한다면, Cassandra는 partition index와 Bloom filter를 사용해 관련 SSTable을 확인해야 할 수 있다. 따라서 `follower_count`의 고빈도 update가 만드는 SSTable 수는 profile 조회의 disk access 가능성에도 영향을 줄 수 있다.

또한 partition 또는 row 수준의 tombstone은 선택한 column의 가시성에 영향을 줄 수 있으므로, query projection과 무관하게 고려되어야 한다.

### `follower_count`까지 조회

```sql
SELECT name, email, follower_count
FROM users
WHERE user_id = ?;
```

이 경우에는 `follower_count`의 모든 관련 version을 확인하고 최신 timestamp를 선택해야 한다.

```text
follower_count
  10 @100
  11 @200
  12 @300
  → 12 @300 반환
```

따라서 `follower_count`를 포함한 query가 일반적으로 더 많은 data block 읽기, cell deserialize, timestamp 비교, merge 작업을 수행할 가능성이 크다.

차이는 항상 크지는 않다.

- 최신 mutation이 memtable 또는 cache에 있을 수 있다.
- compaction이 오래된 version을 이미 정리했을 수 있다.
- `name`, `email`도 자주 변경됐다면 그 column들 역시 여러 version을 reconcile해야 한다.

## Counter의 의미

`follower_count`가 절대값을 설정하는 값이 아니라 `+1`, `-1` 누적값이라면 일반 `int`의 read-modify-write는 동시 요청에서 값 유실을 일으킬 수 있다.

```text
현재 값: 100
요청 A: 100을 읽고 101 기록
요청 B: 100을 읽고 101 기록
결과: 101
```

Cassandra의 `counter` type은 increment와 decrement를 위한 특수 column이다.

```sql
CREATE TABLE follower_count_by_user (
  user_id uuid PRIMARY KEY,
  follower_count counter
);

UPDATE follower_count_by_user
SET follower_count = follower_count + 1
WHERE user_id = ?;
```

counter는 일반 column과 다른 제약이 있다.

- counter table의 non-primary-key column은 모두 counter여야 한다.
- counter는 TTL을 지원하지 않는다.
- counter mutation은 idempotent하지 않다. timeout 뒤 무조건 재시도하면 중복 증가할 수 있다.
- 한 `user_id` counter에 update가 몰리면 여전히 hot partition이다.

매우 높은 increment 트래픽을 분산해야 한다면 sharded counter를 사용할 수 있다.

```sql
CREATE TABLE follower_count_by_user_shard (
  user_id uuid,
  shard tinyint,
  follower_count counter,
  PRIMARY KEY ((user_id, shard))
);
```

```text
shard = hash(follower_id) % 16
```

팔로우 및 언팔로우는 같은 `follower_id`로 같은 shard를 계산하여 증감한다. 전체 count 조회는 16개 shard를 읽어 합산해야 한다.

## 설계 시 확인할 질문

1. 이 table이 지원해야 하는 조회 조건, 정렬, 범위는 무엇인가?
2. 모든 정상 query가 partition key를 지정하는가?
3. 가장 바쁜 partition의 row 수, 데이터 크기, read/write rate는 얼마인가?
4. 시간 버킷으로 partition 성장을 제한해야 하는가?
5. write shard가 필요한 수준의 트래픽인가? 그 shard 수를 application이 안정적으로 관리할 수 있는가?
6. 조회 시 여러 partition/shard를 읽어 병합하는 비용은 허용 가능한가?
7. 고빈도 변경 column과 안정적인 profile column을 같은 table에 둘 때 발생할 read/write churn을 이해하고 있는가?
8. 동시 update에서 last-write-wins가 업무 요구사항에 맞는가? compare-and-set이 필요하면 LWT가 필요한가?

## 참고 문서

- [Apache Cassandra: Architecture overview](https://cassandra.apache.org/doc/stable/cassandra/architecture/overview.html)
- [Apache Cassandra: Data modeling introduction](https://cassandra.apache.org/doc/stable/cassandra/developing/data-modeling/intro.html)
- [Apache Cassandra: CQL data definition](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/ddl.html)
- [Apache Cassandra: Compaction overview](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/compaction/overview.html)
- [Apache Cassandra: Bloom filters](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/bloom_filters.html)
- [Apache Cassandra: Counter columns](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/counter-column.html)
