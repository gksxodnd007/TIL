# Monolithic Architecture vs Micro Service Architecture (MSA)

## Monolithic Architecture

- 서비스가 제공하는 모든 기능을 하나의 component가 제공하는 방식

장점
  - 모든 코드가 동일한 환경에서 동작되기 때문에 개발 환경이 단순하다.
  - 트랜잭션 관리가 수월하다.
  - end to end 테스트가 용이하다.
  - error 추적이 쉽다.

단점
  - 기술이 강제 될 수 있다. -> 폴리글랏 프로그래밍이 불가능하다.
  - 프로젝트가 커질수록 빌드 / 배포 시간이 길어진다.
  - 부하가 많은 component만 별도로 scale out이 불가능 하므로 리소스를 효율적으로 사용하지 못한다.
  - 하나의 기능 장애가 전파되어 전체 장애로 이어진다.


## Micro Service Architecture (MSA)

- 서비스 지향 아키텍쳐(SOA)에서 진화한 아키텍쳐로 애자일 개발 방식
- 급변하는 기술 트랜드 등 최근 IT시장의 요구사항 및 환경에 대응하기 위해 나온 방법

장점
  - 유연하고 확장성있는 서비스 개발이 가능하다.
  - 기술 선택의 자유도가 높아진다. -> 플리글랏 프로그래밍이 가능하다.
  - 부하가 많은 component만 별도로 sale out이 가능하므로 리소스를 효율적으로 사용할 수 있다.
  - 하나의 기능 장애가 전체로 전파되지않고 격리가 가능하다. -> fault tolerance한 시스템이 가능하다.

단점
  - 기술의 남용으로 인해 개발 환경이 복잡해 질 수 있다.
  - 트랜잭션 관리가 Monolithic 구조에 비해 어렵다.
  - end to end 테스트가 어렵다.
  - 모니터링 시스템이 잘 구성되어있지 않다면 error를 추적하기 어렵다.

# Anatomy MicroService Architecture (MSA)

## MSA Component
<img src="https://user-images.githubusercontent.com/27043428/112759656-fba93d80-902e-11eb-8109-7eab92932195.png" width=500 />

### Outer Architecture

#### API gateway
- endpoint 단일화
- 인증, 인가 및 공통 로직 분리
- traffic routing

#### Service Mesh (네트워크 통신 구간 제어)
- Service Discovery
- Load Balancing
- Dynamic Request Routing
- Circuit Breaking
- Retry and Timeout
- TLS

#### Telemetry
- Metric 측정
- 모니터링

#### Backing Services
- 네트워크를 통해서 사용할 수 있는 모든 서비스
- My SQL과 같은 데이터베이스, 캐쉬 시스템, SMTP 서비스 등 어플리케이션과 통신하는 attached resource들을 지칭하는 포괄적인 개념

#### CI/CD Automation
- 지속적인 통합(Continuous Integration), 지속적인 전달(Continuous Delivery), 지속적인 배포(Continuous Deployment)를 자동화 하는 구성 요소로 배포가 잦은 MSA 시스템에 필수 요소

### Inner Architecture
- 정해진 패턴이 없고 각 상황에 맞는 구조를 선택
- 트랜잭션 경계를 잘 나눠야 하며, 트랜잭션 관리가 어렵다면 서비스를 잘 분리했는지에 대해 회고가 필요

## MSA Integration
MSA는 분리된 서비스들간의 협업을 통해 기능을 제공한다. 이때 어떻게 협업(통합)이 이루어지는 것이 좋을지 고민이 필요하다.
네트워크가 완전히 은폐될 정도로 원격 호출을 추상화하면 안된다. 그리고 **서버와 보조를 맞춘 클라이언트의 업그레이드 없이도 서버 인터페이스를 발전시킬 수 있는지 확인해야한다.**

### 트랜잭션

- strong consistency: 모든 시간에 데이터가 동기화되어 있어야한다.
  - **2PC(2 phase commit) pattern**: prepare phase and commit phase, commit 준비가 완료되면 동시에 모든 db에 commit이 적용된다. 이기종 database가 활용될 때 사용되며 성능이 감소할 수 있다.
- eventual consistency: 특정 시간동안은 데이터가 불일치 할 수 있지만, 특정 시간이 지나면 데이터가 동기화되어있다.
  - **SAGA pattern**: 각 트랜잭션이 단일 서비스 내의 데이터를 갱신하는 일련의 로컬 트랜잭션 방식. 첫번째 트랜잭션이 완료되고 난 후 두번째 트랜잭션은 이전의 작업 완료에 의해 트리거 되는 방식이다.
    - Choreography(Event) : 각 로컬트랜잭션이 이벤트를 발생시고 다른 서비스가 트리거링 하는 방식.
    - Orchestration(Command) : 오케스트레이터가 어떤 트랜잭션을 수행할 건지 알려주는 방식

참고
- https://developers.redhat.com/blog/2018/10/01/patterns-for-distributed-transactions-within-a-microservices-architecture
- https://www.howtodo.cloud/microservice/2019/06/19/microservice-transaction.html
- https://www.geeksforgeeks.org/eventual-vs-strong-consistency-in-distributed-databases

### MSA Integration design patterns
- bulkhead: 장애 격리
  - connection pool 분리
  - thread pool 분리
- circuit breaker: fast fail over

## MSA Best Practice

- 하나의 repository는 하나의 서비스에서만 사용한다. 다른 도메인의 데이터가 필요할 경우 REST 통신, RPC 통신등의 네트워크 호출을 통해 가져온다.
- REST 통신을 기본으로 하되 공유 라이브러리 방식은 지양해야한다. 공유라이브러리 방식을 채택하게되면 공유 라이브러리를 사용하는 클라이언트측 코드는 빈번하게 재컴파일/재빌드 후 재배포가 이루어져야한다.
- 도메인 경계가 불분명한 개발 단계에서는 Monolithic으로 시작하는 것이 더 좋은 선택이 될 수 있다. 시스템이 발전하면서 도메인 경계가 분명 해질 때 MSA로 전환하는 것이 좋다. MSA로의 전환은 package 분리부터 시작되어야한다. 도메인 별로 package를 분리하고 다른 도메인과의 통신이 필요할 경우 service class를 직접 의존하여 통신하기 보다는 adaptor (connector) layer를 두어 메서드를 호출하는 것이 좋다. 추후 도메인이 완전하게 분리되어 네트워크 통신이 필요할 경우 해당 adaptor만 메서드 호출 방식에서 REST 통신으로 변경하면 된다. 인터페이스만 잘 설계한다면 메서드 호출 방식이든 네트워크를 통한 호출이든 변경점은 크게 발생하지 않는다.

## Monolithic TO MSA

- API Gateway를 활용하여 legacy 고립 시키기
  - ref: https://woowabros.github.io/r&d/2017/06/13/apigateway.html
- Strangler Pattern -> 점진적 분리 -> legacy 고립 (strangler: 목을 조이다)
  - https://microservices.io/refactoring
  - https://microservices.io/patterns/refactoring/strangler-application.html
- 서비스 분리
  - 여러가지 방법들이 많지만 대부분 DDD를 통하여 `도메인 별로 서비스를 나누는 것`을 추천함 (bounded context, aggregate)
  - 도메인을 나누는데 의견충돌이 발생한다면 A기능의 장애로 인해 B기능까지 마비가 되는 것이 적합한 상황인가?를 고민해보자. `A기능의 장애가 B기능의 장애까지 전파될 필요가 없다면 A와 B를 분리`

참고
- [마이크로 서비스 아키텍쳐 사례 분석 글](https://yobi.navercorp.com/plasma/posts/12)
- [Building Microservices](http://www.yes24.com/Product/Goods/36551677)
- [스프링 5.0 마이크로서비스 2/e](http://www.yes24.com/Product/Goods/58255540)

MSA에 대한 Gartner 인터뷰
- https://www.comworld.co.kr/news/articleView.html?idxno=49710
