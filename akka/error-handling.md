### Akka Error Handling

#### Actor Error Handling

- Application의 최상위 액터는 /user 경로에 만들어지며 사용자 가디언(user guardian)에 의해 감독된다.
- 사용자 가디언의 기본 감독 전략은 Exception이 발생하면 자식을 재시작하는 것이다. 다만, 어떤 액터가 (내부 이유로) 종료됐거나 초기화 도중 실패했음을 알리는 내부 예외가 발생하면 해당 액터를 종료시킨다.
- 다음은 Actor defaultStrategy & defaultDecider 코드 이다.

```scala
final val defaultDecider: Decider = {
  case _: ActorInitializationException => Stop
  case _: ActorKilledException         => Stop
  case _: DeathPactException           => Stop
  case _: Exception                    => Restart
}

/**
 * When supervisorStrategy is not specified for an actor this
 * is used by default. OneForOneStrategy with decider defined in
 * [[#defaultDecider]].
 */
final val defaultStrategy: SupervisorStrategy = {
  OneForOneStrategy()(defaultDecider)
}
```

- `OneForOneStrategy`: 오직 중단된 액터에게만 적용
- `AllForOneStrategy`: 모든 자식이 같은 운명을 공유해서 같은 복구 방식을 한꺼번에 적용
- Application마다 내고장성에 필요한 조건을 만족하고자 SupervisorStrategy를 직접 만들어야 할 때가 있다. Supervisor의 액터 중단 오류에 대해 취할 수 있는 동작으로는 네 가지가 있다.
  - Resume: 오류를 무시하고 같은 액터 인스턴스로 처리를 진행한다.
  - Restart: 중단된 액터 인스턴스를 제거하고 새로 만든 액터 인스턴스로 교체한다.
  - Stop: 자식 액터를 영원히 종료시킨다.
  - Escalate: 오류를 올려보내서 부모 슈퍼바이저가 필요한 동작을 결정하게 한다.
- 다음은 예제 코드이다.

```scala
// 1. 별도의 decider를 만들지 않을 경우
class ActorSupervisor extends Actor {

  def receive: Receive = {
    case _: String => println("Hello! error handling")
  }

  override def supervisorStrategy = OneForOneStrategy() {
    case _: UserDefinedException => Restart
  }
}

// 2. 별도의 decider가 있을 경우
class ActorSupervisor extends Actor {

  def receive: Receive = {
    case _: String => println("Hello! error handling")
  }

  private def decider: SupervisorStrategy.Decider = {
    case _: UserDefinedException => Restart
  }

  override def supervisorStrategy = OneForOneStrategy()(decider)
}
```

#### Stream Error Handling

- 기본적으로 예외가 발생하면 스트림 처리가 중단된다.
- RunnableGraph의 실체화한 값은 발생한 예외를 포함하는 Future가 된다.
- 액터 감독 전략 정의와 비슷하게 스트림 감독 전략을 정의할 수 있다.

```scala
import akka.stream.ActorAttributes
import akka.stream.Supervision

val decider: Supervision.Decider = {
  case _: InvalidSchemaException => Supervision.Resume //InvalidSchemaException 발생 시 계속 진행
  case _ => Supervision.Stop
}

val deserialize: Flow[Array[Byte], GenericRecord, NotUsed] = {
  Flow[Array[Byte]]
    .map(AvroSerializer.deserialize)
    .withAttributes(ActorAttributes.supervisionStrategy(decider))
}
```

- 위와 같이 SupervisionStrategy는 withAttributes를 통해 전달되며 모든 그래프 컴포넌트에서 이 메서드를 사용할 수 있다. 다음과 같이 ActorMaterializerSettings를 사용하면 그래프 전체에 대해 감독 전략을 설정할 수도 있다.

```scala
val decider: Supervision.Decider = {
  case _: InvalidSchemaException => Supervision.Resume
  case _ => Supervision.Stop
}
```

- Stream은 SupervisionStrategy로 Resume, Stop, Restart를 지원한다.

```
1. 예외를 잡아서 오류를 표현하는 타입의 값으로 만들어서 다른 원소들과 함께 스트림을 통해 전달하는 방식이 있다.
UnDeserializeRequest 케이스 클래스를 도입하여 Request와 UnDeserializeRequest가 공통의 sealed된 RequestFrame 트레이트를 확장하게 한다.
이 방식을 사용하면 패턴 매칭으로 오류 여부를 검사할 수 있어 결과적으로 전체 흐름은 Flow[ByteString, RequestFrame, NotUsed]가 될 것이다.

2. Either 타입을 사용해서 오류를 left 타입으로, 이벤트를 right 타입으로 표현할 수도 있다.
이때 전체 흐름은 Flow[ByteString, Either[Error, Request], NotUsed]가 되며, 캣츠(cats)의 Xor타입을 적용할 수 도 있다.
```
