# G1GC와 (Generational) ZGC

이 문서는 HotSpot JVM의 G1GC와 ZGC를 객체 이동, 참조 처리, 세대 관리, latency 관점에서 비교한다. 여기서 말하는 latency는 단순히 GC 로그의 `Pause` 시간이 아니라 애플리케이션 요청의 end-to-end latency를 뜻한다.

## 핵심 요약

두 수집기 모두 살아 있는 객체만 다른 메모리 단위로 복사하고 기존 단위를 통째로 회수하는 **moving/compacting collector**다. 차이는 객체를 이동시키는 시점과, 이전 주소를 가리키는 참조를 처리하는 방법이다.

| 항목 | G1GC | ZGC / Generational ZGC |
| --- | --- | --- |
| 힙 관리 단위 | 동일 크기의 Region | 객체 크기에 따른 Page type |
| 객체 이동 | Young/Mixed GC의 STW evacuation | concurrent relocation |
| 이전 참조 처리 | STW 안에서 일괄 갱신 | colored pointer와 load barrier로 지연 보정 |
| 세대 관리 | Eden, Survivor, Old Region | Generational ZGC에서는 Young/Old Page 및 Young Page age |
| 주된 trade-off | 더 긴 STW pause 가능성 | barrier, concurrent GC CPU, heap headroom 비용 |

## 용어

- **Live object**: GC root에서 도달 가능하여 회수하면 안 되는 객체.
- **Evacuation / relocation**: live object를 새 위치에 복사해 기존 Region/Page를 비우는 작업. G1에서는 보통 evacuation, ZGC에서는 relocation이라고 부른다. 본질적으로는 같은 종류의 객체 이동이다.
- **Collection Set (CSet)**: G1이 이번 STW GC에서 evacuate할 Region 집합.
- **Relocation Set**: ZGC가 relocation할 Page 집합.
- **STW (Stop-The-World)**: Java 애플리케이션 스레드 전체를 safepoint에서 멈추는 구간.
- **Allocation stall**: 빈 공간이 부족해 객체 할당을 시도한 Java 스레드가 GC의 공간 확보를 기다리는 상태. STW와는 다르지만, 많은 worker가 동시에 막히면 서비스 전체가 멈춘 것처럼 보일 수 있다.

## 1. G1GC의 GC 과정

G1은 Region 기반의 세대별 collector다. 대부분의 새 객체는 Eden Region에 할당되고, Young GC 또는 Mixed GC에서 CSet에 속한 Region의 객체를 **STW로 evacuate**한다.

### Young GC

```text
Eden Region + 현재 Survivor Region을 CSet으로 선택
  ↓ STW
GC root와 Remembered Set을 시작점으로 live object 식별
  ↓
live object를 새 Survivor 또는 Old Region으로 복사
  ↓
참조를 새 주소로 갱신
  ↓
기존 Eden/Survivor Region 전체를 free Region으로 반환
```

죽은 객체는 복사하지 않으므로, CSet Region을 통째로 재사용할 수 있다. 이 작업은 garbage collection이면서 동시에 compaction이다.

### Concurrent Marking과 Mixed GC

G1은 Old Region의 live data를 파악하기 위해 concurrent marking을 수행한다. 이후 회수 가치가 높은 Old Region 일부를 Young Region과 함께 CSet에 넣어 **Mixed GC**로 evacuate한다.

```text
Concurrent Marking
  ↓ Old Region별 live bytes 계산
Mixed GC (STW)
  ↓ Young Region + 선택된 Old Region evacuation
```

따라서 G1은 marking은 concurrent하게 할 수 있어도, 일반적인 Young/Mixed 객체 이동과 참조 갱신은 STW다. 실제 pause는 대략 다음 요소에 의해 결정된다.

```text
G1 pause ≈ root/Remembered Set 스캔
         + CSet의 live object 복사
         + 참조 갱신
         + 기타 고정 비용
```

`-XX:MaxGCPauseMillis`는 엄격한 상한이 아니라, Young 크기와 CSet 크기를 조절하는 pause-time 목표다.

## 2. ZGC의 GC 과정

초기 ZGC(non-generational ZGC)는 heap 전체를 하나의 세대로 취급하는 concurrent compacting collector다. 긴 STW 작업을 피하기 위해 marking, relocation, remapping의 무거운 작업을 Java 스레드와 병행한다.

```text
짧은 Pause Mark Start
  ↓
Concurrent Mark / Remap
  ↓
짧은 Pause Mark End
  ↓
Relocation Set 준비 및 선택 (concurrent)
  ↓
짧은 Pause Relocate Start
  ↓
Concurrent Relocation
```

Pause는 완전히 0이 아니다. safepoint 전환, 일부 root 처리 등 매우 짧은 STW 단계가 존재한다. 다만 live object 대량 복사와 참조 갱신을 STW 밖으로 옮겼기 때문에 pause가 heap 크기나 live set 증가에 훨씬 덜 민감하다.

### Colored pointer와 load barrier

ZGC는 HotSpot 내부 객체 참조의 비트를 활용한 **colored pointer**로 참조의 GC 상태를 표현한다. 애플리케이션이 객체 참조를 읽을 때 JIT이 삽입한 **load barrier**가 이를 빠르게 검사한다.

```text
객체 참조 load
  ↓
현재 GC 상태에서 정상(good) 참조인가?
  ├─ 예: fast path로 그대로 사용
  └─ 아니오: slow path
       ├─ relocation/forwarding 정보를 조회해 새 주소 획득
       ├─ 현재 phase에 맞게 참조 상태를 remap
       └─ 가능하면 참조 필드도 새 값으로 갱신(self-healing)
```

따라서 Page A의 객체가 Page B로 옮겨진 뒤에도, 일시적으로 A를 가리키는 stale reference를 허용할 수 있다. 나중에 그 참조를 사용하면 load barrier가 B를 반환한다. 접근되지 않은 live reference는 후속 concurrent mark/remap의 그래프 순회에서 정리된다. self-healing은 정확성의 유일한 기반이 아니라, 같은 stale reference가 반복해서 slow path를 타지 않게 하는 성능 최적화다.

### ZGC의 relocation

```text
Relocation Set Page A
  [live X] [garbage] [live Y]
        ↓ concurrent relocation
새 Page B
  [live X] [live Y]

Page A 전체 회수
```

G1 evacuation과 마찬가지로 garbage는 복사하지 않는다. 차이는 G1이 참조 갱신까지 끝낼 때 애플리케이션을 멈추는 반면, ZGC는 stale reference를 barrier로 안전하게 해석하며 concurrent relocation을 한다는 점이다.

## 3. Generational ZGC의 GC 과정

Generational ZGC는 ZGC의 concurrent relocation 모델을 유지하면서 heap을 Young과 Old로 분리한다. 단명 객체가 대부분이라는 generational hypothesis를 활용해, 빈번한 Young collection이 Old 전체를 매번 mark하지 않고도 Young garbage를 회수하게 한다.

```text
Young collection
  Young Page를 대상으로 mark/relocate
  → 단명 객체를 빠르게 회수

Old collection
  Old Page를 대상으로 mark/relocate
  → Young collection보다 훨씬 낮은 빈도로 수행
```

Generational ZGC에서는 load barrier 외에 **store barrier**도 중요하다. Old 객체가 Young 객체를 참조하거나 concurrent marking 중 참조가 변경될 때, store barrier가 GC가 필요한 참조 관계를 놓치지 않도록 기록·처리한다.

```java
oldOrder.customer = youngCustomer; // store barrier가 필요한 GC 정보를 유지
```

이것은 Java 코드에 메서드 호출이 추가된다는 뜻이 아니다. interpreter/JIT이 해당 참조 read/write 지점에 필요한 barrier 기계어를 생성한다. 일반적인 경우에는 짧은 fast path이며, GC 상태상 처리가 필요할 때만 slow path를 수행한다.

> 참고: Generational ZGC는 JDK 21에 도입되었고, JDK 23에서 기본 ZGC 모드가 되었다. non-generational ZGC 모드는 JDK 24에서 제거되었다.

## 4. Non-generational ZGC가 높은 객체 할당율에서 G1보다 느려질 수 있는 이유

ZGC의 "low latency"는 긴 **GC pause**가 작다는 뜻이지, 모든 workload에서 요청 latency와 throughput이 항상 G1보다 좋다는 뜻은 아니다.

대부분의 서버 workload에서는 매우 많은 객체가 Eden에 할당된 뒤 곧 죽는다.

```text
높은 allocation rate
  ↓
대부분의 객체가 짧은 시간 안에 garbage가 됨
```

G1은 Young GC에서 Eden/Survivor 중심으로 이 단명 객체를 회수한다. 반면 non-generational ZGC는 Young/Old 구분이 없어서, 새 쓰레기를 회수하는 cycle에도 전체 heap 관점의 marking과 relocation을 수행해야 한다.

| 비용 또는 현상 | Non-generational ZGC에서의 영향 |
| --- | --- |
| 반복적인 전체 live graph marking | 큰 Old live set이 있어도 매 GC cycle 비용에 포함 |
| frequent concurrent relocation | CPU와 memory bandwidth 사용량 증가 |
| load barrier/remap | 애플리케이션 실행 경로에 추가 비용 |
| GC worker와 application worker 경쟁 | CPU가 포화되면 요청 처리 시간이 증가 |
| heap headroom 부족 | `Allocation Stall`로 실제 요청이 대기 |

가장 위험한 상태는 다음이다.

```text
allocation rate > concurrent GC의 reclaim rate
  ↓
free heap headroom 감소
  ↓
GC가 back-to-back으로 실행
  ↓
할당 스레드가 Allocation Stall
  ↓
p99/p999 요청 latency 상승
```

Allocation stall은 STW가 아니다. 할당을 시도한 스레드가 공간이 생길 때까지 대기하는 현상이다. 하지만 worker pool의 많은 스레드가 동시에 stall되면, 사용자 관점에서는 전역 pause와 유사한 장애로 보일 수 있다.

Generational ZGC는 Young GC가 단명 객체를 집중적으로 회수해 Old 전체 cycle의 빈도를 낮춤으로써 이 문제를 크게 완화한다. 그래도 ZGC는 충분한 heap headroom과 concurrent GC가 따라갈 CPU 자원이 필요하다.

## 5. Generational ZGC의 Young GC 과정

Generational ZGC의 Young GC도 copying/compacting collection이다. "dead object만 Page에서 삭제하고 live object와 빈 hole을 남기는 sweep 방식"이 아니다.

```text
Young source Page
  [live A] [garbage] [live B]
        ↓ concurrent relocation
destination Young Page 또는 Old Page
  [live A] [live B]

source Page 전체 회수
```

흐름을 age와 함께 보면 다음과 같다.

```text
Eden Page (age 0)
  └─ 생존 → Survivor age 1 Page

Survivor age 1 Page
  └─ 생존 → Survivor age 2 Page

Survivor age N Page
  └─ 생존 → 다음 age의 Survivor Page 또는 Old Page로 promotion
```

G1과 비교하면 논리적 copying 흐름은 유사하다.

```text
G1: Eden + Survivor(from) → Survivor(to) / Old
ZGC: Young source Page     → 다음-age Survivor Page / Old Page
```

그러나 ZGC에는 미리 고정된 두 개의 Survivor semi-space를 교대하는 전통적 `from/to` 공간 모델이 없다. 필요에 따라 destination Page를 할당하고, source Page의 live object를 옮긴 뒤 source Page를 반환한다. 따라서 결과적으로는 flip과 유사하지만, 물리적 공간 운영 단위는 Page와 relocation set이다.

## 6. G1GC의 Region과 ZGC의 Page 차이

| 구분 | G1 Region | ZGC Page |
| --- | --- | --- |
| 크기 | JVM 기동 시 정해지는 동일 크기(보통 1~32MB) | Small/Medium/Large 등 객체 크기에 맞는 Page type |
| 주요 역할 | Eden/Survivor/Old 역할을 맡는 수집 단위 | allocation, relocation, OS 메모리 관리 단위 |
| 수집 단위 | CSet에 넣어 STW evacuation | Relocation Set에 넣어 concurrent relocation |
| cross-reference 처리 | Remembered Set/card table이 핵심 | colored pointer + load/store barrier가 핵심 |
| 기존 단위 회수 | evacuation 후 Region 전체 반환 | relocation 후 Page 전체 반환 |
| 세대 표현 | Region의 역할로 표현 | Generational ZGC에서는 Page의 Young/Old 소속과 age로 표현 |

큰 힙에서 G1 pause가 항상 길어지는 것은 아니다. 핵심은 heap 전체 크기보다 한 번의 CSet에서 처리하는 live bytes, Remembered Set 스캔량, 참조 갱신량이다. 다만 큰 heap은 live set·Old Region·cross-region reference가 함께 증가하기 쉬워 tail pause 위험을 키운다.

## 7. G1GC의 객체 age 관리과 Generational ZGC의 age 관리

### G1: 객체 단위 age

G1은 객체 header(mark word)의 age 정보를 사용한다. Young GC에서 살아남아 복사되는 객체의 age를 증가시키고, tenure threshold와 Survivor 공간 압력 등을 바탕으로 객체별 promotion 여부를 판단한다. 같은 Survivor Region에도 서로 다른 age의 객체가 함께 들어갈 수 있다.

```text
하나의 G1 Survivor Region
  [object age 1] [object age 3] [object age 6]
```

### Generational ZGC: Page 단위 age

Generational ZGC는 `ZPageAge`처럼 **Page의 age**를 관리한다. 일반적인 Young allocation/relocation 흐름에서는 같은 Page의 객체가 같은 Young 생존 이력을 공유하도록 배치된다.

```text
Survivor age 2 Page
  [object A: page age 2]
  [object B: page age 2]
  [object C: page age 2]
```

그 Page를 Young relocation source로 처리하면 live object는 모두 다음 age 그룹의 destination Page로 이동한다. 남은 객체 양에 따라 destination은 하나가 아니라 여러 Page일 수 있지만, destination의 age는 동일한 다음 age다.

```text
Survivor age 2 source Page
  [live A] [dead] [live B]
       ↓
Survivor age 3 destination Page(s)
  [live A] [live B]
```

age가 충분히 높아졌거나 정책상 promotion이 유리하면, 해당 live object들은 Old Page로 옮겨진다. promotion threshold는 workload의 allocation rate, survivor 비율, collection 비용 등에 따라 적응적으로 결정될 수 있다.

| 항목 | G1 | Generational ZGC |
| --- | --- | --- |
| age 관리 단위 | 객체 | Page |
| 하나의 Survivor 단위 안의 age | 서로 섞일 수 있음 | 일반적으로 동일 age로 묶음 |
| Young 생존 시 | 객체별 age 증가 | Page age에 따라 다음 age Page로 relocation |
| promotion | 객체별 판단 | Page age 기반 정책과 relocation을 통해 Old Page로 이동 |

## 운영 관점의 선택 기준

- 수십~수백 ms 수준의 pause를 수용할 수 있고, CPU 효율과 안정적인 일반 서버 throughput이 중요하면 G1은 여전히 좋은 기본 선택이다.
- 큰 heap에서 p99/p999 latency에 매우 민감하고 충분한 CPU 및 heap 여유가 있다면 Generational ZGC가 유리할 수 있다.
- collector 선택은 pause 평균값만 보지 말고, 요청 latency percentile, allocation rate, live set, CPU saturation, GC cycle 빈도, `Allocation Stall`을 함께 비교해야 한다.

## 확인할 로그와 지표

```bash
-Xlog:gc*,safepoint
```

- G1: `Evacuate Collection Set`, Remembered Set/heap-root 스캔, object copy, Mixed GC 빈도와 pause를 확인한다.
- ZGC: Young/Old cycle 빈도, concurrent mark/relocate 지속 시간, free heap headroom, `Allocation Stall`, concurrent GC thread CPU를 확인한다.
- 공통: GC 이벤트 시각과 애플리케이션 p99/p999 latency, CPU saturation, allocation rate를 같은 타임라인에서 비교한다.

## 참고 자료

- [OpenJDK ZGC 프로젝트](https://wiki.openjdk.org/spaces/zgc/pages/329646/ZGC)
- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439)
- [JEP 474: ZGC Generational Mode by Default](https://openjdk.org/jeps/474)
- [JEP 490: Remove the Non-Generational Mode of ZGC](https://openjdk.org/jeps/490)
- [OpenJDK ZGC load barrier 및 relocation 발표 자료](https://cr.openjdk.org/~pliden/slides/ZGC-OracleDevLive-2020.pdf)
