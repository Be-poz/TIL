# JPA 연관관계 매핑에 대해

본 내용은 자바 ORM 표준 JPA 프로그래밍 책을 토대로 정리했습니다.  

이 글은 기존에 JPA 연관관계 매핑에 대해 알고 있다는 가정하에 내용을 간단히 복기하기 위한 글입니다.  

***

### 단방향 연관관계

```JAVA
@Entity
public class Member {
    @Id
    @GeneratedValue
    private String id;

    @Column(name = "USERNAME")
    private String name;

    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
}
```

Member 와 Team의 간단한 단방향 연관관계이다.  

Member 객체에서 그냥 setTeam 이던 간에 그냥 Team 객체를 집어넣으면 된다.  

***

### 양방향 연관관계

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```

위와는 달리 Team에서 Member에 접근하기 위해서 List 형식으로 Member를 집어넣어 주었다.  

``mappedBy="변수명"`` 을 통해서 어떤 녀석하고 매핑이 되는지 알려준다. Member 객체 내의 team 변수명이므로 위와 같이 작성해주었다.  이 ``mappedBy``는 사용하는 때가 있다.  

객체의 양방향 관계는 사실 양방향 관계가 아니라 서로 다른 단방향 관계 2개다.  

테이블은 외래 키 하나로 두 테이블의 연관관계를 관리한다.  

양방향에는 매핑 규칙이 있는데, 다음과 같다.  

* 객체의 두 관계 중 하나를 연관관계의 주인으로 지정
* **연관관계의 주인만이 외래 키를 관리(등록, 수정)**
* **주인이 아닌쪽은 읽기만 가능**
* 주인은 mappedBy 속성 사용X
* 주인이 아니면 mappedBy 속성으로 주인 지정

``mappedBy``는 철자그대로 by에 의해 mapped 당한 것이다.  

위의 예제에서는 Member가 주인이다. 오직 주인만이 DB에 쓸 수 있다. Team에서 아무리 Member를 집어 넣어봤자 DB에 들어가지 않는다. 읽을 수만 있다.  

외래 키가 있는 곳을 주인으로 정하는 것이 좋다. 여기서는 Member.team이 연관관계의 주인이다.  

Team 을 살짝 건드렸더니 List 내의 Member에 이상이 생긴다거나 그럴 수도 있고 성능이슈도 있다. 그렇기에 외래 키가 있는 곳을 주인으로 정하는 것이 좋다.  



양방향 매핑 시 많이 하는 실수의 예가 있다.  

```java
Member member = new Member();
member.setUsername("member1");
em.persist(member);

Team team = new Team();
team.setName("TeamA");
team.getMembers().add(member);
em.persist(team);

em.flush();
em.clear();
```

다음의 경우에 Member 테이블에 team 이 null로 되어있다. Member가 주인인데 setTeam을 안해주었기 때문이다. 반드시 member.setTeam을 해주자.  

```java
Team team = new Team();
team.setName("TeamA");
em.persist(team);

Member member = new Member();
member.setUsername("member1");
member.setTeam(team);
em.persist(member);

em.flush();
em.clear();

Team findTeam = em.find(Team.class, team.getId());
List<Member> members = findTeam.getMembers();
```

다음과 같은 코드를 실행 시에 members 안에 값이 들어있다.  

flush(), clear()를 통해서 db에 다 집어 넣어주었기 때문이다. 하지만 이 두 메서드가 없다면,  

1차 캐시에서 값을 가져왔을텐데 아직 team.getMembers.add(member); 를 하지 않았기 때문에 값이 없다는 것을 알 수 있다. 그래서 보통 편의 메서드라는 것을 만드는데, setTeam 을 하는 과정에서 team.getMembers.add() 또한 처리를 해주는 것이다. 

그리고, 무한 루프 또한 조심을 해야한다.  

각 객체 당 toString()을 호출하게되면 Team은 각 Member를 이 각 Member는 Team을 .. 이렇게 무한 루프가 돌아가게된다. JSON 의 경우에서도 서로의 entity를 계속해서 호출하게 된다. 따라서, lombok의 toString은 조심해서 써야할 것이고, JSON의 경우에서는 entity로 반납하지 말아야 한다. 이 무한 루프의 위험성도 있고, 해당 스펙이 바뀌었을 경우 파급효과가 크기 때문이다.  

양방향 매핑은 반대 방향으로 조회 기능이 추가된 것 뿐이다. 단방향 매핑만으로도 연관관계 매핑은 완료가 된 것이다. 그러니 일단 단방향 매핑을 먼저하고 양방향은 필요할 때 추가하면 된다.  

위의 경우는 다대일 관계이다. 다 에 외래키를 갖는거다. 일대다는 일 에 외래키를 갖는 것인데 생각하지 말자  

일대일 관계를 보자. 단방향의 경우 다대일 단방향과 같다. 양방향도 뭐 ``mappedBy``를 적용해주면 된다.  

일대일 관계는 주 테이블에 외래 키를 넣는지 대상 테이블에 외래 키를 두는지에 따라 갈린다.  

**주 테이블에 외래 키**  

* 주 객체가 대상 객체의 참조를 가지는 것 처럼 주 테이블에 외래 키를 두고 대상 테이블을 찾음
* 객체지향 개발자 선호
* JPA 매핑 편리
* 장점: 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인 가능
* 단점: 값이 없으면 외래 키에 NULL 허용

**대상 테이블에 외래 키**  

* 대상 테이블에 외래 키가 존재
* 전통적인 데이터베이스 개발자 선호
* 장점: 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경할 때 테이블 구조 유지
* 단점: 프록시 기능의 한계로 **지연 로딩으로 설정해도 항상 즉시 로딩됨**

다대다 관계는 쓰는걸 무조건적으로 지양해야한다. 그 사이에 다른 테이블을 둬서 다대일 관계를 2개를 만드는 것이 낫다.  

***

### 상속관계 매핑

관계형 데이터베이스는 상속 관계가 불가능하다.  

상속관계 매핑이란 객체의 상속과 구조와 DB의 슈퍼타입 서브타입 관계를 매핑하는 것이다.  

슈퍼타입 서브타입 논리 모델을 실제 모델로 구현하는 방법은 다음과 같다.  

* 각각 테이블로 변환 -> 조인 전략
* 통합 테이블로 변환 -> 단일 테이블 전략
* 서브타입 테이블로 변환 -> 구현 클래스마다 테이블 전략

#### 조인전략

조인전략은 엔티티 각각을 모두 테이블로 만들고 자식 테이블이 부모 테이블의 기본 키를 받아서 기본 키 + 외래 키로 사용하는 전략이다.  DTYPE 컬럼을 구분 컬럼으로 사용한다.  

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn
public abstract class Item {

    @Id @GeneratedValue
    private Long id;

    private String name;
    private int price;
}

@Entity
@DiscriminatorValue("A")
public class Album extends Item{

    private String artist;
}
```

다음과 같이 구현한다.테이블이 Item, Album 따로 생긴다. Album은 Item을 상속받은 형태이다.  

``@DiscriminatorColumn``을 작성하면 Item 테이블 내에 DTYPE 이라는 컬럼이 생기는데, 이곳에 상속받은 테이블들의 엔티티 명이 들어서게 된다. ``@DiscriminatorValue("value")`` 를 통해 그 명을 바꿀 수도 있다. 위의 코드에서는 A라고 적었으니 DTYPE에 A가 들어가게 된다.  

테이블이 정규화되고, 저장공간이 효율적이고, 외래 키 참조 무결성 제약조건을 활용할 수 있다는 장점이 있다.  

그러나, 조회할 때 조인이 많이 사용되어 성능이 저하될 수 있고, 조회 쿼리가 복잡하고 등록 시에 insert를 2번 실행한다는 단점이 있다.  

#### 단일 테이블 전략

이 전략은 한 테이블 내에 모든 컬럼을 다 담고 DTYPE 으로 구분하는 방법이다.  

이 방법은 위 코드에서 ``@Inheritance(strategy = InheritanceType.SINGLE_TABLE)``으로 수정하면 적용된다.  장점은 성능이 잘나오고 쿼리문을 1번만 실행하면 된다.  

그러나, 자식 엔티티가 매핑한 컬럼을 모두 null을 허용해야 한다는 단점이 있다. 그리고 단일 테이블에 모든 것을 저장하므로 테이블이 커질 수 있고 상황에 따라서는 조회 성능이 떨어질 수도 있다.  

단일 테이블 전략은 ``@DiscriminatorColumn`` 을 사용하지 않아도 DTYPE이 자동으로 포함된다.  

#### 구현 클래스마다 테이블 전략

이 전략은 상속받는 각 테이블에 Item의 컬럼들이 다 들어간 상태로 테이블이 각각 만들어진다. 컬럼이 나뉘는 조인전략과는 사뭇 다르다. 그리고 부모 클래스인 Item은 아예 만들어지지 않는다.  각 테이블의 pk는 Item 클래스의 id를 사용하게 된다.  

``@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)`` 로 사용하면된다.  

이 전략은 서브 타입을 구분해서 처리할 때 효과적이고 not null 제약조건을 사용할 수 있어서 좋다.  

하지만, 여러 자식 테이블을 함께 조회할 때 성능이 너무 느리다.(UNION을 사용해야 한다). 그리고, 자식 테이블을 통합해서 쿼리하기가 어렵다.  

**이 전략은 DB 설계자와 ORM 전문가 둘 다 추천하지 않는 전략이다. 조인이나 단일 테이블 전략을 고려해야한다.**  

***

### @MappedSupperClass

여러 클래스에 공통된 필드가 있고 묶고 싶을 때에 바로 이 방법을 사용한다.  

```java
@MappedSuperclass
public abstract class BaseEntity {
    
    private String createdBy;
    private LocalDateTime createdDate;
    private String lastModifiedBy;
    private LocalDateTime lastModifiedDate;
}
```

다음과 같이 공통 속성을 담은 클래스를 생성하고 ``@MappedSuperclass``를 붙인다.  

그리고 해당 클래스를 사용할 클래스에서 extends 받아서 사용하면 된다.  

물론 필드에 ``@Column(name = " ")`` 를 이용할 수도 있다.  

* 상속관계 매핑이 아니다.
* 엔티티가 아니고, 테이블과 매핑되지 않는다. (테이블이 생기지 않음)
* 부모 클래스를 **상속 받는 자식 클래스에 매핑 정보만 제공한다.**
* 조회, 검색이 불가능하다.
* 직접 생성해서 사용할 일이 없으므로 **추상 클래스를 권장**한다.

***