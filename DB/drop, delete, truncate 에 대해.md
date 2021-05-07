# drop, delete, truncate 에 대해

### drop

* 테이블 자체를 삭제하는 명령어다.
* 테이블 자체가 모두 지워지고 생성되어 있던 인덱스도 삭제된다.
* 테이블이 사용했던 Storage는 모두 Release 처리된다.
* 오라클 10g 부터는 테이블이 삭제되는 것이 아니라 휴지통 개념처럼 잠시 삭제되고, 테이블 이름이 BIN$...로 변경된다.
* drop table [table_name]

<br/>

### delete

* 데이터만 삭제되고 테이블 용량은 줄어 들지 않는다. 
* 커밋이전에는 롤백이 가능하다.
* 전체 또는 일부 데이터 삭제가 가능하다.
* 데이터를 모두 Delete해도 사용했던 Storage는 Release 처리되지 않는다.

<br/>

### truncate

* 테이블을 최초 생성된 초기상태로 만든다.
* 용량이 줄어들고 인덱스 등도 모두 삭제된다.
* 롤백이 불가능하다.
* 무조건 전체 삭제만 가능하다.
* 삭제 행수를 반환한다.
* 테이블이 사용했던 Storage 중 최초 테이블 생성 시 할당된 Storage만 남기고 Release 처리된다.

<br/>

<img width="840" alt="스크린샷 2021-05-07 오후 4 59 18" src="https://user-images.githubusercontent.com/45073750/117417734-9e6da980-af55-11eb-8c2e-4d657bbc082b.png">
<img width="716" alt="스크린샷 2021-05-07 오후 4 40 38" src="https://user-images.githubusercontent.com/45073750/117417760-a594b780-af55-11eb-9b8a-082ce29aa59c.png">

<br/>

***

### Reference

https://lee-mandu.tistory.com/476  

https://goddaehee.tistory.com/55