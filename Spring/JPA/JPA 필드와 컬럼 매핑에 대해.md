#  JPA 필드와 컬럼 매핑에 대해

본 내용은 자바 ORM 표준 JPA 프로그래밍 책을 토대로 정리했습니다.  

필드와 컬림 패잉용 어노테이션은 다음과 같이 있다.  

``@Column, @Enumerated, @Temporal, @Lob, @Transient, @Access`` 이것들을 간단히 살펴보자.  

***

### @Column

``@Column``은 가장 많이 사용되는 어노테이션이고 기능도 많다. 속성 중에 name, nullable이 주로 사용된다.  

| 속성             | 기능                                                         | 기본값                      |
| :--------------- | ------------------------------------------------------------ | --------------------------- |
| name             | 필드와 매핑할 테이블의 컬럼 이름                             | 객체의 필드 이름            |
| insertable       | 엔티티 저장 시 이 필드도 같이 저장한다. false로 설정하면 db에 저장하지 않는다. false 옵션은 읽기 전용일 때 사용한다. | true                        |
| updatable        | 엔티티 수정 시 이 필드도 같이 수정한다. false로 설정하면 db에 수정하지 않는다. false 옵션은 읽기 전용일 때 사용한다. | true                        |
| table            | 하나의 엔티티를 두 개 이상의 테이블에 매핑할 때 사용한다.    | 현재 클래스가 매핑된 테이블 |
| nullable         | null 값 허용 여부를 설정한다. false 설정시 DDL 생성 시에 not null 제약조건이 붙게된다. | true                        |
| unique           | 유니크 제약조건을 걸 때 사용한다.                            |                             |
| columnDefinition | db 컬럼 정보를 직접 줄 수 있다.                              |                             |
| length           | 문자 길이 제약조건, String 타입에만 사용한다.                | 255                         |
| precision, scale | BigDecimal 타입에 사용한다. precision은 소수점을 포함한 전체 자릿수를, scale은 소수의 자릿수다. | precision=19, sacle=2       |

``@Column(nullable = false)``, ``@Column(unique = true)``, ``@Column(columnDefinition="varchar(100) default 'EMPTY'")``, ``@Column(length=400)``, ``@Collumn(precision=10, scale=2)`` 와 같이 사용한다.  

> ``@Column``을 생략하면 기본값이 적용되는데, nullable 속성일 경우 좀 특이하다.  
>
> int data; 일 때에는 ``data integer not null`` 이라는 DDL이 생성된다. 
>
> Integer data2; 는 ``data2 integer`` 이다.  
>
> 기본 타입에는 null 값 허용이 안되지만, 객체 타입일 때에는 null 허용이 되는데, JPA는 알아서 해당 처리를 해준다. 문제는 ``@Column``을 사용하면 기본값이 ``nullable=true`` 이므로 null 처리가 안된다.
>
> @Column int data3; ``data3 integer`` 이렇게 말이다.
>
> 따라서 다음의 경우에는 ``nullable=false``를 지정하는 것이 안전하다.

***

### @Enumerated

enum 타입의 속성은 value가 있다.  

``EnumType.ORDINAL`` : enum 순서를 db에 저장  

``EnumType.STRING``: enum 이름을 db에 저장  

전자의 경우 enum 파일이 수정되었을 때 큰 혼란을 야기하므로 반드시 후자를 사용하는 것이 좋다.  

``@Enumerated(EnumType.STRING)``다음과 같이 말이다.  

***

### @Temporal

``@Temporal``은 날짜 타입을 매핑할 때 사용되는데,  

이제는 그냥 LocalDate, LocalDateTime 등을 사용하면 된다.  

***

### @Lob

``@Lob``은 db의 BLOB, CLOB 타입과 매핑한다.  

***

### @Transient

``@Transient``는 매핑을 안한다는 뜻을 가지고 있다.  

따라서 db에 저장하지 않고 조회하지도 않는다. 객체에 임시로 어떤 값을 보관하고 싶을 때 사용한다.  

***

### @Access

JPA가 엔티티 데이터에 접근하는 방식을 지정한다.  

``@Access(AccessType.FIELD)`` 와 ``@Access(AccessType.PROPERTY)``가 있다.  

전자는 필드에 직접 접근한다. private 이어도 접근 가능하다.  

후자는 접근자를 이용해 접근하는 방식이다.  

``@Access``를 생략한다면 ``@Id``의 위치를 기준으로 접근 방식이 설정된다.  

```java
@Entity
@Access(AccessType.FIELD)
public class Member{
    @Id
    private String id;
    
    private String data1;
    ...
}
```

다음과 같은 상황에서는 ``@Id``가 필드에 있으므로 ``@Access``를 생략해도 된다.  

```java
@Entity
@Access(AccessType.PROPERTY)
public class Member{
    private String id;
    private String data1;
    ...
    @Id
    public String getId(){
        return id;
    }
    @Column
    public String getData1(){
        return data1;
    }
}
```

이 경우에는 ``@Id``가 프로퍼티에 있으므로 마찬가지로 생략가능하다.  

접근자를 이용해 데이터에 접근을 하고, 이 때에 만약 data1에 ``@Column(name="user_data1")`` 이렇게 적혀있어도 무시되고 data1 로 저장될 것이다.  

```java
@Entity
public class Member{
    @Id
    private String id;
    @Transient
    private String firstName;
    @Transient
    private Stirng lastName;
    
    @Access(AccessType.PROPERTY)
    public String getFullName(){
        return firstName + lastName;
    }
    ...
}
```

다음의 경우에는 기본 필드 접근 방식을 사용하고 getFullName()만 프로퍼티 접근 방식을 사용한다.  

따라서 저장하게되면 회원 테이블의 FULLNAME 컬럼에 firstName + lastName의 결과가 저장된다.  

만약, ``@Id``가 프로퍼티로 되어있고, 필드에 ``@Access(AccessType.FIELD) @Column(name="user_data1")`` 이렇게 data1위에 있다면 이건 필드로 접근을 해서 컬럼명이 적절하게 user_data1 로 사용된다.  

***

