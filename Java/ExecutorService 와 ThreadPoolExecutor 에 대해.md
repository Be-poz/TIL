# ExecutorService 와 ThreadPoolExecutor 에 대해

``ExecutorService service = Executors.newFixedThreadPool(50);``  

위 코드가 뜻하는 것은 무엇일까?  

<br/>

``Executors`` 는 ``Executor``, ``ExecutorService``, ``ScheduledExecutorService``, ``ThreadFactory`` 등을 위한 정적 팩토리 메서드를 지원해주는 클래스다. 내부 메서드를 확인해보면 다음과 같은 팩토리 메서드가 눈에 들어올 것이다.  

```java
Executors.newSingleThreadExecutor();
Executors.newFixedThreadPool();
Executors.newCachedThreadPool();
Executors.newWorkStealingPool();
```

이 메서드들에 대한 차이는 추후 설명하고 가장 흔하게 사용하는 ``newFixedThreadPool()`` 메서드의 내부구조를 살펴보자.  

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
  this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
       Executors.defaultThreadFactory(), defaultHandler);
}
```

내부에서 ``ThreadPoolExecutor`` 클래스의 생성자를 호출하고 있었다. 이 클래스는 스레드 풀을 관리해주는 클래스이다.  
파라미터로 5개의 값을 갖고 있는데 이것들은 다음과 같은 역할을 한다.  

* corePoollSize : 풀 사이즈를 뜻하며, 최초 생성되는 스레드의 사이즈다.  
* maximumPoolSize : 풀에 최대로 유지될 수 있는 스레드의 개수다.
* keepAliveTime : corePoolSize 보다 스레드가 많아져 maximumPoolSize의 값까지 스레드가 생성이 되는데 keepAliveTime 만큼 유지가 되었다가 corePoolSize로 돌아오게된다.
* unit : 시간 단위
* workQueue : corePoolSize가 꽉 찼을 경우 스레드를 담아두는 블록킹 큐  

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```

이제 다른 메서드들을 살펴보자. 
 ``newSingleThreadExecutor()`` 는 corePoolSize와 maxPoolSize가 1인 ``ThreadPoolExecutor``였다.  
``newCachedThreadPool()`` 는 corePoolSize가 0, maxPoolSize가 엄청나게 큰 계속해서 늘어나는 ``Executor`` 다.  
``newWorkStealingPool()`` 는 CPU의 사용가능한 코어의 개수에 대응하는 풀 사이즈를 설정해준다.  

<br/>

이제 ``ExecutorService`` 에 대해 알아보자.  
![image](https://user-images.githubusercontent.com/45073750/132217794-104e4725-26f2-4f4c-989f-9cc9cac36d21.png)

``ExecutorService`` 는 비동기로 실행된다.  
Task를 위임하는 메서드로는 다음과 같이 존재한다.  

* execute(Runnable)
* submit(Runnable)
* submit(Callable)
* invokeAny(...)
* invokeAll(...)

하나씩 살펴보겠다.  

```java
    @Test
    void executeTest() {
        ExecutorService service = Executors.newFixedThreadPool(1);
        service.execute(newRunnable("task"));
        service.shutdown();
    }

    private static Runnable newRunnable(String task) {
        return new Runnable() {
            @Override
            public void run() {
                System.out.println(task);
            }
        };
    }
```

``execute()`` 는 ``Runnable`` 을 받아 비동기로 실행해준다. 결과에 대한 반환을 받는다거나 할 수 없다.  

```java
    @Test
    void submitTest() {
        ExecutorService service = Executors.newFixedThreadPool(5);
        Future future1 = service.submit(newRunnable("submit runnable task"));
        Future future2 = service.submit(newCallable("submit callable task"));
        //future2.cancel();
        System.out.println(future2.isDone());
			  assertThat((String) future.get()).isEqualTo("submit callable task");
        service.shutdown();
    }

    private static Callable newCallable(String task) {
        return new Callable() {
            @Override
            public Object call() throws Exception {
                return task;
            }
        };
    }

    private static Runnable newRunnable(String task) {
        return new Runnable() {
            @Override
            public void run() {
                System.out.println(task);
            }
        };
    }
```

submit은 ``Future`` 이라는 타입으로 결과를 반환받을 수 있다.  
``Future`` 는 ``isDone()`` 메서드를 통해 해당 task가 완료되었는지를 확인할 수가 있다. 위 코드에서는 아마 바로 호출이 되었기 때문에 매우 높은 확률로 ``false`` 일 것이다. ``get()`` 메서드는 ``Callable`` 에 대한 결과를 받아올 수가 있다.  
 ``Runnable`` 타입을 인수로 주었을 경우에는 리턴 타입이 ``Runnable`` 의 리턴 타입이 ``void`` 이기 때문에 ``null`` 을 리턴하게된다.  
``cancel()`` 메서드를 이용해서 작업이 이루어지지 않았더라면 작업을 취소할 수도 있다.  

``submit`` 과 ``execute`` 모두 ``Runnable`` 를 실행시킬 수 있다. ``submit`` 은 위에서 설명해서 알다시피 ``Future`` 를 반환한다.  
이것을 이용해서 예외처리를 해줄 수 있다는 차이점이 있다.

```java
    @Test
    void invokeAnyTest() throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);

        List<Callable<String>> callables = new ArrayList<>();
        callables.add(newCallable("task1"));
        callables.add(newCallable("task2"));
        callables.add(newCallable("task3"));

        String result = service.invokeAny(callables);
        System.out.println(result);
        service.shutdown();
    }
```

``<T> T invokeAny(Collection<? extends Callable<T>> tasks)`` 메서드는 ``Callable`` 컬렉션을 넘기고 그 중에서 가장 먼저 처리된 아무 반환 값을 뽑아온다. 이것이 왜 필요할까 의문을 품을 수 있다.  

만약 똑같은 결과를 반환하는 여러 서버한테 모두 호출을 해서 가장 빠른 결과값을 가지고 일을 처리하고 싶을 때에 사용할 수가 있다.  

```java
    @Test
    void invokeAllTest() throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);

        List<Callable<String>> callables = new ArrayList<>();
        callables.add(newCallable("task1"));
        callables.add(newCallable("task2"));
        callables.add(newCallable("task3"));

        List<Future<String>> futures = service.invokeAll(callables);
        service.shutdown();
    }
```

``invokeAll`` 메서드는 예상할 수 있듯이 모든 ``Callable`` 에 대한 ``Future`` 리스트를 받는 메서드이다.  

```java
ExecutorService service = Executors.newFixedThreadPool(5);
service.shutDown();
List<Runnable> runnables = service.shutdownNow();
```

``shutDown()`` 은 더 이상 task 들을 받지 않지만, 기존에 들어있는 작업들은 모두 끝마치고 종료된다.  
``shutDownNow()`` 는 곧장 종료된다. 아직 실행되지 않은 작업들에 대해서 List 형식으로 반환을 한다. 현재 실행중인 작업은 바로 끝마칠 수도 아니면 끝까지 돌아갈 수도 있다.  

``ExecutorService`` 와 ``ThreadPoolExecutor`` 에 대해 알아보았다.

***

### REFERENCE

http://tutorials.jenkov.com/java-util-concurrent/executorservice.html  

https://www.youtube.com/watch?v=Nb85yJ1fPXM&list=PLL8woMHwr36EDxjUoCzboZjedsnhLP1j4&index=13&ab_channel=JakobJenkov  

https://www.youtube.com/watch?v=MB_qCXBSgK0&list=PLL8woMHwr36EDxjUoCzboZjedsnhLP1j4&index=14&ab_channel=JakobJenkov  

http://wonwoo.ml/index.php/post/2254  

https://emong.tistory.com/221  

https://stackoverflow.com/questions/4016091/what-is-the-difference-between-submit-and-execute-method-with-threadpoolexecutor