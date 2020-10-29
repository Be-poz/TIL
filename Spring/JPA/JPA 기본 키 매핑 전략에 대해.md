# JPA 기본 키 매핑 전략에 대해

``@Id`` 에 대한 기본키 매핑 전략은 여러 방법이 있다. 크게는 직접할당하는 방법과 자동생성 방법이 있다.  

***

### 직접할당 방법

```java
@Id
@Column(name = "id")
private String id;
```

다음의 경우는 그냥 직접 할당하는 방법이다. 식별자 값을 반드시 넣고 저장을 해야한다.  

***

### 자동할당 방법

#### IDENTITY 전략

이 전략은 기본 키 생성을 데이터베이스에 위임하는 전략이다.  

주로 MySQL, PostgreSQL, SQL Server, DB2 에서 사용한다.  

예로 들어 MySQL 의 AUTO_INCREMENT 기능은 데이터베이스가 기본 키를 자동으로 생성해준다.  

```JAVA
@Entity
public class Board{
    @Id
    @GeneratedValue(startegy = GenerationType.IDENTITY)
    private Long id;
    ...
}
```

다음과 같이 사용하면 된다.  

IDENTITY 전략은 데이터베이스에 INSERT를 한 후에야 그 값을 알 수가 있다.  

엔티티가 영속 상태가 되려면 식별자가 반드시 필요한데, 이 전략은 INSERT를 해야 그 값을 알 수 있으므로 특이하게 em.persist()를 호출하는 즉시 INSERT SQL이 데이터베이스에 전달이된다. 그리고 그 값을 바로 반환받게 된다. 따라서 이 전략은 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.  



#### SEQUENCE 전략

데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트인데, 이 전략은 이 시퀀스를 사용해서 기본 키를 생성한다. 이 전략은 시퀀스를 지원하는 오라클, PostgreSQL, DB2, H2 데이터베이스에서 사용할 수 있다.  

```java
@Entity
@SequenceGenerator(
	name = "BOARD_SEQ_GENERATOR",
    sequenceName = "BOARD_SEQ",	//매핑할 데이터베이스 시퀀스 이름
    initialValue = 1, allocationSize = 1)
public class Board{
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                   generator = "BOARD_SEQ_GENERATOR")
    private Long id;
    ...
}
```

다음과 같이 사용하면 된다.  속성은 다음과 같다.  

* name: 식별자 생성기 이름
* sequenceName: 데이터베이스에 등록되어 있는 시퀀스 이름
* initialValue: DDL 생성 시에만 사용됨, 시퀀스 DDL을 생성할 때 처음 시작하는 수
* allocationSize: 시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용됨)
* catalog, schema: 데이터베이스 catalog, schema 이름

SEQUENCE 전략은 em.persist() 가 호출이 되면 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회한다. 그리고 조회한 식별자를 엔티티에 할당한 후에 엔티티를 영속성 컨텍스트에 저장한다.  

시퀀스는 테이블을 만들면서 같이 만든다.  

그런데 문제는, 이 시퀀스를 받아오기 위해서 데이터베이스와 연결이 되어야 하는데 이 부분에서 성능 문제가 발생할 수가 있다. 그래서 ``allocationSize``를 통해 시퀀스를 한 번에 들고오는 작업을 하는 것이다.  

``initialValue``의 기본값은 1, ``allocationSize``의 기본값은 50으로 설정되어있다.  

처음에 딱 돌리면 한 번만 데이터베이스와 연결해서 1부터 50까지 들고온 후에, 해당 시퀀스를 이용하고 더 필요하다면 그때가 되어서야 데이터베이스와 연결하는 식으로 동작하여 성능을 최적화 시킨다.  



#### TABLE 전략

TABLE 전략은 키 생성 전용 테이블을 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어서 데이터베이스 시퀀스를 흉내내는 전략인데, 잘 사용하지 않으므로 생략하겠다.  

***