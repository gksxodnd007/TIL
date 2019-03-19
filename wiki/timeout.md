### Timeout에 관한 정리

#### Connection Timeout
- 클라이언트가 서버측으로 connection을 맺길 원하지만 서버의 장애 상황으로 connection조차 맺어지지 못할 때 발생하는 timeout이다.

#### Read Timeout
- 클라이언트와 서버가 connection은 맺어졌지만 I/O작업이 길어지거나 락이 걸려 요청이 처리되지 못하고 있을 때 클라이언트는 더 이상 기다리지 못하고 커넥션을 끊는다. 이런 상황을 Read Timeout 이라고 하는데 java에서는 SocketTimeout Exception이 떨어진다.
> JDBC Driver SocketTimeout은 OS의 SocketTimeout 설정에 영향을 받는다. JDBC Driver SocketTimeout을 설정하지 않아도 네트워크 장애 발생 이후 30분이 지나면 JDBC Connection Hang이 복구되는 것은 OS의 SocketTimeout 설정때문이다.

#### Statement Timeout
- 네트워크 연결 장애에 대한 timeout을 담당하는 것이 아니다.
- Statement 하나가 얼마나 오래 수행되어도 괜찮은지에 대한 한계 값이다. JDBC API인 Statement에 타임아웃 값을 설정하며, 이 값을 바탕으로 JDBC 드라이버가 StatementTimeout을 처리한다. JDBC API인 java.sql.Statement.setQueryTimeout(int timeout) 메서드로 설정한다.
> 네트워크 장애에 대비하는 타임아웃은 JDBC Driver SoecketTimeout이 처리해야 한다.

#### Transaction Timeout
- 프레임워크나 애플리케이션 레벨에서 유효한 타임아웃이다.
- 간단히 설명하면 "StatementTimeout x N(Statement 수행 수) + α(가비지 컬렉션 및 기타)"라고 할 수 있다. 전체 Statement 수행 시간을 허용할 수 있는 최대 시간 이내로 제한하려 할 때 TransactionTimeout을 사용한다.
> Statement 한 개를 수행할 때 0.1초가 필요하다면, 몇 개 안 되는 Statement를 수행할 때에는 문제가 없다. 그러나 Statement 10만 개를 수행할 때에는 일만 초(약 7시간)가 필요하다. TransactionTimeout은 이런 경우에 사용할 수 있다.

#### JDBC Driver's SocketTimeout
- JDBC Driver Type4는 소켓을 사용하여 DBMS에 연결하는 방식이고, 애플리케이션과 DBMS 사이의 ConnectTimeout 처리는 DBMS에서 하지 않는다.
- JDBC 드라이버의 SocketTimeout 값은 DBMS가 비정상으로 종료되었거나 네트워크 장애(기기 장애 등)가 발생했을 때 필요한 값이다. TCP/IP의 구조상 소켓에는 네트워크의 장애를 감지할 수 있는 방법이 없다. 그렇기 때문에 애플리케이션은 DBMS와의 연결 끊김을 알 수 없다. 이럴 때 SocketTimeout이 설정되어 있지 않다면 애플리케이션은 DBMS로부터의 결과를 무한정 기다릴 수도 있다(이러한 Connection을 Dead Connection이라고 부르기도 한다). 이러한 상태를 방지하기 위해 소켓에 타임아웃을 설정해야 한다. SocketTimeout은 JDBC 드라이버에서 설정할 수 있다. SocketTimeout을 설정하면 네트워크 장애 발생 시 무한 대기 상황을 방지하여 장애 시간을 단축할 수 있다.
> 단, SocketTimeout 값을 Statement의 수행 시간 제한을 위해 사용하는 것은 바람직하지 않다. 그러므로 SocketTimeout 값은 StatementTimeout 값보다는 크게 설정해야 한다. SocketTimeout값이 StatementTimeout보다 작으면, SocketTimeout이 먼저 동작하므로 StatementTimeout 값은 의미가 없게 되어 동작하지 않는다.
- SocketTimeout에는 아래 두 가지 옵션이 있고, 드라이버별로 설정 방법이 다르다.
  - Socket Connect 시 타임아웃(connectTimeout): Socket.connect(SocketAddress endpoint, int timeout) 메서드를 위한 제한 시간
  - Socket Read/Write의 타임아웃(socketTimeout): Socket.setSoTimeout(int timeout) 메서드를 위한 제한 시간

#### OS level socket timeout 설정
- SocketTimeout이나 ConnectTimeout을 설정하지 않으면 네트워크 장애가 발생해도 애플리케이션이 대부분 이를 감지할 수 없다. 따라서 연결이 되거나 데이터를 읽을 수 있을 때까지 애플리케이션이 무한정 기다리게 된다. 그러나 서비스에서 발생한 실재 장애 상황에서는 30분 후에 애플리케이션(WAS)이 재연결을 시도하여 문제가 해결되는 경우가 많다. OS에서도 SocketTimeout 시간을 설정할 수 있기 때문이다. 해당 설정 값을 통해 OS 레벨에서 네트워크 연결 끊김을 확인하는 것이다. 만약 리눅스 서버의 KeepAlive 체크 수행 주기가 30분이면 SocketTimeout 설정을 0으로 해도 네트워크 장애로 인한 DBMS 연결 장애 지속 시간이 30분을 넘지 않는 것이다. Linux 서버에서 KeepAlive 체크 수행 주기는 tcp_keepalive_time로 조정할 수 있다.
