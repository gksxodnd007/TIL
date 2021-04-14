### Callback Method
- 어떤 함수의 인자로 함수를 전달하고 특정 이벤트가 발생하면 인자로 전달된 함수를 실행시킨다. 여기서 인자로 전달되는 함수를 callback 메서드라고 지칭한다.
- 비동기 패러타임에서 callback 메서드는 중요하다. 특정 task가 끝났을 때 후처리를 어떻게 할건지 설계해야한다. 호출자로 결과를 반환하여 호출자가 처리할 수 있고(non-blocking & synchronous) 호출자로 반환하지 않고 callback 메서드를 정의하여 넘겨주면 후처리를 callback 메서드를 통해 처리한다.(non-blocking & asynchronous) 후자의 방법을 강력하게 추천한다.
