# Spring 빈 설정

### 필드명이 빈 이름과 동일하면 자동 주입
```java
//HubertBean.java
@Component
public class HubertBean {
  public HubertBean() {}
}

//HubertConfig.java
@Configuration
public class HubertConfig {

  @Bean
  public void beanMethod(HubertBean hubertBean) {
    ...
  }
}
```
이럴 경우 beanMethod에 HubertBean이 자동 주입 됨. 빈의 이름과 파라미터명 혹은 필드이름과 같으면 자동주입이 되는 상황

참고 링크 : https://stackoverflow.com/questions/48715714/bean-declaration-with-qualifier-doesnt-work
