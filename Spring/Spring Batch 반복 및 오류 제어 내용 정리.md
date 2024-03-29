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

![image](https://user-images.githubusercontent.com/45073750/190651723-de0c9a10-64aa-4bbc-8ecb-5de10a8db741.png)

![image](https://user-images.githubusercontent.com/45073750/188876042-bdffab14-524d-4887-90e5-f4af4c640d7a.png)

``FaultTolerantChunkProvider`` 의 ``read()`` 메서드를 보면 catch 구문에 ``shouldSkip`` 메서드를 이용해 skip 여부를 확인하는 구문이 있는 것을 확인할 수가 있다.  

<img width="933" alt="image" src="https://user-images.githubusercontent.com/45073750/188878145-6709b330-ce6d-4da2-8cae-19ce2bc1615c.png">

![image](https://user-images.githubusercontent.com/45073750/188877549-6f1ff41b-ea9c-413d-9392-f6abb2786fd7.png)

그리고  ``FaultTolerantChunkProcessor`` 의 ``transform`` 메서드 내부에서 ``RetryTemplate`` 의 ``execute`` 를 호출하는 것을 확인할 수가 있다. 해당 구문 위쪽에 ``RetryCallback`` 가 정의되어 있다.  

일단 크게 대략 이 정도의 흐름이고 흐름은 뒷 쪽에서 차차 살펴보도록 하겠다.  

<Br/>

## Skip

* Skip은 데이터를 처리하는 동안 설정된 Exception이 발생했을 경우, 해당 데이터 처리를 건너뛰는 기능
* ``ItemReader`` 과정에서 예외가 발생하면 해당 아이템만 스킵하고 진행
* ``ItemProcessor`` 과정에서 예외가 발생하면 다시 Chunk의 처음으로 돌아가서 read 하게되고(이 때 read 할 item 들은 캐싱이 되어있음) 이전 process 과정에서 예외가 발생한 아이템은 체크가 되어있기 때문에 해당 아이템을 제외한 나머지 아이템들을 가지고 처리하게 됨
* ``ItemWriter`` 과정에서 예외 발생 시 ``ItemProcessor``와 마찬가지로 처음부터 돌아가서 동작을 하고 예외가 발생한 아이템은 건너뛰게 된다. 이 과정에서 원래라면 List<Item> 이렇게 리스트 형식으로 writer에 들어올 것을 processor에서 요소를 하나씩 처리하고 writer에 보내게 된다. processor에서 1, 3, 5 값이 있다면 한 번에 3개를 writer에 보내는 것이 아니라 processor에서 1처리 -> writer 1처리, processor에서 3처리 -> writer에서 3처리 ... 이런식으로 동작하게된다.
* Skip은 내부적으로 ``SkipPolicy`` 를 통해서 구현되어 있다.
* 스킵 대상에 포함된 예외인지 여부, 스킵 카운터를 초과 했는지 여부에 따라 Skip 가능 여부를 판별한다.

``Step`` -> ``RepeatTemplate`` -> ``Chunk`` -> ``Exception`` -> ``SkipPolicy`` -> ``Classifier`` -> skip?  
위의 흐름을 따라간다. ``StepBuilderFactory`` 를 이용해 ``.skip(class)``, ``noSkip(class)``, ``skipLimit(count)`` 를 설정할 수 있고 이 경우에는 ``LimitCheckingItemSkipPolicy`` 가 생성되어 작동하게 된다.  

내부적으로 ``SubclassClassifier<Throwable, Boolean>`` 을 가지고 있으며 특정 클래스에 대한 스킵 여부 boolean 값을 가지고 있으며 skip? 물음에 대하여 해당 boolean 값을 보거나 skipLimit을 체크한다.(true면 skip 한다)  

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
                                     if (i == 3) {
                                         System.out.println("error occurred item: " + i);
                                         throw new SkippableException("skip"); //커스텀 예외
                                     }
                                     System.out.println("ItemReader: " + i);
                                     return i > 20 ? null : String.valueOf(i);
                                 }
                             })
                             .processor(itemProcess())
                             .writer(itemWriter())
                             .faultTolerant()
                             .skip(SkippableException.class)
                             .skipLimit(2)
                             .build();
}

private ItemWriter<? super String> itemWriter() {
    return new SkipItemWriter();
}

private ItemProcessor<? super String, String> itemProcess() {
    return new SkipItemProcessor();
}

---
public class SkipItemProcessor implements ItemProcessor<String, String> {
    
    @Override
    public String process(String item) throws Exception {
        if (item.equals("6") || item.equals("7")) {
            System.out.println("error occurred item: " + item);
            throw new SkippableException("Process failed cnt :");
        }

        System.out.println("ItemProcess : " + item);
        return String.valueOf(Integer.parseInt(item) * -1);
    }
}

---
  
public class SkipItemWriter implements ItemWriter<String> {

    @Override
    public void write(List<? extends String> items) throws Exception {

        for (String item : items) {
            if (item.equals("-12")) {
                System.out.println("error occurred item: " + item);
                throw new SkippableException("Write failed cnt : ");
            }
            System.out.println("ItemWriter: " + item);
        }
    }
}
```

위의 코드를 돌렸을 때 결과는 다음과 같이 나온다.  

```text
ItemReader: 1
ItemReader: 2
error occurred item: 3					// 3에서 첫 에러 발생
ItemReader: 4
ItemReader: 5
ItemReader: 6
ItemProcess : 1
ItemProcess : 2
ItemProcess : 4
ItemProcess : 5
error occurred item: 6						// 6에서 두번째 에러 발생
ItemProcess : 1										// 해당 chunk 다시 시작, 에러난 item인 6은 pass
ItemProcess : 2
ItemProcess : 4
ItemProcess : 5
ItemWriter: -1
ItemWriter: -2
ItemWriter: -4
ItemWriter: -5
ItemReader: 7
ItemReader: 8
ItemReader: 9
ItemReader: 10
ItemReader: 11
error occurred item: 7					// Processor에서 7 만나자마자 에러 발생, 3번째 에러이고 limit 넘었으므로 종료
```

이제 limit을 3으로 설정하고 돌리면 다음과 같이 나오게 된다.  

```text
ItemReader: 1
ItemReader: 2
error occurred item: 3
ItemReader: 4
ItemReader: 5
ItemReader: 6
ItemProcess : 1
ItemProcess : 2
ItemProcess : 4
ItemProcess : 5
error occurred item: 6
ItemProcess : 1
ItemProcess : 2
ItemProcess : 4
ItemProcess : 5
ItemWriter: -1
ItemWriter: -2
ItemWriter: -4
ItemWriter: -5
ItemReader: 7
ItemReader: 8
ItemReader: 9
ItemReader: 10
ItemReader: 11
error occurred item: 7
ItemProcess : 8
ItemProcess : 9
ItemProcess : 10
ItemProcess : 11
ItemWriter: -8
ItemWriter: -9
ItemWriter: -10
ItemWriter: -11
ItemReader: 12
ItemReader: 13
ItemReader: 14
ItemReader: 15
ItemReader: 16
ItemProcess : 12
ItemProcess : 13
ItemProcess : 14
ItemProcess : 15
ItemProcess : 16
error occurred item: -12
ItemProcess : 12
error occurred item: -12
```

내부 코드 흐름을 간단히 살펴보자면,  

<img width="482" alt="image" src="https://user-images.githubusercontent.com/45073750/196469919-21e09ad6-b2c5-455e-8e9c-a770d061b6db.png">

``.faultTolerant()`` 를 입력하면 위의 FaultTolerant 파트에서 언급했듯이 ``SimpleStepBuilder`` 대신 ``FaultTolerantStepBuilder`` 를 사용하게 된다. 

<img width="861" alt="image" src="https://user-images.githubusercontent.com/45073750/196470325-2d354f16-a477-4a91-9290-3ab4c4df22d4.png">

``.skip()`` 을 통해 스킵될 예외들을 추가해주게 되고,  

<img width="977" alt="image" src="https://user-images.githubusercontent.com/45073750/196470505-3a7d08ef-d347-4b31-93bb-55296a8baf65.png">

<img width="928" alt="image" src="https://user-images.githubusercontent.com/45073750/196470554-55f79bfc-5973-4412-824e-92c27d99c6a0.png">

``FaultTolerantStepBuilder``에서 ``FaultTolerantChunkProvider`` 를 만드는 과정에서 ``SkipPolicy``를 생성하게되고 내가 설정한 ``skipLimit``과 skip할 exception에 대한 정보가 들어가있는 map이 생성자 파라미터로 들어가게 된다. 위의 코드에서는 따로 ``SkipPolicy``를 설정해주지 않았으므로 default 값인 ``LimitCheckingItemSkipPolicy`` 를 이용한 것이다. 

<img width="668" alt="image" src="https://user-images.githubusercontent.com/45073750/196471289-856325ef-3eea-4978-b108-92a3361350b9.png">

그리고 이 ``SkipPolicy`` 에는 ``shouldSkip`` 메서드가 존재하는데(위의 코드는 ``LimitCheckingItemSkipPolicy`` 의 ``shouldSkip`` 메서드이다)  

<img width="860" alt="image" src="https://user-images.githubusercontent.com/45073750/196471506-6e238be6-d627-468a-889e-47ccd3d51400.png">

``FaultTolerantChunkProvider`` 의 ``read`` 메서드에서 ``shouldSkip`` 을 호출하는 것을 확인할 수가 있다.  
위의 흐름은  ``ItemWriter``의 경우의 흐름이고 ``ItemProcessor``와 ``ItemWriter``는 조금 다르다. 이는 ``Retry`` 파트에서 살펴보겠다.  

```java
.skipPolicy(limitCheckingItemSkipPolicy())
  
---
  
@Bean
public SkipPolicy limitCheckingItemSkipPolicy() {
    Map<Class<? extends Throwable>, Boolean> exceptionClass = new HashMap<>();
    exceptionClass.put(SkippableException.class, true);

    return new LimitCheckingItemSkipPolicy(3, exceptionClass);
}
```

위와 같이 ``SkipPolicy``를 따로 정해서 ``.skip()``, ``.skipLimit()`` 대신 넣어줄 수도 있다. 물론 ``SkipPolicy`` 인터페이스를 커스텀하게 구현해서 넣어도 된다.  

<img width="541" alt="image" src="https://user-images.githubusercontent.com/45073750/196473296-1ea7b459-4b07-423e-a55c-724d15a36abe.png">

스프링 배치가 기본적으로 제공하는 ``SkipPolicy``는 위와 같다.  

<Br/>

## Retry

* Retry는 ItemProcess, ItemWriter에서 설정된  Exception이 발생했을 경우, 지정한 정책에 따라 데이터 처리를 재시도하는 기능
* Skip과 마찬가지로 Retry를 함으로써, 배치수행의 빈번한 실패를 줄일 수 있게 함

``RepeatOperations``와 ``RepeatTemplate`` 이 있는 것 처럼 Retry 또한 ``RetryOperations``와 ``RetryTemplate``이 있고 내부적으로  ``RetryCallback``와 ``RecoveryCallback`` 을 수행하게된다.  

대략적인 흐름은 ``Step`` -> ``RepeatTemplate`` -> ``RetryTemplate`` -> ``RetryCallback``/``RecoveryCallback`` -> ``Chunk``  
-> ``Exception`` -> ``RetryPolicy`` -> ``BackOffPolicy``/``Step``  와 같다.  

``RetryTemplate`` 에서 retry 할 예외인지, retryCount는 limit을 넘었는지 여부에 따라 ``RetryCallback`` 과 ``RecoveryCallback`` 으로 분기를 타게되고, ``RetryPolicy`` 에서도 마찬가지로 판단 후에 ``Step`` 반복문 처음부터 다시 시작할 것인지 아니면 ``Step``을 종료할 것인지 판단하게 된다.  

사용방법은 ``Skip``과 비슷하다. ``FaultTolerant`` 설명할 때 있었던 예시코드인데 다시보고 파악해보자.  

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

스프링 배치에서 기본적으로 제공하는  ``RetryPolicy``는 다음과 같다.  

<img width="726" alt="image" src="https://user-images.githubusercontent.com/45073750/197381934-b6e5c69e-c987-4739-bb42-0a6f70a68c4b.png">

default는  ``SimpleRetryPolicy``로 내가 설정한 retry 할 예외들, retryLimit 등이 파라미터로 들어가게 된다.  

이제 코드를 살펴보면서 흐름을 파악해보겠다. 일단 기본 코드는 다음과 같이 생성해놓았다.  

```java
@Bean
public Step step1() {
    return stepBuilderFactory.get("step1")
                             .<String, String>chunk(5)
                             .reader(reader())
                             .processor(processor())
                             .writer(items -> items.forEach(System.out::println))
                             .faultTolerant()
                             .retry(RetryableException.class)
                             .retryLimit(2)
                             .build();
}

@Bean
public ListItemReader<String> reader() {
    List<String> items = new ArrayList<>();
    for (int i = 0; i < 30; i++) {
        items.add(String.valueOf(i));
    }
    return new ListItemReader<>(items);
}

@Bean
public ItemProcessor<? super String, String> processor() {
    return new RetryItemProcessor();
}

---
  
public class RetryItemProcessor implements ItemProcessor<String, String> {

    private int cnt = 0;

    @Override
    public String process(String item) throws Exception {
        cnt++;
        throw new RetryableException();
    }
}

---
  
public class RetryableException extends Exception {

    public RetryableException() {
        super();
    }

    public RetryableException(String message) {
        super(message);
    }
}
```

이제 실행시켜보면서 살펴보겠다.  

<img width="933" alt="image" src="https://user-images.githubusercontent.com/45073750/197386130-577c662f-3efb-4b50-8eec-64c12ce95482.png">

<img width="1093" alt="image" src="https://user-images.githubusercontent.com/45073750/197386255-e5207f08-cc72-4cbd-9700-a7bd5d0536d7.png">

``ChunkOrientedTasklet``에서 호출한 ``FaultTolerantChunkProcessor`` 에서 아이템 수만큼 for문을 돌면서 ``RetryCallback``과 ``RecoveryCallback``이 생성되고 ``RetryTemplate`` 에서 이를 가지고 수행하게 된다. 즉 아이템마다 ``RetryCallback``과 ``RecoveryCallback`` 을 가지게 된다는 것이다. 그리고 이 아이템마다 retry를 따로 측정하기 때문에 이 내용은 꼭 기억하고 있자. 하이라이트 된 코드의 ``DefaultRetryState``은 아이템의 retry정보가 담긴 ``RetryContext``를 찾을 때 필요한 정보를 담고 있는 객체고 현재 아이템 값으로 key를 생성해 넣어주고 이 객체는 밑의 ``doExecute()`` 메서드에서 사용하게 된다.

<img width="1252" alt="image" src="https://user-images.githubusercontent.com/45073750/197387193-9aca97d6-f280-45c5-a8b3-cbd0b7636a07.png">

<img width="1598" alt="image" src="https://user-images.githubusercontent.com/45073750/197387320-3290c2cf-361d-46a1-ac72-e53252626b40.png">

``doExecute()`` 메서드 내에서 retry 할 수 있는지 체크를 하고  while문을 돌게되는데 이때 위에서 ``doExecute()`` 메서드 3번째 코드라인에서 생성된 context를 이용하게 된다. 이 context는 몇 번의 시도를 했는지, 최대 limit은 몇인지 등에 대한 상태정보를 가지고 있다. 해당 정보를 이용해 retry 할 수 있는지 체크하게되는 것이다.  

그리고 ``retryCallback.doWithRetry(context)`` 를 호출하고 이는 아까 정의되어있던 ``RetryCallback`` 메서드를 호출하게되고 내부적으로 ``doProcess``를 호출하고 결국 ``ItemProcessor`` 로직을 타게된다.  

<img width="590" alt="image" src="https://user-images.githubusercontent.com/45073750/197387566-c7de224c-d88c-4714-81c9-a1bae8fb998d.png">

예외가 던져지면,  

<img width="859" alt="image" src="https://user-images.githubusercontent.com/45073750/197387633-d0c41ad2-b468-421a-9970-2fbb08297258.png">

``RetryCallback``에서  exception catch를 하게되고, 예외가 발생하게되면 이번에는 ``RecoveryCallback`` 의 흐름을 타게된다.  

<img width="837" alt="image" src="https://user-images.githubusercontent.com/45073750/197388861-71fe3e38-7cb8-4dc8-a252-381e743770b3.png">

``RecoveryCallback`` 에서는 ``Skip``에 대한 여부가 나오게 된다. ``Skip`` 까지 설정해두었다면 해당 아이템을 item iterator에서 remove하고  다시 돌게될 것이다.  
이 내용은 밑에서 한 번 더 다루겠다.

<img width="932" alt="image" src="https://user-images.githubusercontent.com/45073750/197387697-03a7b3bb-b688-4816-9556-6894a4e8de2f.png">

``RetryTemplate``에서도 마찬가지로 로  catch 흐름을 타게된다. retry 할 수 있는 여부로 판단하여 ``BackOffPolicy`` 흐름을 타야하는지 판단한다.  

<img width="886" alt="image" src="https://user-images.githubusercontent.com/45073750/197387783-a5297d1f-a153-4822-8ae1-b4f49e0167ff.png">

이후 다시 ``Step``의 처음부터 시작하게된다. (위의 코드는 ``ChunkOrientedTasklet`` 의 ``execute()``)  

``Skip``의 경우에는 해당 아이템을 체크해두고 다시 chunk로 돌아가서 해당 아이템을 제외하고 재시도한다면,  
``Retry``는 ``Skip``과 같은 데이터의 조작없이 다시 reader부터 처음부터 처리하게된다.  

```java
@Override
public String process(String item) throws Exception {
    System.out.println("item process: " + item);
    if (item.equals("2") || item.equals("3")) {
        cnt++;
        System.out.println("processor occurred! failed cnt: " + cnt);
        throw new RetryableException();
    }
    return item;
}
```

``RetryItemProcessor`` 의 내용을 다음과 같이 수정하고 돌리면 다음과 같은 결과가 출력이 된다.  

```text
item process: 0
item process: 1
item process: 2
processor occurred! failed cnt: 1
item process: 0
item process: 1
item process: 2
processor occurred! failed cnt: 2
item process: 0
item process: 1
종료...
```

``Skip``은 예외가 발생한 아이템을 체크하고 건너뛰었지만 ``Retry``는 청크의 처음부터 다시 돌아가기 때문에 위와같은 상황이 발생하게된다.  ``Skip`` 까지 추가하고 살펴보겠다.  

```java
 .retry(RetryableException.class)
 .retryLimit(2)
```

``Skip``을 붙이고 돌리면 정상적으로 마무리가 된다. 출력결과는 다음과 같다.  

```java
item process: 0
item process: 1
item process: 2
processor occurred! failed cnt: 1
item process: 0
item process: 1
item process: 2
processor occurred! failed cnt: 2
item process: 0
item process: 1
item process: 3
processor occurred! failed cnt: 3
item process: 0
item process: 1
item process: 3
processor occurred! failed cnt: 4
item process: 0
item process: 1
item process: 4
0
1
4
item process: 5
item process: 6
item process: 7
item process: 8
item process: 9
5
6
7
8
9
```

예외 2번이 터지고 이후  ``RecoveryCallback`` 콜백 메서드를 돌면서 예외가 터진 아이템을 skip 하게 된다. 이때 ``ItemProcessor`` 에서는  ``Skip`` 시에 해당 아이템을  Item Iterator 에서 remove 하고 진행하게 된다.  

이후 다시 돌면서 아이템 '3' 을 마주하게되고 ``RetryCallback``과 ``RecoveryCallback``은 앞서 말한 것 처럼 아이템마다 할당되기 때문에 다시 retry 2번을 거치고 skip 흐름에서 item remove를 하고 이후 진행하게된다.  

<img width="853" alt="image" src="https://user-images.githubusercontent.com/45073750/197407179-d9a32ec4-9eba-4efe-8c86-e9d7a0720151.png">

<br/>

마지막으로 ``ItemReader``, ``ItemWriter``, ``ItemProcessor``의 전체 흐름 이미지로 마무리하도록 하겠다.  

``ItemReader`` 

![image](https://user-images.githubusercontent.com/45073750/197408137-64c4ae9c-df15-4c26-b003-5ee4504cd4ea.png)

``ItemProcessor``

![image](https://user-images.githubusercontent.com/45073750/197408172-8b40b11a-341f-4115-9fd9-94c881ed95c4.png)

``ItemWriter``

![image](https://user-images.githubusercontent.com/45073750/197408195-97171047-17c4-4892-b9fc-1bdcd090e11d.png)

---

### REFERENCE

정수원님 스프링 배치 강의
