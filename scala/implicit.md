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
