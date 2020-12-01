## JVM Garbage Collector

### Serial GC VS Parallel GC
- Young GC를 Serial GC는 Single CPU를 이용하여(싱글스레드) 처리한다. 반면에 Parallel GC는 (Multi CPU환경에서) 병렬로 처리한다.
- Parallel GC도 Old GC는 Single CPU를 이용하여(싱글스레드로) GC작업을 진행하게된다.(Mark-Sweep-Compaction)
> Mark: 살아있는 객체 표시, Sweep: 살아있는 객체들로 부터 객체들을 찾아다니면서 garbage들을 식별한다. Compaction: 쓰레기 객체들을 지우고 살아있는 객체들과 비어있는 공간을 분리한다. -XX:+UseSerialGC or -XX:+UseParallelGC


### Parallel Old GC
- Parallel GC와 가장 큰 차이점은 Old영역들을 잘게 나누어 Old GC도 병렬로 처리한다는 것이다.
- Parallel Old GC는 Sweep phase 대신에 Summary라는 phase가 이를 대신하는데, Summary phase에서는 object를 훑는 것이 아니라 region을 훑는다. (Compaction단계에서 살아있는 객체들을 시작 영역으로 모아두기 때문)
> -XX:+UseParallelOldGC

Parallel Old GC & Parallel GC 참고 글
- https://sarc.io/index.php/java/478-gc-useparallelgc-useparalleloldgc

### Concurrent Mark Sweep GC
- low-latency collector로도 알려져 있으며 heap memory 영역의 크기가 클 때 적합하다. CMS GC는 다음 옵션을 통해 사용할 수 있다.
> -XX:+UseConcMarkSweepGC

- CMS GC는 대부분 어플리케이션 스레드와 동시에 작동한다. 기본적으로 가용 스레드 절반을 동원해 GC 동시 단계를 수행하고, 나머지 절반은 어플리케이션 스레드가 자바 코드를 실행하는 사용된다. 따라서 CMS GC 수행 도중에 새로운 객체가 할당된다. 만약 CMS GC 실행 도중 Eden영역이 꽉 차버리면 어플리케이션의 스레드가 더 이상 진행될 수 없어 STW가 발생하는 Young GC가 일어난다. 이 때 Young GC는 코어 절반만 사용(나머지 절반은 CMS GC에 사용)하므로 ParallelOldGC Young GC보다 더 오래걸린다. 이때 일부 객체는 Old영역으로 승격된다. CMS GC가 실행되는 동안에 승격된 객체는 Old영역으로 이동시켜야하는데, 두 수집기기 간에 조정이 필요한 부분이다. 이런 이유로 CMS는 조금 다른 Young GC(`-XX:-UseParNewGC`)를 사용한다.
- Young 영역에 대한 GC는 기본적으로 Parallel Collector와 동일하지만 다음 option으로 조금 더 향상된 Young GC를 적용할 수 있다.
> -XX:-UseParNewGC -> java 8부터는 -XX:+UseConcMarkSweepGC -XX:-UseParNewGC 이 조합이 Deprecate 되었다.
그냥 -XX:+UseConcMarkSweepGC만 사용하면 된다.

- CMS GC는 다음 단계를 거친다.
  - 초기 표시 단계 (STW): 매우 짧은 대기 시간으로 살아 있는 객체를 찾는 단계
  - 컨커런트 표시 단계: 서버 수행과 동시에 살아 있는 객체에 표시를 해 놓는 단계
  - 재표시(remark) 단계: 컨커런트 표시 단계에서 표시하는 동안 변경된 객체에 대해서 다시 표시하는 단계
  - 컨커런트 스윕 단계: 표시되어 있는 쓰레기를 정리하는 단계
- CMS는 Compaction단계를 거치지 않기 때문에 왼쪽으로 메모리를 몰아 놓는 작업을 수행하지 않는다. 따라서 Old 영역에 객체보다 큰 여유 공간이 있을지라도 fragment로 인해 객체가 Old 영역으로 승격되지 못하는 경우가 발생한다. 이럴 경우에는 ParallelOldGC가 수행되어 Compaction이 이루어진다.
  - 위와 같은 상황을 Concurrent Failure Mode(이하 CMF)라고 한다. CMS GC는 대부분 어플리케이션 스레드와 동시에 작동한다. 실행 객체의 할당압이 높거나 객체의 크기가 커서 조기승격이 자주 발생 할 때 CMF가 발생할 확률이 높아진다. 이를 해결하기 위해서는 `-XX:CMSInitiatingOccupancyFraction=<N>` 옵션을 주어 CMS가 언제 수집을 시작할지 설정할 수 있다. 기본적으로 최초의 CMS Full GC는 heap이 75% 찼을 때 시작된다. CMS가 제때 수행되어 Old 영역을 정리해 준다면 CMF발생 빈도를 낮출 수 있다.

CMS GC 참고 글
- http://insightfullogic.com/2013/May/07/garbage-collection-java-3/
- https://blogs.oracle.com/poonam/understanding-cms-gc-logs
- https://stackoverflow.com/questions/2918124/how-to-reduce-java-concurrent-mode-failure-and-excessive-gc

### JVM GC 관련 링크
- GC 대상 search 알고리즘: https://d2.naver.com/helloworld/329631
- GC 알고리즘: https://d2.naver.com/helloworld/1329
- JVM GC 튜닝 가이드 문서: https://docs.oracle.com/en/java/javase/11/gctuning/introduction-garbage-collection-tuning.html#GUID-326EB4CF-8C8C-4267-8355-21AB04F0D304
