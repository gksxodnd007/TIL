## Anatomy Spring Cloud Gateway

Spring Cloud Gateway는 Webflux를 기반으로 동작하기 때문에 먼저 Webflux가 http 요청을 어떻게 처리하는지 간단하게 분석해보자.

### DispatcherHandler

- Spring Webflux는 Spring MVC와 유사하게 front controller pattern을 통해 요청을 처리한다. WebHandler를 구현한 DispatcherHandler를 통해 요청을 프로세싱하는 shared algorithm을 제공한다. 요청을 실제로 처리하는 bean 설정을 통해 등록된 component들에 의해 처리된다.
- DispatcherHandler는 요청을 실제로 처리하는 component들을 spring configuration을 통해 찾는다. 이를 위하여 DispatcherHandler는 ApplicationContextAware를 구현한다.

### Special Bean Types

- DispatcherHandler는 WebFlux Framework가 제공하고 관리하는 특별한 Bean들에게 실제 요청을 위임한다.

| Bean type | Explanation |
|:---------:|-------------|
| `HandlerMapping` | Map a request to a handler. The mapping is based on some criteria, the details of which vary by `HandlerMapping` implementation — annotated controllers, simple URL pattern mappings, and others. The main `HandlerMapping` implementations are `RequestMappingHandlerMapping` for `@RequestMapping` annotated methods, `RouterFunctionMapping` for functional endpoint routes, and `SimpleUrlHandlerMapping` for explicit registrations of URI path patterns and WebHandler instances. |
| `HandlerAdaptor` | Help the `DispatcherHandler` to invoke a handler mapped to a request regardless of how the handler is actually invoked. For example, invoking an annotated controller requires resolving annotations. The main `purpose` of a HandlerAdapter is to shield the `DispatcherHandler` from such details. |
| `HandlerResultHandler` | Process the result from the handler invocation and finalize the response. |

### Processing

- 각 `HandlerMapping`은 matching handler가 있는지 찾는다. 첫 번째로 matching된 핸들러를 찾으면 그것을 사용한다.
- Handler를 찾았다면 `HandlerAdapter`를 통해 호출한다. return 값은 `HandlerResult`이다.
- `HandlerResult`는 적절한 `HandlerResultHandler`를 통해 응답을 직접 반환하거나 뷰를 랜더링한다.

### Result Handling

- HandlerAdapter의 호출을 통해 return 되는 값은 HandlerResult로 wrapping 되어있다. 다음 테이블은 이용가능한 HandlerResultHandler들이다.

| Result Handler Type | Return Values | Default Order |
|:-------------------:|:-------------:|:-------------:|
| `ResponseEntityResultHandler` | `ResponseEntity`, typically from `@Controller` instances | 0 |
| `ServerResponseResultHandler` | `ServerResponse`, typically from functional endpoints | 0 |
| `ResponseBodyResultHandler` | Handle return values from `@ResponseBody` methods or `@RestController` classes. | 100 |
| `ViewResolutionResultHandler` | `CharSequence`, `View`, `Model`, `Map`, `Rendering`, or any other `Object` is treated as a model attribute. | `Interger.MAX_VALUE` |

### Spring Cloud Gateway
<div id="init">
	<h4>Init Application</h4>
</div>

1. SpringApplication.run(Application.class, args)
2. SpringApplication#refreshContext(context)
3. SpringApplication#refresh(context)
4. AbstractApplicationContext#refresh()
5. AbstractApplicationContext#finishRefresh()
6. AbstractApplicationContext#publishEvernt(new ContextRefreshedEvent(this))
7. RouteRefreshListner#onApplicationEvent(event) -> ContextRefreshedEvent를 받아 reset 호출
8. RefreshRoutesEvent 이벤트 발행
9. CachingRouteLocator에서 RefreshRoutesEvent를 구독
10. CachingRouteLocator#fetch() 호출
11. cache route 데이터 refresh

#### Actuator Refresh Endpoint
- build.gardle
```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```
- application.yml
```yaml
management:
  endpoint:
    gateway:
      enabled: true
  endpoints:
    web:
      exposure:
        include: gateway
```
- actuator 의존성 추가 후 위와 같이 property를 설정하면 spring cloud gateway에서 라우트 정보를 조회하고 갱신할 수 있는 endpoint를 열어준다.

> To clear the routes cache, make a POST request to /actuator/gateway/refresh. The request returns a 200 without a response body.

- GatewayControllerEndpoint는 AbstractGatewayControllerEndpoint를 상속받는다.
- AbstractGatewayControllerEndpoint의 refresh 메서드를 보면 RefreshRoutesEvent를 발행하고 Mono.empty를 return한다. [Init Application](#init)의 8번 부터 다시 실행된다.

#### Request Handling

- Spring Cloud Gateway는 AbstractHandlerMapping을 구현하는 `RoutePredicateHandlerMapping`를 통해 요청을 라우팅한다.
- Spring Cloud Gateway에서 http 요청을 처리하는 핵심 flow 정리하면 다음과 같다.

1. `ReactorNetty의 ChannelHandler`
2. `DefaultWebFilterChain.filter(...)`에서 `WebHandler.handler` 호출
3. `WebHandler`의 구현체인 `DispatcherHandler`에서 `HandlerMapping`의 `getHandler`들을 호출
4. `RoutePredicateHandlerMapping`의 `lookupRoute`를 호출하여 Route정보를 discovering 하여 `ServerWebExchange`에 attribute로 해당 정보들을 저장 (각 Route의 Predicate들을 호출)
> RequestMappingHandlerMapping의 order가 RoutePredicateHandlerMapping보다 높기 때문에 @RequestMapping이 붙은 메서드가 우선적으로 search되어 사용된다.
5. `NettyRoutingFilter`에서 `ServerWebExchange`에 저장된 Route정보들을 꺼내 forwarding하며 forwarding 전후로 등록된 `GatewayFilter`들을 적용함

### RouteLocator

- Spring Cloud Gateway에서 http 요청을 처리하는 flow에서 **4번 과정**이 핵심이다.
- 아무런 커스텀 구현이 없는 상황에서 Spring Cloud Gateway는 `CachingRouteLocator`를 통해 Route정보들을 가져오며 Spring Context가 Boot될 때 기본으로 등록된 모든 `RouteLocator들의 getRoutes`를 `CachingRouteLocator`에서 호출하여 route정보를 로컬에 캐싱하게된다. 해당 정보는 CachingRouteLocator의 refresh 메서드를 통해 바뀔수 있으며 Spring에서 제공하는 ApplicationEvent를 통해 해당 메서드가 호출되게 만들 수 있다.

```java
...
@Bean
@Primary
public RouteDefinitionLocator routeDefinitionLocator(
		List<RouteDefinitionLocator> routeDefinitionLocators) {
	return new CompositeRouteDefinitionLocator(
			Flux.fromIterable(routeDefinitionLocators));
}

@Bean
public ConfigurationService gatewayConfigurationService(BeanFactory beanFactory,
		@Qualifier("webFluxConversionService") ObjectProvider<ConversionService> conversionService,
		ObjectProvider<Validator> validator) {
	return new ConfigurationService(beanFactory, conversionService, validator);
}

@Bean
public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
		List<GatewayFilterFactory> gatewayFilters,
		List<RoutePredicateFactory> predicates,
		RouteDefinitionLocator routeDefinitionLocator,
		ConfigurationService configurationService) {
	return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates,
			gatewayFilters, properties, configurationService);
}

@Bean
@Primary
@ConditionalOnMissingBean(name = "cachedCompositeRouteLocator")
public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
	return new CachingRouteLocator(
			new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
}
...
```
위의 코드는 GatewayAutoConfiguration 클래스의 일부 코드이며 cachedCompositeRouteLocator에 `@Primary` 애너테이션이 붙어있는걸 확인할 수 있다.

PS. 우리팀은 동적으로 라우트 정보를 추가하고 제거해야한다. 기본 구현체를 이용하면 event를 통해 refresh를 제어해야하므로 코드 복잡도가 증가한다. 그래서 우리는 `CachingRouteLocator`를 우리의 목적에 맞게 커스텀하여 사용할 예정이다.
