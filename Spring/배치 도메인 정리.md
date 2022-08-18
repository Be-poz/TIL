1. BatchAutoConfiguration
   * 스프링 배치가 초기화 될 때 자동으로 실행되는 설정 클래스
   * Job을 수행하는 JobLauncherApplicationRunner 빈을 생성
2. SimpleBatchConfiguration
   * JobBuilderFactory와 StepBuilderFactory 생성
   * 스프링 배치의 주요 구성 요소 생성 - 프록시 객체로 생성됨
3. BatchConfigurerConfiguration
   * BasicBatchConfigurer
     * SimpleBatchConfiguration에서 생성한 프록시 객체의 실제 대상 객체를 생성하는 설정 클래스
     * 빈으로 의존성 주입 받아서 주요 객체들을 참조해서 사용할 수 있다
   * JpaBatchConfigurer
     * JPA 관련 객체를 생성하는 설정 클래스

@EnableBatchProcessing -> SimpleBatchConfiguration -> BatchConfigurerConfiguration -> BatchAutoConfiguration  



JobLauncherApplicationRunner 가 job을 실행  
Job이 내부 Step 들을 실행  
Step 들이 내부 taskLet을 실행  

<br/>

JobLauncher가 Job과 JobParameter를 가지고 JobRepository를 통해 DB에 동일한 JobInstance가 있는지 확인한다. (처음 실행하는건지 이전에 한 번이라도 실행한 Job인지를 확인)  
존재한다면 기존 JobInstance 리턴 존재하지 않다면 새로운 JobInstance를 생성  
기존 JobInstance를 반환한다면 Job을 실행하지 않고 예외를 발생시킨다.  

<br/>

JobParameter는 JobInstance를 구분하기 위한 용도, JobParameters와 JobInstance는 1:1 관계  

```java
//@Component
public class JobRunner implements ApplicationRunner {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job job;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
                .addString("name", "user1")
                .addLong("seq", 2L)
                .addDate("date", new Date())
                .addDouble("age", 100.5)
                .toJobParameters();

        jobLauncher.run(job, jobParameters);
    }
}
```

![image](https://user-images.githubusercontent.com/45073750/179698035-e762a40e-0576-44f7-9451-ebc885a7c3eb.png)

BATCH_JOB_EXECUTION_PARAMS 테이블에 위와 같이 들어가게됨.  
`` java -jar batchExample-0.0.1-SNAPSHOT.jar 'name=user3' 'seq(long)=5L' 'date(date)=2021/01/19' 'age(double)=131.5'`` 이렇게 어플리케이션 실행 시 주입하는 방법도 있음  

```java
@Bean
public Step step1() {
    return stepBuilderFactory.get("Step1")
                             .tasklet((StepContribution contribution, ChunkContext chunkContext) -> {
                                 JobParameters jobParameters = contribution.getStepExecution().getJobExecution().getJobParameters();
                                 System.out.println(jobParameters.getString("name"));
                                 System.out.println(jobParameters.getLong("seq"));
                                 System.out.println(jobParameters.getDate("date"));
                                 System.out.println(jobParameters.getDouble("age"));

                                 Map<String, Object> jobParameters1 = chunkContext.getStepContext().getJobParameters();

                                 System.out.println("Hello Spring Batch, step1");
                                 return RepeatStatus.FINISHED;
                             })
                             .build();
}
```

contribution 이나 chunkContext 를 이용해 가져올 수 있음  

<Br/>

JobExecution은 JobInstance에 대한 한 번의 시도를 의미하는 객체, Job 실행 중에 발생한 정보들을 저장하고 있는 객체  
JobExecution은 FAILED / COMPLETED 등의 Job 실행 결고 상태를 가지고 있고, COMPLETED면 JobInstance 실행이 완료된 것으로 간주해서 재실행이 불가능하지만 FAILED면 가능하다. JobParameter가 동일한 값으로 Job을 실행할지라도 JobInstance를 계속 실행할 수 있다. 즉 COMPLETED 될 때 까지 하나의 JobInstance 내에서 여러 번의 시도가 생길 수 있다.  

JobInstance와 JobExecution은 1:N 관계  

JobLauncher가 Job과 JobParameter를 가지고 JobRepository를 통해 DB에 동일한 JobInstance가 있는지 확인한다. (처음 실행하는건지 이전에 한 번이라도 실행한 Job인지를 확인)  
존재한다면 기존 JobInstance 리턴 존재하지 않다면 새로운 JobInstance를 생성  
기존 JobInstance를 반환한다면 Job을 실행하지 않고 예외를 발생시킨다.  

위의 JobInstance 흐름에서 추가적으로  

새로운 JobInstance가 생성이 되면 새로운 JobExecution도 생성됨  
기존 JobInstance를 반환할 때 예외 발생시킨다 했는데 정확히는 해당 JobInstance의 BatchStatus가 COMPLETED 일 때에 예외가 발생하고 FAILED 일 경우에는 새로운 JobExecution이 생성되고 재실행 가능하게된다.

<br/>

StepExecution은 Step에 대한 한 번의 시도를 의미하는 객체, Step 실행 중에 발생한 정보들을 저장하고 있는 객체  
각 Step 별로 생성되고, Job이 재시작 하더라도 이미 성공한 Step은 재실행되지 않고 실패한 Step만 실행된다.  
이전 단계 Step이 실패해서 현재 Step이 실행하지 않았다면(도달 안함) StepExecution을 생성하지 않음  
StepExecution 중 하나라도 실패하면 JobExecution은 실패한다  

<br/>

StepContribution은 청크 프로세스의 변경 사항을 버퍼링 한 후 StepExecution 상태를 업데이트하는 도메인 객체  
TaskletStep이 StepExecution을, StepExecution이 StepContribution을 생성, TaskletStep이 execute(contribution, chunkContext) 를 이용하여 chunkOrientedTasklet을 실행 그리고 이것이 ItemReader, ItemProcessor 등을 수행하고 이 청크 프로세스의 변경 사항을 StepContribution에 버퍼링한다. 그리고 StepExecution이 완료되는 시점에 apply 메서드를 호출하여 StepContribution을 이용하여 상태를 최종 업데이트한다.  

<br/>

ExecutionContext는 Map 형식으로, StepExecution 또는 JobExecution 객체의 상태를 저장하는 공유 객체  
Step 범위에서는 각 Step의 StepExecution에 저장돼서 Step간 서로 공유가 안되지만, Job은 Job 내의 Step 간 서로 공유 가능  

```java
@Component
public class ExecutionContextTasklet1 implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        System.out.println("step1 is executed");

        ExecutionContext jobExecutionContext = contribution.getStepExecution().getJobExecution().getExecutionContext();
        ExecutionContext stepExecutionContext = contribution.getStepExecution().getExecutionContext();

        String jobName = chunkContext.getStepContext().getStepExecution().getJobExecution().getJobInstance().getJobName();
        String stepName = chunkContext.getStepContext().getStepExecution().getStepName();

        if (jobExecutionContext.get("jobName") == null) {
            jobExecutionContext.put("jobName", jobName);
        }

        if (stepExecutionContext.get("stepName") == null) {
            stepExecutionContext.put("stepName", stepName);
        }

        System.out.println("jobName: " + jobExecutionContext.get("jobName"));
        System.out.println("stepName: " + stepExecutionContext.get("stepName"));

        return RepeatStatus.FINISHED;
    }
}

@Component
public class ExecutionContextTasklet2 implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        System.out.println("step2 is executed");

        ExecutionContext jobExecutionContext = chunkContext.getStepContext().getStepExecution().getJobExecution().getExecutionContext();
        ExecutionContext stepExecutionContext = chunkContext.getStepContext().getStepExecution().getExecutionContext();

        System.out.println("jobName: " + jobExecutionContext.get("jobName"));
        System.out.println("stepName: " + stepExecutionContext.get("stepName"));

        String jobName = chunkContext.getStepContext().getStepExecution().getJobExecution().getJobInstance().getJobName();
        String stepName = chunkContext.getStepContext().getStepExecution().getStepName();

        if (stepExecutionContext.get("stepName") == null) {
            stepExecutionContext.put("stepName", stepName);
        }

        if (jobExecutionContext.get("jobName") == null) {
            jobExecutionContext.put("jobName", jobName);
        }

        return RepeatStatus.FINISHED;
    }
}
```

대충 이런식이다.  

<br/>

JobRepository는 배치 작업 중의 정보를 저장하는 저장소 역할이다. @EnableBatchProcessing 을 선언하면 JobRepository가 자동으로 빈으로 생성됨  

<br/>

JobLauncher는 배치 Job을 실행시키는 역할을 한다. Job과 Job Parameters를 인자로 받으며 요청된 배치 작업을 수행한 후 최종 client에게 JobExecution을 반환한다.  

스프링 부트 배치에서는 JobLauncherApplicationRunner가 자동적으로 JobLauncher를 실행시킨다.  

taskExecutor를 SyncTaskExecutor로 설정하여 동기적 실행을 할 수 있고,  
taskExecutor를 SimpleAsyncExecutor로 설정하여 비 동기적 실행을 할 수 있다.  

전자는 스케줄러에 의한 배치처리에 적합하고 후자는 HTTP 요청에 의한 배치처리에 적합하다.(배치처리 시간이 길어도 상관 무 vs 유)  

<br/>

