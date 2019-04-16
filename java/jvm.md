## JVM 세팅

- example
> java -jar -Dspring.profiles.active=production -server -XX:+UseParallelGC -Xms512m -Xmx512m -XX:NewSize=256m -XX:MaxNewSize=256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/analysis ./kk-0.0.1-SNAPSHOT.jar &

- Xmx512m : heap memory를 최대 512mb까지 이용(default: os memory / 4)
- Xms512m : heap memory init 512mb
- XX:+HeapDumpOnOutOfMemoryError
- XX:HeapDumpPath

### 주의
- .jar파일명 앞에 jvm 옵션을 주어야함.
- application parameter는 그 뒤에 주어야함.
