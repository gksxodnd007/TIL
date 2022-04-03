### 멤버 확장 함수

- 지양해야하는 패턴
- 리시버 타입에 같은 이름의 프로퍼티가 있다면 리시버 타입의 프로퍼티 값으로 오버라이딩 된다.

```kotlin
fun main(args: Array<String>) {
    Sut().apply { Dut().sum() }
}

class Sut(val a: Int = 10, val b: Int = 20) {

    fun Dut.sum(): Int {
        val sum = a + b
        println(sum) // Dut의 a와 Sut의 b가 이용되어 50 출력

        return sum
    }
}

class Dut(val a: Int = 30)
```

- 리시버 타입(Dut)의 프로퍼티가 private 접근제어자라면 오버라이드 되지 않고 멤버 주인(클래스) 객체의 값을 이용

```kotlin
fun main(args: Array<String>) {
    Sut().apply { Dut().sum() }
}

class Sut(val a: Int = 10, val b: Int = 20) {

    fun Dut.sum(): Int {
        val sum = a + b
        println(sum) // Dut의 a접근제어자가 private이므로 Sut의 a, b가 이용되어 30 출력

        return sum
    }
}

class Dut(private val a: Int = 30)
```
