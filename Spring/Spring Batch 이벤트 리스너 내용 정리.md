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

### SkipListener / RetryListener

#### SKipListener

```java
public interface SkipListener<T,S> extends StepListener {

	/**
	 * Callback for a failure on read that is legal, so is not going to be
	 * re-thrown. In case transaction is rolled back and items are re-read, this
	 * callback will occur repeatedly for the same cause.  This will only happen
	 * if read items are not buffered.
	 * 
	 * @param t cause of the failure
	 */
	void onSkipInRead(Throwable t);

	/**
	 * This item failed on write with the given exception, and a skip was called
	 * for. 
	 * 
	 * @param item the failed item
	 * @param t the cause of the failure
	 */
	void onSkipInWrite(S item, Throwable t);

	/**
	 * This item failed on processing with the given exception, and a skip was called
	 * for. 
	 * 
	 * @param item the failed item
	 * @param t the cause of the failure
	 */
	void onSkipInProcess(T item, Throwable t);

}
```

``SkipListener``는 skip이 일어났을 때에 어디서 일어났는지에 따라 호출이 되게끔하는 리스너이다.  

```java
  @Bean
  public Step step1() throws Exception {
      return stepBuilderFactory.get("step1")
              .<Integer, String>chunk(10)
              .reader(listItemReader())
              .processor(new ItemProcessor<Integer, String>() {
                  @Override
                  public String process(Integer item) throws Exception {
                      if (item == 4) {
                          throw new CustomSkipException("process skipped");
                      }
                      System.out.println("process : " + item);
                      return "item" + item;
                  }
              })
              .writer(new ItemWriter<String>() {
                  @Override
                  public void write(List<? extends String> items) throws Exception {
                      for (String item : items) {
                          if (item.equals("item5")) {
                              throw new CustomSkipException("write skipped");
                          }
                          System.out.println("write : " + item);
                      }
                  }
              })
              .faultTolerant()
              .skip(CustomSkipException.class)
              .skipLimit(3)
              .listener(customSkipListener)
              .build();
  }

  @Bean
  public ItemReader<Integer> listItemReader() {
      List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9,10);
      return new LinkedListItemReader<>(list);
  }

------------------------------------------------------------------------------
@Component
public class CustomSkipListener implements SkipListener {

	@Override
	public void onSkipInRead(Throwable t) {
		System.out.println(">> onSkipInRead : "+ t.getMessage());
	}

	@Override
	public void onSkipInWrite(Object item, Throwable t) {
		System.out.println(">> onSkipInWrite : "+ item);
		System.out.println(">> onSkipInWrite : "+ t.getMessage());
	}

	@Override
	public void onSkipInProcess(Object item, Throwable t) {
		System.out.println(">> onSkipInProcess : "+ item);
		System.out.println(">> onSkipInProcess : "+ t.getMessage());
	}
}
```

위와 같이  batch configuration이 있다. reader에서는 item이 3일 때에 예외, processor, writer에서는 각각 4, 5에서 예외가 발생하고 있는 상황이다. 이를 돌려보면 다음과 같은 결과가 나오게 된다.  

```
read : 1
read : 2
read : 4
read : 5
read : 6
read : 7
read : 8
read : 9
read : 10
process : 1
process : 2
process : 1
process : 2
process : 5
process : 6
process : 7
process : 8
process : 9
process : 10
write : item1
write : item2
process : 1
write : item1
>> onSkipInProcess : 4
>> onSkipInProcess : process skipped
>> onSkipInRead : read skipped : 3
process : 2
write : item2
>> onSkipInRead : read skipped : 3
process : 5
process : 6
write : item6
>> onSkipInWrite : item5
>> onSkipInWrite : write skipped
>> onSkipInRead : read skipped : 3
process : 7
write : item7
>> onSkipInRead : read skipped : 3
process : 8
write : item8
>> onSkipInRead : read skipped : 3
process : 9
write : item9
>> onSkipInRead : read skipped : 3
process : 10
write : item10
>> onSkipInRead : read skipped : 3
```

``ItemReader``에서 skip이 발생하면 해당  item을 제외하고 진행되며 ``ItemProcessor``와 ``ItemWriter`` 에서는 chunk의 처음으로 돌아가서 스킵된 아이템을 제외한 나머지 아이템들을 가지고 처리하게 되는 것을 까먹지 말아야 한다.  

``ItemReader``로 읽어들인 아이템들은 캐싱이 되어있으므로 processor의  skip처리 때에 다시 read 호출이 되지 않는 것을 확인할 수가 있다.  

``ItemWriter``의 경우에는 chunk의 처음부터 다시 동작하는 것이 맞긴하지만 processor와 writer 단계에서 item을 한 번에 처리하는 것이 아닌 요소 별로 처리를 하게된다.  

그리고 skip된 아이템들에 대한 정보를 출력해주는 것을 확인할 수가 있다. 

<Br/>

#### RetryListener

```java
public interface RetryListener {

	/**
	 * Called before the first attempt in a retry. For instance, implementers can set up
	 * state that is needed by the policies in the {@link RetryOperations}. The whole
	 * retry can be vetoed by returning false from this method, in which case a
	 * {@link TerminatedRetryException} will be thrown.
	 * @param <T> the type of object returned by the callback
	 * @param <E> the type of exception it declares may be thrown
	 * @param context the current {@link RetryContext}.
	 * @param callback the current {@link RetryCallback}.
	 * @return true if the retry should proceed.
	 */
	<T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback);

	/**
	 * Called after the final attempt (successful or not). Allow the interceptor to clean
	 * up any resource it is holding before control returns to the retry caller.
	 * @param context the current {@link RetryContext}.
	 * @param callback the current {@link RetryCallback}.
	 * @param throwable the last exception that was thrown by the callback.
	 * @param <E> the exception type
	 * @param <T> the return value
	 */
	<T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable);

	/**
	 * Called after every unsuccessful attempt at a retry.
	 * @param context the current {@link RetryContext}.
	 * @param callback the current {@link RetryCallback}.
	 * @param throwable the last exception that was thrown by the callback.
	 * @param <T> the return value
	 * @param <E> the exception to throw
	 */
	<T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable);

}
```

* open: 재시도 전 매번 호출됨, false를 반환하면  retry 시도를 하지 않음
* close: 재시도 후 매번 호출됨
* onError: 재시도 실패 시마다 호출됨

```java
@RequiredArgsConstructor
@Configuration
public class RetryListenerConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final CustomRetryListener customRetryListener;

    @Bean
    public Job job(){
        return jobBuilderFactory.get("batchJob")
                .incrementer(new RunIdIncrementer())
                .start(step1())
                .build();
    }

    @Bean
    public Step step1(){
        return stepBuilderFactory.get("step1")
                .<Integer, String>chunk(10)
                .reader(listItemReader())
                .processor(new CustomItemProcessor())
                .writer(new CustomItemWriter())
                .faultTolerant()
                .retry(CustomRetryException.class)
                .retryLimit(2)
                .listener(customRetryListener)

                .build();
    }

    @Bean
    public ItemReader<Integer> listItemReader() {
        List<Integer> list = Arrays.asList(1,2,3,4);
//        List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9,10);
        return new LinkedListItemReader<>(list);
    }
}

---------------------------------------------------------------------------------------
public class CustomItemProcessor implements ItemProcessor<Integer,String> {

    int count = 0;

    @Override
    public String process(Integer item) throws Exception {

        if(count < 2) {
            if (count % 2 == 0) {
                count = count + 1;

            } else if (count % 2 == 1) {
                count = count + 1;
                throw new CustomRetryException();
            }
        }
        return String.valueOf(item);
    }
}

public class CustomItemWriter implements ItemWriter<String> {
    int count = 0;
    @Override
    public void write(List<? extends String> items) throws CustomRetryException {
        for (String item : items) {
            if(count < 2) {
                if (count % 2 == 0) {
                    count = count + 1;

                } else if (count % 2 == 1) {
                    count = count + 1;
                    throw new CustomRetryException();
                }
            }
            System.out.println("write : " + item);
        }
    }
}

@Component
public class CustomRetryListener implements RetryListener{

	@Override
	public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
		System.out.println("open");
		return true;
	}

	@Override
	public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
		System.out.println("close");
	}

	@Override
	public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
		System.out.println("Retry Count: " + context.getRetryCount());
	}
}
```

위와 같은 코드일 때 실행시키면  

```
open
close
open
Retry Count: 1
close
open
close
open
close
open
close
open
close
open
write : 1
Retry Count: 1
close
open
close
open
close
open
close
open
close
open
write : 1
write : 2
write : 3
write : 4
close
```

위와 같이 출력이 된다.  

Retry의 경우  ``ChunkIterator``를 통해  ``RetryTemplate``을 호출해 내부 로직을 for문을 돌리게 되므로,  
처음에 open close 후 retry 한 번 발생하고 이후 다시 process 과정을 거친다. write 에서도 마찬가지로 1을 write하고 2에서 예외가 발생하여 retry가 일어나게 되고 다시 청크의 처음부터 돌아가서 process작업을 한 후 write를 정상적으로 작성하게 된다.  

---

### REFERENCE

정수원님 스프링 배치 강의
