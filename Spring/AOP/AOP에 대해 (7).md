# AOP에 대해 (7)

포인트컷 지시자의 종류는 다음과 같다.  

* ``execution``: 메서드 실행 조인 포인트를 매칭
* ``within``: 특정 타입 내의 조인 포인트를 매칭
* ``args``: 인자가 주어진 타입의 인스턴스인 조인 포인트
* ``this``: 스프링 빈 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트
* ``target``: Target 객체(스프링 AOP 프록시가 가르키는 실제 대상)를 대상으로 하는 조인 포인트
* ``@target``: 실행 객체의 클래스에 주어진 타입의 어노테이션이 있는 조인 포인트
* ``@within``: 주어진 어노테이션이 있는 타입 내 조인 포인트
* ``@annotation``: 메서드가 주어진 어노테이션을 가지고 있는 조인 포인트를 매칭
* ``@args``: 전달된 실제 인수의 런타임 타입이 주어진 타입의 어노테이션을 갖는 조인 포인트
* ``bean``: 스프링 전용 포인트컷 지시자, 빈의 이름으로 포인트컷을 지정

<br/>

바로 코드로 살펴보겠다. 먼저 기본 코드는 다음과 같다.  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ClassAop {
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAop {
    String value();
}

public interface MemberService {

    String hello(String param);
}


@ClassAop
@Component
public class MemberServiceImpl implements MemberService {

    @Override
    @MethodAop("test value")
    public String hello(String param) {
        return "ok";
    }

    public String internal(String param) {
        return "ok";
    }
}
```

```java
@Slf4j
public class ExecutionTest {

    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    Method helloMethod;

    @BeforeEach
    public void init() throws NoSuchMethodException {
        helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
    }

    @Test
    void printMethod() {
        log.info("helloMethod={}", helloMethod);
    }
}
```

패키지 구조는 다음과 같다.  

<img src="https://user-images.githubusercontent.com/45073750/148675604-4d8912ae-da42-4ded-88c6-11502459ef03.png" alt="image" style="zoom:60%;" />

### execution

먼저 ``execution`` 문법을 살펴보겠다. 기본 틀은 다음과 같다.  
``execution(접근제어자? 반환타입 선언타입?메서드이름(파라미터) 예외?)``  
여기서 ?는 생략할 수 있다는 뜻이다. ``*`` 와 같은 패턴을 지정할 수도 있다. 코드로 살펴보겠다.  

```java
@Test
void exactMatch() {
  pointcut.setExpression("execution(public String bepoz.member.MemberServiceImpl.hello(String))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
/*
접근제어자?: public
반환타입: String
선언타입?: bepoz.member.MemberServiceImpl
메서드이름: hello
파라미터: (String)
예외?: 생략
*/

@Test
void allMatch() {
  pointcut.setExpression("execution(* *(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
/*
접근제어자?: 생략
반환타입: *
선언타입?: 생략
메서드이름: *
파라미터: (..)
예외?: 생략

파라미터의 .. 은 파라미터 타입과 수가 상관없다는 뜻
*/

@Test
void nameMatch() {
  pointcut.setExpression("execution(* hello(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void nameMatchStar1() {
  pointcut.setExpression("execution(* hel*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void nameMatchStar2() {
  pointcut.setExpression("execution(* *el*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void nameMatchFalse() {
  pointcut.setExpression("execution(* wrongName(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}

@Test
void packageExactMatch1() {
  pointcut.setExpression("execution(* bepoz.member.MemberServiceImpl.hello(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
//패키지에서 '.'은 정확하게 해당 위치의 패키지, '..'은 해당 위치의 패키지와 그 하위 패키지도 포함

@Test
void packageExactMatch2() {
  pointcut.setExpression("execution(* bepoz.member.*.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void packageExactMatchFalse() {
  pointcut.setExpression("execution(* bepoz.*.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}
/*
MemberServiceImpl은 bepoz.member 에 있다. 하지만 위의 경우와 같이 bepoz.*.* 은 bepoz 패키지에 존재해야만 하므로 False가 나오게 되는 것이다. 반면 bepoz..*.* 이었더라면 True가 된다.
*/

@Test
void packageMatchSubPackage1() {
  pointcut.setExpression("execution(* bepoz.member..*.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void packageMatchSubPackage2() {
  pointcut.setExpression("execution(* bepoz..*.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void typeExactMatch() {
  pointcut.setExpression("execution(* bepoz.member.MemberServiceImpl.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void typeExactMatchSuperType() {
  pointcut.setExpression("execution(* bepoz.member.MemberService.*(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
// 부모 타입또한 매칭 된다는 것을 확인할 수가 있다.

@Test
void typeMatchInternal() throws NoSuchMethodException {
  pointcut.setExpression("execution(* bepoz.member.MemberService.*(..))");
  Method internalMethod = MemberServiceImpl.class.getMethod("internal", String.class);

  assertThat(pointcut.matches(internalMethod, MemberServiceImpl.class)).isFalse();
}
//부모 타입에 있는 메서드만 허용한다는 것을 알 수가 있다.

@Test
void argsMatch() {
  pointcut.setExpression("execution(* *(String))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
//String 한정

@Test
void argsMatchNoArgs() {
  pointcut.setExpression("execution(* *())");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}
//파라미터가 없는 경우

@Test
void argsMatchStar() {
  pointcut.setExpression("execution(* *(*))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
//모든 타입 허용, 1개의 파라미터 (*,* 은 모든 타입 허용, 2개의 파라미터임)

@Test
void argsMatchAll() {
  pointcut.setExpression("execution(* *(..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
//모든 타입 허용, 0개 이상의 파라미터

@Test
void argsMatchComplex() {
  pointcut.setExpression("execution(* *(String, ..))");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
//String 타입으로 시작, 이후 0개 이상의 모든 타입 허용 파라미터
```

### within

이번에는 ``within`` 지시자를 코드로 살펴보겠다. ``within``은 특정 타입 내의 조인 포인트에 대한 매칭을 제한한다.  
``execution`` 에서 타입 부분만 사용한다고 보면 된다.  

```java
@Test
void withinExact() {
  pointcut.setExpression("within(bepoz.member.MemberServiceImpl)");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void withinStar() {
  pointcut.setExpression("within(bepoz.member.*Service*)");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void withinSubPackage() {
  pointcut.setExpression("within(bepoz..*)");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void withinSuperType() {
  pointcut.setExpression("within(bepoz.member.MemberService)");
  assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}
//within은 execution과 다르게 부모타입으로 지정할 수가 없다.
```

### args

이번에는 ``args``다. ``args`` 는 실제 넘어온 파라미터 객체 인스턴스를 보고 판단한다.  
``execution`` 은 파라미터 타입이 정확하게 매칭되어야 하지만, ``args``는 부모 타입을 허용한다.  
``execution``은 클래스에 선언된 정보를 기반으로 판단하지만, ``args``는 위에 말한대로 실제 넘어온 파라미터 객체 인스턴스를 보고 판단하기 때문이다. 코드로 살펴보겠다.  

```java
public class ArgsTest {

    Method helloMethod;

    @BeforeEach
    public void init() throws NoSuchMethodException {
        helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
    }

    private AspectJExpressionPointcut pointcut(String expression) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(expression);

        return pointcut;
    }

    @Test
    void args() {
        assertThat(pointcut("args(String)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(Object)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args()")
                .matches(helloMethod, MemberServiceImpl.class)).isFalse();
        assertThat(pointcut("args(..)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(*)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(String, ..)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
    }

    @Test
    void argsVsExecution() {
        assertThat(pointcut("args(String)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(java.io.Serializable)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(Object)")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();

        assertThat(pointcut("execution(* *(String))")
                .matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("execution(* *(java.io.Serializable))")
                .matches(helloMethod, MemberServiceImpl.class)).isFalse();
        assertThat(pointcut("execution(* *(Object))")
                .matches(helloMethod, MemberServiceImpl.class)).isFalse();
    }
}
```

### @target, @within

``@target``: 실행 객체의 클래스에 주어진 타입의 어노테이션이 있는 조인 포인트  
``@within``: 주어진 어노테이션이 있는 타입 내 조인 포인트  

말이 어려운데 간단히 말하면 ``@within``은 어노테이션이 걸린 해당 메서드에만 적용이 되고,  
``@target``은 부모 클래스의 메서드까지 적용이 된다.  

그냥 코드로 살펴보자.  

```java
@Slf4j
@Import({AtTargetAtWithinTest.Config.class})
@SpringBootTest
public class AtTargetAtWithinTest {

    @Autowired
    Child child;

    @Test
    void success() {
        log.info("child Proxy={}", child.getClass());
        child.childMethod();
        child.parentMethod();
    }

    static class Config {

        @Bean
        public Parent parent() {
            return new Parent();
        }

        @Bean
        public Child child() {
            return new Child();
        }

        @Bean
        public AtTargetAtWithinAspect atTargetAtWithinAspect() {
            return new AtTargetAtWithinAspect();
        }
    }

    static class Parent {
        public void parentMethod() {}
    }

    @ClassAop
    static class Child extends Parent {
        public void childMethod() {}
    }

    @Slf4j
    @Aspect
    static class AtTargetAtWithinAspect {

        @Around("execution(* bepoz..*(..)) && @target(bepoz.member.annotation.ClassAop)")
        public Object atTarget(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@target] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        @Around("execution(* bepoz..*(..)) && @within(bepoz.member.annotation.ClassAop)")
        public Object atWithin(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@within] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}

/*
[@target] void bepoz.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@within] void bepoz.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@target] void bepoz.pointcut.AtTargetAtWithinTest$Parent.parentMethod()
```

앞서 생성해둔 ``ClassAop`` 를 ``Child`` 에 걸어둔 상태이다.  
``@target`` 을 걸어둔 Advice는 ``Parent`` 의 메서드에도 호출이 된 반면,  
``@within`` 을 걸어둔 Advice는 ``Child`` 에만 적용이 되었다.  

### @annotation

``@annotation`` 은 메서드가 주어진 어노테이션을 가지고 있는 조인 포인트를 매칭한다.  

```java
@Slf4j
@Import(AtAnnotationTest.AtAnnotationAspect.class)
@SpringBootTest
public class AtAnnotationTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class AtAnnotationAspect {

        @Around("@annotation(bepoz.member.annotation.MethodAop)")
        public Object doAtAnnotation(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@annotation] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}

/*
[@annotation] String bepoz.member.MemberServiceImpl.hello(String)
```

### bean

``bean``: 스프링 전용 포인트컷 지시자, 빈의 이름으로 지정한다.  

```java
@Slf4j
@Import(BeanTest.BeanAspect.class)
@SpringBootTest
public class BeanTest {

    @Autowired
    OrderService orderService;

    @Test
    void success() {
        orderService.orderItem("itemA");
    }

    @Aspect
    static class BeanAspect {
        @Around("bean(orderService) || bean(*Repository)")
        public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[bean] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}
/*
[bean] void bepoz.order.OrderService.orderItem(String)
OrderService.orderItem()
[bean] String bepoz.order.OrderRepository.save(String)
OrderRepository.save()
```

### 매개변수 전달

``this, target, args, @target, @within, @annotation, @args`` 를 이용해서 Advice에 매개변수를 전달할 수 있다.  

```java
@Slf4j
@Import(ParameterTest.ParameterAspect.class)
@SpringBootTest
public class ParameterTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class ParameterAspect {

        @Pointcut("execution(* bepoz.member..*.*(..))")
        private void allMember() {}

        @Around("allMember()")
        public Object logArgs1(ProceedingJoinPoint joinPoint) throws Throwable {
            Object arg1 = joinPoint.getArgs()[0];
            log.info("[logArgs1]{}, arg={}", joinPoint.getSignature(), arg1);
            return joinPoint.proceed();
        }

        @Around("allMember() && args(arg,..)")
        public Object logArgs2(ProceedingJoinPoint joinPoint, Object arg) throws Throwable {
            log.info("[logArg2]{} arg={}", joinPoint.getSignature(), arg);
            return joinPoint.proceed();
        }

        @Before("allMember() && args(arg,..)")
        public void logArgs3(String arg) {
            log.info("[logArgs3] arg={}", arg);
        }

        @Before("allMember() && this(obj)")
        public void thisArgs(JoinPoint joinPoint, MemberService obj) {
            log.info("[this]{} obj={}", joinPoint.getSignature(), obj.getClass());
        }

        @Before("allMember() && target(obj)")
        public void targetArgs(JoinPoint joinPoint, MemberService obj) {
            log.info("[target]{} obj={}", joinPoint.getSignature(), obj.getClass());
        }

        @Before("allMember() && @target(annotation)")
        public void atTarget(JoinPoint joinPoint, ClassAop annotation) {
            log.info("[@target]{} obj={}", joinPoint.getSignature(), annotation);
        }

        @Before("allMember() && @within(annotation)")
        public void atWithin(JoinPoint joinPoint, ClassAop annotation) {
            log.info("[@within]{} obj={}", joinPoint.getSignature(), annotation);
        }

        @Before("allMember() && @annotation(annotation)")
        public void atAnnotation(JoinPoint joinPoint, MethodAop annotation) {
            log.info("[@annotation]{} annotationValue={}", joinPoint.getSignature(), annotation.value());
        }
    }
}
```

![image](https://user-images.githubusercontent.com/45073750/148685566-3a236348-7683-4cfa-a0f3-a2d76d421c28.png)

``this`` 는 프록시 객체를 전달 받고, ``target`` 은 실제 대상 객체를 전달 받는다.  
다른 것들은 코드로 다 추측이 가능할 것이다.  

### this, target

먼저 코드부터 살펴보겠다.  

```java
@Slf4j
@Import(ThisTargetTest.ThisTargetAspect.class)
//@SpringBootTest(properties = "spring.aop.proxy-target-class=false") //JDK 동적 프록시
@SpringBootTest(properties = "spring.aop.proxy-target-class=true") //CGLIB
public class ThisTargetTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class ThisTargetAspect {

        @Around("this(bepoz.member.MemberService)")
        public Object doThisInterface(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[this-interface] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        @Around("target(bepoz.member.MemberService)")
        public Object doTargetInterface(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[target-interface] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        @Around("this(bepoz.member.MemberServiceImpl)")
        public Object doThis(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[this-impl] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        @Around("target(bepoz.member.MemberServiceImpl)")
        public Object doTarget(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[targets-impl] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}

/*
CGLIB인 경우
memberService Proxy=class bepoz.member.MemberServiceImpl$$EnhancerBySpringCGLIB$$fd0677f3
[targets-impl] String bepoz.member.MemberServiceImpl.hello(String)
[target-interface] String bepoz.member.MemberServiceImpl.hello(String)
[this-impl] String bepoz.member.MemberServiceImpl.hello(String)
[this-interface] String bepoz.member.MemberServiceImpl.hello(String)

JDK 동적 프록시인 경우
memberService Proxy=class com.sun.proxy.$Proxy51
[targets-impl] String bepoz.member.MemberService.hello(String)
[target-interface] String bepoz.member.MemberService.hello(String)
[this-interface] String bepoz.member.MemberService.hello(String)
```

``this``: 스프링 빈 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트  
``target``: Target 객체(스프링 AOP 프록시가 가르키는 실제 대상)를 대상으로 하는 조인 포인트  

``this`` 와 ``target`` 은 ``*`` 이 불가능하고, 정확하게 타입을 지정해야 한고, 부모 타입을 허용한다.  

이제 위 코드의 결과와 엮어서 설명을 해보겠다.  

알다시피, JDK 동적 프록시는 인터페이스를 구현한 프록시 객체를 생성하고 CGLIB은 구체 클래스를 상속 받아서 프록시 객체를 생성한다. 위 코드에서 ``spring.aop.proxy-target-class`` 옵션을 사용해서 CGLIB, JDK 동적 프록시 어떤 것을 선택할 것인지 정해주었다(yml에 입력해도됨). true는 CGLIB, false는 JDK 동적 프록시다. 스프링 부트는 기본적으로 default로 true를 채택하고있다.  

먼저, JDK 동적 프록시를 사용하는 경우를 살펴보겠다.  

``this(bepoz.member.MemberService)``: proxy 객체로 판단하고, ``this`` 는 부모 타입을 허용하기 때문에 AOP가 적용된다.  
``target(bepoz.member.MemberService)``: target 객체로 판단하고, ``target``은 부모 타입을 허용하기 때문에 AOP가 적용된다.  

``this(bepoz.member.MemberServiceImpl)``: proxy 객체로 판단한다. JDK 동적 프록시로 만들어진 객체는 ``MemberService`` 를 기반으로 만든 새로운 클래스이기 때문에 ``MemberServiceImpl`` 를 알지 못한다. 따라서 AOP 적용 대상이 아니다.  
``target(bepoz.member.MemberServiceImpl)``: target 객체로 판단한다. target 객체가 ``MemberServiceImpl`` 타입이므로 AOP 적용 대상이다.  

![image](https://user-images.githubusercontent.com/45073750/148779304-97ea1324-fc10-400f-afaf-9126eb0b6971.png)

이번에는 CGLIB 프록시를 사용하는 경우를 살펴보겠다.  

``this(bepoz.member.MemberService)``: proxy 객체로 판단하고, ``this`` 는 부모 타입을 허용하기 때문에 AOP가 적용된다.  
``target(bepoz.member.MemberService)``: target 객체로 판단하고, ``target``은 부모 타입을 허용하기 때문에 AOP가 적용된다.  

``this(bepoz.member.MemberServiceImpl)``: proxy 객체로 판단한다. JDK 동적 프록시로 만들어진 객체는 ``MemberServiceImpl`` 를 상속받아 생성된 클래스이기 때문에 ``MemberServiceImpl`` 를 안다(부모이다). 따라서 AOP 적용 대상이다.다.  
``target(bepoz.member.MemberServiceImpl)``: target 객체로 판단한다. target 객체가 ``MemberServiceImpl`` 타입이므로 AOP 적용 대상이다.  

![image](https://user-images.githubusercontent.com/45073750/148779764-891852df-78b5-4a8d-ac6b-b03fa74eee82.png)

<br/>

이러한 이유로 인해 위 코드의 결과 로그가 저렇게 찍힌 것이다.  

이제 스프링 AOP의 문제점 및 주의사항에 대해 알아보겠다.  
8편에서 계속...  

[AOP에 대해 (1)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(8).md)

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)