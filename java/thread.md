### Thread Scheduling

- 자바의 쓰레드는 user thread 이지만 kernel thread와 매핑된다.
- kernel thread는 os에서 스케줄링하기 때문에 jvm에 thread scheduler가 필요 없다. 하지만 이는 vendor마다 다르고, oracle, openJDK는 kernel thread와 매핑되게 구현되어 있다.
> thread library가 thread 스케줄링을 관리하는 것을 green thread라고 하는데 이러면 cpu가 multi core일때 병렬성을 획득 할 수 없다.
