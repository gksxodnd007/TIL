### Singleton 객체 생성 idom

```java
public class Singleton {

    private Singleton() {}

    private static class LazyHolder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```

위의 코드는 Singleton객체를 on demand(필요할 때 생성) 방식으로 구현한 java의 싱글톤 패턴 idiom이다.
이외에 싱클톤 객체를 생성하는 방법들이 몇 가지 있지만 on demand 방식이 아니거나 double checked locking을 사용하게 되는데double checked locking은 고질적인 문제를 갖고 있는 java의 anti pattern이다.
java는 클래스 로더가 클래스 정보를 메모리에 올릴 때 multi thread 환경에서 안전하게 초기화 해준다.
이에 더해 클래스 로더는 동적으로 클래스를 로딩하기 때문에 실제 해당 클래스를 사용하는 시점에 메모리에 올린다.
이러한 클래스 로더의 특징을 활용하여 java singleton idom이 위와 같은 코드로 안착되었다.
이러한 기법을 piggy back(편승)기법 이라고 한다. 어떠한 구현체의 장점에 얹혀서 그 장점으로 보는 혜택을 얻어간다. 라는 의미이다.
