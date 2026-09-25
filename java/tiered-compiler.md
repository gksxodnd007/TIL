# HotSpot Tiered Compilation과 Compiler 로그

이 문서는 HotSpot JVM의 Tiered Compilation 동작, `-XX:-TieredCompilation`의 의미, 그리고 `-XX:+PrintCompilation`에서 관찰할 수 있는 컴파일 코드 상태 전이 로그를 운영 관점에서 정리한다.

## 핵심 요약

- Tiered Compilation은 빠른 C1 컴파일과 고도 최적화 C2 컴파일을 결합한 기본 JIT 전략이다.
- `-XX:-TieredCompilation`은 JIT 전체를 끄는 설정이 아니다. C1 중심의 다단계 컴파일을 끄고, 인터프리터에서 충분히 hot하다고 판단된 코드를 C2가 컴파일하게 한다.
- `made not entrant`와 `made zombie`는 컴파일된 네이티브 코드(`nmethod`)의 상태 전이 로그다. 로그 출력 자체나 상태 전이 자체가 주된 CPU 비용은 아니다.
- Tiered 모드에서 C1 컴파일, C1 코드 실행과 프로파일 수집, C2 재컴파일이 반복되면 시작 구간 CPU 사용량이 커질 수 있다.
- 다수의 짧거나 I/O-bound인 batch JVM이 동시에 기동되는 환경에서는 `-XX:-TieredCompilation`이 aggregate startup CPU burst를 낮추는 유효한 선택일 수 있다. 개별 batch의 전체 실행 시간과 총 CPU time으로 반드시 검증한다.

## 컴파일러와 Tier

HotSpot에는 두 주요 JIT 컴파일러가 있다.

| 컴파일러 | 특징 | 주된 역할 |
| --- | --- | --- |
| C1 (Client Compiler) | 컴파일이 빠르고 최적화가 비교적 가볍다. | 빠른 warm-up과 실행 프로파일 수집 |
| C2 (Server Compiler) | 컴파일 비용은 크지만 공격적인 최적화를 수행한다. | 장기 실행 코드의 peak performance 향상 |

Tiered Compilation은 인터프리터, C1, C2를 결합한다. 일반적인 Tier는 다음과 같이 이해할 수 있다.

| Tier | 실행 주체 | 목적 |
| --- | --- | --- |
| 0 | Interpreter | 바이트코드를 실행하면서 호출 횟수, loop back-edge, 타입, 분기 등의 정보를 수집 |
| 1 | C1 | 프로파일 없이 빠르게 네이티브 코드 생성 |
| 2 | C1 | 제한된 프로파일을 수집하며 실행 |
| 3 | C1 | 풍부한 프로파일을 수집하며 실행 |
| 4 | C2 | 누적된 프로파일을 사용해 고도 최적화 |

기본 흐름은 다음과 같다.

```text
Bytecode
  -> Interpreter: 실행 + 프로파일 수집
  -> C1: 빠른 컴파일 및 프로파일링
  -> C2: 충분히 hot한 코드의 고도 최적화
```

C2는 인라이닝, 탈가상화, escape analysis, loop 최적화 등을 수행할 수 있다. 이러한 최적화는 관찰된 타입이나 분기 분포가 이후에도 유지된다는 가정에 기초한다.

## Deoptimization과 nmethod 상태

JIT가 생성한 네이티브 코드 한 버전을 HotSpot 내부에서는 `nmethod`라고 부른다. 하나의 Java 메서드도 Tier 3과 Tier 4 등 여러 nmethod를 가질 수 있다.

예를 들어 C2가 어떤 interface 호출의 실제 구현체가 항상 하나라고 보고 인라이닝했는데, 이후 다른 구현체가 등장하면 해당 가정이 깨질 수 있다. JVM은 기존 최적화 코드를 무효화하고 일반적인 실행 경로로 되돌린 뒤 필요하면 다시 컴파일한다. 이 과정을 deoptimization이라고 한다.

`nmethod`의 대표적 상태 전이는 다음과 같다.

```text
entrant
  -> not entrant
  -> zombie
  -> Code Cache에서 회수
```

- `entrant`: 새 호출이 이 컴파일 코드를 진입점으로 사용할 수 있다.
- `not entrant`: 새 호출은 이 코드로 들어오지 않는다. 이미 실행 중인 activation은 잠시 남아 있을 수 있다.
- `zombie`: 실행 중인 activation도 없어져 Code Cache에서 회수 가능한 상태다.

`not entrant`가 되었다고 해서 항상 현재 실행 중인 모든 프레임이 즉시 deopt되는 것은 아니다. nmethod 무효화와 실제 activation의 deoptimization은 구분해야 한다.

## `-XX:+PrintCompilation` 로그 읽기

다음 옵션은 메서드가 컴파일될 때와 nmethod 상태가 전이될 때 로그를 출력한다.

```bash
-XX:+PrintCompilation
```

예시:

```text
  742  381       3  com.example.OrderService::find
  911  381       3  com.example.OrderService::find   made not entrant
  912  427       4  com.example.OrderService::find
 1250  381       3  com.example.OrderService::find   made zombie
```

- `381`, `427`: 메서드 ID가 아니라 컴파일 결과의 compile ID다.
- `3`, `4`: Tiered Compilation이 켜진 환경에서의 컴파일 level이다. 각각 C1 프로파일링 코드와 C2 코드로 읽을 수 있다.
- `(N bytes)`: 원본 bytecode 크기이며 생성된 네이티브 코드 크기가 아니다.

### `made not entrant`

해당 nmethod를 새 호출의 진입점으로 더 이상 사용하지 않게 했다는 의미다.

가장 흔한 정상 사례는 상위 tier 코드로의 교체다.

```text
100  23  3  Foo::run
300  67  4  Foo::run
301  23  3  Foo::run   made not entrant
```

Tier 3 C1 코드가 Tier 4 C2 코드로 대체되었으므로 기존 C1 nmethod가 not entrant가 된 것이다. 이 로그 하나만으로 deoptimization 문제나 성능 이상을 결론 내리면 안 된다.

그 밖에는 C2 최적화 가정의 붕괴, 새 클래스 로딩으로 인한 의존성 무효화, Code Cache 회수 등이 원인이 될 수 있다.

### `made zombie`

이미 not entrant 상태였던 nmethod에 더 이상 실행 중인 activation이 남아 있지 않아, Code Cache sweeper가 회수 가능한 상태로 표시했다는 의미다.

대부분은 정상적인 정리 완료 신호다. `made zombie`가 많다는 사실 자체는 문제의 증거가 아니다.

### `deoptimized`

`PrintCompilation`만으로는 deoptimization 사유를 안정적으로 알 수 없다. `made not entrant`는 nmethod 무효화 신호이고, `deoptimized`는 실행 중이던 특정 activation을 인터프리터 또는 낮은 tier의 프레임으로 복원한 실제 사건을 뜻한다.

원인(`uncommon trap`의 reason/action, 대상 method, bytecode index)을 조사해야 한다면 다음을 사용한다.

```bash
-XX:+UnlockDiagnosticVMOptions
-XX:+LogCompilation
-XX:LogFile=/var/log/app/hotspot.xml
```

`LogCompilation`은 상세 XML 로그를 많이 생성하므로, 상시 운영보다 재현 환경이나 짧은 진단 구간에 적합하다. 운영 영향이 낮은 원인 분석에는 JFR의 `jdk.Deoptimization` 이벤트도 유용하다.

## `-XX:-TieredCompilation`의 동작

다음 옵션은 Tiered Compilation을 비활성화한다.

```bash
-XX:-TieredCompilation
```

이는 JIT 전체를 비활성화하는 `-Xint`와 다르다. 일반적인 HotSpot Server VM에서는 다음 흐름이 된다.

```text
Bytecode
  -> Interpreter: 실행 + 호출/loop 프로파일 수집
  -> 충분히 hot한 method 또는 loop
  -> C2 컴파일
```

따라서 다음처럼 단순화할 수 있다.

```text
기본 Tiered Compilation: Interpreter -> C1 -> C2
-XX:-TieredCompilation: Interpreter -> C2
```

C2 컴파일은 프로파일이 완벽하게 축적된 뒤에만 발생하는 것은 아니다. 호출 횟수와 loop back-edge 등의 hotness 임계치에 따라 컴파일될 수 있고, 보유한 프로파일의 양에 따라 가능한 최적화가 달라진다. 짧게 끝나거나 임계치에 도달하지 못한 코드는 끝까지 인터프리터로 실행될 수 있다.

또한 이 옵션은 `made not entrant`와 `made zombie`를 완전히 없애지 않는다. C2 코드도 최적화 가정이 깨지거나, 새 코드로 교체되거나, Code Cache 회수 대상이 되면 같은 상태 전이를 거칠 수 있다. 다만 C1 -> C2 승격에 따른 전이는 제거한다.

## 동시 기동 batch 환경에서의 판단

여러 JVM이 같은 시점에 부팅되면, 각 JVM의 JIT CPU 비용이 노드에서 합산된다.

```text
각 JVM의 비용
  C1 컴파일
  + C1 코드 실행 및 프로파일 수집
  + C2 컴파일
  + 기존 C1 nmethod 무효화 및 Code Cache 정리

수백 JVM 동시 기동
  = 위 비용이 노드 CPU에서 동시에 발생
```

I/O-bound bulk batch처럼 peak CPU 성능보다 동시 기동 시 CPU burst 억제가 중요한 워크로드라면, `-XX:-TieredCompilation`은 C1 기반 단계와 C1 -> C2 교체 비용을 줄이는 실용적인 선택이 될 수 있다.

단, 이 설정은 startup CPU를 줄이는 대가로 초기 실행 성능 또는 전체 수행 시간을 악화시킬 수 있다. 적용 여부는 다음을 비교해 결정한다.

- 동시 기동 구간의 node CPU peak 및 CPU throttling
- batch 전체 수행 시간의 p50/p95
- batch 하나당 총 CPU time 또는 CPU-seconds
- timeout, retry, 오류율 및 산출물 정확성
- 동시 실행 batch 수가 최대인 구간의 node saturation

CPU peak가 낮아지고 batch SLA와 총 CPU 비용이 유지된다면, 해당 batch군에는 적절한 설정이다. 특히 request보다 훨씬 높은 CPU burst를 허용하는 수백 개 batch가 동시에 기동되는 환경에서는, 개별 JVM의 startup 최적화보다 노드 전체의 안정성이 더 중요한 목표가 될 수 있다.

## Kubernetes 리소스와 구분할 점

Kubernetes 기본 scheduler는 Pod의 실시간 CPU 사용량이 아니라 `resources.requests.cpu` 합계로 배치 가능 여부를 판단한다. 실제 CPU 사용량이 request보다 높으면 node runtime contention, throttling, kubelet/runtime 지연 등이 발생할 수 있지만, 기본 scheduler의 `Insufficient cpu` 판정 자체와는 구분해서 분석해야 한다.

따라서 batch 기동 문제가 있었다면 다음을 함께 확인한다.

- Pod event의 실제 `FailedScheduling` reason
- Pod의 CPU request와 limit
- node allocatable CPU와 같은 시간대의 request 합계
- CPU usage, CPU throttling, node load, kubelet latency
- cluster autoscaler 또는 custom scheduler/admission control의 존재

JVM 옵션은 application-level CPU burst를 낮추는 대책이고, 동시 실행 수 제한, 기동 jitter, 정확한 request/limit 정책, batch node pool 용량 계획은 cluster-level 대책이다. 두 대책은 경쟁 관계가 아니라 함께 적용할 수 있다.
