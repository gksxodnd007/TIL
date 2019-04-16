### 람다 표현식

- 자바8에서 람다 표현식은 익명메서드라고도 한다. 즉, 다음은 같은 코드다
```java
public static void main(String... args) {
  Thread thread = new Thread(new Runnable() {
    @Override
    public void run() {
      System.out.println("i love supercell");
    }
  })

  thread.start();
}
```

```java
public static void main(String... args) {
  Thread thread = new Thread(() -> System.out.println("i love supercell"));

  thread.start();
}
```

- 여기서 의문점이 하나 생긴다. 그럼 리스트의 forEach()문에 람다 표현식을 사용하면 늘 new 연산자를 사용해서 heap memory와 Cpu자원을 소모하는 것일까???(Kotlin에서는 람다식안에 포획된 변수가 없으면 무명 객체를 반복 사용한다.)
```java
list.forEach(() -> System.out.println("hello"));
```

검색해보니 다음과 같은 답변이 있었다.
> It is equivalent but not identical. Simply said, if a lambda expression does not captures values, it will be a singleton that is re-used on every invocation. The behavior is not exactly specified. The JVM is given a big freedom how to implement it. Currently, Oracle’s JVM creates (at least) one instance per lambda expression (i.e. doesn’t share instance between different identical expressions) but creates singleton for all expression which don’t capture values.

**즉, 자바에서도 람다식 본문에 외부의 변수를 포획하지 않으면 재사용 된다는 얘기다.** 하지만 이는 Oracle jdk에서 구현해놓은 방식이고 이에대해서는 자율성이 주어진다고 되어있다.

여기서 또 의문점이 하나 생긴다. 그럼 람다식을 선언할 때마다 클래스가 하나 생성되어야 하는 것인가? ClassLoader가 죽어나겠구나. 이와 관련해서는 다음과 같은 답변이 있었다.

> As each anonymous inner class would be loaded it would take up room in the JVM’s meta-space.

일단 meta-space를 사용한다고한다. meta-space는 java 8 이전에 perm 영역을 대체하는 것으로 native memory를 사용한다. (OOME가 사라진 이유) 그리고 java8부터 새로 나온 자바바이트코드 명령어인 invokedynamic. 이 명령어가 람다식을 호출하면 생기는 자바바이트코드이다.

- 관련 링크
  - https://stackoverflow.com/questions/27524445/does-a-lambda-expression-create-an-object-on-the-heap-every-time-its-executed
  - https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.4
  - https://stackoverflow.com/questions/23983832/is-method-reference-caching-a-good-idea-in-java-8/23991339#23991339
  - https://www.infoq.com/articles/Java-8-Lambdas-A-Peek-Under-the-Hood
