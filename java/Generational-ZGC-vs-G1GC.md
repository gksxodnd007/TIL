# Generational ZGC의 짧은 Pause와 G1 Young GC의 차이

**대상:** JDK 21 HotSpot (Generational ZGC: JEP 439)  
**목적:** Generational ZGC가 pause time을 매우 짧게 유지하는 핵심 메커니즘과 G1 GC의 Young GC 모델을 비교한다.

---

## Executive summary

Generational ZGC의 낮은 지연시간은 “marking을 concurrent로 한다”는 한 가지 이유로 설명되지 않는다. 결정적인 설계는 **객체 이동(relocation)과 stale reference 처리까지 애플리케이션 실행 중에 수행**하는 것이다. 참조를 읽는 경로의 **load barrier**가 이전 주소를 새 주소로 해석하고, 포인터에 담긴 GC 상태 정보인 **colored pointer**가 이 판정을 빠르게 가능하게 만든다.

G1도 concurrent marking을 수행하지만, Young GC와 Mixed GC의 객체 evacuation은 stop-the-world(STW) pause에서 수행한다. 따라서 live object 복사량, 외부 참조 탐색, remembered-set 처리량이 pause에 직접 반영된다.

Generational ZGC는 ZGC의 concurrent relocation 모델을 유지한 채 young/old 세대를 분리했다. young object를 자주, 저렴하게 수집해 CPU와 메모리 여유 공간 요구량을 줄이지만, load/store barrier 및 concurrent worker를 위한 비용은 지불한다.

---

## 1. 두 collector의 근본 모델

| 관점 | G1 GC | Generational ZGC |
|---|---|---|
| Heap 구성 | 동일 크기 Region, young/old 논리 세대 | young/old 논리 세대와 Region |
| Marking | 상당 부분 concurrent | 상당 부분 concurrent |
| 객체 이동 | Young/Mixed STW evacuation | concurrent relocation |
| 참조 갱신 | pause 동안 eager update | load barrier 기반 lazy/self-healing update |
| Young 수집의 주된 pause 비용 | root/remembered set scan, 복사, 참조 갱신 | 짧은 phase 전환 및 root 관련 작업 |
| 지연시간 특성 | pause 목표를 예측·제어하지만 workload 변화에 영향 | live set/heap 크기와 pause의 결합이 훨씬 약함 |

### G1: 정지 중에 객체를 옮기는 collector

G1은 회수 효율이 좋은 Region을 Collection Set으로 선택하고, STW pause에서 해당 Region의 live object를 destination Region으로 복사한다. 참조도 이 pause에 처리한 뒤 source Region을 즉시 재사용한다. Region 선택으로 pause 목표에 맞추려 하지만, copy 및 scan 비용은 본질적으로 pause budget에 들어간다.

### Generational ZGC: 실행 중에 객체를 옮기는 collector

ZGC는 relocation 대상 객체를 복사하는 동안에도 application thread를 실행시킨다. 예전 주소를 가진 참조가 남아 있어도 해당 참조를 load할 때 새 주소로 해석할 수 있으므로, 전체 heap의 모든 참조를 한 번에 고치는 STW 작업이 필요하지 않다.

---

## 2. 낮은 pause를 만드는 핵심 메커니즘

### 2.1 Colored pointer: 참조 자체에 GC 상태를 표현

ZGC는 객체 참조의 일부 비트를 GC metadata로 사용한다. 이 정보로 barrier는 현재 참조가 relocation 전 주소인지, 현재 GC phase에서 이미 확인된 참조인지 등을 빠르게 판별한다. application과 GC가 동시에 객체를 다루면서도 안전성을 지키는 기반이다.

중요한 점은 Java 프로그램이나 JNI API가 colored pointer를 직접 보지 않는다는 것이다. HotSpot의 object access/barrier 경로가 이를 관리하며, runtime에 노출되는 일반 object pointer는 dereference 가능한 형태로 제공된다.

### 2.2 Load barrier: stale pointer를 사용 시점에 보정

객체 A가 `old-address`에서 `new-address`로 이동해도, 모든 A 참조를 즉시 찾을 필요가 없다. application thread가 A의 참조를 읽을 때 JIT가 삽입한 load barrier가 동작한다.

```text
reference load
  ├─ 최신 주소/정상 metadata  → fast path로 그대로 사용
  └─ relocation 전 주소      → forwarding 정보로 새 주소 해석
                               └─ 가능한 경우 참조도 새 주소로 갱신 (self-healing)
```

따라서 “이동이 끝나기 전에 모든 포인터를 고쳐야 한다”는 moving GC의 전제가 사라진다. 이후 접근 과정에서 stale pointer가 점진적으로 해소된다.

### 2.3 Concurrent marking과 concurrent relocation

ZGC의 일반적인 흐름은 다음과 같다.

```text
짧은 STW phase 전환/root 처리
        ↓
concurrent marking: live object 판별
        ↓
relocation set 선택
        ↓
concurrent relocation: live object 복사
        ↓
load barrier를 통한 lazy remapping 및 source region 회수
```

핵심은 live object graph traversal와 object copy가 애플리케이션을 멈춘 pause의 주 작업이 아니라는 점이다. 이 때문에 ZGC의 pause는 보통 sub-millisecond에서 수 ms 수준이며, 수집해야 할 live data가 커졌다고 그만큼 pause가 선형으로 늘어나는 구조가 아니다.

### 2.4 Generational ZGC가 추가한 store barrier

Generational ZGC는 young과 old를 독립적으로 수집해야 하므로 inter-generational reference를 추적한다. 이를 위해 store barrier를 추가했다.

- load barrier: stale pointer의 주소 보정과 metadata 제거에 집중한다.
- store barrier: 새 reference를 colored 형태로 저장하고, marking 및 old-to-young remembered set을 유지한다.
- marking 부담 일부를 store barrier로 옮겨, 더 빈번히 실행되는 load barrier의 fast path를 단순화한다.

이것은 pause 시간만을 위한 변화가 아니라, generational 모델에서도 높은 throughput을 유지하기 위한 설계다.

---

## 3. G1 Young GC와 Generational ZGC Young GC

### 3.1 G1 Young GC: STW evacuation

```text
[애플리케이션 정지]
1. Thread root 및 old → young remembered set 스캔
2. Eden/Survivor의 live object를 Survivor 또는 Old로 복사
3. 복사된 객체를 향하는 참조를 갱신
4. source young Region을 즉시 재사용
[애플리케이션 재개]
```

G1에서 Young GC의 Collection Set은 Young Region으로 구성된다. 살아남은 객체는 age에 따라 Survivor 또는 Old Region으로 evacuation된다. G1은 per-region remembered set을 유지하며, 보통 512-byte card 단위의 근사 위치를 기록한다. GC pause에서 해당 card 범위를 스캔해 정확한 참조를 찾는다.

따라서 다음이 Young pause에 직접 영향을 준다.

- young generation 크기
- 살아남은 객체의 양 및 promotion 양
- object copy와 pointer update 비용
- old-to-young reference 수와 card scan 비용
- destination 공간 부족 또는 evacuation failure

### 3.2 Generational ZGC Young GC: concurrent mark 후 concurrent relocation

```text
짧은 pause: phase 전환/root 관련 작업
        ↓
concurrent young marking
        ↓
concurrent relocation of selected young regions
        ↓
load barrier가 stale reference를 lazy update
```

Young collection은 young generation의 live object graph를 중심으로 처리한다. Old에서 Young으로 향하는 reference는 remembered set을 root로 취급한다. relocation은 application과 동시에 수행되며, Old 영역에 남은 old-to-young stale reference 역시 load barrier가 나중에 보정한다.

Generational ZGC는 live object를 먼저 mark하여 생존 집합을 확정한 후 relocation한다. density가 높은 young Region은 굳이 비싼 relocation을 하지 않고 제자리에 age시키거나 old로 promotion할 수 있다. 반면 G1 Young Collection Set에 포함된 Young Region의 생존 객체는 pause 내 evacuation해야 한다.

### 3.3 Remembered set의 차이

| 항목 | G1 | Generational ZGC |
|---|---|---|
| 추적 대상 | Collection Set 밖에서 Collection Set으로 들어오는 참조 | young collection 시 old-to-young 참조 |
| 기록 단위 | 보통 card(기본 512 bytes) 기반의 근사 위치 | object field 주소마다 bitmap bit로 기록하는 정밀 위치 |
| 기록 시점 | write barrier가 card dirty | store barrier가 field location 기록 |
| 수집 중 처리 | STW pause에서 card 범위를 스캔 | double-buffered bitmap snapshot을 concurrent 처리 |
| trade-off | barrier/metadata 비용을 낮추되 pause scan 비용 발생 | 정밀한 bitmap과 barrier 복잡도 대신 pause 작업을 축소 |

Generational ZGC는 remembered-set bitmap을 두 벌로 둔다. Young GC 시작 시 active bitmap과 GC가 읽을 bitmap을 atomically swap한다. application은 새 bitmap에 계속 기록하고 GC는 이전 snapshot을 concurrent로 읽고 비운다. 이 분리는 application thread와 GC thread가 같은 remembered-set 자료구조를 두고 기다리는 일을 줄인다.

---

## 4. Allocation stall은 STW가 아니다

ZGC에서 allocation stall은 새 객체를 할당하려는 thread가 즉시 할당 가능한 공간을 얻지 못할 때 발생한다. 그 thread는 concurrent GC가 공간을 회수할 때까지 대기한다.

```text
Thread A:  allocation 요청 ── stall ─────────── 재개
Thread B:  계속 실행 가능 ─────────────────────
GC worker:                 concurrent GC/relocation
```

이는 safepoint에서 모든 Java thread를 정지시키는 STW pause와 다르다. 그러나 힙 여유 공간이 완전히 소진되면 많은 request thread가 동시에 stall에 걸려 서비스 전체가 멈춘 것처럼 관측될 수 있다. GC log의 `Allocation Stall (thread-name) Nms`는 해당 할당 thread가 지연된 시간으로 해석해야 한다.

주요 원인은 작은 `-Xmx`, 높은 allocation burst, CPU 포화로 인한 GC worker 실행 부족, 또는 concurrent GC의 reclaim 속도를 넘는 allocation rate다. Generational ZGC는 young을 더 자주 수집해 이 위험을 낮추지만, heap headroom 및 CPU가 충분해야 한다는 조건은 남는다.

---

## 5. 운영상 결론

Generational ZGC의 강점은 “GC가 공짜”라는 데 있지 않다. 큰 비용을 **STW pause에서 concurrent worker와 barrier 실행 비용으로 이동**시킨 데 있다.

- 매우 낮은 tail latency가 최우선이고 CPU/heap headroom을 확보할 수 있으면 Generational ZGC가 유리하다.
- 수십~수백 ms pause가 허용되고, CPU 효율 및 일반적인 서버 throughput이 우선이면 G1도 강력한 선택이다.
- ZGC에서 pause가 짧다는 사실만 확인하지 말고, allocation stall, GC CPU, allocation rate, available heap을 같이 봐야 한다.
- G1에서 Young pause가 길다면 young size, survivor/promotion, remembered-set scan, copy 시간을 확인해야 한다.

## References

1. OpenJDK, *JEP 439: Generational ZGC* — https://openjdk.org/jeps/439
2. Oracle, *Garbage-First (G1) Garbage Collector*, Java SE 21 GC Tuning Guide — https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html
3. OpenJDK ZGC developers mailing list, *Big hiccups with ZGC* (Allocation Stall explanation) — https://mail.openjdk.org/pipermail/zgc-dev/2018-November/000503.html
