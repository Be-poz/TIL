# JPA 값 타입에 대해

**JPA의 데이터 타입 분류**  

* 엔티티 타입
  * @Entity로 정의 하는 객체
  * 데이터가 변해도 식별자로 지속해서 추적 가능 EX) 회원 엔티티의 키나 나이 값을 변경해도 식별자로 인식 가능
* 값 타입
  * int, Integer, String 처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
  * 식별자가 없고 값만 있으므로 변경 시 추적 불가 



**값 타입 분류**  

* 기본값 타입
  * 자바 기본 타입 (int, double)
  * 래퍼 클래스(Integer, Long)
  * String
* 임베디드 타입(embedded type, 복합 값 타입)
* 컬렉션 값 타입(collection value type)

***

### 임베디드 타입

임베디드 타입을 사용하면 여러 필드들을 묶어서 표현할 수 있다.  

```java
public class Member{
    @Id @GeneratedValue
    private Long id;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
}
// 이것을
@Embeddable
public class Period {

    private LocalDateTime startDate;
    private LocalDateTime endDate;
}
//Period 클래스로 만들고 @Embeddable 을 붙여준다.
public class Member{
    @Id @GeneratedValue
    private Long id;
    @Embedded
    private Period workPeriod;
}
//그리고 다음과 같이 사용해주면 된다. DB에는 startDate, endDate 이렇게 필드로 들어가게 된다.
//만약 동일한 Period 클래스가 Embedded 된다면 오류가 난다. 필드명이 같기 때문에 이럴 떄에는 속성을 재정의 해주면 된다.
public class Member{
    @Id @GeneratedValue
    private Long id;
    @Embedded
    private Period workPeriod;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "startDate", column = @Column(name = "COMPNAY_STARTDATE")),
        @AttributeOverride(name = "endDate", column = @Column(name = "COMPANY_ENDDATE"))
    })
    private Period companyPeriod;
}
//다음과 같이 @AttributeOverrides, @AttributeOverride 를 이용하면 된다.
//임베디드 타입 안에는 엔티티나 임베디드 타입이 들어갈 수도 있다.
//임베디드 타입이 null이면 매핑한 컬럼 값은 모두 null이 된다.
```

***

### 불변 객체

```java
int a = 10;
int b = a; //기본 타입은 값을 복사
b=4;

Address a = new Address("old");
Address b = a; //객체 타입은 참조를 전달
b.setCity("new") //a 값도 바뀌게 됨
```

* 객체 타입은 수정할 수 없게 만들면 부작용을 원천 차단할 수 있다.
* 값 타입은 불변 객체(immutable object)로 설계해야한다.
* 불변 객체: 생성 시점 이후 절대 값을 변경할 수 없는 객체
* 생성자로만 값을 설정하고 수정자(setter)를 만들지 않으면 된다.
* Integer, String은 자바가 제공하는 대표적인 불변객체이다.

***

### 값 타입의 비교

값 타입을 비교할 떄에는 **동일성 비교인 == 비교**와 **동등성 비교인 equals()** 비교가 있다.  

**동일성 비교**는 인스턴스의 참조 값을 비교하고, **똥등성 비교**는 equals()를 사용하여 값을 비교한다.  

```java
        Period p = new Period("a","b");
        Period p2 = new Period("a","b");
        System.out.println(p.equals(p2));
//다음의 경우는 true가 나오게 된다. equals 를 반드시 Period 클래스 내에서 오버라이딩 해주어야 한다. hash 와 함께
        Period p = new Period("a","b");
        Period p2 = new Period("a","b");
        System.out.println(p==p2);
//다음의 경우에는 false가 나오게 된다. 참조 값을 비교하였기 때문이다.
```

***

### 값 타입 컬렉션

일반적인 엔티티와의 매핑이 아닌 값 타입을 컬렉션으로 두어서 사용하려고 할 때에는 관계형 데이터베이스의 테이블은 컬럼 안에 컬렉션을 포함할 수 없기 때문에 별도의 테이블을 추가하고 ``@CollectionTable``를 사용해서 추가한 테이블을 매핑해야 한다. 이 때에 pk 값은 하나의 필드가 아니라 해당 테이블의 모든 필드값을 pk로 가진다. 하나의 필드만 pk 로 가지게되면 그것은 곧 엔티티이기 때문이다.  

```java
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOODS", joinColumns =
                @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<>();

    @ElementCollection
    @CollectionTable(name = "PERIOD", joinColumns =
                @JoinColumn(name = "MEMBER_ID"))
    private List<Period> periodHistory = new ArrayList<>();
```

``@ElementCollection``을 통해 값 타입 컬렉션이라는 것을 알려주고, ``@CollectionTable``를 통해 해당 테이블의 이름과 조인되는 컬럼을 입력해준다. String 같은 경우엔 필드가 하나여서 FOOD_NAME 으로 지정해 준 것이다.  

```java
            Member member = new Member();
            member.setName("member1");
            member.setWorkPeriod(new Period("a", "b"));

            member.getFavoriteFoods().add("치킨");
            member.getFavoriteFoods().add("피자");
            member.getFavoriteFoods().add("햄버거");

            member.getPeriodHistory().add(new Period("c", "d"));
            member.getPeriodHistory().add(new Period("e", "f"));

            em.persist(member);
            tx.commit();
```

다음과 같은 코드를 돌렸고 결과는 이렇게 나왔다.  

![value_type_collection](https://user-images.githubusercontent.com/45073750/97797008-6c088a00-1c5c-11eb-8402-807f0fc0eed6.PNG)

보면 값타입들을 별도로 persist 하지 않았는데 다 저장된 것을 볼 수 있다.  

**즉, 값 타입 컬렉션은 영속성 전이 + 고아 객체 제거 기능을 필수로 가진다.**  

이제 member를 em.find 하게되면  

```java
Hibernate: 
    select
        member0_.MEMBER_ID as MEMBER_I1_2_0_,
        member0_.USERNAME as USERNAME2_2_0_,
        member0_.TEAM_ID as TEAM_ID5_2_0_,
        member0_.a as a3_2_0_,
        member0_.b as b4_2_0_ 
    from
        Member member0_ 
    where
        member0_.MEMBER_ID=?
```

다음과 같이 나오게 되는데, 값 타입 컬렉션은 지연로딩을 한다는 것을 알 수가 있다.  

``@ElementCollection`` 의 기본값이 LAZY로 잡혀져있기 때문이다.  

값 타입은 수정 시에 새로 아예 넣어야 된다. 그러니깐 무슨 말이냐면,  

```java
findMember.getHomeAddress().setCity("newCity");
//이렇게 하지말고
Address a = findMember.getHomeAddress();
findMember.setHomeAddress(new Address("newCity",a.getStreet(),a.getZipcode()));
//다음과 같이 새로운 객체를 넣어줘야 한다는 것이다.

findMember.getFavoriteFoods().remove("치킨");
findMember.getFavoriteFoods().add("한식");
// 이 경우에는 String 이므로 remove를 통해 삭제하고 add 한다.

findMember.getAddressHistory().remove(new Address("old1", "street", "10000"));
findMember.getAddressHistory().add(new Address("newCity1", "street", "10000"));
//이 경우에는 보통 equals를 통해 찾는다. 따라서, equals와 hashcode를 제대로 구현해주어야 한다.그리고 새 Address를 add를 해준다.
```

흥미로운 점은 값 타입 컬렉션에서 remove를 하고 add를 할 때에 발생한다.  

값 타입 컬렉션에 변경 사항이 발생하면, 주인 엔티티와 연관된 모든 데이터를 삭제하고, 값 타입 컬렉션에 있는 현재 값을 모두 다시 저장한다. 왜냐하면, 값 타입은 식별자라는 개념이 없고 단순한 값들의 모을이므로 값을 변경해버리면 데이터베이스에 저장된 원본 데이터를 찾기가 어렵기 때문이다. 예로들어서 AddressHistory 에 old1,street,10000의 값을 가진 Address가 들어가있는 상황에서 그것을 remove하면 모두 다 삭제하고 나머지 Address들을 다시 insert를 하는 작업이 이루어진다는 것이다.  따라서, **값 타입 컬렉션 대신에 일대다 관계를 고려**하는 것이 좋다.

***