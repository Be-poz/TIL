# ComponentScan에 대해
본 내용은 김영한 님의 스프링 핵심 원리 강의 내용을 토대로 정리했습니다.  

스프링 빈을 등록할 때에 ``@Bean``을 사용해서 등록하고는 했다.  
하지만 이런 빈이 수십, 수백개가 되면 문제가 생길 것이다. 그래서 설정 정보가 없어도 자동으로 스프링 빈을 등록하는 컴포넌트 스캔을 제공한다.  
```java
@Configuration
@ComponentScan(
        //@Configuration 내부에 Component가 있음. 우리는 Configuration 에서 Bean 으로 등록해줬기 때문에 충돌할 수 있으므로 filter로 뺐다.
        excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Configuration.class)
)
public class AutoAppConfig {
}
```
사용하기 위해서는 ``@ConponentScan``을 붙여주면 된다.  
기존의 코드는 다음과 같았다.  
```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService(){
        System.out.println("AppConfig.memberService");
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public OrderService orderService() {
        System.out.println("AppConfig.orderService");
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public MemberRepository memberRepository() {
        System.out.println("AppConfig.memberRepository");
        return new MemoryMemberRepository();
    }

    @Bean
    public DiscountPolicy discountPolicy() {
//      return new FixedDiscountPolicy();
        return new RateDiscountPolicy();
    }
}
```
AutoAppConfig를 보면 ``@Bean``으로 등록한 클래스가 없는 것을 알 수 있을 것이다.  
컴포넌트 스캔은 ``@Component``가 붙은 클래스를 스캔해서 스프링 빈으로 등록해준다.  
위의 경우 ``excludeFilters``를 붙여준 이유는 ``@Configuration``안에도 ``@Component``가 들어가 있는데, AppConfig에서 등록한 빈과  
AutoAppConfig에서 등록되는 빈들과 충돌이 일어날 수 있어서 제외시켰다.  

```java
@Component
public class MemberServiceImpl implements MemberService{

    private final MemberRepository memberRepository;

    @Autowired
    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```
다음과 같이 빈으로 등록할 클래스 위에 ``@Component``를 붙여주었다. 그리고 의존관계는 ``@Autowired``를 통해 해결해 주었다.  
```java
public class AutoAppConfigTest {

    @Test
    void basicScan(){
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AutoAppConfig.class);
        MemberService memberService = ac.getBean(MemberService.class);
        assertThat(memberService).isInstanceOf(MemberService.class);

        OrderServiceImpl bean = ac.getBean(OrderServiceImpl.class);
        MemberRepository memberRepository = bean.getMemberRepository();
        System.out.println("memberRepository = " + memberRepository);
    }
}
```
``AnnotationConfigApplicationContext``를 이용해 AutoAppConfig를 넘겨주고 테스트를 실행해보았고 잘 동작한다는 것을 알 수 있었다.  

``@ComponentScan``은 ``@Component``가 붙은 모든 클래스를 스프링 빈으로 등록한다.  
빈 이름은 클래스명에서 앞글자만 소문자로 바꿔 등록하게 되는데, 따로 지정하고 싶다면 ``@Component("이름")``이런식으로 부여하면 된다.  
``@Autowired``를 사용하면 스프링 컨테이너가 자동으로 해당 스프링 빈을 찾아서 주입한다.  
이때 조회전략의 기본은 같은 타입을 찾아 주입하는 것이다.  
***
컴포넌트 스캔의 스캔은 지정한 클래스의 패키지를 탐색 시작 위치가 된다.(그래서 보통 프로젝트 최상단에 설정 정보를 둔다.)  
``@ComponentScan(basePackages="패키지명")``으로 필요한 위치부터 탐색을 하도록 설정도 가능하다.  

스프링 부트의 시작 정보인 ``@SpringBootApplication``에는 ``@ComponentScan``이 들어있기 때문에 시작 루트 위치에 두는 것이 관례이다.  

``@Controller``,``@Service``,``@Repository``,``@Configuration``등은 소스코드에 ``@Component``를 포함하고 있기 때문에,  
``@ComponentScan``을 통해 빈 등록이 이루어질 수 있는 것이다.  
***
컴포넌트 스캔은 스캔 대상을 지정할 수도 있고 제외할 대상을 지정할 수도 있다.(AutoAppConfig 보면 Configuration.class를 제외한 것 처럼)  
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyExcludeComponent {
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyIncludeComponent {
}

@MyIncludeComponent
public class BeanA {
}

@MyExcludeComponent
public class BeanB {
}

public class ComponentFilterAppConfigTest {

    @Test
    void filterScan() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);
        BeanA beanA = ac.getBean("beanA", BeanA.class);
        assertThat(beanA).isNotNull();

        assertThrows(
                NoSuchBeanDefinitionException.class,
                () -> ac.getBean("beanB", BeanB.class));
    }

    @Configuration
    @ComponentScan(
            includeFilters = @Filter(type = FilterType.ANNOTATION, classes = MyIncludeComponent.class),
            excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = MyExcludeComponent.class)
    )
    static class ComponentFilterAppConfig{

    }
}
```
``@Component``내부 코드에 있는 어노테이션들을 따서 따로 MyIncludeComponent와 MyExcludeComponent이라는 어노테이션을 생성해주었다.  
그리고 BeanA와 BeanB클래스에 달아주고, ComponentFilterAppConfig에 ``includeFilters``와 ``excludeFilters``로 설정 해주었다.  

FilterType에는 5가지 옵션이 있다.  
* ANNOTATION: 기본값, 어노테이션을 인색해서 동작한다.  
* ASSIGNABLE_TYPE: 지정한 타입과 짓ㄱ 타입을 인식해서 동작한다.
* ASPECTJ: AspectJ 패턴을 사용한다.
* REGEX: 정규 표현식
* CUSTOM: ``TypeFilter``라는 인터페이스를 구현해서 처리한다.

``@Component``면 충분하기 때문에 ``includeFilters``를 사용하는 일은 거의 없다. ``excludeFilters``또한 자주 사용하지는 않는다.  
***
동일한 빈 이름이 등록이 되면 충돌이 발생하게 된다.  

컴포넌트 스캔에 의해 자동으로 스프링 빈이 등록되는데, 그 이름이 같은 경우 ``ConflictionBeanDefinitionException``예외가 발생하게된다.  
만약 ``@Component``를 이용한 자동 빈등록과 ``@Bean``을 이용한 수동 빈등록의 이름이 같을 경우에는 수동 빈 등록이 자동 빈을 오버라이딩 한다.  
그때의 로그 ``Overriding bean definition for bean 'memoryMemberRepository' with a different definition: replacing``  

스프링 부트에서는 수동 빈과 자동 빈 등록 시에 충돌이 발생할 경우 에러가 나게끔 해준다.
***