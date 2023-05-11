# Atomic, Volatile, Synchronized 에 대해

멀티 쓰레드의 경우에 공유하는 필드에 대해서 thread-safe를 보장해주어야 한다.  
``public static int idx = 0;`` 이런식으로 두는 것은 thread-safe 하지않다.  

``Atomic``, ``Volatile``, ``Synchronized`` 에 대해 알아보자.  

```java
public class CounterBasic {

    private static int idx = 0;

    public static int increase() {
        return idx++;
    }

    public static int idx() {
        return idx;
    }
}

public class CounterSynchronized {

    private static int idx = 0;

    public static synchronized int increase() {
        return idx++;
    }

    public static int idx() {
        return idx;
    }
}

public class CounterVolatile {

    private static volatile int idx = 0;

    public static int increase() {
        return idx++;
    }

    public static int idx() {
        return idx;
    }
}

public class CounterAtomic {
    private static AtomicInteger idx = new AtomicInteger(0);

    public static int increase() {
        return CounterAtomic.idx.getAndIncrement();
    }

    public static int idx() {
        return idx.get();
    }
}
```

다음과 같은 코드를 두었고 이에 대한 테스트 코드를 작성해보았다.  

```java
class CounterTest {

    private static final int N_VALUE = 10000;

    @Test
    @DisplayName("기본 형태 Counter의 멀티스레드 테스트")
    public void basicCounterMultiThreadTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(200);
        CountDownLatch latch = new CountDownLatch(N_VALUE);

        for (int i = 0; i < N_VALUE; i++) {
            service.execute(() -> {
                CounterBasic.increase();
                latch.countDown();
            });
        }

        latch.await();
        assertThat(CounterBasic.idx()).isNotEqualTo(N_VALUE);
    }

    @Test
    @DisplayName("Volatile 형태 Counter의 멀티스레드 테스트")
    public void volatileMultiThreadTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(200);
        CountDownLatch latch = new CountDownLatch(N_VALUE);

        for (int i = 0; i < N_VALUE; i++) {
            service.execute(() -> {
                CounterVolatile.increase();
                latch.countDown();
            });
        }

        latch.await();
        assertThat(CounterVolatile.idx()).isNotEqualTo(N_VALUE);
    }

    @Test
    @DisplayName("Synchronized 형태 Counter의 멀티스레드 테스트")
    public void synchronizedMultiThreadTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(200);
        CountDownLatch latch = new CountDownLatch(N_VALUE);

        for (int i = 0; i < N_VALUE; i++) {
            service.execute(() -> {
                CounterSynchronized.increase();
                latch.countDown();
            });
        }

        latch.await();
        assertThat(CounterSynchronized.idx()).isEqualTo(N_VALUE);
    }
  
    @Test
    @DisplayName("AtomicInteger 형태 Counter의 멀티스레드 테스트")
    public void AtomicIntegerMultiThreadTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(200);
        CountDownLatch latch = new CountDownLatch(N_VALUE);

        for (int i = 0; i < N_VALUE; i++) {
            service.execute(() -> {
                CounterAtomic.increase();
                latch.countDown();
            });
        }

        latch.await();
        assertThat(CounterAtomic.idx()).isEqualTo(N_VALUE);
    }
}
```

위에서부터 살펴보자.  
단순히 static int를 이용하면 CounterBasic의 경우 assert문을 보면 thread-safe 하지 않게 돌아간 것을 확인할 수가 있다.        

<br/>

그렇다면 CounterVolatile은 어떨까? 값이 제대로 올라가지 않았다.  
volatile은 대체 어떤 키워드일까 ??  
volatile은 변수를 읽거나 쓸 때에 Main Memory에서 읽고 쓴다는 뜻을 가진 키워드다.  

![image](https://user-images.githubusercontent.com/45073750/133931024-06c01934-d950-431f-b541-659e33aafcfd.png)

![image](https://user-images.githubusercontent.com/45073750/133931115-875a0356-d167-4cbb-9294-6c05b9d0efdc.png)

쓰레드1에서 counter의 값을 올린다고 하더라도 Main Memory에 반영이 되지 않았기 때문에 쓰레드2에서 여전히 값이 0이다. 이런 상황에서 문제가 생길 것이다. 하지만, volatile 키워드를 붙이게되면 Main Memory에 읽고 쓰기 때문에 불일치에 대한 이슈를 방지할 수가 있다.  

하지만, volatile은 문제가 있다. 위의 테스트코드에서 알 수 있듯이 멀티쓰레드 환경에서 적절하지 않다.  
![image](https://user-images.githubusercontent.com/45073750/133931181-8cd96f87-e8a5-41f5-9eaf-e47892a646ed.png)

쓰레드1의 값이 반영되기 이전에 쓰레드2에서 값을 가져와서 연산을 하는 경우다. 위의 코드와 똑같은 상황이라고 볼 수 있다.  
이런 경우에 문제가 생긴다. 그러니 volatile은 멀티 쓰레드 환경에서 사용하면 안된다.(하나의 쓰레드만 write 하는 경우에는 괜찮다) 변수의 값 일치를 원할 때에 사용하면 된다. 그러나 Main Memory에서 가져오는 것은 CPU Cache에서 가져오는 것보다 비용이 크니 잘 생각해야 한다.  

<br/>

이번에는 CounterSynchronized를 볼 차례다. 의도한대로 값이 정상적으로 상승했다.  
Synchronized는 하나의 쓰레드만 접근할 수 있게끔 해주는 키워드다. 경쟁상태가 발생하지 않도록 한다는 것이다.  
Synchronized를 적용하는 방법은 여러 가지가 있다. 위 코드에서는 static 메서드에 걸어주었다.  

```java
public class CounterSynchronized {

    private static int idx = 0;

    public static synchronized int increase() {
        return idx++;
    }

    public static int idx() {
        return idx;
    }
}

/* 이런 식으로 메서드에 걸어주는 방법이 있다. 메서드에다가 걸어주게되면 해당 객체가 lock이 걸리게 된다.
메서드 말고 따로 synchronized block 단위로 걸어줄 수도 있다.
*/
   @Test
    @DisplayName("Synchronized 형태 Counter의 멀티스레드 테스트")
    public void synchronizedMultiThreadTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(200);
        CountDownLatch latch = new CountDownLatch(N_VALUE);

        for (int i = 0; i < N_VALUE; i++) {
            service.execute(() -> {
                synchronized(CounterBasic.class) {
                    CounterBasic.increase();
                }
//                CounterSynchronized.increase();
                latch.countDown();
            });
        }

        latch.await();
        assertThat(CounterBasic.idx()).isEqualTo(N_VALUE);
    }

/* 위의 테스트 코드를 synchronized block을 이용해서 CounterBasic을 thread-safe 하게 만들어보았다.
```

Synchronized는 간단하게는 동기화를 위한 키워드라고 생각을 하면된다.  
조금 더 흥미로운 예시는 다른 분의 [블로그 링크](https://tourspace.tistory.com/54)를 보면 되겠다.  

<br/>

마지막으로 대망의 CounterAtomic은 어떨까? assert문을 통해 제대로 값이 올라간 것을 확인할 수가 있었다.  
그럴 수 있었던 이유는 Atomic 클래스는 CAS(Compare And Swap)을 이용하기 때문이다.  
위의 코드에서 사용된 Atomic 메서드는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/133930723-873f2225-5cdb-45b9-adea-830d4a1af0df.png)

![image](https://user-images.githubusercontent.com/45073750/133930729-467b1111-79af-4660-be2c-b57457531648.png)

CAS(Compare And Swap)은 현재 쓰레드에 저장된 값과 Main Memory에 저장된 값을 비교해서 일치하는 경우 새로운 값으로 교체해주고, 일치 하지 않는다면 실패 후 다시 재시도를 하는 알고리즘을 의미한다.  

오잉? Synchronized 그냥 쓰면 되지않나 왜 굳이 이걸?  
=> Synchronized는 위에서 본 것과 같이 블락 전체를 lock 걸어주기 때문에 비용이 크다. 반면, CAS는 그렇지 않다.  
NonBlocking으로 처리할 수 있다는 장점 또한 있다. Atomic 짱이다~!  

Volatile과도 다르다. Volatile은 쓰는 연산을 하는 쓰레드는 2개 이상이면 안된다. 쉽게 말해, 읽기 연산에서만 사용된다고 보면 된다. 하지만, Atomic 클래스들은 여러 쓰레드에서 읽기/쓰기 연산을 할 수가 있다. CAS의 내부 구현에서 Main Memory의 값을 가져오는 용도로 volatile 키워드를 사용하기도 한다.  

<br/>

테스트 코드를 작성하면서 Collection에 대한 동기화의 필요성 또한 느꼈다.  
멀티쓰레드 환경에서는 Concurrent Collection 들을 꼭 활용하자~!  
(짧막상식  ConcurrentHashMap은 CAS와 synchronized 두 개의 방식 모두 사용하고있다)

***

### REFERENCE

http://tutorials.jenkov.com/java-concurrency/volatile.html  

https://tourspace.tistory.com/54  

https://chickenpaella.tistory.com/97  

https://javaplant.tistory.com/23