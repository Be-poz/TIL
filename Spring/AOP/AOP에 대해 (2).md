# AOP에 대해 (2)

템플릿 콜백 패턴을 이용해서 로그 추적기를 적용했었다.  
이번에는 프록시 패턴을 이용해서 원본 코드에 손대지 않고 로그 추적기를 적용해보도록 하겠다.  

3가지의 각기 다른 버전의 예를 가지고 진행하겠다.  

1. 인터페이스를 구현하고, 수동으로 빈 등록
2. 인터페이스 없는 구체 클래스, 수동으로 빈 등록
3. 인터페이스 없는 구체 클래스, 컴포넌트 스캔을 이용하여 빈 등록

<img src="https://user-images.githubusercontent.com/45073750/146638318-3ad330f0-5207-46a0-a7a8-6db6ee84d904.png" alt="image" style="zoom: 67%;" />

다음과 같은 형태이며, V3는 일반적으로 사용하는 ``@Controller``, ``@Service``,``@Repository``를 이용한 클래스다.  
V2는 위의 어노테이션만 빠진 상태이고, ``AppV2Config``에서 빈을 등록해주고 있다.  

```java
@Configuration
public class AppV2Config {

    @Bean
    public BepozControllerV2 bepozControllerV2() {
        return new BepozControllerV2(bepozServiceV2());
    }

    @Bean
    public BepozServiceV2 bepozServiceV2() {
        return new BepozServiceV2(bepozRepositoryV2());
    }

    @Bean
    public BepozRepositoryV2 bepozRepositoryV2() {
        return new BepozRepositoryV2();
    }
}
```

다음과 같이 말이다. ``BepozControllerV2``는 ``@Controller``를 사용하고 있지는 않지만 ``@RequestMapping``은 사용한다.  

```java
@Slf4j
@RequestMapping
@ResponseBody
public class BepozControllerV2 {

    private final BepozServiceV2 bepozService;

    public BepozControllerV2(BepozServiceV2 bepozService) {
        this.bepozService = bepozService;
    }

    @GetMapping("/v2/request")
    public String request(@RequestParam("itemId") String itemId) {
        bepozService.save(itemId);
        return "ok";
    }

    @GetMapping("/v2/no-log")
    public String noLog() {
        return "ok";
    }
}
```

스프링에서는 ``@Controller``나 ``@RequestMapping``이 존재해야만 컨트롤러로 인식하기 때문에 붙여주었다.  
하지만, ``@Controller``를 붙이면 자동 빈 등록이 되버리기 때문에 달아주지 않았다.  

V1은 인터페이스를 구현하는 방식을 사용했다. 이건 코드로 살펴보겠다.  

```java
// 마찬가지로 Controller로 인식을 해야하기 때문에 @RequestMapping을 붙여주었다.
@RequestMapping
@ResponseBody
public interface BepozControllerV1 {

    @GetMapping("/v1/request")
    String request(@RequestParam("itemId") String itemId);

    @GetMapping("/v1/no-log")
    String noLog();
}

@Slf4j
public class BepozControllerV1Impl implements BepozControllerV1 {

    private final BepozServiceV1 bepozService;

    public BepozControllerV1Impl(BepozServiceV1 bepozService) {
        this.bepozService = bepozService;
    }

    @Override
    public String request(String itemId) {
        bepozService.save(itemId);
        return "ok";
    }

    @Override
    public String noLog() {
        return "ok";
    }
}

public interface BepozServiceV1 {
    void save(String id);
}

public class BepozServiceV1Impl implements BepozServiceV1 {

    private final BepozRepositoryV1 bepozRepository;

    public BepozServiceV1Impl(BepozRepositoryV1 bepozRepository) {
        this.bepozRepository = bepozRepository;
    }

    @Override
    public void save(String id) {
        bepozRepository.save(id);
    }
}

public interface BepozRepositoryV1 {
    void save(String id);
}

public class BepozRepositoryV1Impl implements BepozRepositoryV1 {
    @Override
    public void save(String id) {
        if (id.equals("ex")) {
            throw new IllegalStateException("exception thrown!");
        }
        sleep(1000);
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

이제 위의 코드를 원본코드에 손을 대지 않고 로그 추적기를 적용할 수 있어야 한다.  
이를 위해 앞서 말한대로 프록시 패턴을 사용하게 된다.  

<img src="https://user-images.githubusercontent.com/45073750/146638460-b4453a29-99d4-4593-b7c7-3c495e3c3398.png" alt="image" style="zoom:80%;" />

다음과 같이 Client가 Server를 일반적으로 호출을 했었지만,  

<img src="https://user-images.githubusercontent.com/45073750/146638499-ccd2a1cf-f3c6-4e7c-82cc-63970f87b0d4.png" alt="image" style="zoom:80%;" />

다음과 같이 프록시가 껴서 Server 대신 호출이 되게끔 하는 것이 목표다.  
프록시를 사용하게 되면 프록시 내부에서 여러가지 일을 할 수 있다.  

* 접근 제어
  * 권한에 따른 접근 차단
  * 캐싱
  * 지연 로딩
* 부가 기능 추가
  * 원래 서버가 제공하는 기능에 더해서 부가 기능을 수행
  * ex) 요청 값이나 응답 값을 중간에서 변형
  * ex) 실행 시간을 측정해서 추가 로그를 남김

이를 위해서 유의할 점은 Server와 Proxy가 같은 인터페이스를 구현하고 있어야 한다는 것이다.  
Client는 Proxy를 호출하는지 Server를 호출하는지 몰라야 한다. 이렇게 프록시를 사용하게 되면 Server 객체의 원본 코드를 건드리지 않고 추가적인 기능을 추가할 수 있게 된다.  

프록시는 객체 안에서의 개념도 있고, 웹 서버에서의 개념도 있다. 하지만, 근본적인 개념은 같다.  

V1부터 살펴보겠다. 현재 V1의 클래스 의존관계는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/146881801-74acdd06-5670-46e1-bc3b-e46f16ad37e8.png)  

런타임 의존관계는 Client -> ControllerImpl -> ServiceImpl -> RepositoryImpl 과 같을 것이다.  
위의 관계를 프록시를 이용해서 다음과 같이 변경해 보겠다.  

![image](https://user-images.githubusercontent.com/45073750/146882246-f9198d82-39fb-4584-919b-7c45ad895832.png)  

런타임 의존관계는 Client -> ControllerProxy -> ControllerV1Impl -> ServiceProxy -> ServiceV1Impl ... 이 될 것이다.  
Repository 부분은 생략했다. 이제 코드로 살펴보겠다.  

```java
@RequiredArgsConstructor
public class BepozRepositoryInterfaceProxy implements BepozRepositoryV1 {

    private final BepozRepositoryV1 target;
    private final LogTrace logTrace;

    @Override
    public void save(String id) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozRepository.save()");
            target.save(id);
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

```java
@RequiredArgsConstructor
public class BepozServiceInterfaceProxy implements BepozServiceV1 {

    private final BepozServiceV1 target;
    private final LogTrace logTrace;

    @Override
    public void save(String id) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozService.save()");
            target.save(id);
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

```java
@RequiredArgsConstructor
public class BepozControllerInterfaceProxy implements BepozControllerV1 {

    private final BepozControllerV1 target;
    private final LogTrace logTrace;

    @Override
    public String request(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozController.request()");
            String result = target.request(itemId);
            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }

    @Override
    public String noLog() {
        return target.noLog();
    }
}
```

프록시 객체를 만들고 구현한 메서드에서  ``LogTrace``를 사용했다. 이전에는 Impl 객체에 해당 로직을 추가했어야 했다.  
하지만, 프록시 객체 덕분에 Impl 객체를 건드리지 않고 진행할 수 있게 되었다. 그리고 ``AppV1Config``대신 ``InterfaceProxyConfig``를 사용할 것이다.

```java
//이전에 사용하던 config
@Configuration
public class AppV1Config {

    @Bean
    public BepozControllerV1 bepozControllerV1() {
        return new BepozControllerV1Impl(bepozServiceV1());
    }

    @Bean
    public BepozServiceV1 bepozServiceV1() {
        return new BepozServiceV1Impl(bepozRepositoryV1());
    }

    @Bean
    public BepozRepositoryV1 bepozRepositoryV1() {
        return new BepozRepositoryV1Impl();
    }
}

//현재 사용할 config
@Configuration
public class InterfaceProxyConfig {

    @Bean
    public BepozControllerV1 bepozControllerV1(LogTrace logTrace) {
        BepozControllerV1Impl controllerImpl = new BepozControllerV1Impl(bepozServiceV1(logTrace));
        return new BepozControllerInterfaceProxy(controllerImpl, logTrace);
    }

    @Bean
    public BepozServiceV1 bepozServiceV1(LogTrace logTrace) {
        BepozServiceV1Impl serviceImpl = new BepozServiceV1Impl(bepozRepositoryV1(logTrace));
        return new BepozServiceInterfaceProxy(serviceImpl, logTrace);
    }

    @Bean
    public BepozRepositoryV1 bepozRepositoryV1(LogTrace logTrace) {
        BepozRepositoryV1Impl repositoryImpl = new BepozRepositoryV1Impl();
        return new BepozRepositoryInterfaceProxy(repositoryImpl, logTrace);
    }
}
```

실제 객체를 빈 등록하지 않고, 프록시 객체를 빈을 등록하게 된다. 프록시 내부에서는 실제 객체를 참조하고 있다.  
proxy -> target 의 형태를 띄고 있는 것이다. 

이제 V2를 살펴보겠다. V2는 인터페이스가 없는 구체 클래스를 수동 빈 등록해준 상황이었다.  
이번에도 V1과 마찬가지로 다형성을 이용해서 프록시를 구현할 것이다. 다형성은 인터페이스를 구현하든 클래스를 상속하든 상위 타입만 맞으면 된다.  
따라서, 인터페이스를 사용하고 있지 않은 V2이기 때문에 상속을 이용해서 다형성을 이용할 것이다.  

```java
public class BepozRepositoryConcreteProxy extends BepozRepositoryV2 {

    private final BepozRepositoryV2 target;
    private final LogTrace logTrace;

    public BepozRepositoryConcreteProxy(BepozRepositoryV2 target, LogTrace logTrace) {
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public void save(String id) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozRepository.save()");

            target.save(id);
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

```java
public class BepozServiceConcreteProxy extends BepozServiceV2 {

    private final BepozServiceV2 target;
    private final LogTrace logTrace;

    public BepozServiceConcreteProxy(BepozServiceV2 target, LogTrace logTrace) {
        super(null);
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public void save(String id) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozService.save()");

            target.save(id);
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

```java
public class BepozControllerConcreteProxy extends BepozControllerV2 {

    private final BepozControllerV2 target;
    private final LogTrace logTrace;

    public BepozControllerConcreteProxy(BepozControllerV2 target, LogTrace logTrace) {
        super(null);
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public String request(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("BepozController.request()");

            String result = target.request(itemId);
            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }

    @Override
    public String noLog() {
        return target.noLog();
    }
}
```

```java
//이전에 사용하던 config
@Configuration
public class AppV2Config {

    @Bean
    public BepozControllerV2 bepozControllerV2() {
        return new BepozControllerV2(bepozServiceV2());
    }

    @Bean
    public BepozServiceV2 bepozServiceV2() {
        return new BepozServiceV2(bepozRepositoryV2());
    }

    @Bean
    public BepozRepositoryV2 bepozRepositoryV2() {
        return new BepozRepositoryV2();
    }
}


//현재 사용할 config
@Configuration
public class ConcreteProxyConfig {

    @Bean
    public BepozControllerV2 bepozControllerV2(LogTrace logTrace) {
        BepozControllerV2 controllerImpl = new BepozControllerV2(bepozServiceV2(logTrace));
        return new BepozControllerConcreteProxy(controllerImpl, logTrace);
    }

    @Bean
    public BepozServiceV2 bepozServiceV2(LogTrace logTrace) {
        BepozServiceV2 serviceImpl = new BepozServiceV2(bepozRepositoryV2(logTrace));
        return new BepozServiceConcreteProxy(serviceImpl, logTrace);
    }

    @Bean
    public BepozRepositoryV2 bepozRepositoryV2(LogTrace logTrace) {
        BepozRepositoryV2 repositoryImpl = new BepozRepositoryV2();
        return new BepozRepositoryConcreteProxy(repositoryImpl, logTrace);
    }
}
```

상속을 이용한 다형성을 이용해보았다.  
클래스 기반 프록시의 단점은 상속을 이용하기 때문에 자식 클래스를 생성할 때에 ``super()``를 호출해주어야 한다.  
이 상황에서는 부모의 기능을 이용하는 것이 아니기 때문에 파라미터로 null을 넣어 ``super(null)``를 호출한 것을 볼 수가 있다.  
인터페이스 기반의 프록시 생성에서는 이런 상황을 고민하지 않아도 된다는 장점이 있다.  

<br/>

**인터페이스 기반 프록시 vs 클래스 기반 프록시**  

* 클래스 기반 프록시는 상속을 사용하기 때문에 제약사항이 존재한다.
  * 부모 클래스의 생성자를 호출해야함
  * final 클래스면 상속이 불가능함
  * final 메서드면 오버라이딩이 불가능함
* 인터페이스 기반 프록시는 인터페이스가 존재해야 한다.
  * 인터페이스를 사용하는 것은 구현 변경 가능성이 있을 때 효율적이다. 그 뜻은 구현 변경 가능성이 거의 없는 코드에 무작정 인터페이스를 사용하는 것은 실용적이지 않다. 이 외에 인터페이스 도입의 여러 이유가 있긴 하지만 말하고자 하는 바는 인터페이스가 항상 필요한 것은 아니라는 것이다.

프록시를 이용해서 로그 추적기를 달아보았다. 하지만, 너무나도 많은 프록시 클래스들이 생성된다. 만약 적용해야 하는 클래스가 100개라면 클래스도 최소 100개를 만들어줘야 할 것이다. 프록시 클래스를 하나만 만들어서 하는 방법은 없을까?  

이를 위해 동적 프록시를 이용하게 된다.  
3편에서 계속...  

[AOP에 대해 (1)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/AOP/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(8).md)

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)