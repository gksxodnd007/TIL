### Scala 표현식

#### for expression
- 일반적으로 for 표현식은 다음과 같은 형태이다.
> for ( seq ) yield expr

- for expression 에서 yield가 생성하는 컬렉션 타입은 가장 바깥 루프의 제너레이터에 사용된 컬렉션타입을 따른다.
- 여기서 seq는 제너레이터(generator), 정의(definition), 필터(filter)를 나열한 것이다. 연속된 구성요소 사이에는 세미콜론(;)을 넣는다.
```scala
/* example code */
case class Person(val name: String, val age: Int)

object Test {
  def main(args: Array[String]): Unit = {
    val persons = List(Person("hubert", 26), Person("david", 29), Person("Ryun", 29), Person("Freddie", 30))

    val result = for (p <- persons; n = p.name; if (n startWith "h")) yield n
    println(result) // 결과 출력: List("hubert")
  }
}
```

- 제너레이터가 여러 개 있다면 뒤쪽 제너레이터는 앞의 것보다 더 빨리 변화한다. (즉, 이중 loop에서 안쪽 loop에 해당하게 된다.)
```scala
/* example code */
object Test {
  def main(args: Array[String]): Unit = {
    val result: List[(Int, String)] = for (x <- List(1, 2); y <- List("one", "two")) yield (x, y)

    println(result) // 결과 출력: List((1, one), (1, two), (2, one), (2, two))
  }
}
```

- for expression은 컴파일시 map, flatMap, withFilter, foreach 메서드의 조합으로 변환된다. (즉,이 메서들에만 의존한다.)
```scala
/* example pseudo code */

1. for (x <- expr1) body
 :expr1 foreach (x => body)
2. for (x <- expr1; if expr2; y <- expr3) body
 :expr1 withFilter (x => expr2) foreach (x => expr3 foreach (y => body))
3. for (x <- expr1; if expr2) yield body
 :expr1 withFilter (x => expr2) map (x => body)
```
- 어떤 데이터 타입이 for 표현식과 for 루프를 모두 지원하기 위해서는 map, flatMap, withFilter, foreach를 메소드로 정의해야한다. 하지만 이런 메소드 중 일부만을 정의할 수도 있다. 그런 경우에는 for 표현식이나 루프의 기능 중 일부만 사용 가능하다.
  - map만 정의했다면 제너레이터가 1개만 있는 for 표현식을 사용할 수 있다.
  - map과 flatMap을 함께 정의했다면, 제너레이터가 여럿 있는 for 표현식도 사용 가능 하다.
  - foreach를 정의했다면, for 루프(제너레이터 개수는 관계없음)를 사용할 수 있다.
  - withFilter를 정의하면, for 표현식 안에서 if로 시작하는 for 필터 표현식을 사용할 수 있다.
