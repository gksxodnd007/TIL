### Java에서의 Interrupt

작업이나 스레드를 안전하고 빠르고 안정적으로 멈추게 하는 것은 어려운 일이다. 더군다나 자바에는 스레드가 작업을 실행하고 있을 때 강제로 멈추도록 하는 방법이 없다.(Thread.stop(), Thread.suspend()는 문제가 많은 기능으로 사용하지 말아야한다.) 대신 인터럽트(Interrupt)라는 방법을 사용할 수 있게 되어 있는데, 인터럽트는 특정 스레드에게 작업을 멈춰 달라고 요청하는 형태이다. 작업이나 서비스를 실행하는 부분의 코드를 작성할 때 멈춰달라는 요청을 받으면 진행 중이던 작업을 모두 정리한 다음 종료하도록 만들어야한다. 실행 중이던 일을 중단할 때 정상적인 상태에서 마무리하려면 작업을 진행하던 스레드가 직접 마무리하는 것이 가장 적절한 방법이다. 우리는 다음 세 가지의 경우를 대비해야한다.
- 오류가 발생하는 경우
- 종료하는 경우
- 작업을 취소하는 경우

Thread 클래스에는 해당 스레드에 인터럽트를 거는 메소드를 갖고 있으며, 인터럽트가 걸린 상태인지를 확인할 수 있는 메소드도 있다.
- **interrupt() :** 해당하는 스레드에 인터럽트를 거는 역할을 함.
- **isInturrupted() :** 해당 스레드에 인터럽트가 걸려있는지 확인해주는 역할을 함.
- **static interrupted() :** 현재 스레드의 인터럽트 상태를 해제하고, 해제하기 이전의 값이 무엇이었는지를 알려줌. 인터럽트 상태를 해제할 수 있는 유일한 방법.
> 현재 스레드의 인터럽트 상태를 초기화하기 때문에 사용할 때에 상당히 주의를 기울여야한다. interrupted()를 호출했는데 결과 값으로 true가 넘어왔다고 해보자. 만약 인터럽트 요청을 꿀꺽 삼켜버릴 생각이 아니라면 인터럽트에 대응하는 어떤 작업을 진행해야 한다. 예를들어 InterruptedException을 띄우거나 interrupt메소드를 호출해 인터럽트 상태를 다시 되돌려둬야 한다.

Thread.sleep()이나 Object.wait()메소드와 같은 블로킹 메소드는 인터럽트 상태를 확인하고 있다가 인터럽트가 걸리면 즉시 리턴된다. 위 두 메서드는 대기하던 중에 인터럽트가 걸리면 인터럽트 상태를 해제하면서 InterruptedException을 던진다. 여기서 던지는 InterruptedException은 인터럽트가 발생해 대기 중이던 상태가 예상보다 빨리 끝났다는 것을 뜻한다.

```kotlin
// 코틀린 코드이지만 java의 Thread api를 이용하고 있음으로 동작은 똑같다.
@Test
fun test() {
  val task = Thread {
    try {
      Thread.sleep(10000)
    } catch (e: InterruptedException) {
      println("interrupt")
    }
  }

  task.start()
  task.interrupt()
}
```
- 위의 코드는 10초를 대기하지않고 바로 interrupt를 출력하고 종료한다.

스레드가 블록되어 있지 않은 실행 상태에서 인터럽트가 걸린다면, 먼저 인터럽트 상태 변수가 설정되긴 하지만 인터럽트가 걸렸는지 확인하고, 인터럽트가 걸렸을 경우 그에 대응하는 일은 해당 스레드에서 알아서 해야 한다. 말하자면 잊지 말아야 할 일을 책상 앞에 메모해 붙여 둔 것과 같다. InterruptedException이 발생하거나 하지 않기 때문에 해야 할 일을 확인하고 처리하는 것은 당사자가 알아서 할 일이다.
> 특정 스레드의 interrupt()메소드를 호출한다 해도 해당 스레드가 처리하던 작업을 멈추지는 않는다. 단지 해당 스레드에게 인터럽트 요청이 있었다는 메시지를 전달할 뿐이다.