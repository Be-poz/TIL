# AOP에 대해 (3)

프록시를 이용해서 로그 추적기를 적용했었다. 이 과정에서 많은 프록시 클래스들이 생성됐었다.  
이번에는 동적 프록시를 이용해서 프록시 클래스를 계속해서 생성하지 않게끔 해보겠다.  

동적 프록시를 이용하면 개발자가 프록시 클래스를 직접 만들어줄 필요가 없게 된다.  
이름 그대로 동적으로 런타임에 개발자 대신 만들어준다.  

**JDK 동적 프록시는 인터페이스를 기반으로 프록시를 동적으로 만들어주기 때문에, 인터페이스가 필수적이다.**  

JDK 동적 프록시에 적용할 로직은 ``InvocationHandler`` 인터페이스를 구현해서 작성하면 된다.  

![image](https://user-images.githubusercontent.com/45073750/146891303-ba4afc57-2aad-4d55-88a8-05226ccb648d.png)

파라미터로는 프록시 자신, 호출한 메서드, 메서드를 호출할 때 전달한 인수들 이다.  
사용법은 아래 코드와 같다. 리플렉션을 이용하게 되는데, 별도의 설명은 생략하겠다.  

```java
@Slf4j
public class BepozInvocationHandler implements InvocationHandler {

    private final Object target;

    public BepozInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("BepozInvocationHandler.invoke()");
        log.info("chicken is god");

        return method.invoke(target, args);
    }
}
```

```java
public interface AInterface {

    String call();
}

@Slf4j
public class AImpl implements AInterface {

    @Override
    public String call() {
        log.info("AImpl.call()");
        return "a";
    }
}
```

```java
@Slf4j
public class DynamicProxyTest {

    @Test
    public void dynamic() {
        AInterface target = new AImpl();
        BepozInvocationHandler handler = new BepozInvocationHandler(target);

        /*
        Object proxy1 = Proxy.newProxyInstance(
                AInterface.class.getClassLoader(),
                new Class[]{AInterface.class},
                handler);*/

        AInterface proxy = (AInterface) Proxy.newProxyInstance(
                AInterface.class.getClassLoader(),
                new Class[]{AInterface.class},
                handler);

        String result = proxy.call();
        log.info("targetClass={}", target.getClass());
        log.info("proxyClass={}", proxy.getClass());
    }
}

/*
dynamicProxy.BepozInvocationHandler - BepozInvocationHandler.invoke()
dynamicProxy.BepozInvocationHandler - chicken is god
dynamicProxy.AImpl - AImpl.call()
dynamicProxy.DynamicProxyTest - targetClass=class dynamicProxy.AImpl
dynamicProxy.DynamicProxyTest - proxyClass=class com.sun.proxy.$Proxy9
```

``InvocationHandler``를 구현하는 클래스를 생성하고, ``Proxy.newProxyInstance(...)``를 이용해서 프록시를 생성해주었다. 클래스 로더 정보, 인터페이스 그리고 핸들러 로직을 넣어주면 해당 인터페이스를 기반으로 동적 프록시를 생성하고 그 결과를 반환한다.  

위 테스트 코드의 실행 순서는 다음과 같다.  

1. 클라이언트는 JDK 동적 프록시의 ``call()``을 실행
2. JDK 동적 프록시는 ``InvocationHandler.invoke()``를 호출. 이 경우에서는 ``InvocationHandler``를 구현한 ``BepozInvocationHandler``의 ``invoke()``를 호출
3. ``BepozInvocationHandler``가 내부 로직을 수행하고 ``method.invoke(target, args)``를 호출해서 target인 실제 객체 ``AImpl``를 호출
4. ``AImpl``인스턴스의 ``call()`` 실행
5. ``BepozInvocationHandler``로 응답이 돌아옴

로그를 보면 해당 순서에 맞게끔 로그가 찍혀져있고, proxy 클래스는 ``class.sun.proxy.$Proxy9``인 것을 확인할 수가 있다.  
``AImpl``이 아니라 다른 객체를 만들고 동일한 방식으로 프록시 클래스를 만들면 또 별개의 프록시 클래스가 만들어 질 것이다.  
즉, 하나의 ``BepozInvocationHandler``클래스로 여러 프록시 클래스를 생성할 수 있다는 것이다.  

런타임 의존관계는 Client -> proxy -> BepozInvocationHandler -> AImpl 의 관계를 띈다.  

<img src="https://user-images.githubusercontent.com/45073750/147099268-4c662bff-b5ca-41de-b486-9b0bd1abe952.png" alt="image" style="zoom:80%;" />

기존에는 다음과 같은 흐름이었다면 이제는 다음과 같이 변경되었다.  

<img src="https://user-images.githubusercontent.com/45073750/147099181-01c2ca51-1428-4305-a365-9f6accfddb39.png" alt="image" style="zoom:80%;" />

위에서 언급한대로 ``AImpl``이 아니라 ``BImpl``을 생성한다고 해도 ``BepozInvocationHandler``를 가져다가 사용할 수가 있다는 것이다.  

이제 인터페이스를 이용하는 V1의 기존 코드에 적용해보겠다.  

```java
public class LogTraceBasicHandler implements InvocationHandler {

    private final Object target;
    private final LogTrace logTrace;

    public LogTraceBasicHandler(Object target, LogTrace logTrace) {
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TraceStatus status = null;
        try {
            String message = method.getDeclaringClass().getSimpleName() + "." + method.getName() + "()";
            status = logTrace.begin(message);
            Object result = method.invoke(target, args);
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
public class InterfaceProxyConfig {

    @Bean
    public BepozControllerV1 bepozController(LogTrace logTrace) {
        BepozControllerV1Impl controllerImpl = new BepozControllerV1Impl(bepozService(logTrace));
        return new BepozControllerInterfaceProxy(controllerImpl, logTrace);
    }

    @Bean
    public BepozServiceV1 bepozService(LogTrace logTrace) {
        BepozServiceV1Impl serviceImpl = new BepozServiceV1Impl(bepozRepository(logTrace));
        return new BepozServiceInterfaceProxy(serviceImpl, logTrace);
    }

    @Bean
    public BepozRepositoryV1 bepozRepository(LogTrace logTrace) {
        BepozRepositoryV1Impl repositoryImpl = new BepozRepositoryV1Impl();
        return new BepozRepositoryInterfaceProxy(repositoryImpl, logTrace);
    }
}

//현재 사용할 config
@Configuration
public class DynamicProxyBasicConfig {

    @Bean
    public BepozControllerV1 bepozControllerV1(LogTrace logTrace) {
        BepozControllerV1Impl bepozController = new BepozControllerV1Impl(bepozServiceV1(logTrace));

        return (BepozControllerV1) Proxy.newProxyInstance(
                bepozController.getClass().getClassLoader(),
                new Class[]{BepozControllerV1.class},
                new LogTraceBasicHandler(bepozController, logTrace)
        );
    }

    @Bean
    public BepozServiceV1 bepozServiceV1(LogTrace logTrace) {
        BepozServiceV1Impl bepozService = new BepozServiceV1Impl(bepozRepositoryV1(logTrace));

        return (BepozServiceV1) Proxy.newProxyInstance(
                bepozService.getClass().getClassLoader(),
                new Class[]{BepozServiceV1.class},
                new LogTraceBasicHandler(bepozService, logTrace)
        );
    }

    @Bean
    public BepozRepositoryV1 bepozRepositoryV1(LogTrace logTrace) {
        BepozRepositoryV1Impl bepozRepository = new BepozRepositoryV1Impl();

        return (BepozRepositoryV1) Proxy.newProxyInstance(
                bepozRepository.getClass().getClassLoader(),
                new Class[]{BepozRepositoryV1.class},
                new LogTraceBasicHandler(bepozRepository, logTrace));
    }
}
```

클래스 의존관계는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/147105530-6b88ba5b-ec46-43ad-aa5d-32bdbb0c128e.png)

런타임 객체 의존관계는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/147106141-4fdd532e-be53-4a0b-9782-17e38e4d69cd.png)

Controller의 request에 ``v1/no-log``가 존재하고 이 경우에는 로그를 찍으면 안된다.  
이를 위한  config를 추가해보겠다.  

```java
public class LogTraceFilterHandler implements InvocationHandler {

    private final Object target;
    private final LogTrace logTrace;
    private final String[] patterns;

    public LogTraceFilterHandler(Object target, LogTrace logTrace, String[] patterns) {
        this.target = target;
        this.logTrace = logTrace;
        this.patterns = patterns;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        String methodName = method.getName();
        if (!PatternMatchUtils.simpleMatch(patterns, methodName)) {
            return method.invoke(target, args);
        }

        TraceStatus status = null;
        try {
            String message = method.getDeclaringClass().getSimpleName() + "." + method.getName() + "()";
            status = logTrace.begin(message);
            Object result = method.invoke(target, args);
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
@Configuration
public class DynamicProxyFilterConfig {

    private static final String[] PATTERNS = {"request*", "save*"};

    @Bean
    public BepozControllerV1 bepozControllerV1(LogTrace logTrace) {
        BepozControllerV1Impl bepozController = new BepozControllerV1Impl(bepozServiceV1(logTrace));

        return (BepozControllerV1) Proxy.newProxyInstance(
                bepozController.getClass().getClassLoader(),
                new Class[]{BepozControllerV1.class},
                new LogTraceFilterHandler(bepozController, logTrace, PATTERNS)
        );
    }

    @Bean
    public BepozServiceV1 bepozServiceV1(LogTrace logTrace) {
        BepozServiceV1Impl bepozService = new BepozServiceV1Impl(bepozRepositoryV1(logTrace));

        return (BepozServiceV1) Proxy.newProxyInstance(
                bepozService.getClass().getClassLoader(),
                new Class[]{BepozServiceV1.class},
                new LogTraceFilterHandler(bepozService, logTrace, PATTERNS)
        );
    }

    @Bean
    public BepozRepositoryV1 bepozRepositoryV1(LogTrace logTrace) {
        BepozRepositoryV1Impl bepozRepository = new BepozRepositoryV1Impl();

        return (BepozRepositoryV1) Proxy.newProxyInstance(
                bepozRepository.getClass().getClassLoader(),
                new Class[]{BepozRepositoryV1.class},
                new LogTraceFilterHandler(bepozRepository, logTrace, PATTERNS));
    }
}
```

``request``와 ``save``로 시작하는 메서드명인 경우에만 로그를 남기도록 수정해주었다.  
``DynamicProxyBasicConfig``와 비교했을 때, 크게 코드가 바뀐 곳은 없다.  

<br/>

지금까지 V1에 대한 코드 변경을 했다. 인터페이스가 있는 경우의 JDK 동적 프록시를 한 것이다.  
하지만, V2는 상속을 이용해서 프록시를 구현하고 있었다. 클래스만 존재하는 경우에는 어떻게 해야할까?  

이럴 때에는 ``CGLIB``이라는 바이트코드를 조작하는 라이브러리를 이용한다.  
``CGLIB``은 바이트코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공하는 라이브러리이다.  
인터페이스 없이 구체 클래스만 가지고 동적 프록시를 만들어낼 수 있다. 외부 라이브러리이지만, 스프링 프레임워크가 스프링 내부 소스로 포함하고 있다. 따라서 별도의 외부 라이브러리 추가 없이 사용할 수가 있다.  

JDK 동적 프록시에서 ``InvocationHandler``를 제공했듯이, ``CGLIB``에서는 ``MethodInterceptor``를 제공한다.  

```java
public interface MethodInterceptor extends Callback {
    Object intercept(
      Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable;
}
```

* o: ``CGLIB``가 적용된 객체
* method: 호출된 메서드
* objects: 메서드를 호출하면서 전달된 인수들
* methodProxy: 메서드 호출에 이용

사용법은 아래와 같다.  

```java
@Slf4j
public class BepozMethodInterceptor implements MethodInterceptor {

    private final Object target;

    public BepozMethodInterceptor(Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        log.info("BepozMethodInterceptor.invoke()");
        log.info("chicken is god");

        return methodProxy.invoke(target, objects);
      //method를 사용해도 되지만, CGLIB 성능 상 methodProxy를 사용하는 것을 권장한다고 한다.
    }
}
```

```java
@Slf4j
public class ConcreteService {

    public void call() {
        log.info("ConcreteService.call()");
    }
}
```

```java
@Slf4j
public class CglibProxyTest {

    @Test
    public void cglib() {
        ConcreteService target = new ConcreteService();

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(ConcreteService.class);
        enhancer.setCallback(new BepozMethodInterceptor(target));
        ConcreteService proxy = (ConcreteService) enhancer.create();

        proxy.call();
        log.info("targetClass={}", target.getClass());
        log.info("proxyClass={}", proxy.getClass());
    }
}

/* 로그 정보
BepozMethodInterceptor - BepozMethodInterceptor.invoke()
BepozMethodInterceptor - chicken is god
ConcreteService - ConcreteService.call()
CglibProxyTest - targetClass=class com.example.advancedpractice.cglibProxy.ConcreteService
CglibProxyTest - proxyClass=class ConcreteService$$EnhancerByCGLIB$$7df26b60

/*
InvocationHandler는 이렇게 했었다.
        AInterface target = new AImpl();
        BepozInvocationHandler handler = new BepozInvocationHandler(target);
        
        AInterface proxy = (AInterface) Proxy.newProxyInstance(
                AInterface.class.getClassLoader(),
                new Class[]{AInterface.class},
                handler);

        proxy.call();
```

``InvocationHandler`` 사용법과 굉장히 유사하다.  
``CGLIB``은 ``Enhancer``를 이용해서 프록시를 생성한다.  
 ``setSuperClass(...)``를 통해 어떤 클래스를 상속받아서 프록시를 생성할지 정한다.  
 ``setCallback(...)``를 통해 프록시에 적용할 실행 로직을 할당한다.  
``create()``을 통해 프록시를 생성한다.  

``proxyClass=class ConcreteService$$EnhancerByCGLIB$$7df26b60``를 보면 ``CGLIB``을 통해 프록시가 생성된 것을 확인할 수가 있다.  

 ``CGLIB``을 통한 클래스 기반 프록시는 상속을 사용하기 때문에 몇 가지 제약사항이 존재한다.  

* 자식 클래스를 동적으로 생성하기 때문에 부모 클래스에 기본 생성자가 필요하다.
* 클래스에 final이 붙어있으면 상속이 불가능하여 예외가 발생한다.
* 메서드에 final이 붙어있으면 오버라이딩이 할 수 없다 -> ``CGLIB``에서는 프록시 로직이 동작하지 않는다.

그렇다면, 인터페이스가 존재하는 경우에는 ``InvocationHandler``를 적용하고 구체클래스만 존재하는 경우에는 ``MethodInterceptor``를 적용해야 할텐데, 두 기술을 함께 사용할 때에 이것들을 각각 중복으로 만들어서 관리를 해주어야 할까?  

<br/>

스프링은 동적 프록시를 통합해서 편리하게 만들어주는 ``ProxyFactory``라는 기능을 제공한다.  

<img src="https://user-images.githubusercontent.com/45073750/147381483-64b2242f-8058-4376-91ac-8c975c2298a5.png" alt="image" style="zoom:67%;" />

각각의 ``InvoactionHandler``와 ``MethodInterceptor``를 중복으로 만들필요 없이,  
``Advice``라는 새로운 개념을 도입해서 이것만 만들면 알아서 처리되게끔 하였다.  

``ProxyFactory``를 사용하면 ``Advice``를 호출하는 전용 ``InvocationHandler``, ``MethodInterceptor``를 내부에서 사용한다.  

<img src="https://user-images.githubusercontent.com/45073750/147381557-f1749599-a3ff-4143-a2b2-4139dce34982.png" alt="image" style="zoom:67%;" />

<img src="https://user-images.githubusercontent.com/45073750/147381563-1d975264-e9c3-43e2-b398-4ed112cfbe30.png" alt="image" style="zoom:67%;" />

그리고, 앞서 특정 메서드일 경우에만 프록시가 적용되게끔 하였었는데, 스프링은 ``Pointcut``이라는 개념을 이용해서 해결한다.  

이제 코드로 살펴보겠다.  

```java
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {
  
	@Nullable
	Object invoke(@Nonnull MethodInvocation invocation) throws Throwable;

}
```

이 코드는 스프링이 제공하는 ``MethodInterceptor``이다. ``CGLIB``이 제공하는 것과 이름은 같으나 다르니 유의하자.  
``Interceptor``를 상속받고 있고, 이 ``Interceptor`` 상위에는 ``Advice``가 존재한다.  

``MethodInvocaion invocation``에는 메서드를 호출하는 방법, 현재 프록시 객체 인스턴스, 메서드 인수들 등의 정보가 포함되어 있다. 기존에 파라미터로 제공되는 부분들이 이 안으로 모두 들어갔다고 생각하면 된다.  

<img src="https://user-images.githubusercontent.com/45073750/147381728-82f52abc-298c-4041-8198-4b428db65976.png" alt="image" style="zoom:67%;" />

```java
@Slf4j
public class BepozAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("BepozAdvice.invoke()");
        log.info("chicken is god");
        
        return invocation.proceed();
    }
}
```

``Advice`` 생성은 다음과 같이 한다.  
``invocation.proceed()``를 호출하면 ``target``클래스를 호출하고 결과를 받는다. 하지만, 이전에 보았던 ``target`` 클래스의 정보는 보이지 않는다. 해당 정보들은 ``MethodInvocation invocation``안에 모두 포함되어 있다. 그 이유는 ``ProxyFactory``로 프록시를 생성하는 과정에서 target 정보를 파라미터로 전달받기 때문이다. 코드로 살펴보겠다.  

```java
// Jdk DynamicProxy Test를 할 때 사용했던 인터페이스와 구현하는 구체클래스
public interface AInterface {

    String call();
}

@Slf4j
public class AImpl implements AInterface {

    @Override
    public String call() {
        log.info("AImpl.call()");
        return "a";
    }
}

@Slf4j
public class AdviceTest {

    @Test
    public void advice() {
        AInterface target = new AImpl();
        ProxyFactory factory = new ProxyFactory(target);
        factory.addAdvice(new BepozAdvice());

        AInterface proxy = (AInterface) factory.getProxy();
        String result = proxy.call();
        
        log.info("result={}", result);
        log.info("targetClass={}", target.getClass());
        log.info("proxyClass={}", proxy.getClass());


        assertThat(AopUtils.isAopProxy(proxy)).isTrue();
        assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
        assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
    }
}

/*
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
AImpl - AImpl.call()
AdviceTest - result=a
AdviceTest - targetClass=class com.example.advancedpractice.advice.AImpl
AdviceTest - proxyClass=class com.sun.proxy.$Proxy9
```

``new ProxyFactory(target)``코드를 이용해 팩토리를 생성할 때, 생성자에 프록시의 호출 대상을 함께 넘겨준다. 프록시 팩토리는 이 인스턴스 정보를 기반으로 프록시를 만들어낸다. 만약 이 인스턴스에 인터페이스가 있다면 JDK 동적 프록시를 기본으로 사용하고 인터페이스가 없고 구체 클래스만 있다면 ``CGLIB``를 통해서 동적 프록시를 생성한다. 이 경우에는 ``AInterface``를 이용하고 있으므로 JDK 동적 프록시를 생성한다. assert문도 정상적으로 통과되는 것을 확인할 수가 있다.  

``addAdvice(new BepozAdvice())``로 부가 기능 로직을 설정했다.  
``InvocationHandler``나 ``MethodInterceptor``와 유사하다. 이렇게 프록시가 제공하는 부가 기능 로직을 ``Advice``라고 한다.  

``factory.getProxy()``로 프록시 객체를 생성했다.  

이번에는 인터페이스가 없는 구체 클래스만 존재할 시에 ``CGLIB``을 사용하는지 확인해보겠다.  

```java
// CGLIB DynamicProxy Test를 할 때 사용했던 구체클래스
@Slf4j
public class ConcreteService {

    public void call() {
        log.info("ConcreteService.call()");
    }
}

@Test
public void cglibAdvice() {
  ConcreteService target = new ConcreteService();
  ProxyFactory factory = new ProxyFactory(target);
  factory.addAdvice(new BepozAdvice());

  ConcreteService proxy = (ConcreteService) factory.getProxy();
  proxy.call();

  log.info("targetClass={}", target.getClass());
  log.info("proxyClass={}", proxy.getClass());


  assertThat(AopUtils.isAopProxy(proxy)).isTrue();
  assertThat(AopUtils.isJdkDynamicProxy(proxy)).isFalse();
  assertThat(AopUtils.isCglibProxy(proxy)).isTrue();
}

/*
BepozAdvice - BepozAdvice.invoke()
BepozAdvice - chicken is god
ConcreteService - ConcreteService.call()
AdviceTest - targetClass=class com.example.advancedpractice.advice.ConcreteService
AdviceTest - proxyClass=class com.example.advancedpractice.advice.ConcreteService$$EnhancerBySpringCGLIB$$862fd3d1
```

인터페이스 때와 동일하게 흘러간다. 인터페이스가 존재하더라도 ``CGLIB``을 이용하게끔 사용할 수도 있다.  
``factory.setProxyTarget(true);``를 덧붙이면 인터페이스 존재여부와 상관없이 ``CGLIB``을 이용하게 된다.  

``ProxyFactory`` 덕분에 특정 기술에 종속적이지 않게 ``Advice`` 하나로 편리하게 사용할 수 있었다.  
팩토리 내부에서 JDK 동적 프록시인 경우 ``InvocationHandler``가 ``Advice``를,  
``CGLIB``인 경우에는 ``MethodInterceptor``가 ``Advice``를 호출하도록 기능을 개발해두었기 때문이다.  

JDK, CGLIB, Advice 별로 코드를 다시 한 번 비교해보겠다.(인터페이스와 구체 클래스는 생략)  

```java
//Jdk Dynamic Proxy (InvocationHandler 이용)
@Slf4j
public class BepozInvocationHandler implements InvocationHandler {

    private final Object target;

    public BepozInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("BepozInvocationHandler.invoke()");
        log.info("chicken is god");

        return method.invoke(target, args);
    }
}

@Test
public void dynamic() {
  AInterface target = new AImpl();
  BepozInvocationHandler handler = new BepozInvocationHandler(target);

  AInterface proxy = (AInterface) Proxy.newProxyInstance(
    AInterface.class.getClassLoader(),
    new Class[]{AInterface.class},
    handler);

  proxy.call();
  log.info("targetClass={}", target.getClass());
  log.info("proxyClass={}", proxy.getClass());
}
```

```java
//CGLIB Dynamic Proxy (MethodInterceptor 이용)
@Slf4j
public class BepozMethodInterceptor implements MethodInterceptor {

    private final Object target;

    public BepozMethodInterceptor(Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        log.info("BepozMethodInterceptor.invoke()");
        log.info("chicken is god");

        return methodProxy.invoke(target, objects);
    }
}

@Test
public void cglib() {
  ConcreteService target = new ConcreteService();

  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(ConcreteService.class);
  enhancer.setCallback(new BepozMethodInterceptor(target));
  ConcreteService proxy = (ConcreteService) enhancer.create();

  proxy.call();
  log.info("targetClass={}", target.getClass());
  log.info("proxyClass={}", proxy.getClass());
}
```

```java
//Advice 
@Slf4j
public class BepozAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("BepozAdvice.invoke()");
        log.info("chicken is god");

        return invocation.proceed();
    }
}

@Test
public void advice() {
  AInterface target = new AImpl();
  ProxyFactory factory = new ProxyFactory(target);
  factory.addAdvice(new BepozAdvice());

  AInterface proxy = (AInterface) factory.getProxy();
  String result = proxy.call();

  log.info("result={}", result);
  log.info("targetClass={}", target.getClass());
  log.info("proxyClass={}", proxy.getClass());


  assertThat(AopUtils.isAopProxy(proxy)).isTrue();
  assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
  assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
}
```

이제 ``Pointcut``, ``Advisor``에 대해 알아보겠다.  
4편에 계속...  

[AOP에 대해 (1)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)