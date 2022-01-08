# AOP에 대해 (6)

애플리케이션 로직은 크게 핵심 기능과 부가 기능으로 나눌 수 있다.  
앞서 적용했던 ``LogTrace``가 부가 기능이고 다른 비즈니스 코드들이 바로 핵심 기능이라고 볼 수 있다.  

앞에서 겪어봤듯이, 이런 부가 기능을 여러 곳에 적용하기에는 너무 번거롭기 때문에 부가 기능을 핵심 기능에서 분리하고 한 곳에서 관리하도록 했다. 그리고 부가 기능을 어디에 적용할지 선택하는 기능을 만들었다. 부가 기능과 어디에 적용할지 선택하는 기능을 합해서 하나의 모듈로 만들었고 이것이 바로 Aspect다. 앞서 ``@Aspect``로 사용을 해보았다. 스프링이 제공하는 Advisor 도 Advice(부가 기능)과 Pointcut(적용 대상)을 가지고 있어서 개념상 하나의 Aspect다.  

Aspect를 사용한 프로그래밍을 관점 지향 프로그래밍, AOP(Aspect-Oriented Programming)이라 한다.  
AOP는 OOP를 대체하기 위한 것이 아니라 횡단 관심사를 깔끔하게 처리하기 어려운 OOP의 부족한 부분을 보조하는 목적으로 개발되었다.  

<br/>

AOP의 대표적인 구현으로 AspectJ 프레임워크가 있다.  
스프링도 AOP를 지원하지만 대부분 AspectJ의 문법을 차용하고, AspectJ가 제공하는 기능의 일부만 제공한다.  

AOP 적용 방식에는 크게 3가지가 있다.  

* 컴파일 시점
* 클래스 로딩 시점
* 런타임 시점(프록시)

<br/>

### 컴파일 시점

![image](https://user-images.githubusercontent.com/45073750/148240077-89d8c8b3-1053-4383-acbe-82d9292feed9.png)

컴파일 시점은 ``.java`` 파일을 ``.class`` 파일로 변환하는 과정에서 AspectJ 컴파일러가 부가 기능 로직을 붙이는 방식이다.  
컴파일된 ``.class``를 디컴파일 해보면 Aspect 관련 호출 코드가 들어간다.  
위빙이란, 원본 로직에 부가 기능 로직이 추가되는 것을 말한다.  

컴파일 시점의 단점은 특별한 컴파일러가 필요하고 복잡하다는 것이다.  

<Br/>

### 클래스 로딩 시점

![image](https://user-images.githubusercontent.com/45073750/148240382-08ebe856-b389-49db-b9bd-e6f1ec0b736b.png)

클래스 로딩 시점은 ``.class`` 파일을 JVM에 저장하기 전에 코드 조작을 하는 것이다. 많은 모니터링 툴들이 사용하는 방식이다.  
이 시점에 Aspect를 적용하는 것을 로드 타임 위빙이라 한다.  

클래스 로딩 시점의 단점은 로드 타임 위빙이 자바 실행 시 특별한 옵션(java -javaagent)을 통해 클래스 로더 조작기를 지정해야 하는데, 이 부분이 번거롭고 운영하기 어렵다는 점이다.  

<Br/>

### 런타임 시점

![image](https://user-images.githubusercontent.com/45073750/148240763-c9b1b045-86a0-4ed0-a743-885dce274372.png)

앞서 코드로 적용했던 것이 바로 런타임 시점 방식이었다. 프록시 방식의 AOP이다.  
프록시를 사용하기 때문에 AOP 기능에 일부 제약이 있다.(프록시에서 target의 메서드를 호출하기 때문에 생성자 등의 조작이 불가능함(반면 위의 2가지 경우는 가능함). 컴파일 시점 처럼 특별한 컴파일러나 클래스 로딩 시점처럼 클래스 로더 조작기를 설정하지 않아도 된다.  

3가지 방식을 정리하자면, 컴파일 시점과 클래스 로딩 시점은 실제 대상 코드에 Aspect를 통한 부가 기능 호출 코드가 포함된다는 것이다. AspectJ를 직접 사용해야 한다. 런타임 시점은 실제 대상 코드는 그대로 유지되고 프록시를 통해 부가 기능이 적용된다는 것이다. 스프링 AOP가 사용하는 방식이다.  

AOP를 적용할 수 있는 지점을 조인 포인트(Join point)라 한다(이전에 다뤘던 코드에서 ``request()``, ``save()`` 가 바로 조인 포인트다).  

런타임 시점 방식에서 언급했지만, 프록시 방식을 사용하는 스프링 AOP는 메서드 실행 지점에만 AOP를 적용할 수 있다.  
프록시는 메서드 오버라이딩 개념으로 동작하기 때문에, 생성자나 static 메서드, 필드 값 접근에는 프록시 개념이 적용될 수 없다.  
프록시를 사용하는 스프링 AOP의 조인 포인트는 메서드 실행으로 제한된다.  
프록시 방식을 사용하는 스프링 AOP는 스프링 컨테이너가 관리할 수 있는 스프링 빈에만 AOP를 적용할 수 있다.  

스프링은 AspectJ를 직접 사용하는 것이 아니라 AspectJ의 문법을 차용하고 프록시 방식의 AOP를 적용한다.  

<br/>

이제 프로젝트를 다시 파서 해보겠다. aop 관련 라이브러리만 추가해준 상태로 시작한다.  

```java
@Slf4j
@Service
public class OrderService {

    private final OrderReposiotry orderReposiotry;

    public OrderService(OrderReposiotry orderReposiotry) {
        this.orderReposiotry = orderReposiotry;
    }

    public void orderItem(String itemId) {
        log.info("OrderService.orderItem()");
        orderReposiotry.save(itemId);
    }
}

@Slf4j
@Repository
public class OrderReposiotry {

    public String save(String itemId) {
        log.info("OrderRepository.save()");
        if (itemId.equals("ex")) {
            throw new IllegalStateException("Exception Occurred!!!");
        }
        return "ok";
    }
}
```

```java
@Slf4j
@SpringBootTest
public class AopTest {

    @Autowired
    OrderService orderService;

    @Autowired
    OrderReposiotry orderRepository;

    @Test
    void aopInfo() {
        log.info("isAopProxy, orderService={}", AopUtils.isAopProxy(orderService));
        log.info("isAopProxy, orderRepository={}", AopUtils.isAopProxy(orderRepository));
    }

    @Test
    void success() {
        orderService.orderItem("itemA");
    }

    @Test
    void fail() {
        assertThatThrownBy(() -> orderService.orderItem("ex"))
                .isInstanceOf(IllegalStateException.class);
    }
}
```

``aopInfo`` 메서드의 출력값은 다음과 같이 나온다. 둘 다 false로 나온다. 따로 AOP가 적용이 안됐기 때문에 당연하다.  
이제 Aspect 코드를 추가해보자.  

```java
@Slf4j
@Aspect
public class AspectV1 {

    @Around("execution(* bepoz.order..*(..))")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

앞서서 배웠듯이, ``@Around``에는 포인트 컷 표현식을 입력한 것이다.  
현재 나의 패키지 구조는 다음과 같다.  

<img src="https://user-images.githubusercontent.com/45073750/148637398-636e5c49-68b2-4d9a-8cfe-28fdba8c938f.png" alt="image" style="zoom:60%;" />

그리고 테스트 클래스에 ``@Import(AspectV1.class)`` 를 추가해서 ``AspectV1``을 빈 등록 해주었다.  
그 결과 ``isAopProxy``에 대한 출력이 true로 바뀐 것을 확인할 수가 있었다.  
``success()``에 대한 로깅은 다음과 같이 출력된다.  

<img src="https://user-images.githubusercontent.com/45073750/148637438-fdea099d-2090-4417-ad8f-3be6c5d3911a.png" alt="image" style="zoom:50%;" />

위의 포인트 컷 표현식을 다음과 같이 사용할 수도 있다.  

```java
@Slf4j
@Aspect
public class AspectV2 {

    @Pointcut("execution(* bepoz.order..*(..))")
    private void allOrder(){} //pointcut signature


    @Around("allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

``@Pointcut``어노테이션을 달고 있는 ``allOrder()``라는 메서드가 보이고, 해당 메서드를 ``@Around``에서 사용하고 있다.  
이렇게 메서드 이름과 파라미터를 합쳐서 포인트 컷 시그니처라고 부른다. 반환 타입은 void에 코드 내용은 비워둔다.  
포인트 컷 표현식을 여러 메서드에서 재사용 할 수 있고, 메서드 이름으로 의미를 부여할 수 있다는 장점이 있다. 접근 제어자를  public으로 두면 외부의 Advice 클래스에서도 가져다 사용할 수 있다.  

물론 여러 포인트 컷 시그니처를 적용할 수도 있다.  

```java
@Slf4j
@Aspect
public class AspectV3 {

    @Pointcut("execution(* bepoz.order..*(..))")
    private void allOrder(){} //pointcut signature

    @Pointcut("execution(* *..*Service..*(..))")
    private void allService() {}

    @Around("allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }

    @Around("allOrder() && allService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

        try {
            log.info("[Transaction start] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            log.info("[Transaction commit] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            log.info("[Transaction rollback] {}", joinPoint.getSignature());
            throw e;
        } finally {
            log.info("[Resource Release] {}", joinPoint.getSignature());
        }
    }
}
```

``allOrder()``는 ``bepoz.order`` 패키지 하위에 적용이 되었고, ``allService()``는 ``*Service`` 로 인해 네이밍이 Service로 끝나는 것에 적용이 된다(세부 AspectJ 표현은 나중에 다룰 예정). 일반적인 자바 문법과 같이 ``&&``, ``||``, ``!`` 사용이 가능하다.  
``doTransaction``의 경우에는 ``allOrder() && allService()`` 이므로 ``OrderService``에만 Advice가 걸릴 것이다.  

테스트 코드에서 ``orderService.orderItem("itemA")``의 호출 로그 결과를 보면 다음과 같다.  

<img src="https://user-images.githubusercontent.com/45073750/148637597-20f696ce-61b3-4f3c-b090-122bd703cf5b.png" alt="image" style="zoom:50%;" />

로그 값을 살펴보면 먼저 ``doLog()``의 Advice가 호출이 되어 첫 라인이 찍혔다.  
``doTransaction()``이 호출되어 ``[Transaction start]``가 찍히고, 메서드 내부의 ``joinPoint.proceed()``를 통해 target인 ``OrderService``의 ``orderItem()``이 호출되었고, ``OrderService.orderItem`` 메서드 코드의 ``log.info("OrderService.orderItem()")``이 호출이 되어 3번째 라인이 찍혔다.  
그리고, ``orderRepository.save(itemId)``가 호출이 되고 이 ``OrderRepository``는 ``allService()`` 포인트 컷 시그니처에는 걸리지 않으므로(이때, ``OrderRepository``는 프록시 클래스라는 것을 인지하고 있어야 이해하기가 쉽다) ``doLog()``만 적용이 되어 4번째 라인이 찍히고 트랜잭션 관련한 로그가 별도로 찍히지 않는다. 마찬가지로 ``OrderRepository.save()`` 코드 내부의 로깅에 의해 5번째 라인이 찍히게 된다.  
이후 다시 ``OrderService``한테 걸려있던 ``doTransaction()`` 코드로 돌아와 ``Transaction commit``, ``Resource Release`` 로그가 찍히게 되는 것이다.  

앞에서 public을 이용하면 외부 클래스에서도 호출이 가능하다고 했다. 한 번 코드로 확인해보겠다.  

```java
public class Pointcuts {

    @Pointcut("execution(* bepoz.order..*(..))")
    public void allOrder(){} //pointcut signature

    @Pointcut("execution(* *..*Service..*(..))")
    public void allService() {}

    @Pointcut("allOrder() && allService()")
    public void orderAndService() {}
}

@Slf4j
@Aspect
public class AspectV4Pointcut {

    @Around("bepoz.order.aop.Pointcuts.allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }

  @Around("bepoz.order.aop.Pointcuts.orderAndService()")
  public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

    try {
      log.info("[Transaction start] {}", joinPoint.getSignature());
      Object result = joinPoint.proceed();
      log.info("[Transaction commit] {}", joinPoint.getSignature());
      return result;
    } catch (Exception e) {
      log.info("[Transaction rollback] {}", joinPoint.getSignature());
      throw e;
    } finally {
      log.info("[Resource Release] {}", joinPoint.getSignature());
    }
  }
}
```

위와 같이 ``@Pointcut``을 모아둔 ``Pointcuts.class``를 생성하고,  
``AspectV4Pointcut``에서 해당 클래스에 들어있는 포인트 컷 시그니처를 가져다가 사용하였다. 패키지 루트를 써줘야 한다.  

그렇다면, 적용되는 Advice의 순서는 조절할 수 없을까? 있다. 그러나 클래스 단위로만 조절이 가능하다.  

```java
@Slf4j
public class AspectV5Order {

    @Aspect
    @Order(2)
    public static class LogAspect {
        @Around("bepoz.order.aop.Pointcuts.allOrder()")
        public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[log] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }

    @Aspect
    @Order(1)
    public static class TxAspect {
        @Around("bepoz.order.aop.Pointcuts.orderAndService()")
        public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

            try {
                log.info("[Transaction start] {}", joinPoint.getSignature());
                Object result = joinPoint.proceed();
                log.info("[Transaction commit] {}", joinPoint.getSignature());
                return result;
            } catch (Exception e) {
                log.info("[Transaction rollback] {}", joinPoint.getSignature());
                throw e;
            } finally {
                log.info("[Resource Release] {}", joinPoint.getSignature());
            }
        }
    }
}
```

```java
@Slf4j
@SpringBootTest
@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})
public class AopTest {
  ...
```

클래스 단위로 조절을 할 수 있기 때문에 내부 클래스를 이용하였다. ``@Order()``를 이용해서 순서를 지정한다.  

<img src="https://user-images.githubusercontent.com/45073750/148638953-2510b377-f5c9-4725-9a22-82d0c6fda408.png" alt="image" style="zoom:50%;" />

결과는 다음과 같다. ``doLog()``부터 걸리던 앞선 결과와 달리 ``doTransaction()``부터 걸린 것을 확인할 수가 있었다.  
물론 내부 클래스가 아닌 클래스를 아예 나눠서 적용할 수도 있다.  

```java
@Slf4j
@Aspect
@Order(1)
public class TxAspect {

    @Around("bepoz.order.aop.Pointcuts.orderAndService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

        try {
            log.info("[Transaction start] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            log.info("[Transaction commit] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            log.info("[Transaction rollback] {}", joinPoint.getSignature());
            throw e;
        } finally {
            log.info("[Resource Release] {}", joinPoint.getSignature());
        }
    }
}
```

```java
@Slf4j
@Aspect
@Order(2)
public class LogAspect {

    @Around("bepoz.order.aop.Pointcuts.allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

```java
@Slf4j
@SpringBootTest
@Import({LogAspect.class, TxAspect.class})
public class AopTest {
  ...
```



