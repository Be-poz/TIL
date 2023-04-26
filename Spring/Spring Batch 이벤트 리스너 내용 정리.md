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

## JobExecutionListener / StepExecutionListener

```java
public interface JobExecutionListener {

	/**
	 * Callback before a job executes.
	 *
	 * @param jobExecution the current {@link JobExecution}
	 */
	void beforeJob(JobExecution jobExecution);

	/**
	 * Callback after completion of a job. Called after both both successful and
	 * failed executions. To perform logic on a particular status, use
	 * "if (jobExecution.getStatus() == BatchStatus.X)".
	 *
	 * @param jobExecution the current {@link JobExecution}
	 */
	void afterJob(JobExecution jobExecution);

}
```

```java
public interface StepExecutionListener extends StepListener {

	/**
	 * Initialize the state of the listener with the {@link StepExecution} from
	 * the current scope.
	 *
	 * @param stepExecution instance of {@link StepExecution}.
	 */
	void beforeStep(StepExecution stepExecution);

	/**
	 * Give a listener a chance to modify the exit status from a step. The value
	 * returned will be combined with the normal exit status using
	 * {@link ExitStatus#and(ExitStatus)}.
	 *
	 * Called after execution of step's processing logic (both successful or
	 * failed). Throwing exception in this method has no effect, it will only be
	 * logged.
	 *
	 * @param stepExecution {@link StepExecution} instance.
	 * @return an {@link ExitStatus} to combine with the normal value. Return
	 * {@code null} to leave the old value unchanged.
	 */
	@Nullable
	ExitStatus afterStep(StepExecution stepExecution);
}
```

* Job / Step의 성공여부와 상관 없이 호출된다.
* 성공 / 실패 여부는 JobExecution / StepExecution 을 통해 알 수 있다.



```java
public class CustomJobListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("JobExecution.getJobName() : " + jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        System.out.println("JobExecution.getStatus() : " + jobExecution.getStatus());
        long startTime = jobExecution.getStartTime().getTime();
        long endTime = jobExecution.getEndTime().getTime();
        System.out.println("(endTime-startTime) = " + (endTime - startTime));
    }
}

@Component
public class CustomStepListener implements StepExecutionListener {

    @Override
    public void beforeStep(StepExecution stepExecution) {
        System.out.println("stepExecution.getStepName() : " + stepExecution.getStepName());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        System.out.println("stepExecution.getStatus() : " + stepExecution.getStatus());
        return ExitStatus.COMPLETED;
    }
}
```

Job과 Step에 대한 리스너 클래스 2개를 만들었다. StepListener는 빈 등록을 위해 @Component를 붙여주었다.  
어노테이션을 사용하는 경우에는  

```java
public class CustomJobListener {

    @BeforeJob
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("JobExecution.getJobName() : " + jobExecution.getJobInstance().getJobName());
    }

    @AfterJob
    public void afterJob(JobExecution jobExecution) {
        System.out.println("JobExecution.getStatus() : " + jobExecution.getStatus());
        long startTime = jobExecution.getStartTime().getTime();
        long endTime = jobExecution.getEndTime().getTime();
        System.out.println("(endTime-startTime) = " + (endTime - startTime));
    }
}
```

인터페이스 구현이 아니라 어노테이션을 달아주면 된다. 메서드 명은 자유롭게 하면된다.  

```java
@RequiredArgsConstructor
@Configuration
@Slf4j
public class JobAndStepListenerConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final CustomStepListener customStepListener;

    @Bean
    public Job job() throws Exception {
        return jobBuilderFactory.get("batchJob")
                .incrementer(new RunIdIncrementer())
                .start(step1())
                .next(step2())
                .listener(new CustomJobListener())
                .build();
    }

    @Bean
    public Step step1() {
        return stepBuilderFactory.get("step1")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
//                        throw new RuntimeException("failed");
                        return RepeatStatus.FINISHED;
                    }
                })
                .listener(customStepListener)
                .build();
    }

    @Bean
    public Step step2() {
        return stepBuilderFactory.get("step2")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                        return RepeatStatus.FINISHED;
                    }
                })
                .listener(customStepListener)
                .build();
    }
}
```

``.listener()``에 리스너를 등록해주고(어노테이션을 사용한 클래스도 마찬가지) 하면 동작하게 된다.  

<img width="2024" alt="image" src="https://user-images.githubusercontent.com/45073750/229849066-02092885-85b4-4b97-bb39-2afe172700d5.png">

정상적으로 동작하는 것을 확인할 수가 있다.  

<br/>

## ChunkListener / ItemReadListener / ItemProcessorListener / ItemWriterListener 

```java
public interface ChunkListener extends StepListener {

	static final String ROLLBACK_EXCEPTION_KEY = "sb_rollback_exception";

	void beforeChunk(ChunkContext context);
	void afterChunk(ChunkContext context);
	void afterChunkError(ChunkContext context);
}
```

```java
public interface ItemReadListener<T> extends StepListener {

	void beforeRead();
	void afterRead(T item);
	void onReadError(Exception ex);
}
```

```java
public interface ItemProcessListener<T, S> extends StepListener {

	void beforeProcess(T item);
	void afterProcess(T item, @Nullable S result);
	void onProcessError(T item, Exception e);
}
```

```java
public interface ItemWriteListener<S> extends StepListener {

	void beforeWrite(List<? extends S> items);
	void afterWrite(List<? extends S> items);
	void onWriteError(Exception exception, List<? extends S> items);
}
```

인터페이스 시그니처는 위와 같다. 메서드 명으로 파악할 수 있듯이 전/후에 수행할 행동과 에러가 났을 때 수행할 행동을 정의한다.  

```java
@RequiredArgsConstructor
@Configuration
public class ChunkListenerConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final CustomChunkListener customChunkListener;

    @Bean
    public Job job() throws Exception {
        return jobBuilderFactory.get("batchJob")
                                .incrementer(new RunIdIncrementer())
                                .start(step1())
                                .build();
    }

    @Bean
    public Step step1() throws Exception {
        return stepBuilderFactory.get("step1")
                                 .<Integer, String>chunk(10)
                                 .listener(customChunkListener)
                                 .listener(new CustomItemReadListener())
                                 .listener(new CustomItemProcessListener())
                                 .listener(new CustomItemWriteListener())
                                 .reader(listItemReader())
                                 .processor((ItemProcessor) item -> {
//                                     throw new RuntimeException("failed");
                    return "item" + item;
                                 })
                                 .writer((ItemWriter<String>) items -> {
//                                     throw new RuntimeException("failed");
                                 })
                                 .build();
    }

    @Bean
    public ItemReader<Integer> listItemReader() {
        List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9,10);
        return new ListItemReader<>(list);
    }
}
```

위와 같이 리스너를 모두 등록하고 돌려보면(각 리스너에서 현재 어떤 단계인지 출력하게끔 구현해놨다)  

```
>> Before the chunk : step1
>> beforeRead
>> afterRead : 1
>> beforeRead
>> afterRead : 2
...
>> beforeRead
>> afterRead : 10
>> beforeProcess
>> afterProcess : 1
>> afterProcess : item1
>> beforeProcess
>> afterProcess : 2
>> afterProcess : item2
...
>> beforeWrite
>> afterWrite : [item1, item2, item3, item4, item5, item6, item7, item8, item9, item10]
>> After the chunk : step1
>> Before the chunk : step1
>> beforeRead
>> After the chunk : step1
```

위와 같이 나온다. read와 write 옆의 숫자는 reader의 list 내부 item을 출력하게끔 해둔 것이다.  

1청크가 끝난 후 다음 청크 단위를 넘어가려 했지만 reader가 1~10 까지 이므로 요소가 없어서 바로 processor로 넘어가지 않고 마무리 되고 after the chunk가 호출된 것을 확인할 수가 있다.  

```java
.processor((ItemProcessor) item -> {
  throw new RuntimeException("failed");
//return "item" + item;
})

//현재 processorListener의 구현은 아래와 같음
@Component
public class CustomItemProcessListener implements ItemProcessListener {

	@Override
	public void beforeProcess(Object item) {
		System.out.println(">> beforeProcess");
	}

	@Override
	public void afterProcess(Object item, Object result) {
		System.out.println(">> afterProcess : "+ item);
		System.out.println(">> afterProcess : "+ result);
	}

	@Override
	public void onProcessError(Object item, Exception e) {
		System.out.println(">> onProcessError : " + e.getMessage());
		System.out.println(">> onProcessError : " + item);
	}
}

```

processor에서 예외를 던지게끔 변경하면  

```
...
>> afterRead : 10
>> beforeProcess
>> onProcessError : failed
>> onProcessError : 1
>> After the chunk : step1
```

예상한 대로 ``onProcessError``가 호출이 되고 chunk가 끝나게 된다. reader, writer에서의 예외 처리도 위와 동일하게 이루어진다.  

<Br/>

