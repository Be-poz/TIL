# DATA JPA

``@EnableJpaRepositories(basePackages= "jpabook.jpashop.repository")`` 원래라면 다음과 같이 설정을 해주어야 하지만 스프링 부트 사용을 하게 되면 ``@SpringBootApplication`` 에서 위치를 지정하기 때문에 사용할 필요가 없다. 만약 위치가 달라진다면 필요할 것이다.  

``public interface TeamRepository extends JpaRepository<Team, Long>{}``  
이런식으로 repository를 적게 되는데, 해당 repository를 사용하는 곳에서 ``@Autowired``를 통한 의존성 주입은 대체 어떻게 일어나는건지에 대한 의문을 품을 수 있다. 왜냐하면 클래스가 아닌 interface 이기 때문이다.  

의존성이 주입된 repository를 .getClass() 해서 정보를 보면 프록시를 반환한다는 것을 알 수가 있다. 데이터 JPA가 알아서 프록시 클래스를 만들고 주입을 해주기 때문이다. 그리고 ``@Repository``를 생략을 해도된다.  

``JpaRepository`` 인터페이스에는 다음과 같이 존재한다.  

* findAll() : List T
* findAll(Sort) : List T
* findAll(Iterable ID) : List T
* save(Iterable S) : List S
* flush()
* saveAndFlush(T) : T
* deleteInBatch(Iterable T)
* deleteAllInBatch()
* getOne(ID) : T

이 ``JpaRepository``는 ``PagingAndSortingRepository``인터페이스를 상속받는다. 여기에는  다음을 포함하고 있다.  

* findAll(Sort) : Iterable T
* findAll(Pageable) : Page T

그리고 이 인터페이스는 ``CrudRepository`` 인터페이스를 상속받는다.  

* save(S) : S
* findOne(ID) : T
* exists(ID) : boolean
* count() : long
* delete(T)

이런 것들을 포함하고 있고, 최종적으로 ``Repository`` 인터페이스를 상속받는 형태이다.  

여기서 T는 엔티티, ID는 엔티티의 식별자 타입, S는 엔티티와 그 자식 타입이다.  

``delete(T)`` 는 내부에서 ``EntityManager.remove()``를 호출한다.  
``findById(ID)``는 내부에서 ``EntityManager.find()``를 호출한다.  
``getOne(Id)``는 엔티티를 프록시로 조회하고, 내부에서 ``EntityManager.getReference()``를 호출한다.  



DATA JPA는 메서드 이름을 분석해서 JPQL을 생성하고 실행할 수 있다.  
메서드 필터 조건은 [필터 조건](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation) 를 참고하면 된다.  



**``@Query``, Repository 메서드에 쿼리 정의**  

```java
@Query("select m from Member m where m.username =: username and m.age =: age")
List<Member> findUser(@Param("username") String username, @Param("age") int age);
```

``@Query`` 어노테이션을 사용하여 JPQL 쿼리를 작성하여 수행하는 방법이 있다.  

```java
@Query("select m from Member m where m.username in :names")
List<Member> findByNames(@Param("names") List<String> names);
```

다음과 같이 컬렉션일 경우에는 IN 쿼리로 수행할 수 있다.  

```java
List<Member> findByUsername(String name); //컬렉션
Member findByUsername(String name); //단건
Optional<Member> findByUsername(String name); //단건 Optional
```

만약 값이 없을 경우에는 컬렉션은 빈 컬렉션을 반환하고, 단건 조회일 경우에는 null을 반환한다.  
단건조회에서 결과가 2건 이상일 경우에는 ``javax.persistence.NonUniqueResultException`` 이 발생한다.  



**페이징과 정렬 파라미터**  
``org.springframework.data.domain.Sort`` : 정렬 기능  
``org.springframework.data.domain.Pageable`` : 페이징 기능(내부에 ``Sort`` 포함)  



**특별한 반환 타입**  
``org.springframework.data.domain.Page`` : 추가 count 쿼리 결과를 포함하는 페이징  
``org.springframework.data.domain.Slice`` : 추가 count 쿼리 없이 다음 페이지만 확인 가능(내부적으로 limit + 1 조회)  
``List`` : 추가 count 쿼리 없이 결과만 반환  

```java
Page<Member> findByUsername(String name, Pageable pageable);//count 쿼리 사용
Slice<Member> findByUsername(String name, Pageable pageable);//count 쿼리 사용 안함
List<Member> findByUsername(String name, Pageable pageable);//count 쿼리 사용 안함
List<Member> findByUsername(String name, Sort sort);
```

```java
PageRequest pageRequest = PageRequest.of(0,3,Sort.by(Sort.Direction.DESC,"username"));
Page<Member> page = memberRepository.findByAge(10,pageRequest);

List<Member> content = page.getContent();
assertThat(content.size()).isEqualTo(3); //조회된 데이터 수
assertThat(page.getTotalElements()).isEqualTo(5); //전체 데이터 수
assertThat(page.getNumber()).isEqualTo(0); //페이지 번호
assertThat(page.getTotalPages()).isEqualTo(2); //전체 페이지 번호
assertThat(page.isFirst()).isTrue(); //첫번째 항목인가?
assertThat(page.hasNext()).isTrue(); //다음 페이지가 있는가?
```

``PageRequest``는 추상 클래스인 ``AbstractPageRequest``를 상속받고, 해당 추상 클래스는 ``Pageable`` 인터페이스를 implements 한다.  

``PageRequest``는 첫 번째 파라미터에 현재 페이지를, 두 번째 파라미터에는 조회할 데이터 수를 입력한다. 추가적으로 정렬 정보도 줄 수가 있다. 

 

**Page 인터페이스**

```java
public interface Page<T> extends Slice<T> {
	int getTotalPages(); //전체 페이지 수
	long getTotalElements(); //전체 데이터 수
	<U> Page<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

**Slice 인터페이스**  

```java
public interface Slice<T> extends Streamable<T> {
	int getNumber(); //현재 페이지
	int getSize(); //페이지 크기
	int getNumberOfElements(); //현재 페이지에 나올 데이터 수
	List<T> getContent(); //조회된 데이터
	boolean hasContent(); //조회된 데이터 존재 여부
	Sort getSort(); //정렬 정보
	boolean isFirst(); //현재 페이지가 첫 페이지 인지 여부
	boolean isLast(); //현재 페이지가 마지막 페이지 인지 여부
	boolean hasNext(); //다음 페이지 여부
	boolean hasPrevious(); //이전 페이지 여부
	Pageable getPageable(); //페이지 요청 정보
	Pageable nextPageable(); //다음 페이지 객체
	Pageable previousPageable();//이전 페이지 객체
	<U> Slice<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

내부 구현이 이렇기 때문에 Slice로는 토탈 쿼리를 알 수 없는 것이다.  
``Page``에 ``getTotalPages()`` 와 ``getTotalElements()`` 가 있기 때문이다.  
``Slice``는 limit을 +1 한 값을 주기 때문에 더보기 등의 기능에서 쓰일 수가 있다.  
``List``로 받아오게되면 그냥 해당 ``Pageable``에 대한 응답만을 갖고온다. +1 도 하지 않는다.  

count 쿼리는 모든 데이터를 퍼와야 하기 때문에 꽤나 부담이 크다.  
이 count 쿼리를 따로 분리해서 사용할 수도 있다.  

```java
@Query(value = "select m from Member m", countQuery = "select count(m.username) from Member m")
Page<Member> findMemberAllCountBy(Pageable pageable);
```

해당 count 쿼리를 따로 지정안해주면 조인한 상태로 count를 하게되는데, left outer join 처럼 조인을 해서 count 할 필요가 없을 때가 있다. 그럴 때에 저렇게 따로 지정해서 사용할 수가 있다.  

그리고 엔티티를 그대로 api 리턴해주면 안된다. (해당 필드 값이 바뀌게 되면 api 가 깨짐, 보안문제)  

```java
Page<Member> page = memberRepository.findByAge(10,pageRequest);
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```

다음과 같이 사용할 수가 있다.  

그냥 가장 상위의 값을 간단하게 가져오고 싶을 때에 ``List<Member> findTop3By()`` 처럼 Top,First를 사용하면 된다.  
[관련문서](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.limitquery-
result)  



UPDATE문을 통해 한 번에 수정을 하는 것을 JPA 에서는 벌크성 수정 쿼리라고 부른다.  

```java
@Modifying
@Query("update Member m set m.age = m.age+1 where m.age >=:age")
int bulkAgePlus(@Param("age") int age);

@Test
public void bulkUpdate() throws Exception {
	//given
	memberRepository.save(new Member("member1", 10));
	memberRepository.save(new Member("member2", 19));
	memberRepository.save(new Member("member3", 20));
    memberRepository.save(new Member("member4", 21));
	memberRepository.save(new Member("member5", 40));
	//when
	int resultCount = memberRepository.bulkAgePlus(20);
	//then
	assertThat(resultCount).isEqualTo(3);
}
```

벌크성 수정, 삭제 쿼리는 ``@Modifying`` 어노테이션을 사용해야 한다.  
사용하지 않으면 ``QueryExecutionRequestException`` 이 발생한다.  
그런데, 벌크성 쿼리는 DB에 바로 접근한다는 점이 문제다. 그러니깐, 위의 테스트에서는 1차 캐시에 해당 member1~5 들의 정보가 저장되어 있다. 그렇기에 만약 findByUsername member5 로 정보를 가지고 오게되면 41이 아닌 40이 나온다는 것이다. 이때문에 벌크성 쿼리 후에 ``em.clear()``를 해주어야 한다. ``em.flush()``는 JPQL 쿼리를 실행하기 이전에 수행되기 때문에 생략해줘도 된다.  

``@Modifying(clearAutomatically = true)`` 를 설정하면 자동으로 클리어 연산을 해주게 된다.  
권장하는 것은 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산을 하는 것이고, 만약에 있다면 벌크연산을 수행한 후에 영속성 컨텍스트를 초기화하는 것이다.  



**EntityGraph**  

지연로딩일 때에 해당 정보를 가지고 오려고 할 때에 N+1 문제가 발생할 수 있다. 그래서 이 떄에 페치조인을 사용하게 된다.  

```JAVA
@Query("select m from Memberm left join fetch m.team")
List<Member> findMemberFetchJoin();
```

이제 이것을 ``@EntityGraph``를 통해 쉽게 처리할 수가 있다.  

```java
@Override
@EntityGraph(attributePaths ={"team"})
List<Member> findAll();

@EntityGraph(attributePaths = {"team"})
@Query("select m from Member m")
List<Member> findMemberEntityGraph();

@EntityGraph(attributePaths ={"team"})
List<Member> findByUsername(@Param("username") String username)
```

다음과 같이 하면 간편하게 페치 처리를 할 수가 있다.  



**쿼리힌트**  

```java
Member member = memberRepository.findByUsername("member1");
member.setUsername("member2");

em.flush()
```

일반적으로 해당 정보를 받아올 때에 스냅샷을 찍어서 원본을 관리한다. 그리고 이를 통해 더티체킹을 할 수가 있다. 그런데 만약 값을 수정할 일이 전혀없고 읽기만 한다면 스냅샷을 저장하는 과정 등은 의미없는 비용소모일 뿐일 것이다.  

```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value = "true"))
Member findReadOnlyByUsername(String username);
```

처음의 ``findByUsername`` 은 내부의 Username이 수정되었기 때문에 flush() 에서 update 쿼리가 나가게 된다. 더티체킹을 했기 때문이다.  
그런데, 밑의 코드는 hibernate에서 제공하는 ``org.hibernate.readOnly``를 이용해서 스냅샷을 관리하지 않고 그냥 읽기만 하기 때문에 Username 등 필드가 변경이 되어도 flush()를 수행할 때에 update 쿼리를 날리지 않는다.  

모든 연산에 쿼리힌트를 적용하는 것은 의구심이 드는 방법이라고 한다. 하지만, 대량의 읽기 연산의 경우에는 해당 설정을 통해 최적화를 이루어낼 수 있다.  



**Auditing**  

등록일, 수정일, 등록자, 수정자 등을 간편하게 추가할 수가 있다.  

``@EnableJpaAuditing`` 을 스프링 부트 설정 클래스에 적용한다. 그냥 main 있는 곳에 넣으면 된다.  
``@EntityListeners(AUditingEntityListener.class)`` 는 적용할 엔티티에 설정한다.  

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
@Getter
public class BaseEntity {
    
	@CreatedDate
	@Column(updatable = false)
	private LocalDateTime createdDate;
    
	@LastModifiedDate
	private LocalDateTime lastModifiedDate;
    
	@CreatedBy
	@Column(updatable = false)
	private String createdBy;
    
	@LastModifiedBy
	private String lastModifiedBy;
}
```

다음과 같이 사용하면 된다. ``@CreatedDate``,``@LastModifiedDate``,``@CreatedBy``,``@LastModifiedBy`` 를 이용하였다.  

앞의 2개는 ``LocalDateTime.now``를 통해 처리 된다고 치고, 뒤의 등록자 수정자는 어떻게 처리되는 것일까?  

```java
@Bean
public AuditorAware<String> auditorProvider() {	//인터페이스 하나라서 람다로 바꿔 표현했다.
	return () -> Optional.of(UUID.randomUUID().toString());
}
```

다음과 같이 빈 등록을 해주어야 한다.(그냥 main에다 일단 해주었다.)  
뭐 세션에서 꺼내서 넣어줘도 된다. 위에서는 UUID로 꺼냈을 뿐이다.  

등록시간, 수정시간은 필요하지만 등록자, 수정자는 필요 없을 수도 있기 때문에 분리해서 사용하면 된다.  

```java
public class BaseTimeEntity {
    
	@CreatedDate
	@Column(updatable = false)
	private LocalDateTime createdDate;
    
	@LastModifiedDate
	private LocalDateTime lastModifiedDate;
}

public class BaseEntity extends BaseTimeEntity {
    
	@CreatedBy
	@Column(updatable = false)
	private String createdBy;
    
	@LastModifiedBy
	private String lastModifiedBy;
}
```



**파라미터로 페이징 하기**  

이전에 PageRequest를 통한 Page 요청을 한 적이 있다.  이번에는 이런걸 해볼거다.  

```java
@GetMapping("/members")
public Page<Member> list(Pageable pageable) {
	Page<Member> page = memberRepository.findAll(pageable);
	return page;
}
```

파라미터로 Pageable를 받는데 실제는 PageRequest 객체를 생성하는 것이다.  
``/members?page=0&size=3&sort=id,desc&sort=username,desc`` 다음과 같이 요청했다.  
page 는 현재 페이지, size는 페이지 당 데이터 개수, sort는 정렬 조건이다. asc는 생략가능하다.  
``sort=id,desc`` 이거고 ``sort=username,desc``를 추가해준 것이다. 오해 노노  

근데 이제 이 요청에서 사이즈나 페이지 같은 값을 다 안넣고 ``/members`` 로만 했을 때에 default 값이 있는데 이를 변경할 수가 있다.  

```java
spring.data.web.pageable.default-page-size=20 /# 기본 페이지 사이즈/
spring.data.web.pageable.max-page-size=2000 /# 최대 페이지 사이즈/
```

다음과 같이 기본을 20으로 주면 이제 기본 20개가 나오게 될 것이다. 마음가는대로 변경하면 된다.  
위 방법은 글로벌한 설정이고, 이제 세세하게 설정할 수도 있다.  

```java
@RequestMapping(value = "/members_page", method = RequestMethod.GET)
public String list(@PageableDefault(size = 12, sort = “username”,
direction = Sort.Direction.DESC) Pageable pageable) {
...
}
```

``@PageableDefault`` 어노테이션을 사용하면 된다.  



**save() 메서드의 새로운 엔티티 구별 방법**  

save() 메서드는 해당 객체가 새로운 객체인지 기존에 있었던 객체인지 확인을 한다. 전자라면 persist()를 하고 후자는 merge()를 한다. 식별자 생성 전략이 ``@GeneratedValue`` 이면 save 호출 시점에 식별자가 없으므로 새로운 엔티티로 인식해서 정상 동작한다. 그런데 JPA 식별자 생성 전략이 ``@Id``만 사용해서 직접 할당하는 식이면 이미 식별자 값이 있는 상태로 save()를 호출하게 될 것이다. 그러면 이제 식별자 값이 있기 때문에 기존에 존재하던 엔티티로 착각해서 merge()가 호출된다. merge()는 우선 DB를 해당 pk 값으로 호출해서 값을 확인하게 되는데, 당연히 DB에 값이 없을 것이다. 그럼 이때가 되어서야 새로운 엔티티로 인지를 하게되는데 이는 비효율 적이다. 따라서 ``Persistable``를 사용해서 새로운 엔티티 여부 확인을 하는 메서드를 오버라이딩해서 이를 해결할 수가 있다.  

```java
@Transactional
@Override
public <S extends T> S save(S entity){
    if(entityInformation.isNew(entity)){
        em.persist(entity);
        return entity;
    }else{
        return em.merge(entity);
    }
}
```

save의 내부 구현코드이다.  

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {
    
	@Id
	private String id;
    
	@CreatedDate
	private LocalDateTime createdDate;
    
	public Item(String id) {
		this.id = id;
	}
    
	@Override
	public String getId() {
		return id;
	}
    
	@Override
	public boolean isNew() {
		return createdDate == null;
	}
}
```

``Persistable``를 오버라이드해서 ``getId()``와 ``isNew()``를 구현하면 된다. 위의 코드는 생성일자로 구분하는 예시이다.  

***
