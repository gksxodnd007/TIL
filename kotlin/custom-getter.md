### Custom 접근자

- 코틀린은 기본적으로 프로퍼티를 선언시 getter와 setter가 자동으로 생성된다. 여기서 커스텀 getter를 다음과 같이 정의 할 수 있다.
```kotlin
class Person {
  var name: String
  var age: Int
    get() {
      println("execute getter")
      return age
    }
}
```
- field식별자는 커스텀 getter에서 값을 저장해서 사용한다. 게터에서는 field값을 읽을 수 만 있고, 세터에서는 field값을 읽거나 쓸 수 있다. 컴파일러는 디폴트 접근자 구현을 사용하건 직접 게터나 세터를 정의하건 관계없이 게터나 세터에서 field를 사용하는 프로퍼티에 대해 뒷 받침하는 필드를 생성해준다. 다만, field를 사용하지 않는 커스텀 접근자 구현을 정의한다면 뒷받침하는 필드는 존재하지 않는다.(프로퍼티가 val인 경우에는 게터에 field가 없으면 되지만, var인 경우에는 게터나 세터 모두에 field가 없어야 한다.)
```kotlin
class Person {
  var name: String
  var age: Int
    get() {
      println("execute getter")
      return field
    }
    set(value: Int) {
      field = value
    }
}
```
- 다음과 같이 getter를 커스터마이징하면 무한 재귀에 빠지게 된다. 그 이유는 코틀린의 프로퍼티에대해 알고 있다면 이해할 수 있을 것이다.
```kotlin
class Person {
  var name: String
  var age: Int
    get() {
      return this.age
    }
}
```
