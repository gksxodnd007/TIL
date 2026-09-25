# Istio Canary 배포에서 사용자별 버전 고정 라우팅

Canary 배포 중에는 구버전(v1)과 신규 버전(v2) Pod가 동시에 실행된다. 이때 같은 사용자의 요청이 v1과 v2를 오가면 세션 상태, 응답 형식, 기능 플래그 등의 차이로 문제가 생길 수 있다.

이 문서는 API Gateway의 결정적 cohort 할당과 Istio `VirtualService` / `DestinationRule`을 조합해 **같은 사용자를 같은 버전으로 고정**하는 방법을 정리한다.

## 핵심 결론

`DestinationRule`의 `consistentHash`는 일반적인 canary 구성에서 **버전(v1/v2)을 고정하지 못한다**. `VirtualService`의 weight가 먼저 subset을 선택하고, consistent hash는 그 후 선택된 subset 안에서 Pod를 고르는 로드밸런싱 정책이기 때문이다.

사용자별 버전 고정은 다음 흐름으로 구현한다.

```text
Client
  -> API Gateway: session_id 생성 또는 검증
  -> API Gateway: hash(session_id)로 cohort(v1/v2) 결정
  -> API Gateway: 신뢰 가능한 x-release-cohort 헤더 주입
  -> VirtualService: 헤더에 맞는 subset 선택
  -> DestinationRule: subset의 Pod label 및 연결 정책 적용
  -> v1 또는 v2 Pod
```

## Weight 기반 canary와 consistent hash의 관계

다음 weight 라우팅은 **요청 단위**로 대상 version을 선택한다.

```yaml
http:
- route:
  - destination:
      host: my-api.default.svc.cluster.local
      subset: v1
    weight: 90
  - destination:
      host: my-api.default.svc.cluster.local
      subset: v2
    weight: 10
```

따라서 같은 사용자의 첫 요청이 v2로, 다음 요청이 v1로 갈 수 있다. 여기에 `consistentHash`를 추가해도 아래 순서는 바뀌지 않는다.

```text
VirtualService weight
  -> v1 또는 v2 subset 선택
  -> DestinationRule consistentHash
  -> 선택된 subset 내부의 특정 Pod 선택
```

즉 consistent hash가 제공하는 것은 같은 hash key를 가능한 한 같은 **Pod**로 보내는 soft session affinity이지, 같은 **버전**으로 보내는 cohort affinity가 아니다.

또한 Pod 증설/축소, 재배포, 장애 등 endpoint 구성이 변경되면 consistent hash 결과도 일부 바뀔 수 있다. Pod affinity가 별도로 필요하지 않다면 version sticky routing을 위해 consistent hash를 추가할 필요는 없다.

## 결정적 cohort 할당

API Gateway가 인증된 사용자 세션을 식별하고, 고정된 hash 규칙으로 version을 계산한다.

```text
bucket = hash(session_id, stable_salt) % 100

bucket 0-9   -> v2 (10% canary)
bucket 10-99 -> v1
```

같은 `session_id`는 항상 같은 bucket을 가지므로, 별도의 `session_id -> version` 저장소 없이도 같은 version으로 결정된다.

Canary 비율을 올릴 때는 기존 canary bucket을 유지하며 범위만 확장한다.

```text
10%: bucket 0-9   -> v2
25%: bucket 0-24  -> v2
```

이렇게 하면 기존 v2 사용자는 v2에 남고, 추가 bucket의 사용자만 v2로 이동한다.

운영 중에는 hash 함수, salt, bucket 경계를 변경하지 않는다. 변경하면 기존 사용자의 cohort가 대량으로 재배정된다.

## API Gateway 역할

Gateway는 다음을 수행한다.

1. `session_id`가 없으면 생성하고 클라이언트에 cookie로 전달한다.
2. 요청의 session ID로 cohort를 계산한다.
3. 클라이언트가 보낸 `x-release-cohort`는 제거한다.
4. 신뢰 가능한 계산 결과만 내부 요청 헤더에 다시 주입한다.

예시 응답 cookie:

```http
Set-Cookie: session_id=<random-or-signed-value>; Secure; HttpOnly; SameSite=Lax
```

내부 요청 헤더:

```http
x-release-cohort: v2
```

`x-release-cohort`를 외부 입력 그대로 신뢰하면 사용자가 canary version을 임의 선택할 수 있다. 따라서 이 헤더는 Gateway에서 overwrite하고, 외부 API 계약으로 노출하지 않는 것이 원칙이다.

## Istio subset이란 무엇인가

Istio의 올바른 용어는 `subnet`이 아니라 **`subset`**이다.

- `subnet`: CIDR 기반의 IP 네트워크 대역. 예: `10.0.1.0/24`
- `subset`: 하나의 Service 뒤 Pod를 label로 논리적으로 분류한 Istio의 destination 그룹

예를 들어 하나의 `my-api` Kubernetes Service가 v1/v2 Pod 모두를 endpoint로 가진다고 하자.

```text
my-api Service
├── Pod A: app=my-api, version=v1
├── Pod B: app=my-api, version=v1
└── Pod C: app=my-api, version=v2
```

`DestinationRule`의 subset은 이 endpoint들을 label로 구분한다.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: my-api
spec:
  host: my-api.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

subset은 Deployment나 Service처럼 새 Kubernetes 리소스나 별도 Pod 그룹을 생성하는 것이 아니다. Istio가 Service endpoint 중 label 조건에 맞는 endpoint만 대상으로 삼도록 하는 논리적 이름이다.

## Header 기반 version routing 예시

Gateway가 주입한 헤더를 `VirtualService`가 먼저 match한다. v2가 아닌 모든 cohort는 v1로 보내는 예시다.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: my-api
spec:
  hosts:
  - my-api.default.svc.cluster.local
  http:
  - name: canary-cohort
    match:
    - headers:
        x-release-cohort:
          exact: v2
    route:
    - destination:
        host: my-api.default.svc.cluster.local
        subset: v2
  - name: stable-cohort
    route:
    - destination:
        host: my-api.default.svc.cluster.local
        subset: v1
```

HTTP route는 위에서 아래 순서로 처음 match된 rule을 적용하므로, v2 match rule을 default v1 rule보다 앞에 둔다.

## VirtualService와 DestinationRule의 책임

두 리소스는 서로를 런타임에 순차적으로 조회하는 관계가 아니다. Istiod가 둘을 함께 Envoy 설정으로 변환한다.

| 책임 | 리소스 |
| --- | --- |
| Path, header, JWT claim 등 요청 조건 match | `VirtualService` |
| v1/v2 subset 선택, canary weight | `VirtualService` |
| Timeout, retry, fault injection, traffic mirroring | `VirtualService` |
| subset 이름과 Pod label 조건 정의 | `DestinationRule` |
| Load balancing, connection pool, outlier detection | `DestinationRule` |
| Upstream TLS/mTLS | `DestinationRule` |

요약하면 `VirtualService`는 **어디로 보낼지**, `DestinationRule`은 **선택된 대상에 어떻게 연결할지**를 정의한다.

## Rollout 종료 시 고려사항

v2 canary를 종료하거나 롤백할 때 기존 v2 cohort의 처리 방식을 미리 정해야 한다.

- v2를 승격하는 경우: 전체 사용자가 v2가 되도록 cohort 규칙 또는 VirtualService를 전환한다.
- v2를 롤백하는 경우: v2 cohort를 v1로 명시적으로 전환한 뒤 v2 Pod와 subset을 제거한다.
- 기존 session의 긴 수명이 중요하다면, session TTL과 cohort 정책 변경 시점을 함께 설계한다.

## 참고 자료

- [Istio VirtualService reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [Istio DestinationRule reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/)
