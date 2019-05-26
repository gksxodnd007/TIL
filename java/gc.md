### JVM Garbage Collection

### Serial Collector VS Parallel Collector
- Young GC를 Serial Collector는 Single CPU를 이용하여(싱글스레드) 처리한다. 반면에 Parallel Collector는 (Multi CPU환경에서) 병렬로 처리한다.
- Parallel Collector도 Old GC는 Single CPU를 이용하여(싱글스레드로) GC작업을 진행하게된다.(Mark-Sweep-Compaction)
> Mark: 살아있는 객체 표시, Sweep: 살아있는 객체들로 부터 객체들을 찾아다니면서 garbage들을 식별한다. Compaction: 쓰레기 객체들을 지우고 살아있는 객체들과 비어있는 공간을 분리한다.

### Parallel Compaction Collector
- Parallel Collector와 가장 큰 차이점은 Old영역들을 잘게 나누어 Old GC도 병렬로 처리한다는 것이다.
- Parallel Compaction Collector Sweep phase 대신에 Summary라는 Phase가 이를 대신하는데, Summary phase에서는 object를 훑는 것이 아니라 region을 훑는다. (Compaction단계에서 살아있는 객체들을 시작 영역으로 모아두기 때문)

### JVM GC 관련 링크
- GC 대상 search 알고리즘: https://d2.naver.com/helloworld/329631
- GC 알고리즘: https://d2.naver.com/helloworld/1329
- JVM GC 튜닝 가이드 문서: https://docs.oracle.com/en/java/javase/11/gctuning/introduction-garbage-collection-tuning.html#GUID-326EB4CF-8C8C-4267-8355-21AB04F0D304
