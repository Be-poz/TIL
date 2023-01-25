# Spring Batch Multi Thread Processing 정리

## AsyncItemProcessor, AsyncItemWriter

* Step 안에서 ``ItemProcessor``가 비동기적으로 동작하는 구조
* ``AsyncItemProcessor``와 ``AsyncItemWriter``가 함께 구성이 되어야 함
* ``AsyncItemProcessor``로 부터 ``AsyncItemWriter``가 받는 최종 결과값은 ``List<Future<T>>`` 타입이며 비동기 실행이 완료될 때까지 대기한다
* spring-batch-integration 의존성 필요

<img width="1405" alt="image" src="https://user-images.githubusercontent.com/45073750/210568276-8663c5b7-9386-43a3-9bd6-ddda06bdab00.png">

<img width="1942" alt="image" src="https://user-images.githubusercontent.com/45073750/211556350-9f01ad20-6842-4b12-aa10-f1f89b24bc05.png">



```java
public Step step() throws Exception {
  return stepBuilderFactory.get("step")
    									.chunk(100)
    									.reader(pagingItemReader())	//비동기 아님
    									.processor(asyncItemProcessor())
    /*
    쓰레드 풀 개수만큼 쓰레드 생성되어 비동기로 실행,
    내부적으로 실제 ItemProcessor에게 실행 위임하고 결과를 Future에 저장
    */
    									.writer(asyncItemWriter() // 비동기 실행 결과 값들을 모두 받아오기까지 대기함. 내부적으로 ItemWriter실행
    									.build()
}
```

```java
@Bean
public Step asyncStep1() throws Exception {
    return stepBuilderFactory.get("asyncStep1")
            .<Csutomer, Customer> chunk(100)
            .reader(pagingItemReader())
            .processor(asyncItemProcessor())
            .writer(asyncItemWriter())
            .build();
}

@Bean
public AsyncItemProcessor<? super Customer, ? extends Customer> asyncItemProcessor() {
    AsyncItemProcessor<Customer, Customer> asyncItemProcessor = new AsyncItemProcessor<Customer, Customer>();
    asyncItemProcessor.setDelegate(customItemProcessor());
    asyncItemProcessor.setTaskExecutor(new SimpleAsyncTaskExecutor());

    return asyncItemProcessor;
}

@Bean
public ItemProcessor<Customer, Customer> customItemProcessor() throws InterruptedException {
    return new ItemProcessor<Customer, Customer>() {
        @Override
        public Customer process(Customer item) throws Exception {
            return new Customer(item.getId(), item.getFirstName().toUpperCase(), item.getLastName().toUpperCase(), item.getBirthdate());
        }
    };
}

@Bean
public AsyncItemWriter asyncItemWriter() {
    AsyncItemWriter<Customer> asyncItemWriter = new AsyncItemWriter<>();
    asyncItemWriter.setDelegate(customItemWriter());

    return asyncItemWriter;
}
```

<br/>

## Multi-threaded Step

* Step 내에서 멀티 스레드로 Chunk 기반 처리가 이루어지는 구조
* ``TaskExecutorRepeatTemplate`` 이 반복자로 사용되며 설정한 개수(throttleLimit) 만큼의 스레드를 생성하여 수행한다

<img width="1721" alt="image" src="https://user-images.githubusercontent.com/45073750/211808692-6de956cc-9f94-46a4-82d6-496ace064d53.png">

<img width="1850" alt="image" src="https://user-images.githubusercontent.com/45073750/211813819-870b2729-ab27-45c8-ae85-e1ac2acf4fcf.png">

```java
@Bean
public Step step() throws Exception {
    return stepBuilderFactory.get("step1")
            .<Customer, Customer> chunk(100)
            .reader(pagingItemReader())
            .processor((ItemProcessor<? super Customer, ? extends Customer>) item -> item)
            .writer(customItemWriter())
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(4);
    taskExecutor.setMaxPoolSize(8);
    taskExecutor.setThreadNamePrefix("async-thread");

    return taskExecutor;
}
```

``taskExecutor``를 지정하지않으면 default로는 ``SyncTaskExecutor``가 지정이 된다. 위 코드에서는 ``ThreadPoolTaskExecutor``를 지정해서 청크별로 비동기적으로 동작하게끔 만들었다. 이때, ``ItemReader``는 동기화가 보장되게끔 만들어야 한다. ``JdbcPagingItemReader``나 ``JpaPagingItemReader``를 사용하면 이를 보장해준다.  

<br/>

## Parallel Steps

* SplitState 를 사용해서 여러 개의 Flow 들을 병렬적으로 실행하는 구조
* 실행히 다 완료된 후 FlowExecutionStatus 결과들을 취합해서 다음 단계 결정을 한다

<img width="1852" alt="image" src="https://user-images.githubusercontent.com/45073750/214572985-4bbe5071-5849-4778-9e79-0251b32de5af.png">

```java
public Job job() {
  	return jobBuilderFactory.get("job")
      								.start(flow1())		//flow1 생성
      								.split(TaskExecutor).add(flow2(), flow3()) 
      								//flow2, flow3 생성 후 합침, taskExecutor에서 flow 개수만큼 스레드 생성하여 실행
      								.next(flow4())
      								//위의 split이 완료된 후 실행
      								.end()
      								.build();
}
```

실제 코드로 살펴보자

```java
public Job job() throws RetryableException {
    return jobBuilderFactory.get("Job")
                            .incrementer(new RunIdIncrementer())
                            .start(flow1())
                            .split(taskExecutor()).add(flow2())
                            .end()
                            .build();
}

@Bean
public Flow flow1() {
    TaskletStep step1 = stepBuilderFactory.get("step1")
                                         .tasklet(tasklet())
                                         .build();

    return new FlowBuilder<Flow>("flow1")
            .start(step1)
            .build();
}

@Bean
public Flow flow2() {
    TaskletStep step2 = stepBuilderFactory.get("step2")
                                         .tasklet(tasklet())
                                         .build();

    TaskletStep step3 = stepBuilderFactory.get("step3")
                                         .tasklet(tasklet())
                                         .build();

    return new FlowBuilder<Flow>("flow2")
            .start(step2)
            .next(step3)
            .build();
}

@Bean
public Tasklet tasklet() {
    return new CustomTasklet();
}

public class CustomTasklet implements Tasklet {

    private long sum;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        for (int i = 0; i < 100000000; i++) {
            sum++;
        }
        System.out.printf("%s has been executed on thread %s\n", chunkContext.getStepContext().getStepName(), Thread.currentThread().getName());
        System.out.println("sum: " + sum);
        return RepeatStatus.FINISHED;
    }
}
```

결과는 다음과 같이 나왔다.

<img width="1061" alt="image" src="https://user-images.githubusercontent.com/45073750/214584493-f56f498b-8a52-4e63-a2ae-80d3ba73234b.png">

flow1과 flow2가 병렬로 돌아가는 것을 확인할 수 있고, flow2 내부에서 step2가 끝난 후에 step3를 실행하게끔 하였기 때문에 step2가 끝난 후에 step3이 돌아간 것을 확인할 수가 있었다.  

sum값이 1억, 2억, 3억 이렇게 찍혀야 되는데 이상한 값으로 찍힌 이유는 ``CustomTasklet``을 빈 등록하여서 싱글턴 객체가 되었는데 내부 공유 변수가 있어 데이터 동기화 문제가 일어난 것이다. 빈 등록을 해제해주어 싱글턴으로 사용을 하지 말던가 아니면 동기화 처리를 해주면 된다.  

```java
public class CustomTasklet implements Tasklet {

    private long sum;
    private Object object = new Object();

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        synchronized (object){
            for (int i = 0; i < 100000000; i++) {
                sum++;
            }
            System.out.printf("%s has been executed on thread %s\n", chunkContext.getStepContext().getStepName(), Thread.currentThread().getName());
            System.out.println("sum: " + sum);
        }
        return RepeatStatus.FINISHED;
    }
}
```

간단히 동기화 처리를 해주면 정상적으로 sum이 찍히는 것을 확인할 수가 있다. 하지만, ``synchronized``를 사용함으로써 처리 속도가 그만큼 떨어지게 된다.  

```java
@Bean
public Job job() throws RetryableException {
    return jobBuilderFactory.get("Job")
                            .incrementer(new RunIdIncrementer())
                            .start(flow1())
                            .split(taskExecutor()).add(flow2())
                            .next(flow2())
                            .end()
                            .build();
}
```

이번에는 split이후에 next를 하나 더 넣어서  flow2를 다시 실행해보겠다.  

<img width="1741" alt="image" src="https://user-images.githubusercontent.com/45073750/214585558-1a35b79e-823d-4990-9335-644bcedde409.png">

쓰레드명을 살펴보면 next 단계에서 실행된 flow2는 메인 쓰레드로 처리된 것을 확인할 수가 있다. split 까지 병렬처리를 하다가 병렬 처리하던 쓰레드를 모두 완료한 후에 next로 넘어갔고 여기서 부터는 동기로 이루어지니 메인 쓰레드가 처리한 것으로 유추할 수가 있다.  

<br/>

## Partitioning

