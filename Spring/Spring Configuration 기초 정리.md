# Spring Configuration 기초 정리

## @Configuration과 @Bean

```java
class JavaConfigTest {
    @Test
    void javaConfig() {
        ApplicationContext context = new AnnotationConfigApplicationContext(HelloApplication.class);
        String[] beanDefinitionNames = context.getBeanDefinitionNames();
        System.out.println(Arrays.toString(beanDefinitionNames));

        AuthService authService = context.getBean(AuthService.class);
        assertThat(authService).isNotNull();
    }
}

public class AuthService {
    public String findMemberName() {
        return "사용자";
    }
}

public class AuthenticationPrincipalArgumentResolver {
    private AuthService authService;

    public AuthenticationPrincipalArgumentResolver(AuthService authService) {
        this.authService = authService;
    }

    public String findMemberName() {
        return authService.findMemberName();
    }

    public AuthService getAuthService() {
        return authService;
    }
}

@Configuration
public class AuthenticationPrincipalConfig {

    // TODO: AuthService 빈을 등록하기
    @Bean
    public AuthService authService() {
        return new AuthService();
    }

    // TODO: AuthenticationPrincipalArgumentResolver를 빈 등록하고 authService에 대한 의존성을 주입하기
    @Bean
    public AuthenticationPrincipalArgumentResolver authenticationPrincipalArgumentResolver() {
        return new AuthenticationPrincipalArgumentResolver(authService());
    }
}
```

``@Configuration`` 내부에는 ``@Component``가 들어있어서 ``@SpringBootApplication``어노테이션 내부의 ``@ComponentScan``을 통해 빈 등록이 가능하다. 처음에 나는 ``AuthenticationPrincipalConfig`` 클래스의 메서드들 위에 ``@Bean``을 붙여주지 않은 상태로 테스트를 돌렸었다. 결과는 성공이었다. 성공 후에 어 그러면 ``@Bean`` 생략해도 되는거 아니야? 라고 착각했다. 테스트가 통과했던 이유는 무의식적으로 ``AuthService`` 클래스 위에 ``@Service``어노테이션을 붙여주었기 때문이었다.  

내가 만든 서비스나 레포지토리가 아닌 외부 라이브러리들을 사용하고자 할 때에 해당 라이브러리 클래스에 ``@Service``나 ``@Repository``를 붙일 수 없다. 이 두 어노테이션 또한 ``@Component``가 내부에 들어가 있기에 컴포넌트스캔으로 읽어들여 빈 등록을 할 수가 있는 것이다. 만약 해당 어노테이션이 없는 클래스들을 빈 등록하기 위해서는 ``@Bean``을 통해서 빈 등록을 해야만 한다. 내가 처음에 착각했던 것은 클래스 위에 ``@Configuration``을 붙이면 내부가 다 빈 등록이 되는 것 아닌가? 라고 착각했었다. 해당 어노테이션을 붙이면 정확히는 ``AuthenticationPrincipalConfig``객체가 빈 등록이 된다. 테스트 코드에 이 객체를 찾게끔 하면 테스트과 통과될 것이다. 우리는 Config 라는 클래스를 여러 객체들을 빈 등록 시켜주기 위한 클래스로 사용하게된다. 위의 경우에서는 ``@Service``를 붙이지 않은 ``AuthService``를 ``AuthenticationPrincipalConfig``클래스에서 ``@Bean``을 통해 빈 등록을 해주는 과정인 것이다.  

추가적으로 ``@Configuration``은 [싱글턴 컨테이너](https://github.com/Be-poz/TIL/blob/master/Spring/%EC%8B%B1%EA%B8%80%ED%84%B4%20%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%EC%97%90%20%EB%8C%80%ED%95%B4.md)를 보장하는 역할도 해준다.  

<br/>

## @PropertySource와 @Value

```java
    @Test
    void key() {
        ApplicationContext context = new AnnotationConfigApplicationContext(PropertySourceConfig.class);
        String[] beanDefinitionNames = context.getBeanDefinitionNames();
        System.out.println(Arrays.toString(beanDefinitionNames));

        JwtTokenKeyProvider jwtTokenKeyProvider = context.getBean(JwtTokenKeyProvider.class);
        assertThat(jwtTokenKeyProvider.getSecretKey()).isEqualTo("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIiLCJuYW1lIjoiSm9obiBEb2UiLCJpYXQiOjE1MTYyMzkwMjJ9.ih1aovtQShabQ7l0cINw4k1fagApg3qLWiB8Kt59Lno");
    }

@Configuration
@PropertySource("classpath:application.properties")
public class PropertySourceConfig {

    private final Environment env;

    public PropertySourceConfig(Environment env) {
        this.env = env;
    }

    // TODO: application.properties의 security-jwt-token-secret-key 값을 활용하여 JwtTokenKeyProvider를 빈으로 등록하기
    @Bean
    public JwtTokenKeyProvider jwtTokenKeyProvider() {
        return new JwtTokenKeyProvider(env.getProperty("security-jwt-token-secret-key"));
    }
}

@Component
public class JwtTokenExpireProvider {
    // TODO: application.properties의 security-jwt-token-expire-length 값을 활용하여 validityInMilliseconds값 초기화 하기
    private long validityInMilliseconds;

    public JwtTokenExpireProvider(@Value("${security-jwt-token-expire-length}")long validityInMilliseconds) {
        this.validityInMilliseconds = validityInMilliseconds;
    }

    public long getValidityInMilliseconds() {
        return validityInMilliseconds;
    }
}

//application.properties
security-jwt-token-secret-key= eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIiLCJuYW1lIjoiSm9obiBEb2UiLCJpYXQiOjE1MTYyMzkwMjJ9.ih1aovtQShabQ7l0cINw4k1fagApg3qLWiB8Kt59Lno
security-jwt-token-expire-length= 3600000
```

``@Value("${...}")``을 통해 application.properties의 값을 가져올 수 있다.  
만약 해당 값이 없는 경우에는 ``IllegalArgumentException``이 발생하기 때문에, ``@Value("${keyValue:defaultValue}")``와 같이 디폴트 값을 설정해 줄 수가 있다. 세미콜론 앞에는 공백이 있어서는 안된다. 만약 있게된다면 디폴트 값이 들어가게 될 것이다.  
``@Value``는 properties의 값을 가져올 뿐만 아니라 ``@Value{"bepoz"} String name`` 다음과 같은 방법으로 값을 바로 주입해주는 방법으로 또한 사용할 수가 있다. Map과 List 형식 또한 가져와서 주입해줄 수가 있다.  

``@Value`` 말고 ``@PropertySource``를 이용해서 값을 가져오는 방법도 있다. 위의 코드에 나와있는 것처럼 ``@PropertySource("classpath:Application.properities")``와 같이 경로를 작성하고 ``Environment``를 주입받아 ``getProperty``메서드를 이용해서 속성 값을 가져올 수가 있다.  

``@PropertySource``는 yml 파일은 지원하지 않기 때문에 별도의 추가적인 클래스를 만들어서 설정해주어야 한다.  

```java
public class YamlPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(@Nullable String name, EncodedResource resource) throws IOException {
        Properties propertiesFromYaml = loadYamlIntoProperties(resource);
        String sourceName = name != null ? name : resource.getResource().getFilename();
        return new PropertiesPropertySource(sourceName, propertiesFromYaml);
    }

    private Properties loadYamlIntoProperties(EncodedResource resource) throws FileNotFoundException {
        try {
            YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
            factory.setResources(resource.getResource());
            factory.afterPropertiesSet();
            return factory.getObject();
        } catch (IllegalStateException e) {
            // for ignoreResourceNotFound
            Throwable cause = e.getCause();
            if (cause instanceof FileNotFoundException)
                throw (FileNotFoundException) e.getCause();
            throw e;
        }
    }
}

@PropertySource(value = "classpath:application.yml", factory = YamlPropertySourceFactory.class)

```

위와 같이 말이다.  

***

### Reference

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-using-propertysource  

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-value-annotations  

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-java-basic-concepts  

https://vsh123.github.io/spring%20boot/spring-boot-yml/