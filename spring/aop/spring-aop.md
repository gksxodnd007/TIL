### AOP (Aspect Oriented Programming)

- order 값이 낮은 advice가 higher priority를 갖는다.
- 같은 class 안에서 같은 지점에 pointCut될 advice 메서드에 order를 지정하면 효과가 없다. 이럴 경우에는 메서드 이름의 사전 순서로 실행이 되는 것을 확인하였다.
- higer priority를 갖는 advice는 메서드 진입 전 가장 먼저 실행되고 메서드 실행 후에는 가장 마지막에 실행된다. 즉, @Before advice는 priority가 높은 순서대로 @After advice는 priority가 낮은 순서대로 실행된다.
