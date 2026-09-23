# Redis 운영을 위한 메모리·복제·영속화 이해

이 문서는 Redis를 운영할 때 자주 만나는 `fork()`, Copy-on-Write(COW), Linux 메모리 overcommit 정책, RDB 기반 replication full sync의 관계를 정리한다. Redis Cluster에서도 각 primary-replica 복제 쌍에 동일하게 적용된다.

## 1. Redis가 `fork()`를 사용하는 이유

Redis는 메인 프로세스가 클라이언트 요청을 계속 처리하는 동안, 데이터셋의 일관된 시점 복사본을 만들기 위해 child process를 생성한다.

주요 사용 사례는 다음과 같다.

- `BGSAVE`: RDB snapshot 생성
- `BGREWRITEAOF`: AOF 재작성
- replica의 **full sync**를 위한 RDB 생성

중요한 점은 RDB persistence를 비활성화해도, disk 기반 replication full sync가 필요하면 primary는 RDB 생성을 위해 `fork()`할 수 있다는 것이다.

## 2. `fork()`와 virtual memory mapping

각 프로세스는 물리 RAM을 직접 다루지 않고, 독립적인 가상 주소 공간을 사용한다. virtual mapping은 이 가상 주소 영역이 물리 페이지, 파일, 또는 아직 할당되지 않은 익명 메모리와 연결되는 관계를 말한다.

예를 들어 heap, stack, shared library, `mmap()` 영역은 모두 virtual mapping에 해당한다. virtual mapping 크기와 실제 RAM 사용량(RSS)은 다를 수 있다.

```text
Virtual mapping: 20 GiB
RSS:              8 GiB
```

`fork()`는 parent의 virtual mappings를 child에 복제한다. 이때 보통 데이터 페이지 전체를 즉시 복사하지 않고 COW를 사용하므로, 초기 비용은 주로 page table 복제다.

## 3. Copy-on-Write(COW)

`fork()` 직후 parent와 child는 같은 물리 페이지를 공유한다. 커널은 writable private page를 write-protected COW 상태로 표시한다.

```text
fork 직후

Parent ─┐
        ├─ 물리 페이지 A: value = A
Child  ─┘
```

둘 중 한 프로세스가 페이지를 처음 수정하면 write page fault가 발생한다. 커널은 수정하려는 프로세스에 새 물리 페이지를 할당하고 기존 내용을 복사한 뒤, 그 새 페이지에 write를 수행하게 한다.

```text
Parent가 B로 write한 경우

Parent → 새 물리 페이지 B: value = B
Child  → 기존 물리 페이지 A: value = A
```

이후 기존 페이지 A는 child만 참조한다. 따라서 child가 다시 write하면 새 페이지를 만들 필요 없이 A를 writable로 바꿔 직접 수정할 수 있다. 반대로 child가 첫 write를 했다면 parent가 기존 페이지를 단독 소유한다.

> 단, 다른 fork child, KSM, 특수 mapping 등 추가 참조자가 있으면 페이지가 계속 공유될 수 있다.

### Redis에서 COW가 만드는 효과

RDB child는 데이터셋을 읽어 RDB 포맷으로 직렬화하며, 일반적으로 Redis 데이터셋을 수정하지 않는다. Primary parent는 계속 client write를 처리한다.

- child는 fork 시점의 이전 페이지를 유지한다.
- parent가 수정하는 페이지는 COW로 새 사본이 생긴다.
- child는 일관된 point-in-time snapshot을 만든다.
- background 작업 중 parent의 write가 많을수록 COW 메모리가 증가한다.

## 4. Linux `vm.overcommit_memory`

`vm.overcommit_memory`는 Linux가 가상 메모리 요청을 얼마나 엄격하게 사전 허용할지 정하는 전역 VM 정책이다. `malloc()`, writable anonymous/private `mmap()`, `fork()` 등이 영향을 받는다.

이는 실제 물리 RAM을 즉시 할당하는 정책이 아니다. 요청을 받은 시점에 "나중에 실제 메모리가 필요해져도 감당할 수 있는가"를 어디까지 검사할지를 정한다.

| 값 | 정책 | 특징 |
| --- | --- | --- |
| `0` | heuristic overcommit, 기본값 | 명백한 과도한 요청은 거절할 수 있음 |
| `1` | always overcommit | 충분성 사전 검사를 사실상 통과시킴 |
| `2` | strict / never overcommit | `CommitLimit`을 넘는 예약을 거절 |

`2`에서 기본적인 commit limit은 다음과 같다.

```text
CommitLimit = Swap + (RAM - HugeTLB pages) × vm.overcommit_ratio
```

### Redis에서 `vm.overcommit_memory=1`을 권장하는 이유

`fork()` 직후 실제 추가 RAM은 작지만, parent와 child가 향후 shared writable page를 모두 수정할 가능성은 이론적으로 존재한다. `0`에서는 이 잠재 COW 비용에 대한 휴리스틱 판단으로 `fork()`가 `ENOMEM`으로 거절될 수 있다.

`1`은 이 잠재 비용 때문에 `fork()`를 사전 거절하지 않도록 한다.

```text
vm.overcommit_memory=0
  → 잠재 메모리 사용량을 검사
  → 위험하다고 판단하면 fork() = ENOMEM

vm.overcommit_memory=1
  → 잠재 메모리 충분성 검사를 사실상 통과
  → fork() 실행
  → 실제 COW page가 필요한 시점에 물리 RAM을 할당
```

`1`은 물리 메모리를 미리 확보하거나 `fork()` 성공을 절대 보장하는 설정이 아니다. 실제 COW, allocator fragmentation, 다른 프로세스 사용량으로 RAM 또는 swap이 고갈되면 container OOM kill 또는 host OOM이 발생할 수 있다.

권장 설정 예시:

```conf
# /etc/sysctl.d/99-redis.conf
vm.overcommit_memory = 1
```

```bash
sudo sysctl --system
sysctl vm.overcommit_memory
```

컨테이너 환경에서는 대체로 host/node VM 정책이므로, Pod 내부 설정만으로 바뀌지 않을 수 있다. node OS 설정과 cgroup memory limit을 함께 확인한다.

## 5. RDB persistence와 replication full sync

Replica는 연결이 끊긴 뒤 primary의 replication backlog 범위 안에서 누락분을 받을 수 있으면 partial sync를 수행한다. 이때 RDB 생성은 필요 없다.

다음과 같은 경우에는 full sync가 필요하다.

- 새 replica가 처음 연결됨
- replication backlog가 부족함
- replica의 replication ID 또는 offset이 primary가 보유한 history와 맞지 않음

기본적인 disk 기반 full sync 흐름은 다음과 같다.

```text
1. Replica가 PSYNC로 연결
2. Partial sync 불가 → full sync 결정
3. Primary가 fork()하여 RDB snapshot 생성
4. Child가 RDB 파일을 생성
5. Primary가 RDB를 replica로 전송
6. Replica가 RDB를 로드
7. Primary가 fork 이후 축적된 write command stream을 전송
8. Replica가 명령을 적용하고 이후 지속적인 replication stream을 수신
```

RDB는 fork 시점의 snapshot이므로 단독으로는 이후 write가 빠진 과거 상태다. 그러나 primary가 RDB 생성·전송 중 발생한 변경 명령을 순서대로 전송하므로 replica는 같은 replication offset까지 따라잡을 수 있다.

## 6. `repl-diskless-sync yes`

diskless replication은 primary가 full sync용 RDB 파일을 디스크에 쓰지 않도록 하는 옵션이다.

```conf
# primary
repl-diskless-sync yes
```

동작은 다음과 같다.

```text
Replica full sync 요청
  → Primary가 fork()
  → Child가 공유 데이터셋을 RDB 포맷으로 직렬화
  → RDB 바이트를 파일 대신 replication TCP socket으로 직접 전송
  → Primary는 신규 write를 replication stream에 축적
  → Replica는 RDB 로드 후 누적 command stream을 적용
```

따라서 diskless는 "RDB를 만들지 않는다"가 아니라 **"RDB 파일을 만들지 않고 RDB 바이트 스트림을 직접 전송한다"**는 뜻이다.

### diskless replication의 운영 특성

- primary에서 full-sync용 RDB 파일 디스크 I/O와 디스크 공간 요구를 줄인다.
- `fork()`와 COW는 여전히 발생한다.
- 완성된 RDB 전체를 primary 메모리에 별도 버퍼로 유지하는 구조가 아니다. Child가 데이터를 직렬화하면서 socket으로 스트리밍한다.
- 느린 replica 또는 네트워크는 child의 실행 시간을 늘릴 수 있다. 실행 시간이 길수록 COW가 누적될 가능성이 커진다.
- `repl-diskless-sync-delay`는 첫 full-sync 요청 뒤 잠시 대기해 여러 replica가 하나의 child/RDB stream에 합류하도록 돕는다.

Replica의 수신·로드 방식은 별도 설정인 `repl-diskless-load`가 제어한다.

| 값 | Replica 동작 |
| --- | --- |
| `disabled` | 받은 RDB를 디스크에 저장한 뒤 로드 |
| `on-empty-db` | DB가 비어 있을 때 socket에서 메모리로 직접 로드 |
| `swapdb` | 새 DB를 비동기로 로드하고 완료 뒤 기존 DB와 교체 |

Primary와 replica 모두의 full-sync RDB 디스크 I/O를 줄이려면 두 옵션을 함께 검토한다.

```conf
# primary
repl-diskless-sync yes

# replica
repl-diskless-load swapdb
```

## 7. Full sync 성공과 replication lag

Replica의 다음 로그는 일반적으로 RDB 전송·로드가 성공하고 replication command stream 상태로 전환됐다는 뜻이다.

```text
MASTER <-> REPLICA sync: Finished with success
```

하지만 primary는 계속 write를 받을 수 있으므로, 이 로그 자체가 그 순간 최신 primary offset까지 완벽히 catch-up했다는 절대적 의미는 아니다. 실제 lag 여부는 replication offset을 비교해 확인한다.

```bash
# primary에서 master_repl_offset 확인
redis-cli INFO replication

# replica에서 slave_repl_offset 확인
redis-cli INFO replication
```

두 offset이 같으면 해당 시점에서 replica는 primary의 replication stream을 따라잡은 상태다. 이후 새 write가 발생하면 비동기 replication 특성상 다시 작은 lag가 생길 수 있다.

## 8. 컨테이너 메모리 용량 계획

`maxmemory`는 Redis 데이터셋에 대한 제한이지, 컨테이너 전체 메모리 사용량의 상한은 아니다. 특히 fork 기반 background 작업에서는 COW 때문에 메모리 사용량이 크게 증가할 수 있다.

```text
maxmemory = 10 GiB

평상시:       Redis dataset이 최대 약 10 GiB
fork 직후:     parent와 child가 페이지를 공유
write-heavy:   변경된 페이지가 COW로 복제
최악에 가까움: 추가 COW가 dataset 크기에 근접 가능
```

따라서 `container memory limit >= maxmemory × 2`는 COW 최악 상황을 고려한 보수적인 출발점이다. 그러나 이 값만으로 충분하다고 단정할 수는 없다.

추가로 고려할 항목:

- jemalloc fragmentation과 Redis object overhead
- replication backlog
- client/replica output buffer
- cluster bus 및 일반 network buffer
- Lua/function 관련 메모리
- RDB/AOF 작업의 부수 메모리
- 컨테이너 runtime 및 cgroup accounting
- host/node의 다른 workload

Persistence를 끄더라도 disk 기반 full sync 또는 diskless full sync가 있다면 fork/COW 위험은 남는다. `repl-diskless-sync yes`는 primary의 디스크 I/O를 줄이지만 COW 메모리 요구를 제거하지 않는다.

## 9. 운영 점검 항목

### Linux / container

```bash
sysctl vm.overcommit_memory
grep -E 'MemAvailable|SwapFree|CommitLimit|Committed_AS' /proc/meminfo
```

- `MemAvailable`: 현재 물리 메모리 관점의 예상 여유
- `Committed_AS`: 시스템이 장래 사용 가능하도록 약속한 virtual memory의 근사치
- `CommitLimit`: strict overcommit mode (`2`)에서 적용되는 상한

컨테이너에서는 cgroup memory limit/current usage 및 OOM event도 별도로 확인한다.

### Redis

```bash
redis-cli INFO memory
redis-cli INFO persistence
redis-cli INFO replication
```

특히 관찰할 값:

- `used_memory`, `used_memory_rss`, `mem_fragmentation_ratio`
- `current_cow_size`, `current_cow_peak`, `current_fork_perc`
- `rdb_last_bgsave_status`, `aof_last_bgrewrite_status`
- `master_repl_offset`, `slave_repl_offset`
- `master_link_status`, `master_sync_in_progress`

## 10. 핵심 정리

1. Redis의 background persistence와 full sync는 `fork()`를 사용한다.
2. `fork()` 직후 parent와 child는 데이터 페이지를 COW로 공유한다.
3. shared COW page에 처음 write한 프로세스만 새 물리 페이지를 얻는다.
4. COW는 child에게 일관된 RDB snapshot을 제공하고, parent는 write를 계속 처리하게 한다.
5. `vm.overcommit_memory=1`은 잠재 COW 비용 때문에 `fork()`가 사전 거절되는 위험을 줄인다.
6. `1`은 물리 메모리를 미리 확보하지 않는다. 실제 COW에 대비한 RAM/cgroup 여유는 별도로 필요하다.
7. `repl-diskless-sync yes`는 primary RDB 파일을 없애지만 `fork()`와 COW는 없애지 않는다.
8. `maxmemory × 2`는 유용한 보수적 출발점이며, production에서는 COW·fragmentation·buffer를 포함해 추가 여유를 계획한다.

## References

- [Redis administration: Linux setup tips](https://redis.io/docs/latest/operate/oss_and_stack/management/admin/)
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis FAQ: Background saving fails with a fork() error](https://redis.io/docs/latest/develop/get-started/faq/#background-saving-fails-with-a-fork-error-on-linux)
- [Linux kernel: Overcommit Accounting](https://docs.kernel.org/mm/overcommit-accounting.html)
- [Linux kernel: `/proc/sys/vm` documentation](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)
- [Linux `fork(2)` manual](https://man7.org/linux/man-pages/man2/fork.2.html)
- [Linux kernel COW write-fault implementation](https://github.com/torvalds/linux/blob/master/mm/memory.c#L3640-L3646)
