# AOP에 대해 (8)

이번에는 프록시 방식의 AOP가 야기하는 문제점을 살펴보겠다.  

## 내부 호출 문제

앞서 정리한 것을 토대로 우리는 다음과 같이 프록시가 동작한다는 것을 알 수가 있었다.  
client -> proxy -> target 호출의 순서로 말이다.  

그렇다면 다음과 같은 코드는 어떨까?  

```java
@Slf4j
@Component
public class CallServiceV0 {

    public void external() {
        log.info("call external");
        internal();
    }

    public void internal() {
        log.info("call internal");
    }
}

@Slf4j
@Aspect
public class CallLogAspect {

    @Before("execution(* bepoz.internalcall..*.*(..))")
    public void doLog(JoinPoint joinPoint) {
        log.info("aop={}", joinPoint.getSignature());
    }
}

@Slf4j
@Import(CallLogAspect.class)
@SpringBootTest
class CallServiceV0Test {

    @Autowired
    CallServiceV0 callServiceV0;

    @Test
    void external() {
        callServiceV0.external();
    }
}
```

이 경우에 ``CallServiceV0Test`` 에서 주입받은 ``CallServiceV0``  빈은 프록시일 것이다. 그리고 ``external()`` 을 호출하면 내부적으로 target을 호출한 것일테고 말이다. 그런데, ``external()`` 에서 내부적으로 ``internal()`` 을 호출하고 있다.  

이 상황에서 당연히 Advice가 호출되지 않는다. 프록시 객체가 아닌 target 객체의 ``internal()`` 이 호출되었으니 말이다.  
만약, 프록시 방식의 AOP가 아니라 컴파일 타임, 클래스 로딩 시점에서의 위빙을 이용한 AOP 방식이었으면 문제되지 않았을 것이다. 물론 사용방법이 복잡해진다는 단점은 있다.  

### 대안1 자기 자신 주입

```java
@Slf4j
@Component
public class CallServiceV1 {

    private CallServiceV1 callServiceV1;

    @Autowired
    public void setCallServiceV1(CallServiceV1 callServiceV1) {
        this.callServiceV1 = callServiceV1;
    }

    public void external() {
        log.info("call external");
        callServiceV1.internal();
    }

    public void internal() {
        log.info("call internal");
    }
}
```

자기 자신을 빈 주입 받는 것이다. 프록시를 주입받는 것이다. 그리고 그걸 통해서 호출한다.  
생성자를 통한 주입은 불가능하다(빈 생성을 하려는데 그 빈을 주입 받는다는게 말이 안됨). 하지만, 수정자 주입은 주입 시점이 달라 가능하다(스프링 부트 2.6 부터는 수정자로도 불가능하다. 사용하려면 ``spring.main.allow-circular-references=true`` 를 사용해야함).  

### 대안2 지연 조회

```java
@Slf4j
@Component
public class CallServiceV2 {

//    private final ApplicationContext ac;
    private final ObjectProvider<CallServiceV2> callServiceProvider;

//    public CallServiceV2(ApplicationContext ac) {
//        this.ac = ac;
//    }

    public CallServiceV2(ObjectProvider<CallServiceV2> callServiceProvider) {
        this.callServiceProvider = callServiceProvider;
    }

    public void external() {
        log.info("call external");
//        CallServiceV2 callServiceV2 = ac.getBean(CallServiceV2.class);
        CallServiceV2 callServiceV2 = callServiceProvider.getObject();
        callServiceV2.internal();
    }

    public void internal() {
        log.info("call internal");
    }
}
```

대안1 에서 생성자 주입이 실패하는 이유가 자기 자신을 생성하면서 주입해야 하기 때문에 실패했다.  
대안2 에서는 빈을 지연해서 조회를 한다. ``ApplicationContext`` 를 이용할 수도 있지만, 사용하려는 기능과 비교해 너무나도 많은 기능을 제공하는 클래스이기 때문에 ``ObjectProvider`` 를 사용했다.(``ObjectProvider`` 는 프로토타입 스코프 빈을 학습할 때에 잠시 봤었다. [링크](https://github.com/Be-poz/TIL/blob/master/Spring/%EB%B9%88%20%EC%8A%A4%EC%BD%94%ED%94%84%EC%97%90%20%EB%8C%80%ED%95%B4.md))    

``ObjectProvider`` 는 객체를 스프링 컨테이너에서 조회하는 것을 스프링 빈 생성 시점이 아니라 실제 객체를 사용하는 시점으로 지연할 수 있다.  

### 대안3 구조 변경

가장 나은 대안은 내부 호출이 발생하지 않도록 구조를 변경하는 것이다.  

```java
@Slf4j
@Component
public class InternalService {

    public void internal() {
        log.info("call internal");
    }
}

@Slf4j
@Component
public class CallServiceV3 {

    private final InternalService internalService;

    public CallServiceV3(InternalService internalService) {
        this.internalService = internalService;
    }

    public void external() {
        log.info("call external");
        internalService.internal();
    }
}
```

다음과 같이 구조를 변경한다면 문제되지 않을 것이다. 또는 client에서 ``external()`` 과 ``internal()`` 을 모두 호출하는 방법도 있겠다.  

<br/>

