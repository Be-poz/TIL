# ThreadLocal에 대해

ThreadLocal은 무엇일까?  
코드로 먼저 확인해보자   

```java
public class Shared {

    private String nonDuplicatableId;

    public void responseYourId() {
        allocate();
        System.out.println(nonDuplicatableId);
    }

    private void allocate() {
        if (nonDuplicatableId == null) {
            nonDuplicatableId = UUID.randomUUID().toString().substring(0, 6);
        }
    }
}
```

```java
class SharedTest {

    private static final Shared shared = new Shared();
    private static final int COUNT = 2;

    @Test
    public void sharedTest() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(COUNT);
        ExecutorService executorService = Executors.newFixedThreadPool(COUNT);

        for (int i = 0; i < COUNT; i++) {
            executorService.execute(shared::responseYourId);
            latch.countDown();
        }
        latch.await();
    }
}
```

위 테스트코드의 결과는 어떻게 나올까?  
``Shared shared = new Shraed();`` 로 선언을 해두고 2개의 쓰레드에서 ``responseYourId``를 호출했다.  
``allocate()`` 메서드 내부로직이 null일 때에 ``String nonDuplicatableId``에 값을 할당하기 때문에  
똑같은 필드 변수가 print 될 것이다.  

여러 쓰레드에서 접근하는 클래스에 상태를 두는 경우에는 동시성 문제를 항상 염두해서 코드를 작성해야한다.  
가령, ``int count = 0`` 변수가 존재하는데 여러 쓰레드에서 이 값을 변경하고 return 받는 경우에 문제가 발생할 것이다.  
위 코드의 예시와는 조금 동떨어지는 얘기긴하지만 어쨋든 동시성 이슈를 고려해야 한다.  

그런데, 위 코드에서 각 쓰레드마다 각자 본인만의 값을 갖고 싶다면 어떻게 해야할까??  
쉽게 말해서, 각 쓰레드마다 ``nonDuplicatableId``의 값이 달랐으면 좋겠다는 것이다.  
하지만 단순히 String 필드로 운용하게 되면 동시성 문제를 마주하게 될 것이다.  

이때 사용하는 것이 바로 ``ThreadLocal`` 이다. 쓰레드별로 지역 변수를 할당하게끔 해준다.  
코드를 변경해보겠다.  

```java
public class Shared {

    private final ThreadLocal<String> nonDuplicatableId = new ThreadLocal<>();

    public void responseYourId() {
        allocate();
        System.out.println(nonDuplicatableId.get());
    }

    private void allocate() {
        if (nonDuplicatableId.get() == null) {
            nonDuplicatableId.set(UUID.randomUUID().toString().substring(0, 6));
        }
    }
}
```

위와 같이 변경했다. ``ThreadLocal<T>`` 의 사용법은 간단하다.  
``new ThreadLocal<>();``을 선언하고 ``get()``을 이용해 값을 꺼내오고, ``set()``을 이용해 값을 넣고, ``remove()``를 이용해 제거한다.  

**유의해야 할 점**은 사용을 다 한 후에는 무조건 ``remove()``를 이용해서 제거해주어야 한다는 것이다.  
그 이유는 쓰레드 별로 값을 할당하기 때문에 쓰레드 풀을 사용할 시에 요청 사용자가 변경되어도 동일 쓰레드를 이용하면서 이전에 저장된 값을 사용하게 되는 경우가 발생하기 때문이다.  

<br/>

3줄 요약  

1. 쓰레드 별로 변수를 관리하고 싶을 때에 ``ThreadLocal``을 사용
2. ``get()``, ``set()``, ``remove()``를 이용해서 사용
3. ``remove()``를 호출하지 않으면 쓰레드 풀을 사용할 때에, 동일한 쓰레드를 이용하는 별개의 사용자한테서 문제가 발생할 수 있음

---

### REFERENCE

https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8