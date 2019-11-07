### Guice With Scala
- [Getting Started](https://github.com/google/guice/wiki/GettingStarted)

- Future를 사용할 때 ExecutionContext를 implict해서 사용할 수 있다.
```scala
@Singleton
class UserRepository @Inject()(implict ec: ExecutionContext) {

  ...
}
```

- 여기서 UserRepository를 다음과 같이 UserService에 의존성을 주입하려면 ExecutionContext의 구현체가 없다고 컴파일에러가 발생한다.
```scala
@Singleton
class UserService @Injecd()(val userRepository: UserRepository)(implicit ec: ExecutionContext) {

  ...
}

class GuiceModule extends AbstractModule {

  override def configure(): Unit = {
    bind(classOf[UserService])
  }
}
```

- ExecutionContext는 다음과 같이 trait로 정의되어 있다. ExecutionContext는 비동기 프로그래밍 모델을 추상화해서 쉽게 사용하기 위해 Scala에서 제공하는 trait이다.
- ExecutionContextExecutor는 ExecutionContext와 java의 Executor를 상속하는 trait이다.
```scala
trait ExecutionContext {

  def execute(runnable: Runnable): Unit

  def reportFailure(@deprecatedName('t) cause: Throwable): Unit

  def prepare(): ExecutionContext = this
}

private[scala] class ExecutionContextImpl private[impl] (final val executor: Executor, final val reporter: Throwable => Unit) extends ExecutionContextExecutor {
  require(executor ne null, "Executor must not be null")
  override final def execute(runnable: Runnable): Unit = executor execute runnable
  override final def reportFailure(t: Throwable): Unit = reporter(t)
}
```

- ExecutionContext의 구현체인 ExecutionContextImpl는 접근제어자가 private[scala]이다. 이를 다른 패키지에서 사용할 수 없다. 따라서 구현체가 없다고 컴파일 에러가 발생하는 것이다. 이를 다음과 같이 해결하였다.
```scala
@Singleton
class MyExecutionContextImpl extends ExecutionContextExecutorService {

  private lazy val numThreads: Int = ConfigLoaderHelper.getByPrefix("global.execution-context").getInt("numThreads")
  implicit lazy val ec: ExecutionContextExecutorService = ExecutionContext.fromExecutorService(Executors.newFixedThreadPool(numThreads))

  override def execute(command: Runnable): Unit = {
    ec.execute(command)
  }

  override def reportFailure(cause: Throwable): Unit = {
    ec.reportFailure(cause)
  }

  override def shutdown(): Unit = {
    ec.shutdown()
  }

  override def shutdownNow(): util.List[Runnable] = {
    ec.shutdownNow()
  }

  override def isShutdown: Boolean = {
    ec.isShutdown
  }

  override def isTerminated: Boolean = {
    ec.isTerminated
  }

  override def awaitTermination(timeout: Long, unit: TimeUnit): Boolean = {
    ec.awaitTermination(timeout, unit)
  }

  override def submit[T](task: Callable[T]): Future[T] = {
    ec.submit(task)
  }

  override def submit[T](task: Runnable, result: T): Future[T] = {
    ec.submit(task, result)
  }

  override def submit(task: Runnable): Future[_] = {
    ec.submit(task)
  }

  override def invokeAll[T](tasks: util.Collection[_ <: Callable[T]]): util.List[Future[T]] = {
    ec.invokeAll(tasks)
  }

  override def invokeAll[T](tasks: util.Collection[_ <: Callable[T]], timeout: Long, unit: TimeUnit): util.List[Future[T]] = {
    ec.invokeAll(tasks, timeout, unit)
  }

  override def invokeAny[T](tasks: util.Collection[_ <: Callable[T]]): T = {
    ec.invokeAny(tasks)
  }

  override def invokeAny[T](tasks: util.Collection[_ <: Callable[T]], timeout: Long, unit: TimeUnit): T = {
    ec.invokeAny(tasks, timeout, unit)
  }
}
```
- 위와 같이 기능을 위임하여 구현체를 @Singleton으로 정의한 후 다음과 같이 binding해주면 문제를 해결할 수 있다.
```scala
class GuiceModule extends AbstractModule {

  override def configure(): Unit = {
    bind(classOf[ExecutionContext]).to(classOf[MyExecutionContextImpl])
    bind(classOf[UserService])
  }
}
``
