### Spring Cloud Zuul

## API Gateway
- Microservice Architecture(이하 MSA)에서 언급되는 컴포넌트 중 하나이며, 모든 클라이언트 요청에 대한 end point를 통합하는 서버이다. 마치 프록시 서버처럼 동작한다. 그리고 인증 및 권한, 모니터링, logging 등 추가적인 기능이 있다. 모든 비지니스 로직이 하나의 서버에 존재하는 Monolithic Architecture와 달리 MSA는 도메인별 데이터를 저장하고 도메인별로 하나 이상의 서버가 따로 존재한다. 한 서비스에 한개 이상의 서버가 존재하기 때문에 이 서비스를 사용하는 클라이언트 입장에서는 다수의 end point가 생기게 되며, end point를 변경이 일어났을때, 관리하기가 힘들다.

**Authentication and Security** : 클라이언트 요청시, 각 리소스에 대한 인증 요구 사항을 식별하고 이를 만족시키지 않는 요청은 거부
**Insights and Monitoring** : 의미있는 데이터 및 통계 제공
**Dynamic Routing** : 필요에 따라 요청을 다른 클러스터로 동적으로 라우팅
**Stress Testing** : 성능 측정을 위해 점차적으로 클러스터 트래픽을 증가
**Load Shedding** : 각 유형의 요청에 대해 용량을 할당하고, 초과하는 요청은 제한
**Static Response handling** :클러스터에서 오는 응답을 대신하여 API GATEWAY에서 응답 처리

### Spring Cloud Ribbon
- 클라이언트 측 로드 밸런서
- 여러 서버를 라운드로빈 방식의 부하 분산 기능을 제공(여러 알고리즘 사용 가능)
- Spring Cloud Config와 결합하여, 서버 목록을 제공받아 사용할 수 있음

### Spring Cloud Eureka
- 서버 개수 동적 조절 (elastic service에서 URL을 환경설정 파일에 정적으로 미리 지정하는 것은 적합하지 않음)
- IP주소는 예측할 수 없으며, 어떤 파일에서 정적으로 관리하기가 어려움

### Spring Cloud Hystrix
- Netflix에서 Circuit Breaker Pattern을 구현한 라이브러리. Micro Service Architecture에서 장애 전파 방지를 할 수 있음.
- 외부 api를 호출하는 메서드를 한번 wrapping해서 외부서비스 에러가 임계치에 도달했을 때 빠른 장애 전파 및 임시로 응답 처리함.
> Circuit Breaker Pattern : The basic idea behind the circuit breaker is very simple. You wrap a protected function call in a circuit breaker object, which monitors for failures. Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error, without the protected call being made at all. Usually you'll also want some kind of monitor alert if the circuit breaker trips.
- example
```java
@SpringBootApplication
@EnableCircuitBreaker
@EnableHystrixDashboard
public class DemoApplication {

  public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
  }
}

public class DemoService {

  @Autowired
  RestTemplate restTemplate;

  @Bean
  public RestTemplate restTemplate() {
      return new RestTemplate();
  }

  // GetItem command
  @HystrixCommand(fallbackMethod = "getFallback")
  public List<User> getItem(String name)  {
    List<User> usersList = new ArrayList<User>();

    List<Item> itemList = (List<Item>) restTemplate.exchange("http://localhost:8082/users/"+name+"/items", HttpMethod.GET, null ,new ParameterizedTypeReference<List<Item>>() {}).getBody();
    usersList.add(new User(name,"myemail@mygoogle.com",itemList));

    return usersList;
  }

  // fall back method
  // it returns default result
  @SuppressWarnings("unused")
  public List<User> getFallback(String name){
    List<User> usersList = new ArrayList<User>();
    usersList.add(new User(name,"myemail@mygoogle.com"));

    return usersList;
  }
}
```
