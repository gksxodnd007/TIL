## 트레이트 (Trait)

- Scala의 Trait는 Java의 Interpace와 비슷하면서도 다르게 동작한다. 먼저 기본적이 내용, Syntax 부터 정리해보자.

### 트레이트 정의 및 사용
```scala
trait Manager {

  def lookup(): Unit
}
```
- trait라는 키워드를 이용하여 정의하며 별도의 클래스나 트레이트를 상속하고 있지 않으므로 Manager 트레이트는 AnyRef가 부모 클래스이다.
- trait는 주 생성자를 가질 수 없다. 다음과 같은 코드는 컴파일 에러를 일으킨다.
```scala
trait Manager(val name: String) { // 컴파일 에러
  ...
}
```
- trait는 메서드를 선언 뿐만 아니라 구현할 수도 있다. 또한, 트레이트를 상속하여 구현하려면 다음과 같이 코드를 작성하면 된다.
```scala
trait TaskManager extends Manager {

  def run(body: => Unit): Unit = {
    println("TaskManager running")
    body
  }

  override def lookup(): Unit = {
    println("TaskManager task lookup")
  }
}
```
> Java8 이전에는 인터페이스에 default method가 없었으므로 트레이트를 컴파일하게 되면 trait의 구현된 메서드는 이를 상속 또는 구현하는 구현체에 inlininge되어 있게 된다. Java8이후 부터는 default method로 컴파일 되지 않을까 추측해본다.(이는 찾아보지 않았음.)

여기까지는 Java의 인터페이스와 크게 다르지 않다. 하지만 이후에 설명할 부분부터 Java의 인터페이스와는 다를 뿐더러 일반적인 언어에서 지원하는 다중상속과도 다르게 동작한다.

### Mixin (믹스인)
- 트레이트는 기능들을 쌓아 올릴 수 있다. 디자인패턴중 하나인 Decorator Pattern을 떠올리면 된다. 다음과 같은 코드를 통해 설명을 이어가도록 하겠다. Syntax는 주석을 통하여 전달하도록 하겠다.
```scala
abstract class Manager { // 트레이트가아닌 추상클래스임에 주의

  def lookup(): Unit = {
    println("Base lookup")
  }
}

// 슈퍼클래스로 Manager를 선언하였다. 이 선언은 TaskManager 트레이트가 Manager를 상속한 클래스에만 Mixin될 수 있다는 것이다.
trait TaskManager extends Manager {

  override def lookup(): Unit = {
    println("TaskManager task lookup")
    super.lookup()
  }
}

trait JobManager extends Manager {

  override def lookup(): Unit = {
    println("JobManager job lookup")
    super.lookup()
  }
}

// extends Manager가 빠져있다면 즉, Manager 클래스를 상속하지 않았다면 컴파일 에러가 발생한다.
trait ManagerImpl extends Manager with TaskManager with JobManager {

  override def lookup(): Unit = {
    println("Impl lookup")
    super.lookup()
  }
}

object Test {

  def main(args: Array[String]): Unit = {
    val manager = new ManagerImpl
    manager.lookup()
  }
}

/*
  결과 출력:
  Impl lookup
  JobManager job lookup
  TaskManager task lookup
  Base lookup
*/
```
- 자바의 인터페이스를 위의 ManagerImpl클래스와 같이 구현하였다면 컴파일에러가 발생하였을 것이다. `super.lookup()`의 super는 컴파일 시점에 정해지게 되는데 어떤 인터페이스의 lookup메서드를 호출할지 모호하기 때문이다.
- 스칼라에서는 위와 같은 경우를 Mixin이라고 부르며 super를 컴파일 시점에 정하지않고 동적으로 binding하기 때문에 선형화를 작업의 결과를 토대로 자신의 다음 super의 lookup메서드를 호출한다. 선형화는 다음과 같이 진행된다.

1. ManagerImpl은 자신의 슈퍼클래스나 Mixin한 트레이트보다 앞에 위치한다.
2. Manager는 AnyRef를 상속하며 AnyRef는 Any를 상속한다.
3. TaskManager는 Manager를 상속한다. (2번 에서 상속한 Manager, AnyRef, Any는 중복이므로 제외한다.)
4. JobManager는 Manager를 상속한다. (2번 에서 상속한 Manager, AnyRef, Any는 중복이므로 제외한다.)

1번을 제외한 나머지 2,3,4 => 4,3,2로 역순하여 super를 binding한다.
`ManagerImpl.lookup -> JobManager.lookup -> TaskManager.lookup -> Manager.lookup`

PS. 컴파일러에게 의도적으로 super의 메소드를 호출했다는 사실을 알려주기 위해, 이런 메소드를 abstract override로 표시해야한다. 그런 표시는 클래스에는 사용할 수 없고, 트레이트의 멤버에만 사용할 수 있다. abstract override 메소드가 어떤 트레이트에 있다면, 그 트레이트는 반드시 abstract override가 붙은 메소드에 대한 구체적 구현을 제공하는 클래스에 Mixin해야한다.
