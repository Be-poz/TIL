# AOP에 대해 (4)

* ``Pointcut``: 부가 기능 적용 여부를 판단하는 필터링 로직, 주로 클래스와 메서드 이름으로 필터링 한다. 이름 그대로 어떤 포인트(point)에 기능을 적용할지 말지에 대해 잘라서(cut) 구분하는 것이다.
* ``Advice``: 프록시가 호출하는 부가 기능, 프록시 로직이다.
* ``Advisor``: ``Pointcut`` 1개 + ``Advice`` 1개

조언(``Advice``)를 어디(``Pointcut``)에 할 것인가?  
조언자(``Advisor``)는 어디(``Pointcut``)에 조언(``Advice``)을 해야할지 알고 있다.  

<br/>

코드를 통해 바로 살펴보겠다. 이번에는 ``save()``와 ``find()`` 두 메서드를 가지고 있는 인터페이스와 클래스를 이용해보겠다.  

```java
public interface ServiceInterface {
    void save();

    void find();
}

@Slf4j
public class ServiceImpl implements ServiceInterface {

    @Override
    public void save() {
        log.info("save 호출");
    }

    @Override
    public void find() {
        log.info("find 호출");
    }
}
```

```java
@Test
public void advisor() {
  ServiceInterface target = new ServiceImpl();
  ProxyFactory factory = new ProxyFactory(target);
  DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(Pointcut.TRUE, new BepozAdvice());

  factory.addAdvisor(advisor);
  ServiceInterface proxy = (ServiceInterface) factory.getProxy();

  proxy.save();
  proxy.find();
}
/*
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
ServiceImpl - save 호출
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
ServiceImpl - find 호출
```

``Advisor`` 인터페이스의 가장 일반적인 구현체인 ``DefaultPointcutAdvisor``를 생성했다. 파라미터로 ``Pointcut``과 ``Advice``를 전달하면된다. ``Poinrtcut.TRUE``는 항상 true를 전달하는 ``Pointcut``이다. ``addAdvisor``를 통해 팩토리에 추가해준다.  

<img src="https://user-images.githubusercontent.com/45073750/147382875-e62b6efc-f8ac-4e79-9d9f-95f747af7c34.png" alt="image" style="zoom:67%;" />

다음과 같이 참고하게 될 것이다.  
이제 ``Pointcut``을 직접 만들어서 메서드에 따라 ``Advice``를 적용시킬지 말지에 대해 판별하게 해보겠다.  

``Pointcut``은 ``ClassFilter``와 ``MethodMatcher`` 이렇게 두 개의 타입으로 이루어진다.  
클래스와 메서드가 둘 다 맞아야지만 true를 반환한다.  

```java
public interface Pointcut {

	/**
	 * Return the ClassFilter for this pointcut.
	 * @return the ClassFilter (never {@code null})
	 */
	ClassFilter getClassFilter();

	/**
	 * Return the MethodMatcher for this pointcut.
	 * @return the MethodMatcher (never {@code null})
	 */
	MethodMatcher getMethodMatcher();


	/**
	 * Canonical Pointcut instance that always matches.
	 */
	Pointcut TRUE = TruePointcut.INSTANCE;

}
```

```java
@Slf4j
public class MyPointcut implements Pointcut {

    @Override
    public ClassFilter getClassFilter() {
        log.info("MyPointcut.getClassFilter()");
        return ClassFilter.TRUE;
    }

    @Override
    public MethodMatcher getMethodMatcher() {
        log.info("MyPointcut.getMethodMatcher()");
        return new MyMethodMatcher();
    }
}
```

```java
@Slf4j
public class MyMethodMatcher implements MethodMatcher {

    private final String MATCH_NAME = "save";

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        boolean result = method.getName().equals(MATCH_NAME);
        log.info("포인트컷 호출 method={} targetClass={}", method.getName(), targetClass);
        log.info("포인트컷 결과 result={}", result);
        return result;
    }

    @Override
    public boolean isRuntime() {
        return false;
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass, Object... args) {
        return false;
    }
}
```

```java
@Test
public void customAdvisor() {
  ServiceInterface target = new ServiceImpl();
  ProxyFactory factory = new ProxyFactory(target);
  DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(new MyPointcut(), new BepozAdvice());

  factory.addAdvisor(advisor);
  ServiceInterface proxy = (ServiceInterface) factory.getProxy();

  proxy.save();
  proxy.find();
}
/*
MyPointcut - MyPointcut.getClassFilter()
MyPointcut - MyPointcut.getMethodMatcher()
MyMethodMatcher - 포인트컷 호출 method=save targetClass=class com.example.advancedpractice.advice.ServiceImpl
MyMethodMatcher - 포인트컷 결과 result=true
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
ServiceImpl - save 호출
MyPointcut - MyPointcut.getClassFilter()
MyPointcut - MyPointcut.getMethodMatcher()
MyMethodMatcher - 포인트컷 호출 method=find targetClass=class com.example.advancedpractice.advice.ServiceImpl
MyMethodMatcher - 포인트컷 결과 result=false
ServiceImpl - find 호출
```

결과 로그를 보면 ``find()`` 호출 시에는 내가 생성한 ``MyMethodMatcher``에서 false를 return 하기 때문에 부가 ``BepozAdvice`` 호출이 되지 않는 것을 확인할 수가 있다.  

이번에는 ``BepozPointcut``이 아닌 스프링에서 제공하는 ``NamedMatchMethodPointcut``을 사용해보겠다.  

```java
@Test
public void nameMatchMethodPointcut() {
  ServiceInterface target = new ServiceImpl();
  ProxyFactory factory = new ProxyFactory(target);
  NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
  pointcut.setMappedNames("save");

  DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new BepozAdvice());

  factory.addAdvisor(advisor);
  ServiceInterface proxy = (ServiceInterface) factory.getProxy();

  proxy.save();
  proxy.find();
}
/*
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
ServiceImpl - save 호출
ServiceImpl - find 호출
```

이렇게 스프링은 여러 ``Pointcut``을 제공한다.(``JdkRegexpMethodPointcut``, ``TruePointcut`` ...)  
그 중에서 aspectJ 표현식을 사용하는 ``AspectJExpressionPointcut``을 특히나 많이 사용하게 된다.  

이제 여러 ``Advisor``를 하나의 target에 적용시켜보자.  
단순히 프록시 여러개를 생성해서 하는 방법이 있다.  

```java
@Test
public void manyProxies() {
  ServiceInterface target = new ServiceImpl();
  ProxyFactory factory = new ProxyFactory(target);
  DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice1());
  factory.addAdvisor(advisor);
  ServiceInterface proxy = (ServiceInterface) factory.getProxy();

  ProxyFactory secondFactory = new ProxyFactory(proxy);
  DefaultPointcutAdvisor secondAdvisor = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice2());
  secondFactory.addAdvisor(secondAdvisor);
  ServiceInterface proxy2 = (ServiceInterface) secondFactory.getProxy();

  proxy2.save();
}

@Slf4j
static class Advice1 implements MethodInterceptor {

  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    log.info("Advice1.invoke()");
    return invocation.proceed();
  }
}

@Slf4j
static class Advice2 implements MethodInterceptor {

  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    log.info("Advice2.invoke()");
    return invocation.proceed();
  }
}
/*
AdvisorTest$Advice2 - Advice2.invoke()
AdvisorTest$Advice1 - Advice1.invoke()
ServiceImpl - save 호출
```

이러한 방식은 적용해야할 ``Advisor``의 개수가 늘어날수록 그 숫자만큼의 프록시를 생성해야만 한다.  
스프링은 이러한 문제를 해결하기 위해 하나의 프록시에 여러 ``Advisor``를 적용할 수 있게 만들었다.  

```java
@Test
public void manyProxied() {
  ServiceInterface target = new ServiceImpl();
  ProxyFactory factory = new ProxyFactory(target);
  DefaultPointcutAdvisor advisor1 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice1());
  DefaultPointcutAdvisor advisor2 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice2());

  factory.addAdvisor(advisor1);
  factory.addAdvisor(advisor2);
  ServiceInterface proxy = (ServiceInterface) factory.getProxy();
  proxy.save();
}
/*
AdvisorTest$Advice1 - Advice1.invoke()
AdvisorTest$Advice2 - Advice2.invoke()
ServiceImpl - save 호출

앞의 코드는 advice2 부터 호출이 되었었다. 첫 번째 프록시가 target을 감싸고 후에 이 프록시를 다시 감쌌었기 때문에
```

``addAdvisor``를 해준 순서대로 호출이 된 것을 확인할 수가 있다.  
하나의 target에 여러 AOP를 적용해도 스프링은 하나의 프록시만 생성해서 진행한다.  

이제 다시 원래의 코드에 ``Advice``를 적용해보자.  
먼저 V1부터 진행하겠다. V1은 인터페이스가 존재하는 경우의 프록시 코드였다.  

```java
public class LogTraceAdvice implements MethodInterceptor {

    private final LogTrace logTrace;

    public LogTraceAdvice(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

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
}
```

```java
//이전에 사용하던 config
@Configuration
public class DynamicProxyFilterConfig {

    private static final String[] PATTERNS = {"request*", "save*"};

    @Bean
    public BepozControllerV1 orderControllerV1(LogTrace logTrace) {
        BepozControllerV1Impl bepozController = new BepozControllerV1Impl(bepozServiceV1(logTrace));

        return (BepozControllerV1) Proxy.newProxyInstance(
                bepozController.getClass().getClassLoader(),
                new Class[]{BepozControllerV1.class},
                new LogTraceFilterHandler(orderController, logTrace, PATTERNS)
        );
    }

    @Bean
    public BepozServiceV1 bepozServiceV1(LogTrace logTrace) {
        BepozServiceV1Impl bepozService = new BepozServiceV1Impl(bepozRepositoryV1(logTrace));

        return (BepozServiceV1) Proxy.newProxyInstance(
                bepozService.getClass().getClassLoader(),
                new Class[]{BepozServiceV1.class},
                new LogTraceFilterHandler(orderService, logTrace, PATTERNS)
        );
    }

    @Bean
    public BepozRepositoryV1 bepozRepositoryV1(LogTrace logTrace) {
        BepozRepositoryV1Impl bepozRepository = new BepozRepositoryV1Impl();

        return (BepozRepositoryV1) Proxy.newProxyInstance(
                bepozRepository.getClass().getClassLoader(),
                new Class[]{BepozRepositoryV1.class},
                new LogTraceFilterHandler(orderRepository, logTrace, PATTERNS));
    }
}

//현재 사용할 config
@Slf4j
@Configuration
public class ProxyFactoryConfigV1 {

    @Bean
    public BepozControllerV1 orderController(LogTrace logTrace) {
        BepozControllerV1 bepozController = new BepozControllerV1Impl(bepozService(logTrace));
        ProxyFactory factory = new ProxyFactory(bepozController);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozControllerV1 proxy = (BepozControllerV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozController.getClass());

        return proxy;
    }

    @Bean
    public BepozServiceV1 bepozService(LogTrace logTrace) {
        BepozServiceV1 bepozService = new BepozServiceV1Impl(bepozRepository(logTrace));
        ProxyFactory factory = new ProxyFactory(bepozService);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozServiceV1 proxy = (BepozServiceV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozService.getClass());

        return proxy;
    }

    @Bean
    public BepozRepositoryV1 bepozRepository(LogTrace logTrace) {
        BepozRepositoryV1Impl bepozRepository = new BepozRepositoryV1Impl();
        ProxyFactory factory = new ProxyFactory(bepozRepository);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozRepositoryV1 proxy = (BepozRepositoryV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozRepository.getClass());

        return proxy;
    }

    private Advisor getAdvisor(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
				pointcut.setMappedNames("request*", "save*");

        LogTraceAdvice advice = new LogTraceAdvice(logTrace);

        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}

/*
ProxyFactory proxy=class com.sun.proxy.$Proxy51 target=class com.example.advancedpractice.csr.proxy.v1.BepozRepositoryV1Impl

로딩 시에 다음과 같이 로그가 찍힌다. 
인터페이스를 이용하기 때문에 Jdk 동적 프록시가 이용된 것을 확인할 수가 있다.
```

이제 V2인 구체 클래스만 존재하고 수동 빈 등록의 경우 코드를 확인하겠다.  

```java
@Slf4j
@Configuration
public class ProxyFactoryConfigV2 {

    @Bean
    public BepozControllerV2 bepozController(LogTrace logTrace) {
        BepozControllerV2 bepozController = new BepozControllerV2(bepozService(logTrace));
        ProxyFactory factory = new ProxyFactory(bepozController);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozControllerV2 proxy = (BepozControllerV2) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozController.getClass());

        return proxy;
    }

    @Bean
    public BepozServiceV2 bepozService(LogTrace logTrace) {
        BepozServiceV2 bepozService = new BepozServiceV2(bepozRepository(logTrace));
        ProxyFactory factory = new ProxyFactory(bepozService);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozServiceV2 proxy = (BepozServiceV2) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozService.getClass());

        return proxy;
    }

    @Bean
    public BepozRepositoryV2 bepozRepository(LogTrace logTrace) {
        BepozRepositoryV2 bepozRepository = new BepozRepositoryV2();
        ProxyFactory factory = new ProxyFactory(bepozRepository);
        factory.addAdvisor(getAdvisor(logTrace));
        BepozRepositoryV2 proxy = (BepozRepositoryV2) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), bepozRepository.getClass());

        return proxy;
    }

    private Advisor getAdvisor(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("request*", "save*");

        LogTraceAdvice advice = new LogTraceAdvice(logTrace);

        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}

/*
ProxyFactory proxy=class com.example.advancedpractice.csr.proxy.v2.BepozRepositoryV2$$EnhancerBySpringCGLIB$$644d9afa target=class com.example.advancedpractice.csr.proxy.v2.BepozRepositoryV2

로딩 시에 다음과 같이 로그가 찍힌다. 
구체클래스만 이용하기 때문에 CGLIB 동적 프록시가 이용된 것을 확인할 수가 있다.
```

이렇게 V1과 V2가 적용이 끝났다. 원본 코드에 손대지 않고 프록시를 이용해서 부가 기능 적용을 할 수가 있었다.  
그러나, 지금과 같은 방식의 문제점은 설정 코드가 많다는 것이다.  

controller, service, repository 이렇게 3개의 빈을 등록하는데 동적 프록시 생성 코드를 만들었어야 했다. 이 개수가 많아진다면 설정 코드의 작업량이 더 많아질 것이다. 그리고 V3처럼 컴포넌트 스캔으로 빈을 자동등록 하는 경우 이미 빈이 등록되었기 때문에 프록시 적용이 안된다. 이를 해결하기 위해 빈 후처리기를 이용해야 한다.  

5편에서 계속...  

[AOP에 대해 (1)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(8).md)

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)

