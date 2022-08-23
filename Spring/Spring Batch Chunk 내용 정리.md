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

![image](https://user-images.githubusercontent.com/45073750/186176439-6ffb0f73-c4bb-41c8-bce9-6c5834ceaf19.png)

다음과 같은 코드를 가지고 있고 실행 순서는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/186176673-c35a2553-b144-460b-a578-3e16928f50dc.png)

``ChunkOrientedTasklet``의 ``execute`` 메서드 호출  

![image](https://user-images.githubusercontent.com/45073750/186176850-793f1274-9677-410f-992a-4538a05cc055.png)

``ChunkProvider`` provide 에서 ``Chunk<I>`` 생성 하고 read 호출을 하는데 이게 설정한 ItemReader의 ``read`` 메서드를 호출하게 된다. (위의 사진은 ``ChunkProvider`` 의 구현체인 ``SimpleChunkProvider`` 이다)  

![image](https://user-images.githubusercontent.com/45073750/186177192-be8d6f74-3f5e-4de7-aa88-d4dadf393f8f.png)

``CHunkOrientedTasklet`` 는 생성된 ``Chunk<I>`` 를 ``ChunkProcessor`` 의 ``process`` 를 호출하면서 넘겨준다.  
이 곳에서 ``transform`` 를 호출해서 item의 개수만큼 ``doProcess`` 메서드를 호출한다.``doProcess`` 는 ``ItemProcessor`` 의 ``process`` 메서드를 호출하게 된다.  

이렇게 만들어진 ``Chunk<O>`` 를 ``write`` 호출 시에 넘겨주면서 최종적으로 ``ItemWriter`` 의 ``write``  메서드를 호출하게 된다.  



