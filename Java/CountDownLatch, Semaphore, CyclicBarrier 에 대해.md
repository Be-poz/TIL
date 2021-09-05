# CountDownLatch, Semaphore, CyclicBarrier 에 대해

CountDownLatch, Semaphore, CyclicBarrier는 자바에서 제공해주는 동기화 클래스이다.  
이 클래스들을 이용해서 멀티 스레드와 관련된 코드들을 핸들링 할 수가 있다.  

## CountDownLatch

```java
    @Test
    void countDownLatchTest() throws InterruptedException {
        int numberOfThreads = 10;
        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService service = Executors.newFixedThreadPool(numberOfThreads);
        for (int i = 0; i < numberOfThreads; i++) {
            service.execute(() -> {
                latch.countDown();
            });
        }
        latch.await();
//        latch.await(10, TimeUnit.SECONDS);
        System.out.println("finish");
    }
```

CountDownLatch는 ``await()`` 메서드를 통해 코드의 진행을 멈춘다.  
스레드에서 원하는 횟수(위 코드에서는 10) 만큼 ``countDown()`` 이 호출되어야 코드가 마저 진행이 된다.  
위 코드에서 for문이 10번 미만으로 돈다면 ``latch.await()`` 에서 멈춰 빠져나가지 못한다.  
``await()`` 메서드는 주석처리된 코드에서 볼 수 있듯이 tiemOut을 따로 줄 수가 있다.  

이런 기능이 왜 필요할까?? 그 이유는 코드로 확인해보겠다.  

```java
    @Test
    void countDownLatchTest() throws InterruptedException {
        int numberOfThreads = 10;
        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService service = Executors.newFixedThreadPool(numberOfThreads);
        for (int i = 0; i < numberOfThreads; i++) {
            service.execute(() -> {
                System.out.println("thread start!!");
                System.out.println("thread end!!");
            });
        }
        System.out.println("finish");
    }
/*
thread start!!
thread start!!
thread end!!
thread start!!
...
finish
...
thread start!!
thread end!!
thread end!!
```

위의 코드 결과를 보면 알 수 있듯이 뒤죽박죽으로 나온다. print 사이에 다른 코드가 없어서 그렇지 있다면 더욱 뒤죽박죽이 될 것이다.  
결과를 보면 알 수 있듯이 비동기로 돌아가기 때문이다. 코드를 살짝 변형해보겠다.  

```java
    static class MyCounter {

        private int count;

        public void increment() {
            int temp = count;
            count = temp + 1;
        }

        public int getCount() {
            return count;
        }
    }

    @Test
    void countDownLatchTest() throws InterruptedException {
        int numberOfThreads = 10;
        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService service = Executors.newFixedThreadPool(numberOfThreads);
        MyCounter counter = new MyCounter();
        for (int i = 0; i < numberOfThreads; i++) {
            service.execute(() -> {
                counter.increment();
            });
        }
        assertThat(counter.getCount()).isEqualTo(numberOfThreads);
    }
```

위의 테스트 코드는 깨지게 된다. 비동기이기 때문에 ``counter.increment()`` 가 10번이 다 호출되기도 전에 끝나버리게 되는 것이다.  
이 메서드 앞에 다른 메서드들이 있다면 더더욱 문제가 될 것이다.  

```java
    @Test
    void countDownLatchTest() throws InterruptedException {
        int numberOfThreads = 10;
        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService service = Executors.newFixedThreadPool(numberOfThreads);
        MyCounter counter = new MyCounter();
        for (int i = 0; i < numberOfThreads; i++) {
            service.execute(() -> {
                counter.increment();
                latch.countDown();
            });
        }
        latch.await();
        assertThat(counter.getCount()).isEqualTo(numberOfThreads);
    }
```

이런 경우에 이제 CountDownLatch 를 이용해서 확실하게 모든 쓰레드가 ``counter.increment()`` 를 호출을 하는 것을 보장받을 수가 있게된다. 이 코드에서는 쓰레드 바깥에서 조절을 했지만 쓰레드 내에서 필요에 따라 호출을 해서 적절하게 사용할 수 있을 것이다.  

<br/>

## Semaphore

```java
    @Test
    void semaphoreTest() throws InterruptedException {
        Semaphore semaphore = new Semaphore(10);
        for (int i = 0; i < 10; i++) {
            semaphore.acquire();
        }

        semaphore.acquire();
    }
```

Semaphore 는 ``acquire()`` 와 ``release()`` 메서드를 주로 사용한다.  
CountDownLatch가 했듯이 수량을 정해둔다. 해당 값 횟수까지는 ``acquire()`` 이 호출되어도 상관없지만 이후에는 대기상태에 들어가게 된다. 위 코드는 for 문 이후에 추가로 호출된 ``semaphore.acquire()`` 로 인해 대기상태에 들어가게 된다.  

```java
    @Test
    void semaphoreTest() throws InterruptedException {
        Semaphore semaphore = new Semaphore(10);
        for (int i = 0; i < 10; i++) {
            semaphore.acquire();
        }
        assertThat(semaphore.tryAcquire()).isFalse();
        assertThat(semaphore.tryAcquire(0)).isTrue();
        assertThat(semaphore.tryAcquire(5)).isFalse();
        semaphore.release(10);
        assertThat(semaphore.tryAcquire(10)).isTrue();
    }
```

``tryAcquire()`` 메서드는 남은 횟수가 있는지에 대한 boolean 결과를 리턴하는 메서드다. 파라미터가 없을 때에는 1 이상 남았는지를 체크한다. 위의 코드에서 볼 수 있듯이 ``tryAcquire(0)`` 을 체크했을 때에는 true를 리턴한 것을 확인할 수가 있다.  

``release()`` 메서드는 시도 횟수를 늘리는 메서드다. default 증가 횟수는 1이며, ``release(10)`` 은 10번의 횟수를 늘려준 것을 확인할 수가 있다.  

Semaphore 를 이용해서 컬렉션의 사이즈를 제한한다던지 응용이 가능하다.  
(List 를 두고 add 메서드에서 ``acquire()``를 호출하게끔 remove에서 ``release()`` 를 호출하게끔 해서 특정 사이즈 이상 늘어날 수 없도록 조절)  

<br/>

## CyclicBarrier

CyclicBarrier는 CountDownLatch와 광장히 유사하다. 다른점은 CountDownLatch는 각 스레드가 ``countDown()`` 호출 후에 ``await()`` 을 만나지 않는다면 대기 상태에 빠지지 않고 할 일을 하지만, CyclicBarrier는 모든 스레드들이 대기상태에 빠지게 된다는 것이다.  

```java
    @Test
    void cyclicBarrierTest() throws InterruptedException {
        CyclicBarrier barrier = new CyclicBarrier(5);
        ExecutorService service = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 4; i++) {
            service.execute(() -> {
                try {
                    barrier.await();
                    System.out.println("bepoz");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            });
        }
        Thread.sleep(100);
        assertThat(barrier.getNumberWaiting()).isEqualTo(4);
    }
```

``barriergetNumberWaiting()`` 메서드는 메서드명에서 유추할 수 있듯이 현재 대기중인 스레드의 수를 반환해준다.  
``CyclicBarrier barrier = new CyclicBarrier(5);`` 다음과 같이 선언했기 때문에 5 개의 스레드가 대기상태에 들어가야 그제서야 대기가 끝나게 된다. CountDownLatch는 단순히 ``countDown()`` 메서드만 호출하고 ``await()`` 가 호출되지 않는 이상 본인의 할 일을 할 수 있었던 것과 달리 CyclicBarrier는 모두가 대기한다.  (``Thread.sleep(100)`` 은 ``await()`` 이 호출되기도 전에 assert 문으로 넘어가버려서 추가함)  

for 문의 반복 횟수를 5로 바꿔주면 정상적으로 그 다음 로직들을 실행하게된다.  
또 차이점이 하나 더 있다.  

```java
    @Test
    void cyclicBarrierTest() throws InterruptedException, BrokenBarrierException {
        CyclicBarrier barrier = new CyclicBarrier(5);
        ExecutorService service = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5 ; i++) {
            service.execute(() -> {
                try {
                    barrier.await();
                    System.out.println("bepoz");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            });
        }
        Thread.sleep(100);
        System.out.println("a");
        barrier.await();
        System.out.println("b");
    }
```

CountDownLatch는 한 번 Count를 모두 소진시키면 ``await()`` 만나도 그냥 통과하는 반면,  
CyclicBarrier는 다시 시작된다. 즉, 위 코드에서 ``a`` 출력이 되고나서 ``await()`` 를 만나고 또 다시 5개의 스레드가 ``await()`` 을 호출할 때 까지 대기상태에 머무르게 된다.  

***

### REFERENCE

https://www.baeldung.com/java-cyclicbarrier-countdownlatch  

https://multifrontgarden.tistory.com/266