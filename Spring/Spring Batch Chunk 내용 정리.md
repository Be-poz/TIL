# Spring Batch Chunk 내용 정리

## Chunk

* Chunk란 여러 개의 아이템을 묶은 하나의 덩어리 블록을 의미
* 한 번에 하나씩 아이템을 입력 받아 Chunk 단위의 덩어리로 만든 후 Chunk 단위로 트랜잭션을 처리함. 즉, Chunk 단위의 Commit과 Rollback이 이루어짐
* 일반적으로 대용량 데이터를 한 번에 처리하는 것이 아닌 청크 단위로 쪼개어서 더 이상 처리할 데이터가 없을 때 까지 반복해서 입출력하는데 사용됨

* ``Chunk<I>`` vs ``Chunk<O>``
  * ``Chunk<I>``는 ``ItemReader``로 읽은 하나의 아이템을 Chunk에서 정한 개수만큼 반복해서 저장하는 타입
  * ``Chunk<O>``는 ``ItemReader``로부터 전달받은 ``Chunk<I>``를 참조해서 ``ItemProcessor``에서 적절하게 가공, 필터링한 다음 ``ItemWriter``에 전달하는 타입

``Chunk<I>``는 내부적으로 ``List<Item>`` 를 가지고 있으며 Chunk Size를 넘어서지 않았다면 ``ItemReader``에서 Item을 가지고 온다. 만약 Chunk Size 만큼 ``ItemReader``가 반복해서 채웠다면 ``Chunk<I>`` 를 ``ItemProcessor`` 에 전달하게 된다.  

``ItemProcessor`` 는 아이템의 개수만큼 iterate 하며 transform하여 ``Chunk<O>`` 를 만들고 ``ItemWriter`` 한테 전달된다.  

위의 Chunk 일련의 작업들은 하나의 트랜잭션을 묶여 관리된다.  

``Chunk`` 내부에는 대략 다음과 같은 정보가 있다.  

* ``List<Item>`` : 아이템들
* ``List<SkipWrapper>``: 오류 발생 시 스킵한 아이템 저장
* ``List<Exception>``: 스킵 시 예외 목록 저장
* ``ChunkIterator iterator(return new ChunkIterator(items))``: Inner class

<br/>

* ``ChunkOrientedTasklet`` 은 스프링 배치에서 제공하는 Tasklet의 구현체로서 Chunk 지향 프로세싱을 담당하는 도메인 객체다.  
* ``ItemReader``, ``ItemWriter``, ``ItemProcessor`` 를 사용해 Chunk 기반의 데이터 입출력 처리를 담당한다.
* ``TaskletStep`` 에 의해서 반복적으로 실행되며 ``ChunkOrientedTasklet`` 이 실행될 때마다 매번 새로운 트랜잭션이 생성되어 처리가 이루어진다.
* exception이 발생할 경우, 해당 Chunk는 롤백되며 이전에 커밋한 Chunk는 완료된 상태가 유지된다.
* 내부적으로 ItemReader를 핸들링 하는 ``ChunkProvider``와 ``ItemProcessor``, ``ItemWriter`` 를 핸들링하는 ``ChunkProcessor`` 타입의 구현체를 가진다.

호출 순서는 다음과 같다.  

``TakletStep`` --``execute()``--> ``ChunkOrientedTasklet`` --``provide()``--> ``ChunkProvider`` --``read()``--> ``ItemReader`` (Chunk Size 만큼 반복) --> ``ChunkOrientedTasklet`` --``process(inputs)``--> ``ChunkProcessor`` --``process()``--> ``ItemProcessor`` (``ItemReader``가 전달한 아이템 개수만큼 반복) --> ``ChunkProcessor`` --``write(items)``--> ``ItemWriter``  

``execute()`` 이후의 프로세스를 읽을 Item이 없을 때 까지 Chunk 단위로 반복된다. 즉, ``write(items)`` 호출 후 다시 ``provide()`` 를 호출하게 되는데, 읽을 Item이 없다면 종료된다.  

코드와 함께 살펴보겠다.  

![image](https://user-images.githubusercontent.com/45073750/186176439-6ffb0f73-c4bb-41c8-bce9-6c5834ceaf19.png)

```java
public Step chunkStep() {
  return stepBuilderFactory.get("chunkStep")
          .<I, O>chunk(10)							//chunk size 설정
          .<I, O>chunk(CompletionPolicy)//chunk 프로세스 완료를 위한 정책 설정 클래스 지정
          .reader(itemReader())					//ItemReader 구현체 설정
          .writer(itemWriter())					//ItemWriter 구현체 설정
          .processor(itemProcessor())		//ItemProcessor 구현체 설정
          .stream(ItemStream())					//재시작 데이터를 관리하는 콜백에 대한 스트림 등록
          .readerIsTransactionQueue()
    //Item이 JMS, Message Queue Server와 같은 트랜잭션 외부에서 읽혀지고 캐시할 것인지 여부, 기본값은 false
          .listener(ChunkListener)			
    //Chunk 프로세스가 진행되는 특정 시점에 콜백 제공받도록 ChunkListener 설정
          .build();
}
```

![image](https://user-images.githubusercontent.com/45073750/186176673-c35a2553-b144-460b-a578-3e16928f50dc.png)

``ChunkOrientedTasklet``의 ``execute`` 메서드 호출  

* ``chunkContext.getAttribute(INPUTS_KEY);`` : Chunk 처리 중 예외가 발생하여 재 시도할 경우 다시 데이터를 읽지 않고 버퍼에 담아 놓았떤 데이터를 가지고 온다. 
* ``chunkProvider.provide(contribution)``: Item을 Chunk size 만큼 반복해서 읽어 ``Chunk<I>`` 에 저장하고 반환한다.
* ``chunkContext.setAtribute(INPUTS_KEY, inputs);``: Chunk를 캐싱하기 위해 ChunkContext 버퍼에 담는다.
* ``chunkContext.removeAttribute(INPUTS_KEY);``: Chunk 단위 입출력이 완료되면 버퍼에 저장한 Chunk 데이터를 삭제한다.
* ``return RepeatStatus.continueIf(!inputs.isEnd());``: 읽을 Item이 더 존재하는지 체크해서 존재하면 Chunk 프로세스를 반복하고 null 일 경우 RepeatStatus.FINISHED 반환하고 Chunk 프로세스를 종료한다.

![image](https://user-images.githubusercontent.com/45073750/186176850-793f1274-9677-410f-992a-4538a05cc055.png)

``ChunkProvider`` provide 에서 ``Chunk<I>`` 생성 하고 read 호출을 하는데 이게 설정한 ItemReader의 ``read`` 메서드를 호출하게 된다. (위의 사진은 ``ChunkProvider`` 의 구현체인 ``SimpleChunkProvider`` 이다)  

더 이상 읽을 Item이 없는 null 일 경우 ``RepeatStatus.FINISHED`` 를 리턴하며  반복문 종료 및 전체 Chunk 프로세스를 종료한다.

![image](https://user-images.githubusercontent.com/45073750/186177192-be8d6f74-3f5e-4de7-aa88-d4dadf393f8f.png)

``CHunkOrientedTasklet`` 는 생성된 ``Chunk<I>`` 를 ``ChunkProcessor`` 의 ``process`` 를 호출하면서 넘겨준다.  
이 곳에서 ``transform`` 를 호출해서 item의 개수만큼 ``doProcess`` 메서드를 호출한다.``doProcess`` 는 ``ItemProcessor`` 의 ``process`` 메서드를 호출하게 된다.  

 ``contribution.incrementFilterCount`` 는 processor에서 필터링 된 아이템 개수를 저장하는 것이다.

이렇게 만들어진 ``Chunk<O>`` 를 ``write`` 호출 시에 넘겨주면서 최종적으로 ``ItemWriter`` 의 ``write``  메서드를 호출하게 된다.  

<br/>

### ItemStream

* ``ItemReader`` 와 ``ItemWriter`` 처리 과정 중 상태를 저장하고 오류가 발생하면 해당 상태를 참조하여 실패한 곳에서 재시작 하도록 지원
* 리소스를 열고 닫아야 하며 입출력 장치 초기화 등의 작업을 해야 하는 경우
* ``ExecutionContext`` 를 매개변수로 받아서 상태 정보를 업데이트 한다
* ``ItemReader`` 및 ``ItemWriter`` 는 ``ItemStream`` 을 구현해야 한다

```java
ItemStream
  
  void open(ExecutionContext executionContext) throws ItemStreamException
  //read, write 메서드 호출전에 파일이나 커넥션이 필요한 리소스에 접근하도록 초기화 작업
  
  void update(ExecutionContext executionContexst) throws ItemStreamException
  //현재까지 진행된 모든 상태를 저장
  
  void close() throws ItemStreamException
  //열려 있는 모든 리소스를 안전하게 해제하고 닫음
```

```java
@Bean
public Job job() {
    return jobBuilderFactory.get("Job")
//                                .incrementer(new RunIdIncrementer())
                            .start(step1())
                            .build();
}

@Bean
public Step step1() {
    return stepBuilderFactory.get("step1")
                             .<String, String>chunk(5)
                             .reader(itemReader())
                             .writer(itemWriter())
                             .build();
}

private CustomItemStreamReader itemReader() {
    List<String> items = new ArrayList<>();

    for (int i = 0; i < 10; i++) {
        items.add("item" + i);
    }

    return new CustomItemStreamReader(items);
}

private CustomItemStreamWriter itemWriter() {
    return new CustomItemStreamWriter();
}

//CustomItemStreamReader
public class CustomItemStreamReader implements ItemStreamReader<String> {

    private final List<String> items;
    private int index = -1;
    private boolean restart = false;

    public CustomItemStreamReader(List<String> items) {
        this.items = items;
        this.index = 0;
    }

    @Override
    public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        String item = null;

        if (index < items.size()) {
            item = items.get(index);
            index++;
        }

        if (index == 6 && !restart) {
            throw new RuntimeException("Restart is required");
        }

        return item;
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        if (executionContext.containsKey("index")) {
            index = executionContext.getInt("index");
            this.restart = true;
        } else {
            index = 0;
            executionContext.put("index", index);
        }
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        executionContext.put("index", index);
    }

    @Override
    public void close() throws ItemStreamException {
        System.out.println("reader close");
    }
}

//CustomItemStreamWriter
public class CustomItemStreamWriter implements ItemStreamWriter<String> {

    @Override
    public void write(List<? extends String> items) throws Exception {
        for (String item : items) {
            System.out.println(item);
        }
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        System.out.println("writer open");
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        System.out.println("writer update");
    }

    @Override
    public void close() throws ItemStreamException {
        System.out.println("writer close");
    }
}
```

위와 같이 ``ItemStreamReader`` 와 ``ItemStreamWriter`` 를 구현하는 커스텀한 reader, writer를 생성하여서 적용해보았다.  

``ExecutionContext`` 를 이용하여 청크 단위가 끝날 때 마다 update를 하게 된다.  

```
// 첫 번째 시도 시
writer open
writer update
item0
item1
item2
item3
item4
writer update

// 두 번째 시도 시
writer open
writer update
item5
item6
item7
item8
item9
writer update
writer update
2022-08-30 01:32:16.396  INFO 39623 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step1] executed in 42ms
reader close
writer close
```

실행 시에 reader가 ``open``  writer  ``open``, reader ``update`` writer ``update`` 한다.  
이후 청크단위가 끝날 때 마다 각 stream 구현체가 ``update`` 메서드를 호출하게 된다.  
종료 시에는 ``update`` 후 ``close`` 하게 된다.  

위에서 ``ItemStreamReader`` 와 같은 인터페이스를 구현한 커스텀한 클래스를 사용하였는데, 흔히 사용하는 reader 클래스들 (ex. ``JpaCursorItemReader``, ``JpaPagingItemReader``)은 이미 상위에 ``ItemStream`` 이 있어서 ``open``, ``close``, ``update`` 메서드를 오버라이딩하여 사용할 수 있다.  

---

### REFERENCE

정수원님 스프링 배치 강의
