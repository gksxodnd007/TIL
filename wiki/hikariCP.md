## HikariCP 옵션

jdbcUrl, username, password는 너무 기본적인 내용이라 생략하겠습니다.
HikariCP설정의 시간 단위는 ms입니다.

- **autoCommit:** auto-commit설정 (default: true)
- **connectionTimeout:** pool에서 커넥션을 얻어오기전까지 기다리는 최대 시간, 허용가능한 wait time을 초과하면 SQLException을 던짐. 설정가능한 가장 작은 시간은 250ms (default: 30000 (30s))
- **idleTimeout:** pool에 일을 안하는 커넥션을 유지하는 시간. 이 옵션은 minimumIdle이 maximumPoolSize보다 작게 설정되어 있을 때만 설정. pool에서 유지하는 최소 커넥션 수는 minimumIdle (A connection will never be retired as idle before this timeout.). 최솟값은 10000ms (default: 600000 (10minutes))
- **maxLifetime:** 커넥션 풀에서 살아있을 수 있는 커넥션의 최대 수명시간. 사용중인 커넥션은 maxLifetime에 상관없이 제거되지않음. 사용중이지 않을 때만 제거됨. 풀 전체가아닌 커넥션 별로 적용이되는데 그 이유는 풀에서 대량으로 커넥션들이 제거되는 것을 방지하기 위함임. 강력하게 설정해야하는 설정 값으로 데이터베이스나 인프라의 적용된 connection time limit보다 작아야함. 0으로 설정하면 infinite lifetime이 적용됨(idleTimeout설정 값에 따라 적용 idleTimeout값이 설정되어 있을 경우 0으로 설정해도 무한 lifetime 적용 안됨). (default: 1800000 (30minutes))
- **connectionTestQuery:** JDBC4 드라이버를 지원한다면 이 옵션은 설정하지 않는 것을 추천. 이 옵션은 JDBC4를 지원안하는 드라이버를 위한 옵션임(Connection.isValid() API.) 커넥션 pool에서 커넥션을 획득하기전에 살아있는 커넥션인지 확인하기 위해 valid 쿼리를 던지는데 사용되는 쿼리 (보통 SELECT 1 로 설정) JDBC4드라이버를 지원하지않는 환경에서 이 값을 설정하지 않는다면 error레벨 로그를 뱉어냄.(default: none)
- **minimumIdle:** 아무런 일을 하지않아도 적어도 이 옵션에 설정 값 size로 커넥션들을 유지해주는 설정. 최적의 성능과 응답성을 요구한다면 이 값은 설정하지 않는게 좋음. default값을 보면 이해할 수있음. (default: same as maximumPoolSize)
- **maximumPoolSize:** pool에 유지시킬 수 있는 최대 커넥션 수. pool의 커넥션 수가 옵션 값에 도달하게 되면 idle인 상태는 존재하지 않음.(default: 10)
- **poolName:** 이 옵션은 사용자가 pool의 이름을 지정함. 로깅이나 JMX management console에 표시되는 이름.(default: auto-generated)
- **initializationFailTimeout:** 이 옵션은 pool에서 커넥션을 초기화할 때 성공적으로 수행할 수 없을 경우 빠르게 실패하도록 해준다. 상세 내용은 한국말보다 원문이 더 직관적이라 생각되어 다음 글을 인용함.
> This property controls whether the pool will "fail fast" if the pool cannot be seeded with an initial connection successfully. Any positive number is taken to be the number of milliseconds to attempt to acquire an initial connection; the application thread will be blocked during this period. If a connection cannot be acquired before this timeout occurs, an exception will be thrown. This timeout is applied after the connectionTimeout period. If the value is zero (0), HikariCP will attempt to obtain and validate a connection. If a connection is obtained, but fails validation, an exception will be thrown and the pool not started. However, if a connection cannot be obtained, the pool will start, but later efforts to obtain a connection may fail. A value less than zero will bypass any initial connection attempt, and the pool will start immediately while trying to obtain connections in the background. Consequently, later efforts to obtain a connection may fail. Default: 1
- **readOnly:** pool에서 커넥션을 획득할 때 read-only 모드로 가져옴. 몇몇의 database는 read-only모드를 지원하지 않음. 커넥션이 read-only로 설정되어있으면 몇몇의 쿼리들이 최적화 됨.(default: false)
- **driverClassName:** HikariCP는 jdbcUrl을 참조하여 자동으로 driver를 설정하려고 시도함. 하지만 몇몇의 오래된 driver들은 driverClassName을 명시화 해야함. 어떤 에러 메시지가 명백하게 표시 되지않는다면 생략해도됨.
- **validationTimeout:** valid 쿼리를 통해 커넥션이 유효한지 검사할 때 사용되는 timeout. 250ms가 설정될 수 있는 최솟값(default: 5000ms)
- **leakDetectionThreshold:** 커넥션이 누수 로그메시지가 나오기 전에 커넥션을 검사하여 pool에서 커넥션을 내보낼 수 있는 시간. 0으로 설정하면 leak detection을 이용하지않음. 최솟값 2000ms (default: 0)

**Statement Cache**
많은 connection pool(including dbcp, vibur, c3p0)라이브러리들은 PreparedStatement caching을 지원하지만 HikariCP는 지원하지않는다. connection pool layer에서 PreparedStatements는 각 커넥션 마다 캐싱된다. 애플리케이션에서 250개의 공통적인 쿼리를 캐싱하고 있고 커넥션 pool size가 20이라면 database에 5000 쿼리 실행계획을 hold하고 있게된다. 대부분의 major database의 jdbc driver들은 이미 설정을 통해 Statement를 캐싱할 수 있다.(PostgreSQL, Oracle, Derby, MySQL, DB2 등등) 즉 우리가 원하는대로 250개의 쿼리 실행계획만 데이터베이스에 캐싱할 수 있음을 의미한다.(connection pool에 설정하는 것이 아닌 driver에 설정함으로써)
> ps. Clever implementations do not even retain PreparedStatement objects in memory at the driver-level but instead merely attach new instances to existing plan IDs.

**Log Statement Text / Slow Query Logging**
Statement Cache와 마찬가지로 대부분의 vendor들은 driver단에서 해결

## Performance Tips
MySQL Performance Tips
- **prepStmtCacheSize:** This sets the number of prepared statements that the MySQL driver will cache per connection. The default is a conservative 25. We recommend setting this to between 250-500.
- **prepStmtCacheSqlLimit:** This is the maximum length of a prepared SQL statement that the driver will cache. The MySQL default is 256. In our experience, especially with ORM frameworks like Hibernate, this default is well below the threshold of generated statement lengths. Our recommended setting is 2048.
- **cachePrepStmts:** Neither of the above parameters have any effect if the cache is in fact disabled, as it is by default. You must set this parameter to true.
- **useServerPrepStmts:** Newer versions of MySQL support server-side prepared statements, this can provide a substantial performance boost. Set this property to true.

A typical MySQL configuration for HikariCP might look something like this:
```properties
jdbcUrl=jdbc:mysql://localhost:3306/codingsquid
user=test
password=test
dataSource.cachePrepStmts=true
dataSource.prepStmtCacheSize=250
dataSource.prepStmtCacheSqlLimit=2048
dataSource.useServerPrepStmts=true
dataSource.useLocalSessionState=true
dataSource.rewriteBatchedStatements=true
dataSource.cacheResultSetMetadata=true
dataSource.cacheServerConfiguration=true
dataSource.elideSetAutoCommits=true
dataSource.maintainTimeStats=false
```
참고 자료 : https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby
