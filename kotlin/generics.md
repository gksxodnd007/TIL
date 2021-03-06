## Kotlin 제네릭스

코틀린에서의 제네릭스는 한 번쯤 정리를 해보고 싶어 글을 쓰게 되었다. 드디어 코틀린을 사용하는데 필요한 뼈대를 익힌 것 같아 이후에는 코틀린과 스프링으로 side project를 한 번 진행해보면 재밌을 것 같다. 이 글은 제네릭스를 자세하게 다루지 않는다. Kotlin 제너릭스에서의 새로운 특징들인 다음의 내용들만 정리한다.

- 제네릭 정의
- 타입 파라미터 제약
- 실체화한 타입 파라미터
- 선언 지점 변성

### 제네릭 정의
**제네릭 함수**
```kotlin
fun <T> genericsFunction(param: T): T {
  ...
}
```
**제네릭 확장 프로퍼티**
- 일반 프로퍼티는 타입 파라미터를 가질 수 없다. 클래스 프로퍼티에 여러 타입의 값을 저장할 수는 없으므로 제네릭한 일반 프로퍼티는 말이 되지 않는다. 다음은 리스트의 마지막 원소 바로 앞에 있는 원소를 반환하는 확장 프로퍼티다.
```kotlin
val <T> List<T>.penultimate: T
  get() = this[size - 2]
```
**제네릭 클래스**
```kotlin
class GenericsClass<T>(val property: T) {
  ...
}
```
### 타입 파라미터 제약
- 타입 파라미터 제약은 클래스나 함수에 사용할 수 있는 타입 인자를 제한하는 기능이다.

**상한 지정**
```kotlin
class Weapon(val damage) {
  ...
}

class WeaponPocket<T : Weapon>(val weapons: T) {
  ...
}
```
- 위와 같은 경우 무기 주머니는 Weapon의 하위 타입만 타입인자로 넘겨줄 수 있다.

**타입 파라미터에 여러 제약 지정**
```kotlin
fun <T> ensureTrailingPeriod(seq: T) where T : CharSequence, T : Appendable {
  ...
}
```

**타입 파라미터를 널이 될 수 없는 타입으로 한정**
- 기본적으로 타입 파라미터에 타입 인자로 널이 될 수 있는 타입이 들어올 수 도있다.
```kotlin
class Processor<T>
  fun process(value: T) {
    value?.hashCode()
  }
```
- 따라서 위와 같이 안전한 호출 연산자를 이용하여야 한다. 다음과 같이 정의하면 널이 될 수 없는 타입으로 한정 할 수 있다.
```kotlin
class Processor<T : Any>
  fun process(value: T) {
    value.hashCode()
  }
```

### 실체화한 타입 파라미터
자바 개발자라면 알고 있겠지만 JVM의 제네릭스는 보통 타입 소서를 사용해 구현된다. 이는 실행 시점에 제네릭 클래스의 인스턴스에 타입 인자 정보가 들어있지 않다는 뜻이다. 코틀린 타입 소거가 실용적인 면에서 어떤 영향을 끼치는지 살펴보고 함수를 inline으로 선언함으로써 이런 제약을 어떻게 우회할 수 있는지 살펴본다. 함수를 inline으로 만들면 타입 인자가 지워지지 않게 할 수 있다.(코틀린에서는 이를 실체화라고 부른다.)
- 자바와 마찬가지로 코틀린 제네릭 타입 인자 정보는 런타임에 지워진다. 이는 제네릭 클래스 인스턴스가 그 인스턴스를 생성할 때 쓰인 타입 인자에 대한 정보를 유지하지 않는다는 뜻이다. 예를 들어 List\<String\> 객체를 만들고 그 안에 문자열을 여럿 넣더라도 실행 시점에는 그 객체를 오직 List로만 볼 수 있다. 그 List 객체가 어떤 타입의 원소를 저장하는지 실행 시점에는 알 수 없다. 이는 저장해야 하는 타입 정보의 크기가 줄어들어서 전반적인 메모리 사용량이 줄어든다는 제네릭 타입 소거의 장점이 있다. 그럼 이제 타입을 실체화하는 코드를 보자.
```kotlin
inline fun <reified T> weaponCheck(value: Weapon): Boolean = value is T
```
- 타입 파라미터는 런타임에 알 수 없는 정보다. 근데 어떻게 런타임에 저 함수가 호출되어 객체가 무기라는 것을 알 수 있을까? 답은 inline과 reified라는 선언 때문이다. 어떤 함수에 inline 키워드를 붙이면 컴파일러는 그 함수를 호출한 식을 모두 함수 본문으로 바꾼다. 함수가 람다를 인자로 사용하는 경우 그 함수를 인라인 함수로 만들면 람다 코드도 함께 인라이닝 되고, 그에 따라 무명 클래스와 객체가 생성되지 않아서 성능이 더 좋아질 수 있다. reified(사전 용어를 찾아보면 구체화된 이라는 뜻이다.)는 이 타입 파라미터가 실행 시점에 지워지지 않음을 표시한다.(참고로 인라인 함수에서만 실체화한 타입 인자를 쓸 수 있다.)

### 선언 지점 변성
- 변성이란 개념부터 알아보자. 변성 개념은 List\<String\>와 List\<Any\>와 같이 기저 타입이 같고 타입 인자가 다른 여러 타입이 서로 어떤 관계가 있는지 설명하는 개념이다. 선언 지점 변성이란 이러한 특징을 선언 시점에 정한다는 의미이다. 선언 시점에 정한다면 이후 모든 곳에서 사용할 때마다 명시하지 않아도 정해진 룰을 따른다. 반대로 사용 지점 변성이란 것이 있는데 이것은 자바의 와일드 카드 타입을 떠올리면 된다. 사용 할 때마다 변성에 대한 특징을 명시하는 것을 말한다.
- A가 B의 하위 타입이면 List\<A\>는 List\<B\>의 하위 타입이다. 그런 클래스나 인터페이스를 **공변적**이라 말한다. 이전과 같은 조건으로 List\<B\>가 List\<A\>의 하위타입이라면 **반공변적**이라 말한다. 예시 코드를 보자.
```kotlin
interface Producer<out T : Weapon> {
  fun produce(): T
}

interface Consumer<in T : Weapon> {
  fun consume(weapon: T)
}
```
- 위와 같이 타입 파라미터의 인자가 return에만 쓰인다면 공변적, paramter로 input값에만 쓰인다면 반공변적이라 한다. 이 두가지의 규칙을 컴파일시점에서 강제 할 수 있는 것이 변성을 이용한 제네릭이다. 이렇게 되면 Producer\<Gun\>은 Producer\<Weapon\>의 하위 타입이고 Consumer\<Weapon\>은 Consumer\<Gun\>의 하위타입이다. 자바의 \<? extends Weapon\>와 \<? super Weapon\> 를 떠올리면 이에대해 이해하기 수월할 것이다.
