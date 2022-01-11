# AOP에 대해 (5)

``Advice``를 이용해서 V1, V2 코드들을 변경했었으나, V3과 같이 컴포넌트 스캔을 이용한 자동 빈 등록의 경우를 해결하지 못했었다.  
이번에는 ``BeanPostProcessor``, 빈 후처리기를 이용해서 해결을 해보려고 한다.  

먼저, 일반적인 스프링 빈 등록은 객체를 생성 후에 스프링 컨테이너 내부의 빈 저장소에 등록을 하는 방식이다.  
빈 후처리기는 객체 생성 후 빈 저장소에 등록하기 전에 다른 무언가의 처리를 해줄 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/147539492-4f0da887-eab1-4b45-9015-49bcd8498726.png)

위의 3번 작업에서 A객체를 다른 객체로 변환시킨다던가 하는 작업 또한 가능하다.  

```java
@Slf4j
public class BeanPostProcessorTest {

    @Test
    public void beanPostProcessor() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(BasicConfig.class);
        A beanA = ac.getBean("beanA", A.class);
        beanA.helloA();

        assertThatThrownBy(() -> ac.getBean(B.class))
                .isInstanceOf(NoSuchBeanDefinitionException.class);
    }

    @Configuration
    static class BasicConfig {
        @Bean(name = "beanA")
        public A a() {
            return new A();
        }
    }

    static class A {
        public void helloA() {
            log.info("hello A");
        }
    }

    static class B {
        public void helloB() {
            log.info("hello B");
        }
    }
}

// hello A
```

클래스 A, B를 생성하고 ``@Configuration``을 통해서 A를 빈 등록을 해준 후에 ``ac.getBean``으로 조회하고 메서드를 호출, 그리고 B 타입의 빈을 조회했을 때에 ``NoSuchBeanDefinitionException``예외가 던져지는 것을 확인할 수가 있다.  

```java
@Test
public void beanPostProcessor() {
  AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(BasicConfig.class);
  B beanA = ac.getBean("beanA", B.class);
  beanA.helloB();

  assertThatThrownBy(() -> ac.getBean(A.class))
    .isInstanceOf(NoSuchBeanDefinitionException.class);
}
// hello B

@Configuration
static class BasicConfig {
  @Bean(name = "beanA")
  public A a() {
    return new A();
  }

  @Bean
  public AToBPostProcessor processor() {
    return new AToBPostProcessor();
  }
}

static class AToBPostProcessor implements BeanPostProcessor {
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    log.info("beanName={} bean={}", beanName, bean);
    if (bean instanceof A) {
      return new B();
    }
    return bean;
  }
}
```

``BeanPostProcessor`` 인터페이스를 구현하고 스프링 빈으로 등록하자, beanA 로 빈을 검색했을 때에 B타입의 빈이 찾아지는 것을 확인할 수가 있다.  

위의 코드에서 ``BeanPostProcessor`` 인터페이스의 ``postProcessBeforeInitialization`` 메서드를 오버라이딩 했다.  
이 메서드 말고 ``postProcessAfterInitialization`` 메서드도 존재하는데, 메서드 명 처럼 초기화 이전에 처리할 것인지 초기화 이후에 처리할 것인지에 대한 차이다.(``@PostConstruct``과 같은 초기화를 말한다. 스프링 빈의 이벤트 라이프 싸이클은 **스프링 컨테이너 생성 -> 스프링 빈 생성 -> 의존관계 주입 -> 초기화 콜백 -> 사용 -> 소멸 전 콜백 -> 스프링 종료** 와 같다.)  

``@PostConstruct`` 또한 내부적으로 스프링에서 ``CommonAnnotationBeanPostProcessor``라는 빈 후처리기에서 ``@PostConstruct``가 붙은 메서드를 호출해서 처리한다.  

이제 실제 코드에 적용해보자.  

```java
@Slf4j
public class PackageLogTraceProxyPostProcessor implements BeanPostProcessor {

    private final String basePackage;
    private final Advisor advisor;

    public PackageLogTraceProxyPostProcessor(String basePackage, Advisor advisor) {
        this.basePackage = basePackage;
        this.advisor = advisor;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        log.info("param beanName={} bean={}", beanName, bean.getClass());

        String packageName = bean.getClass().getPackageName();
        if (!packageName.startsWith(basePackage)) {
            return bean;
        }
        ProxyFactory factory = new ProxyFactory(bean);
        factory.addAdvisor(advisor);
        Object proxy = factory.getProxy();
        log.info("create proxy: target={} proxy={}", bean.getClass(), proxy.getClass());

        return proxy;
    }
}
```

```java
@Slf4j
@Configuration
public class BeanPostProcessorConfig {

    @Bean
    public PackageLogTraceProxyPostProcessor logTraceProxyPostProcessor(LogTrace logTrace) {
        return new PackageLogTraceProxyPostProcessor(
                "com.example.advancedpractice.csr.proxy", getAdvice(logTrace));
    }

    private Advisor getAdvice(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("reqeust*", "save*");

        return new DefaultPointcutAdvisor(pointcut, new LogTraceAdvice(logTrace));
    }
}
```

``BeanPostProcessor`` 을 통해서 빈 후처리기 클래스를 생성하고 빈 등록을 해주었다. v3 패키지 내부의 클래스에만 적용이 되도록 로직을 작성했다.  

스프링은 프록시를 생성하기 위해 빈 후처리기를 만들어서 제공하고 있다.

```java
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

먼저 해당 라이브러리를 추가해야 한다. aspectJ 관련 라이브러리를 등록하게 된다.  
스프링부트를 사용하지 않는다면 ``@EnableAspectJAutoProxy`` 를 사용해서 AOP관련 클래스를 직접 스프링 빈으로 등록시켜줘야 했지만, 스프링부트를 사용한다면 자동으로 등록이 된다.  

![image](https://user-images.githubusercontent.com/45073750/147810218-dd475291-492a-4d0d-b120-b46413e95d82.png)

자동으로 등록된 빈은 바로 위의 클래스다. 자동으로 프록시를 생성해주는 빈 후처리기다.  
스프링 빈으로 등록된 Advisor을 찾아서 자동으로 프록시를 등록해준다.  

![image](https://user-images.githubusercontent.com/45073750/147810476-cb661159-62bd-4d7f-8057-29cd79fa5b76.png)

동작 과정은 다음과 같다.  

1. 생성: 스프링이 빈 대상이 되는 객체를 생성한다. (``@Bean``, 컴포넌트 스캔)
2. 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다.
3. 모든 Advisor 빈 조회: 자동 프록시 생성기 - 빈 후처리기는 스프링 컨테이너에서 모든 Advisor를 조회한다.
4. 프록시 적용 대상 체크: 앞서 조회한 Advisor에 포함되어 있는 포인트컷을 사용해서 해당 객체가 프록시를 적용할 대상인지 아닌지 판단한다. 이때 객체의 클래스 정보는 물론이고, 해당 객체의 모든 메서드를 포인트컷에 하나하나 모두 매칭해본다. 조건이 하나라도 만족하면 프록시 적용 대상이 된다.
5. 프록시 생성: 프록시 적용 대상이면 프록시를 생성하고 반환해서 프록시를 스프링 빈으로 등록한다. 만약 프록시 적용 대상이 아니라면 원본 객체를 반환해서 원본 객체를 스프링 빈으로 등록한다.
6. 빈 등록: 반환된 객체는 스프링 빈으로 등록된다.

코드에 적용해겠다.

```java
//이전에 사용하던 config
@Slf4j
@Configuration
public class BeanPostProcessorConfig {
  
    @Bean
    public PackageLogTraceProxyPostProcessor logTraceProxyPostProcessor(LogTrace logTrace) {
        return new PackageLogTraceProxyPostProcessor(
                "com.example.advancedpractice.csr.proxy", getAdvice(logTrace));
    }

    private Advisor getAdvice(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("reqeust*", "save*");

        return new DefaultPointcutAdvisor(pointcut, new LogTraceAdvice(logTrace));
    }
}

//현재 사용할 config
@Slf4j
@Configuration
public class AutoProxyConfig {

    @Bean
    public Advisor advisor(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("request*", "save*");

        LogTraceAdvice advice = new LogTraceAdvice(logTrace);
        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}
```

이렇게 Advisor만 빈 등록을 하고 빈 후처리기에 대한 빈 등록을 따로 하지 않아도 스프링부트에서 자동으로 등록해주기 때문에 ``localhost:8080/v3/request?itemId=hello`` 를 입력해도 로그가 잘 찍히는 것을 확인할 수가 있다.  
``localhost:8080/v3/no-log``는 포인트 컷에 의해서 로그가 찍히지 않는다.

**포인트 컷은 2가지에 사용된다.**  

1. 프록시 적용 여부 판단 - 생성 단계

   포인트 컷을 사용해서 해당 빈이 프록시를 생성할 필요가 있는지 없는지 체크한다.  
   ``BepozControllerV1`` 같은 경우 메서드에 ``request()``가 있으므로 프록시가 생성된다.  
   만약 조건에 맞는 것이 하나도 없으면 프록시를 생성할 필요가 없기 때문에 생성하지 않는다.

2. 어드바이스 적용 여부 판단 - 사용 단계

   프록시가 호출되었을 때 부가 기능인 어드바이스를 적용할지 말지 포인트컷을 보고 판단한다.  
   ``BepozControllerV1``의 경우 ``request()``는 현재 포인트컷 조건에 만족하므로 프록시는 Advice를 먼저 호출하고, target을 호출한다. ``noLog()``의 경우 포인트컷 조건에 만족하지 않으므로 바로 target을 호출한다.

프록시를 모두 생성하는 것은 비효율적이기 때문에 포인트컷으로 한 번 필터링해서 조건에 부합되는 경우에만 프록시를 만들어주는 것이다.  

이번에는 조금 더 정교한 포인트컷을 만들어 볼 것이다. 현재 조건을 ``request*, save*``로 걸어두었기 때문에 다음과 같은 프록시가 생성되는 것을 확인할 수가 있었다.  

![image](https://user-images.githubusercontent.com/45073750/147820509-09594984-df18-46d8-b89f-830222681ec6.png)

내가 적용하고자한 target이 아닌 다른 곳에서도 포인트컷 조건이 부합이 돼서 프록시가 만들어진 것이다.  
``request*``이기 때문에 ``requestMappingHandlerMapping()`` 또한 잡혀버린 것이다!!  
이를 위해 코드를 추가 및 수정 해보겠다.  

```java
//@Bean
public Advisor advisor1(LogTrace logTrace) {
  NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
  pointcut.setMappedNames("request*", "save*");

  LogTraceAdvice advice = new LogTraceAdvice(logTrace);
  return new DefaultPointcutAdvisor(pointcut, advice);
}

@Bean
public Advisor advisor2(LogTrace logTrace) {
  AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
  pointcut.setExpression("execution(* com.example.advancedpractice.csr.proxy..*(..))");

  LogTraceAdvice advice = new LogTraceAdvice(logTrace);
  return new DefaultPointcutAdvisor(pointcut, advice);
}
```

``AspectJExpressionPointcut``을 사용했다. AspectJ 포인트컷 표현식을 이용해서 정교한 포인트컷을 설정하는 것이다.  
``execution(*com.example.advancedpractice.csr.proxy..*(..))``  

* ``*`` : 모든 반환타입  
* ``com....proxy..``: 해당 패키지와 그 하위 패키지
* ``*(..)``: 모든 메서드 이름 (...)는 파라미터 상관없음

대충 이렇게다. AspectJ 표현식은 차후에 다시 언급하도록 하겠다.  

위의 코드에서의 문제는 ``noLog()``의 경우에도 프록시가 생성된다는 것이다. 이를 위해 더 정교하게 다음과 같이 표현해서 고칠 수가 있다.  

```java
execution(* com.example.advancedpractice.csr.proxy..*(..))
&& !execution(* com.example.advancedpractice.csr.proxy..noLog(..))
```

<br/>

어떤 스프링 빈이 여러개의 Advisor가 제공하는 pointcut의 조건을 모두 만족한다고 프록시 자동 생성기는 프록시를 1개만 생성한다.  
프록시 팩토리가 생성하는 프록시는 내부에 여러 Advisor들을 포함할 수 있기 때문이다.  

1. advisor1의 포인트컷만 만족 -> 프록시 1개 생성, 프록시에 advisor1만 포함
2. advisor1, advisor2의 포인트컷을 둘 다 만족 -> 프록시 1개 생성, 프록시에 advisor1, advisor2 포함
3. advisor1, advisor2의 포인트컷을 아무것도 만족 X -> 프록시 생성하지 않음

<img src="https://user-images.githubusercontent.com/45073750/147821064-df6c55ca-199c-4853-8a52-458bb9c6ea89.png" alt="image" style="zoom:67%;" />

<img src="https://user-images.githubusercontent.com/45073750/147821085-d3330729-d061-4200-bb16-5f1462180a12.png" alt="image" style="zoom:67%;" />

<br/>

이번에는 조금 더 간편하게 Advisor를 생성하는 것을 알아보겠다.  
바로 ``@Aspect`` 어노테이션을 사용하는 것이다. 이 어노테이션은 pointcut과 Advice로 구성되어있는 Advisor의 생성을 간편하게 해준다. 관점 지향 프로그래밍(AOP)을 가능하게하는 AspectJ 프로젝트에서 제공하는 어노테이션이고, 스프링은 이것을 차용해서 프록시를 통한 AOP를 가능하게 한다.  

바로 코드로 살펴보겠다.  

```java
@Aspect
public class LogTraceAspect {

    private final LogTrace logTrace;

    public LogTraceAspect(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

    @Around("execution(* com.example.advancedpractice.csr.proxy..*(..))" +
            "&& !execution(* com.example.advancedpractice.csr.proxy..noLog(..))")
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
        TraceStatus status = null;
        try {
            String message = joinPoint.getSignature().toShortString();
            status = logTrace.begin(message);
            Object result = joinPoint.proceed();
            logTrace.end(status);

            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}

/* 이전에 정의했던 Advice의 override method는 다음과 같았다.
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
  TraceStatus status = null;
  try {
    Method method = invocation.getMethod();
    String message = method.getDeclaringClass().getSimpleName() + "." + method.getName() + "()";
    status = logTrace.begin(message);
    Object result = invocation.proceed();
    logTrace.end(status);
    return result;
  } catch (Exception e) {
    logTrace.exception(status, e);
    throw e;
  }
}
```

```java
//이전에 사용하던 config
@Slf4j
@Configuration
public class AutoProxyConfig {

    @Bean
    public Advisor advisor(LogTrace logTrace) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression("execution(* com.example.advancedpractice.csr.proxy..*(..)) " +
                "&& !execution(* com.example.advancedpractice.csr.proxy..noLog(..))");

        LogTraceAdvice advice = new LogTraceAdvice(logTrace);
        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}

//현재 사용할 config
@Slf4j
@Configuration
public class AopConfig {

    @Bean
    public LogTraceAspect logTraceAspect(LogTrace logTrace) {
        return new LogTraceAspect(logTrace);
    }
}
```

* ``@Aspect``: 어노테이션 기반 프록시를 적용할 때 필요
* ``@Around(표현식)``: 포인트컷 표현식을 넣음, 표현식은 AspectJ 표현식을 사용. 메서드는 Advice가 됨
* ``ProceedingJoinPoint joinPoint``: Advice의 ``MethodInvocation invocation``과 유사한 기능. 내부에 실제 호출 대상, 전달 인자, 그리고 어떤 객체와 어떤 메서드가 호출되었는지 정보가 포함되어 있음
* ``joinPoint.proceed()``: target을 호출. ``invocation.proceed()``와 동일

``@Aspect``가 달려있어도 빈 등록을 해주어야 하기 때문에 Config를 따로 만들었다. 물론 그냥 ``@Component``로 자동 빈 등록을 해주어도 상관없다.  

앞서 살펴본 ``AnnotnationAwareAspectJAutoProxyCreator``는 Advisor를 자동으로 찾아와 필요한 곳에 프록시를 생성하고 적용해주었었다. 추가로 하나의 역할이 더 있다. 바로 ``@Aspect``를 찾아서 Advisor로 만들어주는 것이다. 네이밍에 ``AnnotationAware``이 붙은 이유가 바로 이 때문이다.  

정리하면, 자동 프록시 생성기는 ``@Aspect``를 보고 Advisor로 변환해서 저장하고, Advisor를 기반으로 프록시를 생성한다.  

``@Aspect``를 Advisor로 변환해서 저장하는 과정은 다음과 같다.  

<img src="https://user-images.githubusercontent.com/45073750/147823363-82ee1224-c384-428f-bdd6-96eff564d2b4.png" alt="image" style="zoom:67%;" />

1. 실행: 스프링 어플리케이션 로딩 시점에 자동 프록시 생성기를 호출한다.
2. 모든 ``@Aspect`` 빈 조회: 자동 프록시 생성기는 스프링 컨테이너에서 ``@Aspect`` 어노테이션이 붙은 스프링 빈을 모두 조회한다.
3. Advisor 생성: ``@Aspect`` Advisor builder를 통해 ``@Aspect`` 정보를 기반으로 Advisor를 생성한다.
4. ``@Aspect`` 기반 Advisor 저장: 생성한 Advisor를 ``@Aspect`` Advisor builder 내부에 저장한다.

``@Aspect`` Advisor builder는 ``BeanFactoryAspectJAdvisorBuilder`` 클래스다. ``@Aspect``의 정보를 기반으로 포인트컷, 어드바이스, 어드바이저를 생성하고 보관하는 것을 담당한다. ``@Aspect``의 정보를 기반으로 어드바이저를 만들고, ``@Aspect`` 어드바이저 빌더 내부 저장소에 캐시한다. 캐시에 어드바이저가 이미 만들어져 있는 경우 캐시에 저장된 어드바이저를 반환한다.  

<br/>

어드바이저를 기반으로 프록시를 생성하는 과정은 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/147825214-6f70673b-f5f7-40b8-bce7-71c015da146a.png)

1. 생성: 스프링 빈 대상이 되는 객체를 생성한다.
2. 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다.
3. 1. Advisor 빈 조회: 스프링 컨테이너에서 ``Advisor`` 빈을 모두 조회한다.
   2. ``@Aspect`` 어드바이저 빌더 내부에 저장된 ``Advisor``를 모두 조회한다.
4. 프록시 적용 대상 체크: 조회한 ``Advisor``에 포함되어 있는 포인트컷을 사용해서 해당 객체가 프록시를 적용할 대상인지 아닌지 판단한다. 객체의 클래스 정보, 메서드를 모두 매칭해보고 하나라도 만족하면 프록시 적용 대상이 된다.
5. 프록시 생성 및 빈 등록: 프록시 적용 대상이면 프록시를 생성하고 프록시를 반환한다. 프록시를 스프링 빈으로 등록한다. 만약 적용대상이 아니라면 원본 객체를 스프링 빈으로 등록한다.

지금까지 Controller, Service, Repository에 LogTrace를 이용해서 로깅을 적용했다.  
특정 기능 하나에 관심이 있는 기능이 아닌 여러 기능 사이에 걸쳐서 들어가는 관심이다.  
이것을 **횡단 관심사(cross-cutting concerns)** 라고 한다.  

지금까지 한 결과물을 한 번 정리해보자면,  

<img src="https://user-images.githubusercontent.com/45073750/148085659-cec1129b-cc4d-4fde-beab-ccae1e5aff81.png" alt="image" style="zoom: 50%;" />

인터페이스를 이용하고 수동 빈 등록  V1, 구체클래스를 이용하고 수동 빈 등록 V2, 구체 클래스 이용하고 자동 빈 등록 V3이 있었다.  

<img src="https://user-images.githubusercontent.com/45073750/148085998-5a965fae-2294-48ce-bc77-277678d66278.png" alt="image" style="zoom:50%;" />

그리고 다음과 같이 조금씩 변경시켰다. 이 단계를 정리해보자면,  

1. 인터페이스 구현과 클래스 상속을 이용해 직접 프록시를 만들어 다형성을 이용하여 V1과 V2를 개선
2. Jdk Dynamic Proxy를 이용해서 V1에서 더 이상 직접 프록시 클래스를 만들지 않도록 개선
3. 인터페이스인 경우는 Jdk Dynamic Proxy, 클래스 상속의 경우는 CGLIB을 이용해서 동적 프록시를 사용했다. 두 기술을 함께 사용할 때 중복으로 관리할 수 없으므로 ProxyFactory를 이용하여 개선
4. V3의 경우에는 수동 빈 등록이 아닌 자동 빈 등록이기 때문에 위의 방법 적용 불가. BeanPostProcessor를 이용해서 빈의 초기화 콜백 이전에 프록시를 생성하는 빈 후처리 코드를 넣어 개선
5. 4번의 경우 BeanPostProcessor를 따로 빈 등록을 해주었으나 aspectJ 관련 라이브러리를 추가하여 스프링 부트가 자동으로 빈 등록하게끔 하여 개선
6. @Aspect 을 이용하여 간편히 pointcut, advice를 등록하게끔 개선

이제 스프링의 AOP에 대해서 조금 더 자세히 알아보자.  

6편에서 계속...  

[AOP에 대해 (1)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(8).md)

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)
