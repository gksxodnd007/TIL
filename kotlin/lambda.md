### Kotlin 람다

#### 표현 방법
- 코틀린에서 람다식은 항상 중괄호로 둘러싸여 있다. 인자 목록 주변에 괄호가 없다는 사실을 기억해야한다.

**Java**
```java
(int x, int y) -> x + y
```
**Kotlin**
```kotlin
{ x: Int, y: Int -> x + y }
```

#### 고차 함수
- 고차 함수란 함수를 파라미터 인자로 받고 반환할 수 있음을 의미한다. 그러려면 함수 타입이 필요한데 코틀린에서 함수타입은 다음과 같이 표현한다.
```kotlin
val process: (Data, Data) -> Result = { x, y -> x + y}//변수
fun process(operator: (Data, Data) -> Result): (Data, Data) -> Result //파라미터 혹인 반환
```
- 함수 타입을 정의하려면 위와 같이 함수 파라미터의 타입을 괄호 안에 넣고, 그뒤에 화살표(->)를 추가한 다음, 함수의 반환 타입을 지정하면 된다.
- 변수에 함수타입을 미리 선언했다면 람다식을 표현할 때 타입을 명시하지 않아도 타입을 추론할 수 있다.

#### Inline(인라이닝)
- 함수를 인라이닝 한다는 것은 함수 본문을 컴파일 할 때 함수를 호출한 지점에 풀어주는 것을 의미한다. 다음과 같이 inline키워드를 붙이면된다.
```kotlin
inline fun invoke(function: (Int) -> Unit): Unit {
  function(26)
  function.invoke(26)
}
```
- 위와 같이 람다를 파라미터로 받아서 바로 호출하는 경우 해당 람다식도 인라이닝된다. (단, 람다를 다음과 같이 직접 받아야만 함.)
```kotlin
inline fun invoke(function: (Int) -> Unit): Unit {
  function(26)
  function.invoke(26)
}

fun doSome() {
  invoke { x: Int -> println(x) }
}
```
- 다음과 같은 경우는 람다가 인라이닝 되지 않음.
```kotlin
inline fun firstInvoke(function: (Int) -> Unit): Unit {
  secondInvoke(function)
}

inline fun secondInvoke(function: (Int) -> Unit): Unit {
  function.invoke(26)
}
```
