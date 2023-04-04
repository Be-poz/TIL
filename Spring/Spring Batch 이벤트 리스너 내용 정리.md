# Spring Batch 이벤트 리스너 내용 정리

* Listener는 배치 흐름 중에 Job, Step, Chunk 단계의 실행 전후에 발생하는 이벤트를 받아 용도에 맞게 활용할 수 있도록 제공하는 인터셉터 개념의 클래스
* 이벤트를 받기 위해 Listener를 등록해야 하며 등록은 API 설정에서 각 단계별로 지정 가능하다.



* Job
  * JobExecutionListener - Job 실행 전 후
* Step
  * StepExecutionListener - Step 실행 전 후
  * ChunkListener - Chunk 실행 전 후 (Tasklet 실행 전 후), 오류 시점
  * ItemReadListener - ItemReader 실행 전 후, 오류 시점, item이 null 일 경우 호출 안됨
  * ItemProcessListener - ItemProcessor 실행 전 후, 오류 시점, item이 null 일 경우 호출 안됨
  * ItemWriteListener - ItemWriter 실행 전 후, 오류 시점, item이 null 일 경우 호출 안됨
* SkipListener - 읽기, 쓰기, 처리 Skip 실행 시점, Item 처리가 Skip 될 경우 Skip 된 item을 추적
* RetryListener - Retry 시작, 종료, 에러 시점

![image](https://user-images.githubusercontent.com/45073750/229532116-537bd523-cf08-44d6-87af-fdfaf8c17977.png)

위와 같은 인터페이스를 구현해서 사용하는 방식도 있지만, ``@BeforeRead``와 같은 어노테이션을 이용해서 사용할 수도 있다.  

<br/>

## 
