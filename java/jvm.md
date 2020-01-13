## JVM 실행

- Bootstrap Class Loader가 , Object 클래스들을 비롯하여 자바 API들을 로드한다.
- Bootstrap Class Loader는 JVM을 기동할 때 생성되며, 다른 클래스 로더와 달리 자바가 아니라 네이티브 코드로 구현되어 있다.
> java -jar -Dspring.profiles.active=production -server -XX:+UseParallelGC -Xms512m -Xmx512m -XX:NewSize=256m -XX:MaxNewSize=256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/analysis ./kk-0.0.1-SNAPSHOT.jar &

- Xmx512m : heap memory를 최대 512mb까지 이용(default: os memory / 4)
- Xms512m : heap memory init 512mb
- XX:+HeapDumpOnOutOfMemoryError
- XX:HeapDumpPath

### 주의
- .jar파일명 앞에 jvm 옵션을 주어야함.
- application parameter는 그 뒤에 주어야함.

## JVM 종료

- JVM이 종료되는 두 가지 경우를 생각할 수 있는데, 하나는 예정된 절차대로 종료되는 경우이고, 또 하나는 예기치 못하게 임의로 종료되는 경우이다. 절차에 맞춰 종료되는 경우에는 "일반"스레드가 모두 종료되는 시점, 또는 어디에선가 System.exit메소드를 호출하거나 기타 여러 가지 상황(SIGINT 시그널을 받거나 CTRL+C를 입력한 경우)에 JVM 종료 절차가 시작된다.
- JVM은 모든 일반스레드가 종료되기 전에는 종료되지 않는다.
  - 일반 스레드를 제대로 종료시키지 않으면 JVM 자체가 종료되지 않고 대기하기도 한다.
- ExecutorService의 `shutdown` vs `shutdownNow`
  - `shutdown`: 안전한 종료 절차를 진행하며 종료중 상태로 진입한다. 이 상태에서는 새로운 작업을 등록받지 않으며, 이전에 등록되어 있던 작업(실행되지 않고 대기 중이던 작업도 포함)까지는 모두 끝마칠 수 있다.
  - `shutdownNow`: 강제 종료 절차를 진행한다. 현재 진행 중인 작업도 가능한 한 취소시키고, 실행되지 않고 대기 중이던 작업은 더 이상 실행시키지 않는다.
- 예정된 절차대로 종료되는 경우에 JVM은 가장 먼저 등록되어 있는 모든 종료 훅(shutdown hook)을 실행시킨다. 종료 훅은 Runtime.addShutdownHook 메소드를 통해 등록된 아직 시작되지 않은 스레드를 의미한다. 하나의 JVM에 여러개의 종료 훅을 등록할 수도 있으며, 두 개 이상의 종료 훅이 등록되어 있는 경우에 어떤 순서로 훅을 실행하는지에 대해서는 아무런 규칙이 없다. (종료 훅이 여러 개 등록되어 있는 경우에는 여러 개의 종료 훅이 서로 동시에 실행된다.) JVM은 종료 과정에서 계속해서 실행되고 있는 애플리케이션 내부의 스레드에 대해 중단 절차를 진행하거나 인터럽트를 걸지 않는다. 계속해서 실행되던 스레드는 종료 절차가 끝나는 시점에 강제로 종료된다.
```java
public void start() {
  Runtime.getRuntime().addShutdownHook(new Thread() {
    public void run() {
      ...
    }
  })
}
```
- JVM을 강제로 종료시킬 때(예기치 않은 종료)는 JVM이 스스로 종료되는 것 이외에 종료 훅을 실행하는 등의 어떤 작업도 하지 않는다.

## JVM Internal
- https://d2.naver.com/helloworld/1230
