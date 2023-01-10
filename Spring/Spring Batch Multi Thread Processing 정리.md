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

