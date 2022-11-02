# Spring Batch @JobScope, @StepScope 에 대해

* 스프링 빈에 scope 가 있는 것 처럼 Job과 Step에도 빈 생성과 관련한 scope이다.  
* 해당 스코프가 선언되면 빈 생성이 어플리케이션 구동시점이 아니라 빈의 실행시점에 이루어지게 된다.
  * ``@Values``를 주입해서 빈의 실행 시점에 값을 참조할 수 있어 지연 로딩이 가능해진다.
  * ``@Value("#{jobParameters[파라미터명]}")`` 와 같이 표현식 언어를 런타임 시점에 주입받을 수 있다.
  * ``@Values``를 사용시에 반드시 빈 선언문에 ``@JobScope``, ``@StepScope`` 정의가 필요하다.
* 프록시 모드를 기본으로 잡고있는 스코프이기 때문에 어플리케이션이 구동될 때에 해당 빈의 프록시 객체가 생성되어 실행 시점에 실제 빈을 호출한다.
* 병렬 처리 시에 각 스레드마다 Step 객체가 생성되어 할당되기 때문에 멤버 변수등의 동시적 접근과 같은 동시성 문제를 차단할 수 있다.



### @JobScope

* Step 선언문에 정의
* @Value: jobParameter, jobExecutionContext만 사용 가능



### @StepScope

* Tasklet이나 ItemReader, ItemWriter, ItemProcessor 선언문에 정의
* @Value: jobPArameter, jobExecutionContext, stepExecutionContext 사용 가능
