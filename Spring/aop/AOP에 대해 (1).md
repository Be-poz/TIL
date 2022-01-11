# AOP에 대해 (1)

```java
@RestController
@RequiredArgsConstructor
public class BepozController {

    private final BepozService bepozService;

    @GetMapping("/request")
    public String request(String itemId) {
        bepozService.save(itemId);
        return "ok";
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class BepozService {

    private final BepozRepository bepozRepository;

    public void save(String id) {
        bepozRepository.save(id);
    }
}
```

```java
@Repository
public class BepozRepository {

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

다음과 같은 코드가 있다. 이 코드에 밑의 로그 클래스를 이용해서 로그 추적기를 달아보려고 한다.  
세부 코드 내용은 중요하지 않으니 대충 이런게 있구나 하고 넘기면 된다. 그냥 로그를 위한 클래스라고만 생각하면 된다.  

<img src="https://user-images.githubusercontent.com/45073750/146503443-bd0234a4-5250-4536-a12e-d509bf843205.png" alt="image" style="zoom: 67%;" />

이런 식으로 출력하기 위한 코드일 뿐이다.  

```java
public interface LogTrace {
    TraceStatus begin(String message);

    void end(TraceStatus status);

    void exception(TraceStatus status, Exception e);
}

@Slf4j
@Component
public class LogTracer implements LogTrace {

    private static final String START_PREFIX = "-->";
    private static final String COMPLETE_PREFIX = "<--";
    private static final String EX_PREFIX = "<X-";

    private final ThreadLocal<TraceId> traceIdHolder = new ThreadLocal<>();

    @Override
    public TraceStatus begin(String message) {
        syncTraceId();
        TraceId traceId = traceIdHolder.get();
        Long startTimeMs = System.currentTimeMillis();
        log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX, traceId.getLevel()), message);
        return new TraceStatus(traceId, startTimeMs, message);
    }

    private void syncTraceId() {
        TraceId traceId = traceIdHolder.get();
        if (traceId == null) {
            traceIdHolder.set(new TraceId());
            return;
        }
        traceIdHolder.set(traceId.createNextId());
    }

    @Override
    public void end(TraceStatus status) {
        complete(status, null);
    }

    @Override
    public void exception(TraceStatus status, Exception e) {
        complete(status, e);
    }

    private void complete(TraceStatus status, Exception e) {
        Long stopTimeMs = System.currentTimeMillis();
        long resultTimeMs = stopTimeMs - status.getStartTimeMs();
        TraceId traceId = status.getTraceId();
        if (e == null) {
            log.info("[{}] {}{} time={}ms", traceId.getId(), addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs);
        } else {
            log.info("[{}] {}{} time={}ms ex={}", traceId.getId(), addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs, e.toString());
        }
        releaseTraceId();
    }

    private void releaseTraceId() {
        TraceId traceId = traceIdHolder.get();
        if (traceId.isFirstLevel()) {
            traceIdHolder.remove();
            return;
        }
        traceIdHolder.set(traceId.createPreviousId());
    }

    private static String addSpace(String prefix, int level) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < level; i++) {
            sb.append((i == level - 1) ? "|" + prefix : "|   ");
        }
        return sb.toString();
    }
}
```

```java
public class TraceStatus {

    private TraceId traceId;
    private Long startTimeMs;
    private String message;

    public TraceStatus(TraceId traceId, Long startTimeMs, String message) {
        this.traceId = traceId;
        this.startTimeMs = startTimeMs;
        this.message = message;
    }

    public TraceId getTraceId() {
        return traceId;
    }

    public Long getStartTimeMs() {
        return startTimeMs;
    }

    public String getMessage() {
        return message;
    }
}
```

```java
public class TraceId {

    private String id;
    private int level;

    public TraceId() {
        this.id = createdId();
        this.level = 0;
    }

    public TraceId(String id, int level) {
        this.id = id;
        this.level = level;
    }

    private String createdId() {
        return UUID.randomUUID().toString().substring(0, 8);
    }

    public TraceId createNextId() {
        return new TraceId(id, level + 1);
    }

    public TraceId createPreviousId() {
        return new TraceId(id, level - 1);
    }

    public boolean isFirstLevel() {
        return level == 0;
    }

    public String getId() {
        return id;
    }

    public int getLevel() {
        return level;
    }
}
```

이제 이것들을 기존의 Controller, Service, Repository에 적용을 해보면 다음과 같이 적용될 것이다.  

```java
//세 클래스 모두 LogTrace 타입의 빈을 주입 받은 상태
//BepozController 
@GetMapping("/request")
public String request(String itemId) {
  TraceStatus status = null;
  try {
    status = logTrace.begin("BepozController.request()");
    bepozService.save(itemId);
    logTrace.end(status);
    return "ok";
  } catch (Exception e) {
    logTrace.exception(status, e);
    throw e;
  }
}

//BepozService
public void save(String id) {
  TraceStatus status = null;
  try {
    status = logTrace.begin("BepozService.save()");
    bepozRepository.save(id);
    logTrace.end(status);
  } catch (Exception e) {
    logTrace.exception(status, e);
    throw e;
  }
}

//BepozRepository
public void save(String id) {
  TraceStatus status = null;
  try {
    status = logTrace.begin("BepozRepository.save()");

    if (id.equals("ex")) {
      throw new IllegalStateException("exception thrown!");
    }
    sleep(1000);

    logTrace.end(status);
  } catch (Exception e) {
    logTrace.exception(status, e);
    throw e;
  }
}
```

적용되는 클래스 비즈니스 로직에 여러 코드들이 덧붙여진 것을 확인할 수가 있다.  
코드를 보면 공통적인 로직들이 보인다.  

```java
 TraceStatus status = null;
  try {
			status = trace.begin("message"); 
      //핵심 기능 호출
      trace.end(status);
  } catch (Exception e) {
      trace.exception(status, e);
			throw e; 
  }
```

위와 같은 형태를 보인다.  
이것을 템플릿 메서드 패턴을 이용해서 변하는 것과 변하지 않는 로직을 분리해보겠다.  

```java
public abstract class AbstractTemplate<T> {

    private final LogTrace trace;

    public AbstractTemplate(LogTrace trace) {
        this.trace = trace;
    }

    public T execute(String message) {
        TraceStatus status = null;
        try {
            status = trace.begin(message);

            T result = call();

            trace.end(status);
            return result;
        } catch (Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }

    protected abstract T call();
}
```

```java
//BepozController
//as is
@GetMapping("/request")
public String request(String itemId) {
  TraceStatus status = null;
  try {
    status = logTrace.begin("BepozController.request()");
    bepozService.save(itemId);
    logTrace.end(status);
    return "ok";
  } catch (Exception e) {
    logTrace.exception(status, e);
    throw e;
  }
}

//to be
@GetMapping("/request")
public String request(String itemId) {
  AbstractTemplate<String> template = new AbstractTemplate<>(logTrace) {
    @Override
    protected String call() {
      bepozService.save(itemId);
      return "ok";
    }
  };

  return template.execute("BebozController.request()");
}
```

템플릿 메서드 패턴을 이용해 Controller 코드를 변경해 보았다.  
이런 상황에서는 로그를 남기는 로직에 대한 변경이 있을 경우 ``AbstractTemplate`` 클래스의 코드만 변경하면 될 것이다.  

하지만, 현재 형태는 **상속**을 사용하고 있다. 부모 클래스를 강하게 의존하게 된다. 부모 클래스가 수정되면 자식 클래스에게도 영향을 주게된다. 이를 콜백을 이용해서 해결해 보겠다.  

```java
public interface TraceCallback<T> {
    T call();
}

public class TraceTemplate {

    private final LogTrace logTrace;

    public TraceTemplate(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

    public <T> T execute(String message, TraceCallback<T> callback) {
        TraceStatus status = null;
        try {
            status = logTrace.begin(message);

            T result = callback.call();

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
//BepozController
//as is
@RestController
@RequiredArgsConstructor
public class BepozController {

    private final BepozService bepozService;
    private final LogTrace logTrace;

    @GetMapping("/request")
    public String request(String itemId) {
        AbstractTemplate<String> template = new AbstractTemplate<>(logTrace) {
            @Override
            protected String call() {
                bepozService.save(itemId);
                return "ok";
            }
        };

        return template.execute("BebozController.request()");
    }
}

//to be
@RestController
public class BepozController {

    private final BepozService bepozService;
    private final TraceTemplate template; //따로 빈 등록을 해서 사용해도 된다.

    public BepozController(BepozService bepozService, LogTrace logTrace) {
        this.bepozService = bepozService;
        this.template = new TraceTemplate(logTrace);
    }

    @GetMapping("/request")
    public String request(String itemId) {
        return template.execute("BepozController.request()", () -> {
            bepozService.save(itemId);
            return "ok";
        });
    }
}
```

이렇게 콜백과 템플릿 메서드 패턴을 사용해서 코드를 간결하게 해보았다.  
스프링에서는 이러한 방식의 전략 패턴을 **템플릿 콜백 패턴**이라고 한다.  
GOF 패턴은 아니고, 스프링 내부에서만 이렇게 부른다고 한다. 전략 패턴에서 템플릿과 콜백 부분이 강조된 패턴이라 생각하면 된다.  
스프링에서는 ``JdbcTemplate``, ``RestTemplate``, ``TransactionTemplate``, ``RedisTemplate`` 처럼 다양한 템플릿 콜백 패턴이 사용된다. 스프링에서 이름에 ``XxxTemplate``가 있다면 템플릿 콜백 패턴으로 만들어져 있다고 생각하면 된다.  

여기서 더 나아가 원본코드를 아예 건드리고 싶지 않다는 생각이 든다. 이를 위해 프록시 패턴을 사용하게 된다.  
2편에서 계속...  

[AOP에 대해 (1) - 현재](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(1).md)  
[AOP에 대해 (2)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(2).md)  
[AOP에 대해 (3)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(3).md)  
[AOP에 대해 (4)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(4).md)  
[AOP에 대해 (5)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(5).md)  
[AOP에 대해 (6)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(6).md)  
[AOP에 대해 (7)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  
[AOP에 대해 (8)](https://github.com/Be-poz/TIL/blob/master/Spring/aop/AOP%EC%97%90%20%EB%8C%80%ED%95%B4%20(7).md)  

---

### REFERENCE

[스프링 핵심원리 고급편 - 김영한](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8)

