## ImportSelector
- @Enable 애노테이션의 메타정보를 이용해 @Import에 적용할 @Configuration 클래스를 결정해주는 오브젝트다.
- example
```java
@Configuration
public class ConfigA {

  @Bean
  public MessageA messageA() {
    return new MessageA("messageA");
  }
}

@Configuration
public class ConfigB {

  @Bean
  public MessageB messageB() {
    return new MessageB("messageB");
  }
}

public class MessageSelector extends ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        String message = (String) importingClassMetadata.getAnnotationAttributes(EnableMessage.class.getName()).get("message");

        if ("messageA".equals(message)) {
            return new String[] { ConfigA.class.getName() };
        }

        if ("messageB".equals(message)) {
            return new String[] { ConfigB.class.getName() };
        }

        return new String [] { DefaultMessage.class.getName() };
    }
}

@Retention(value = RetentionPolicy.RUNTIME)
@Target(value = ElementType.TYPE)
@Documented
@Import(value = MessageSelector.class)
public @interface EnableMessage {

    String message() default "";
}

@EnableMessage(message = "messageA")
@SpringBootApplication(scanBasePackages = "com.kakaopay.supermarket.spring.webclient")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
- @EnableMessage 애너테이션의 속성값에 따라 다른 빈을 등록한다.
