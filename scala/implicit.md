### 암시적 변환

- 암시적 변환은 단일 식별자로 스코프 안에 존재해야한다. 컴파일러는 some.convert 같은 형태의 변환을 삽입하지 않는다. 예를 들어, 컴파일러는 x + y를 some.convert(x) + y로 확장하지 않는다. 암시 처리 시 some.convert를 사용하게 만들고 싶다면, 임포트해서 단일 식별자로 가리킬 수 있게 만들어야 한다.

1. 첫 번째 방법 (object class를 이용)
```scala
object Some {

  implicit def convert(x: X): Plusable {
    new Plusable(x)
  }
}

case class X(x: Int)
case class Y(y: Int)

class Test {

  def main(args: Array[String]): Unit = {
    val x: X = X(1)
    val y: Y = Y(1)

    // val sum = x + y // 컴파일 에러

    import Some._

    val sum = x + y // 컴파일 성공 : convert(x) + y 로 변환됨.
  }
}
```

2. 두 번째 방법 (객체를 이용)
```scala
class Some {

  implicit def convert(x: X): Plusable {
    new Plusable(x)
  }
}

case class X(x: Int)
case class Y(y: Int)

class Test {

  def main(args: Array[String]): Unit = {
    val x: X = X(1)
    val y: Y = Y(1)

    // val sum = x + y // 컴파일 에러

    val some: Some = new Some()

    import some._

    val sum = x + y // 컴파일 성공 : convert(x) + y 로 변환됨.

  }
}
```

3. 세 번째 방법 (함수를 이용)
```scala
case class X(x: Int)
case class Y(y: Int)

class Test {

  implicit def convert(x: X): Plusable {
    new Plusable(x)
  }

  def main(args: Array[String]): Unit = {
    val x: X = X(1)
    val y: Y = Y(1)

    val sum = x + y // 컴파일 성공 : convert(x) + y 로 변환됨.

  }
}
```

### 암시 규칙

- 표시 규칙: `implicit`로 표시한 정의만 검토 대상
- 스코프 규칙: 삽입된 implicit 변환은 스코프 내에 단일 식별자로만 존재하거나, 변환의 결과나 원래 타입과 연관이 있어야한다.
  - `단일 식별자`규칙에는 한 가지 예외가 있다. 컴파일러는 원 타입이나 변환 결과 타입의 동반 객체에 있는 암시적 정의도 살펴본다. 예를 들어 Dollar객체를 Euro인자로 취하는 메서드에 전달한다면, Dollar가 원 타입이고 Euro가 결과 타입이다. 따라서 두 클래스(즉 Dollar나 Euro)의 동반 객체 안에 Dollar에서 Euro로 변환하는 암시적 변환을 넣어둘 수 있다.
- 한 번에 하나만 규칙: 오직 하나의 암시적 선언만 사용한다.
- 명시성 우선 규칙: 코드가 그 상태 그대로 타입 검사를 통과한다면 암시를 통한 변환을 시도하지 않는다.

### 암시적 변환 이름 붙이기

- 암시적 변환에는 아무 이름이나 붙일 수 있다. 이름이 문제가 되는 경우는 두 가지뿐이다. 메소드 호출 시 명시적으로 변환을 사용하고 싶은 경우와, 프로그램의 특정 지점에서 사용 가능한 암시적 변환이 어떤 것이 있는지 파악해야 하는 경우가 바로 그 두 경우다.
```scala
object ImplicitConverter {

  implicit def stringWrapper(s: String): IndexedSeq[Char] = {
    ...
  }
  implicit def intToString(x: Int): String = {
    ...
  }
}
```
- 우리의 애플리케이션에서 stringWrapper변환을 사용하고 싶지만, intToString이 정수를 자동으로 문자열로 바꾸는 것을 막고 싶다면 하나의 변환만 임포트하고 나머지는 임포트하지 않을 수 있다.
```scala
import ImplicitConverter.stringWrapper

... //stringWrapper를 암시적으로 사용하는 코드
```
- 이름을 통해서만 특정 암시적 변환은 임포트하고 그 밖의 것은 제외할 수 있다.

### 암시적 파라미터

- 컴파일러가 암시적 요소를 추가하는 다른 위치로는 인자 목록을 들 수 있다. 컴파일러는 때때로 `call(a)` 호출을 `call(a)(b)`로 바꾸거나, `new Service(a)`를 `new Service(a)(b)`로 바꿔서 함수 호출을 완성하는 데 필요한 빠진 파라미터 목록을 채워 넣어준다.
- 컴파일러는 마지막 파라미터 하나만이 아니고, 커링한 마지막 파라미터 목록 전체를 채워 넣는다. 예를 들어 `call`에 빠진 마지막 파라미터 목록이 세 가지 파라미터를 받아야 한다면, 컴파일러는 `call(a)`를 `call(a)(b, c, d)`로 변경한다. 이런식으로 사용하기 위해서는, (b, c, d)에 추가될 식별자인 b, c, d에 대한 정의에 `implicit`을 표시해둘 뿐만 아니라, `call`이나 `Service`클래스 정의의 마지막 파라미터 목록도 implicit로 표시해둬야한다.
```scala
implicit val b = 'b'
implicit val c = 'c'
implicit val d = 'd'

def call(a: Char)(implicit b: Char, c: Char, d: Char)

class Service(val a: Char)(implicit val b: Char, c: Char, d: Char) {
  ...
}
```
> implicit키워드는 개별적인 파라미터가 아니고 전체 파라미터 목록을 범위로 한다는 사실에 유의하라.
