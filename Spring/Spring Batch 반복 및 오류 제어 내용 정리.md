# Spring Batch 반복 및 오류 제어 내용 정리

## Repeat

* Spring Batch는 특정 조건이 충족 될 때 까지(또는 반대) Job 또는 Step을 반복하도록 배치 어플리케이션을 구성 할 수 있다.
* Step의 반복과 Chunk 반복을 RepeatOperation을 사용해서 처리하고 있다.
* 기본 구현체로 RepeatTemplate을 제공한다.

``Job`` -> ``Step`` -> ( ``RepeatTemplate`` -> ``Tasklet`` -> ( ``RepeatTemplate`` -> ``Chunk``) 반복  )반복  

Tasklet은 이제 Chunk 사용 시에 ``ChunkOrientedTasklet`` 를 사용하게 되는데 이곳의 ``ChunkProvider`` 가 ``RepeatTemplate`` 을 가지고 ``ItemReader`` 에서 데이터를 읽어오는 것을 반복한다.  

반복을 종료할 것인지 여부를 결정하는 세 가지 항목  

* RepeatStatus
  * 스프링 배치의 처리가 끝났는지 판별하기 위한 열거형
    * CONTINUABLE: 작업이 남아 있음
    * FINISHED: 더 이상의 반복 없음
* CompletionPolicy
  * RepeatTemplate의 iterate 메서드 안에서 반복을 중단할지 결정
  * 실행 횟수 또는 완료시기, 오류 발생 시 수행 할 작업에 대한 반복여부 결정
  * 정상 종료를 알리는데 사용된다
* ExceptionHandler
  * RepeatCallback 안에서 예외가 발생하면 RepeatTemplate가 ExceptionHandler를 참조해서 예외를 다시 던질지 여부 결정
  * 예외를 받아서 다시 던지게 되면 반복 종료
  * 비정상 종료를 알리는데 사용된다

``RepeatOperation`` <--- ``RepeatTemplate``->``RepeatCallback`` -> ``RepeatStatus`` & ``RepeatContext``  

``RepeatOperation`` 인터페이스는 ``RepeatStatus iterate(RepeatCallback callback)`` 을 가지고 있다.  
이를 ``RepeatTemplate`` 이 구현하고, ``RepeatTemplate`` 에서 ``RepeatCallback`` 이 
 ``RepeatStatus doInIteration(Repeatcontext context)`` 를 호출하게 된다.  

![image](https://user-images.githubusercontent.com/45073750/187495341-ff538615-cec3-4f74-8d42-bf4a7305c9f3.png)
![image](https://user-images.githubusercontent.com/45073750/187495376-9887ceac-2765-43de-ad8b-83461e8b9ffa.png)  

메서드를 타고타고 밑의 메서드에서 ``doinIteration`` 을 호출 

![image](https://user-images.githubusercontent.com/45073750/187495415-3528775f-0fc2-4068-9bba-467f848552eb.png)

결국, ``Step`` -> ``RepeatTemplate`` -> ( ``RepeatCallback`` --(``doInIteration()``)--> ``tasklet`` ) 반복  
``tasklet`` 에서 예외가 발생했다면 ``ExceptionHandler`` 를 통해 예외 정책에 따라 반복여부를 결정할 수 있다.  
예외가 발생하지 않았다면 ``completionPolicy`` 를 참조해서 종료 정책에 따라 반복 여부를 결정할 수 있게된다.  
만약 종료하지 않는다면 ``RepeatStatus`` 를 확인하게된다. 만약 ``FINISHED`` 면 반복문이 종료된다.  
여기서 ``FINISHED`` 가 아니면 반복문이 유지하게 된다.  

<br/>

``CompletionPolicy`` 에는 2개의 시그니처가 있다.  

* boolean isComplete(RepeatContext context, RepeatStatus result)
  * 콜백의 최종 상태 결과를 참조하여 배치가 완료되었는지 확인
* boolean isComplete(RepeatContext context)
  * 콜백이 완료 될 때 까지 기다릴 필요없이 완료되었는지 확인

![image](https://user-images.githubusercontent.com/45073750/187497156-7196e412-cb88-4f48-a7f2-96656e75cab8.png)

스프링 배치에서 기본적으로 지원되는 ``CompletionPolicy``는 위와 같은데  간단히 살펴보자면,  

* ``TimeoutTerminationPolicy``: 반복시점부터 현재시점까지 소요된 시간이 설정된 시간보다 크면 반복종료
* ``CountingCompletionPolicy``: 일정한 카운트를 계산 및 집계해서 카운트 제한 조건이 만족하면 반복종료
* ``SimpleCompletionPolicy``: default 값, 현재 반복 횟수가 Chunk 개수보다 크면 반복종료

<br/>

``ExceptionHandler``  

* void handleException(RepeatContext context, Throwable throwable) throws Throwable
  * 예외를 다시 던지기위한 전략을 허용하는 핸들러

![image](https://user-images.githubusercontent.com/45073750/187498428-0debf4f2-810e-4983-8c76-7ca76a7f399a.png)

스프링 배치에서 기본적으로 지원되는 ``ExceptionHandler``는 위와 같은데  간단히 살펴보자면,  

* ``LogOrrethrowExceptionHandler``: 예외를 로그로 기록할지 아니면 다시 던질 지 결정
* ``RethrownOnThresholdExceptionHandler``: 지정된 유형의 예외가 임계 값에 도달하면 다시 발생
* ``SimpleLimitExceptionHandler``: default, 예외 타입 중 하나가 발견되면 카운트가 증가하고 한계가 초과되었는지 여부를 확인하고 Throwable를 다시 던짐

<br/>

이제 코드 흐름을 따라가 보겠다.  
앞서 언급한 ``Job`` -> ``Step`` -> ( ``RepeatTemplate`` -> ``Tasklet`` -> ( ``RepeatTemplate`` -> ``Chunk``) 반복  )반복을 기억하자.  
그리고 또한  ``TaskletStep`` -> ``ChunkOrientedTasklet`` -> ``ChunkProvider`` -> ``ItemReader`` -> ``ChunkOrientedTasklet`` -> ``ChunkProcessor`` -> ``ItemProcessor`` -> ``ChunkProcessor`` -> ``ItemWriter`` 의 흐름 또한 기억해두자.  

위 2개의 흐름을 합친 최종 정리는 마지막에 정리하겠다.  

먼저 job과 step은 다음과 같다.  

<img width="1150" alt="image" src="https://user-images.githubusercontent.com/45073750/188685262-86d41714-680a-4491-a89c-471e746f9108.png">

이제 배치를 돌려보겠다.  

<img width="899" alt="image" src="https://user-images.githubusercontent.com/45073750/188685827-7bc85d1c-6a81-4992-93db-d52fc9a5c304.png">

``TaskletStep`` 의 ``doExecute`` 에 들어왔다. 흐름상 ``Step`` 에서는 ``RepeatTemplate`` 을 반복시킨다.  
위 코드의 ``stepOperation`` 이 바로 ``RepeatTemplate`` 이고 callback을 정의해둔 것이다.  

<img width="652" alt="image" src="https://user-images.githubusercontent.com/45073750/188686658-53f01fa0-7db5-4017-a493-0db90e8ccf11.png">

``RepeatTemplate`` 은 ``RepeatOperation`` 의 구현체인 것을 기억하고 있을 것이라고 생각한다.  

<img width="701" alt="image" src="https://user-images.githubusercontent.com/45073750/188687003-923a20be-ba93-46e5-a45a-47397f87c778.png">

``TaskletStep`` 에서 호출한 ``RepeatTemplate`` 의 ``iterate`` 메서드에 들어왔다.  
``result = executeInternal(callback);``  에 들어가게되고,  

<img width="761" alt="image" src="https://user-images.githubusercontent.com/45073750/188687331-0f4a2f9f-90ec-493d-b00b-bf52ac6bb678.png">

``executeInternal`` 메서드에서 반복문을 돌게된다. 이후 ``getNextResult`` 를 호출한다.  

<img width="914" alt="image" src="https://user-images.githubusercontent.com/45073750/188687618-c1828541-4a11-4101-8c70-efece7cf2116.png">

이곳에서 callback 의 ``doInIteration`` 을 호출하게되고 이는 곧 ``TaskletStep`` 에서 정의해준 ``StepContextRepeatCallback`` 의 ``doInIteration`` 메서드를 호출하게 된다(2번째 코드 사진에 나와있는 콜백 클래스). 그리고 ``doInIteration`` 에서 오버라이딩 해준 메서드인 ``doInChunkContext`` 를 호출하고 이곳에서 ``TransactionTemplate`` 호출 그리고 ``ChunkOrientedTasklet`` 의 ``execute`` 를 호출하게 된다. (``doInIteration``, ``doInChunkContext``, ``TransactionTemplate`` 의 흐름사진은 생략)  

<img width="957" alt="image" src="https://user-images.githubusercontent.com/45073750/188689810-9bfe9cf8-cb02-406a-b882-103854392015.png">

이제 ``ChunkProvider`` 의 ``provide`` 를 호출한다.

<img width="807" alt="image" src="https://user-images.githubusercontent.com/45073750/188691011-9d0e49dd-ebcb-41bf-99b1-c40b60e77612.png">

이곳에서 부터 ``RepeatTemplate`` 의 반복이 또 다시 이루어지게 된다.  

<img width="869" alt="image" src="https://user-images.githubusercontent.com/45073750/188691607-35dae139-ef32-4d54-a468-2ff8b6ccf42c.png">

동일한 흐름을 쭉 타면서 ``ChunkProvider`` 의 ``provide`` 메서드에 오버라이딩한 메서드 ``doInIteration`` 을 호출하게되고,  

<img width="807" alt="image" src="https://user-images.githubusercontent.com/45073750/188691011-9d0e49dd-ebcb-41bf-99b1-c40b60e77612.png">

이곳에서 ``ItemReader`` 를 통해 읽어들이고 그 값이 null 이면 ``RepeatStatus.FINISHED`` 를 그렇지 않다면 ``RepeatStatus.CONTINUABLE`` 를 리턴하게되고,  

<img width="894" alt="image" src="https://user-images.githubusercontent.com/45073750/188693644-8d7e064a-9b63-45d4-a239-1bac1a7feadc.png">

위에서 한 번 봤던 ``RepeatTemplate`` 의 ``executionInternal`` 의 해당 라인에서 result로 받고 그 밑에 있는 ``isComplete`` 메서드를 통해 ``CompletionPolicy`` 의 완료 조건에 해당되거나 ``RepeatStatus.CONTINUABLE`` 이 아닌 경우 while 문 지속의 판단 기준인 ``running`` 을 false로 바꿔주게된다. 그러면 

<img width="856" alt="image" src="https://user-images.githubusercontent.com/45073750/188695482-8cee372d-5c3d-447d-b231-11043799e0e6.png">

``RepeatStatus.FINISHED`` 를 담은 result 를 return 하게 되고, 

<img width="841" alt="image" src="https://user-images.githubusercontent.com/45073750/188697740-8be37400-4aca-444e-a389-e660e47bf5fd.png">

``ChunkProvider`` 의(크게는 ``ChunkOrientedTasklet`` 의) ``RepeatTemplate`` 반복이 끝이난게 된다.  

<img width="791" alt="image" src="https://user-images.githubusercontent.com/45073750/188698100-8114c938-514e-4af1-9a5c-4937e923a828.png">

``ChunkOrientedTasklet`` 의 남은 흐름을 쭉 타고 최종적으로 ``RepeatStatus.FINISHED`` 를 리턴,  

<img width="1064" alt="image" src="https://user-images.githubusercontent.com/45073750/188698392-eec61e45-2c37-45fa-97d9-448df0941f9e.png">

역순으로 result를 return 하게 된다. 위의 코드는 ``TakletStep`` 이다(도입부와 마찬가지로 ``doInIteration``, ``doInChunkContext``, ``TransactionTemplate`` 의 흐름을 타게된다. 생략함)  

<img width="981" alt="image" src="https://user-images.githubusercontent.com/45073750/188698861-a6ec984d-e785-455d-94bc-d541302dc4ac.png">

return 받은 ``RepeatStatus.FINISHED`` 값을 가지고 다시 ``RepeatTemplate`` 의 남은 흐름을 쭉 타고나서 이것을 호출했던 ``TaskletStep`` 으로 돌아가게 되고 마무리된다.  

위에서 살펴본 흐름을 간략히 정리하자면,  

 ``TaskletStep`` -> {``RepeatTemplate`` ->  ``ChunkOrientedTasklet`` ->  
 ``ChunkProvider`` -> (``RepeatTemplate`` ->  ``ItemReader``) 반복 -> ``ChunkOrientedTasklet`` -> ``ChunkProcessor`` -> ``ItemProcessor`` -> ``ChunkProcessor`` -> ``ItemWriter``} 반복

이라고 볼 수 있겠다. 그러나, 아래와 같이 processor 에 커스텀한 ``RepeatTemplate`` 을 작성했을 경우에 ``ItemProcessor`` 이후에 다시 ``RepeatTemplate`` 이 반복되는 흐름을 가지게 될 것이다. 이는 reader, writer 도 마찬가지다. 그리고 밑의 코드는 ``RepeatStatus.FINISHED`` 를 리턴하는 경우가 없으므로 ``CompletionPolicy`` 의 완료조건에 의해서만 ``RepeatTemplate`` 반복이 끝나게 될 것이다.  

```java
.processor(new ItemProcessor<String, String>() {
     RepeatTemplate repeatTemplate = new RepeatTemplate();

     @Override
     public String process(String item) throws Exception {
         repeatTemplate.setCompletionPolicy(new SimpleCompletionPolicy(3));
         repeatTemplate.iterate(new RepeatCallback() {
             @Override
             public RepeatStatus doInIteration(RepeatContext context) throws Exception {
                 System.out.println("Testing repeatTemplate");
                 return RepeatStatus.CONTINUABLE;
             }
         });
         return item;
     }
 })
```

그리고 ``repeatTemplate.setExceptionHandler();`` 구문을 통해 ``ExceptionHandler`` 를 등록해줄 수도 있다.  

<br/>

## FaultTolerant

* 스프링 배치는 ``Job`` 실행 중에 요류가 발생할 경우 장애를 처리하기 위한 기능을 제공하며 이를 통해 복원력을 향상시킬 수 있다.
* 오류가 발생해도 ``Step`` 이 즉시 종료되지 않고 ``Retry`` 혹은 ``Skip`` 기능을 활성화 함으로써 내결함성 서비스가 가능하도록 한다.
* 프로그램의 내결함성을 위해 ``Skip`` 과 ``Retry`` 기능을 제공한다. 
  * ``Skip``
    * ``ItemReader`` / ``ItemProcessor`` / ``ItemWriter`` 에 적용할 수 있다.
  * ``Retry``
    * ``ItemProcessor`` / ``ItemWriter`` 에 적용할 수 있다.
* ``FaultTolerant`` 구조는 청크 기반의 프로세스 기반위에 ``Skip`` 과 ``Retry`` 기능이 추가되어 재정의 되어 있다.

```java
public Step batchStep() {
  return stepBuilderFactory.get("batchStep")
     .<I,O>chunk(10)
     .reader(ItemReader)
     .writer(ItemWriter)
     .faultTolerant()                          //내결함성 기능 활성화
     .skip(Class<? extends Throwable> type)    //에외 발생 시 Skip 할 예외 타입 설정
     .skipLimit(int skipLimit)                 //Skip 제한 횟수 설정
     .skipPolicy(SkipPolicy skipPolicy)]       //Skip을 어떤 조건과 기준으로 적용 할 것인지 정책 설정
     .noSkip(Class<? extends Throwable> type)  //예외 발생 시 Skip 하지 않을 예외 타입 설정
     .retry(Class<? extends Throwable> type)   //예외 발생 시 Retry 할 예외 타입 설정
     .retryLimit(int retryLimit)               //Retry 제한 횟수 설정
     .retryPolicy(RetryPolicy retryPolicy)     //Retry를 어떤 조건과 기준으로 적용 할 것인지 정책 설정
     .backOffPolicy(BackOffPolicy backOffPolicy) //다시 Retry 하기 까지의 지연시간(ms)을 설정
     .noRetry(Class<? extends Throwable> type) //예외 발생 시 Retry 하지 않을 예외 타입 설정
     .noRollback(Class<? extends Throwable> type) //예외 발생 시 Rollback 하지 않을 예외 타입 설정
     .build();
}
```

``FaultTolerant`` 를 사용하게 될 경우 구조 살짝 바뀌게 된다.  

* ``SimpleStepBuilder`` 를 상속받는 ``FaultTolerantStepBuilder`` 
* ``SimpleChunkProvider`` 를 상속받는 ``FaultTolerantChunkProvider`` 
* ``SimpleChunkProvider`` 를 상속받는 ``FaultTolerantChunkProcessor`` 

이렇게 ``FaultTolerant`` 사용을 위해 기존의 클래스를 상속받는 클래스를 사용하게 된다.  

그리고 read, process, write을 하는 과정에서 예외가 발생하게 되면 skip count 만큼 예외를 건너뛰게 된다.  
process와 write는 retry도 생각해야 한다. ``FaultTolerantChunkProcessor``는 ``RetryTemplate`` 의 ``execute()`` 메서드를 호출하는 방식을 사용하게 된다.(``ChunkProvider``가 ``RepeatTemplate`` 을 호출해서 진행하는 것 처럼)  
이때 에외가 발생하게 되면 retry count 만큼 재시도를 하게되고 다시 예외가 발생하게 되면 skip 흐름으로 들어가 skip count 만큼 예외가 발생하게 된다.  

이제 간단히 흐름을 살펴보겠다.  

```java
@Bean
public Step step1() {
    return stepBuilderFactory.get("step1")
            .<String, String>chunk(5)
            .reader(new ItemReader<String>() {
                int i = 0;

                @Override
                public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
                    i++;
                    if (i == 1) {
                        throw new IllegalArgumentException("this exception should be skipped.");
                    }
                    return i > 3 ? null : "item" + i;
                }
            })
            .processor(new ItemProcessor<String, String>() {
                @Override
                public String process(String item) throws Exception {
                    throw new IllegalStateException("this exception should be retried.");
                }
            })
            .writer(System.out::println)
            .faultTolerant()							//faultTolerant 사용
            .skip(IllegalArgumentException.class)
            .skipLimit(2)
            .retry(IllegalStateException.class)
            .retryLimit(2)
            .build();
}
```



![image](https://user-images.githubusercontent.com/45073750/188872580-266b5455-c988-4e2a-9b6b-548940e5ccce.png)

``faultTolerant()`` 호출을 하면 위와 같이 ``FaultTolerantStepBuilder`` 를 사용하게 된다.  

![image](https://user-images.githubusercontent.com/45073750/188872933-871ad15d-eec5-43e1-b017-918cd0f0bebc.png)

``build()`` 를 호출하게되면 ``FaultTolerantStepBuilder`` -> ``SimpleStepBuilder`` -> ``AbstractTaskletStepBuilder`` 의 흐름을 타며 ``build()`` 를 호출하게 된다. 

![image](https://user-images.githubusercontent.com/45073750/188873319-e32936c6-79ea-4933-9f93-7dfedee03f49.png)

``createTasklet()`` 호출하게 되면 ``FaultTolerantStepBuilder`` 에서 ``createChunkProvider()`` 와 ``createChunkProcessor()`` 를 호출하게되고,  

![image](https://user-images.githubusercontent.com/45073750/188873963-b7c65102-7a02-4b09-b471-74e450f91dfe.png)

해당 메서드에서 ``FaultTolerantChunkProvider`` 와 ``FaultTolerantChunkProcessor``를 생성해주는 것을 확인할 수가 있다.  

reader에 ``i == 1`` 일 경우 exception 을 던지게끔 해놨었는데,  

![image-20220907211512769](/Users/user/Library/Application Support/typora-user-images/image-20220907211512769.png)

![image](https://user-images.githubusercontent.com/45073750/188876042-bdffab14-524d-4887-90e5-f4af4c640d7a.png)

``FaultTolerantChunkProvider`` 의 ``read()`` 메서드를 보면 catch 구문에 ``shouldSkip`` 메서드를 이용해 skip 여부를 확인하는 구문이 있는 것을 확인할 수가 있다.  

<img width="933" alt="image" src="https://user-images.githubusercontent.com/45073750/188878145-6709b330-ce6d-4da2-8cae-19ce2bc1615c.png">

![image](https://user-images.githubusercontent.com/45073750/188877549-6f1ff41b-ea9c-413d-9392-f6abb2786fd7.png)

그리고  ``FaultTolerantChunkProcessor`` 의 ``transform`` 메서드 내부에서 ``RetryTemplate`` 의 ``execute`` 를 호출하는 것을 확인할 수가 있다. 해당 구문 위쪽에 ``RetryCallback`` 가 정의되어 있다.  

일단 크게 대략 이 정도의 흐름이고 흐름은 뒷 쪽에서 차차 살펴보도록 하겠다.  

<Br/>



