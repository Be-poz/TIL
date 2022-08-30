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
예외가 발생하지 않았따면 ``completionPolicy`` 를 참조해서 종료 정책에 따라 반복 여부를 결정할 수 있게된다.  
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

