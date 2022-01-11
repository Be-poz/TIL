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

## 타입 캐스팅 문제

JDK 동적 프록시는 인터페이스가 필수이고, 인터페이스를 기반으로 프록시를 생성한다.  
CGLIB은 구체 클래스를 기반으로 프록시를 생성한다.  

인터페이스가 없는 경우에는 당연히 JDK 동적 프록시를 사용하겠지만, 인터페이스가 있는 경우에는 둘 중 하나를 선택할 수 있다.  

그렇다면 어떤 문제일까 코드로 살펴보겠다.  

```java
@Slf4j
public class ProxyCastingTest {

    @Test
    void jdkProxy() {
        MemberServiceImpl target = new MemberServiceImpl();
        ProxyFactory factory = new ProxyFactory(target);
        factory.setProxyTargetClass(false); //JDK 동적 프록시
      
        //프록시를 인터페이스로 캐스팅 성공
        MemberService memberServiceProxy = (MemberService) factory.getProxy();

        //프록시를 구현 클래스로 캐스팅 실패
        assertThatThrownBy(() -> {
            MemberServiceImpl casting = (MemberServiceImpl) factory.getProxy();
        }).isInstanceOf(ClassCastException.class);
    }

    @Test
    void cglibProxy() {
        MemberServiceImpl target = new MemberServiceImpl();
        ProxyFactory factory = new ProxyFactory(target);
        factory.setProxyTargetClass(true); //CGLIB 프록시
      
        //프록시를 인터페이스로 캐스팅 성공
        MemberService memberServiceProxy = (MemberService) factory.getProxy();

        //프록시를 구현 클래스로 캐스팅 성공
        MemberServiceImpl casting = (MemberServiceImpl) factory.getProxy();
    }
}
```

JDK 동적 프록시를 구현 클래스로의 캐스팅이 실패한 반면,  
CGLIB은 인터페이스, 구현 클래스로의 캐스팅이 모두 성공했다.  

그 이유는 JDK 동적 프록시는 인터페이스를 가지고 프록시를 만들고 CGLIB은 구현클래스를 가지고 프록시를 만들기 때문이다. 앞에서 쭉 봐온대로 말이다.  

<img src="https://user-images.githubusercontent.com/45073750/148958766-fc2855ca-3aa6-468b-902e-3292b48a9d27.png" alt="image" style="zoom:50%;" />

<img src="https://user-images.githubusercontent.com/45073750/148958863-7d8a930e-b8bd-4a5c-a227-44ad75bc1880.png" alt="image" style="zoom:50%;" />

그렇다면 위와 같은 문제가 어떤 상황에서 문제가 되는걸까?  

```java
@Autowired
MemberService memberService;

@Autowired
MemberServiceImpl memberServiceImpl;
```

위와 같이 구현 클래스를 빈 주입받을 때에 문제가 된다.  
CGLIB 방식이라면 문제가 없겠지만, JDK 동적 프록시를 사용하는 경우에는 문제가 발생할 것이다.  

보통의 경우 인터페이스 타입으로 빈 주입을 받는 일이 대부분이겠지만, 테스트 또는 여러가지 이유로 프록시가 적용된 구체 클래스를 직접 주입받아야 하는 경우가 있을 수도 있기 때문이다. 하지만 CGLIB 기반의 프록시 또한 단점이 있다.  

1. 대상 클래스에 기본 생성자 필수
2. 생성자 2번 호출 문제
3. final 키워드 클래스, 메서드 사용 불가

1에대한 답: CGLIB은 구체 클래스를 상속 받기 때문에 자바 문법에 따라 부모 클래스의 생성자도 호출되어야 한다. 따라서 대상 클래스에 기본 생성자가 존재해야 한다.  

2에대한 답: target 생성 시점에 생성자 1번 호출, 프록시 객체를 생성할 때 1번에 대한 답의 과정에서 부모 클래스의 생성자 호출 1번. 이렇게 2번이 호출된다.  

3에대한 답: 상속을 이용하기 위해서는 final이 붙어있어서는 안되기 때문  

이를 위해 스프링은 다음과 같이 해결하였다.  
스프링 4.0부터 ``objenesis`` 라는 라이브러리를 이용해서 기본 생성자 없이 객체 생성을 가능하게끔 만들었다. 이를 통해 기본 생성자 필수 문제를 해결하고,  target 호출 시에만 생성자를 호출하고 프록시 객체 생성 시에는 생성자를 호출하지 않아 생성자 2번 호출 문제를 해결하였다.  

스프링부트 2.0 버전부터는 CGLIB를 기본으로 사용하도록 채택되었다. 이로 인해 구체 클래스 타입의 의존관계 주입 시의 문제를 해결했다. 물론 ``spring.aop.proxy-target-class=false`` 옵션을 이용해서 JDK 동적 프록시를 이용할 수도 있다. final 문제의 경우 AOP를 적용할 대상에는 final 클래스나 final 메서드를 잘 사용하지 않으므로 크게 문제가 되지 않는다.  

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)