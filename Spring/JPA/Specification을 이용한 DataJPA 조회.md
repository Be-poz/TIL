# Specification을 이용한 DataJPA 조회

Specification을 이용하면 쿼리를 이용하는데 있어 여러 조건들을 손쉽게 처리할 수 있고 동적인 처리가 가능하다.  
Specification을 사용하지 않을 때의 코드를 먼저 살펴보겠다.  

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    private int age;

    private LocalDate birth;

    public Member(String name, int age, LocalDate birth) {
        this.name = name;
        this.age = age;
        this.birth = birth;
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public void createMember(String name, int age, LocalDate birth) {
        memberRepository.save(new Member(name, age, birth));
    }

    public List<Member> findAll() {
        return memberRepository.findAll();
    }

    public List<Member> findAllByName(String name) {
        return memberRepository.findAllByName(name);
    }

    public List<Member> findAllByAgeGreaterThan(int age) {
        return memberRepository.findAllByAgeGreaterThan(age);
    }
}
```

```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {

    List<Member> findAllByName(String name);

    List<Member> findAllByAgeGreaterThan(int age);
}
```

```java
@SpringBootTest
class MemberServiceTest {

    @Autowired
    private EntityManager em;

    @Autowired
    private MemberService memberService;

    @BeforeEach
    void setUp() {
        memberService.createMember("kang", 19, LocalDate.of(2022, 2, 27));
        memberService.createMember("kang", 20, LocalDate.of(2022, 2, 27));
        memberService.createMember("kim", 21, LocalDate.of(2022, 2, 27));
        memberService.createMember("lee", 22, LocalDate.of(2022, 2, 27));
        memberService.createMember("min", 23, LocalDate.of(2022, 2, 27));
        em.clear();
    }

    @Test
    public void findAll() {
        assertThat(memberService.findAll()).hasSize(5);

        assertThat(memberService.findAllByName("kang")).hasSize(2);
        assertThat(memberService.findAllByName("min")).hasSize(1);

        assertThat(memberService.findAllByAgeGreaterThan(21)).hasSize(2);
    }

}
```

JpaRepository에 기본적으로 findAll() 이라는 메서드가 제공된다. 하지만, 여기에서 추가적으로 조건을 주고자 메서드 네이밍 지정을 통해서 조건을 추가했다. 하지만 이러한 일들이 정말 많아지고 그 조건 또한 복잡해진다면? 메서드도 많아질 것이고, 메서드명 또한 길어질 것이다.  

이를 Specification을 이용해서 해결해보겠다.  

```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long>, JpaSpecificationExecutor<Member> {

    List<Member> findAllByName(String name);

//    List<Member> findAllByAgeGreaterThan(int age);
}
```

```java
@Getter
public class MemberSearchCriteria {

    private int age;

    public MemberSearchCriteria(int age) {
        this.age = age;
    }
}
```

```java
@RequiredArgsConstructor
public class MemberSpecification implements Specification<Member> {

    private final MemberSearchCriteria criteria;

    @Override
    public Predicate toPredicate(Root<Member> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
        return criteriaBuilder.greaterThan(root.get("age"), criteria.getAge());
    }
}
```

```java
//MemberService
public List<Member> findAllByAgeGreaterThan(MemberSearchCriteria criteria) {
  return memberRepository.findAll(new MemberSpecification(criteria));
}
```

```java
//MemberServiceTest
void setUp() {
  memberService.createMember("kang", 19, LocalDate.of(2022, 2, 27));
  memberService.createMember("kang", 20, LocalDate.of(2022, 2, 27));
  memberService.createMember("kim", 21, LocalDate.of(2022, 2, 27));
  memberService.createMember("lee", 22, LocalDate.of(2022, 2, 27));
  memberService.createMember("min", 23, LocalDate.of(2022, 2, 27));
  em.clear();
}

@Test
public void findAllByAgeGreaterThan() {
  assertThat(memberService.findAllByAgeGreaterThan(new MemberSearchCriteria(21))).hasSize(2);
}
```

기존에 MemberRepository에 ``findAllByAgeGreaterThan(int age)`` 를 선언해서 사용하던 것을 Specification을 이용하는 것으로 변경했다. CriteriaBuilder에 정말 많은 메서드들이 주어지니 상황에 맞게끔 사용하면 된다. 첫 번째 인수에는 어떤 컬럼인지를 알려주는 인수가 들어가게 된다. 두 번째 인수는 그 기준? 이 되는 값이다.  

기존의 MemberRepository에서 ``JpaSpecificationExecutor`` 를 상속받아야 한다. 그 후 Specification 을 구현해서 toPredicate 메서드를 오버라이딩한다. MemberSearchCriteria는 requestDto 라고 보면된다.  

이번엔 findAllByName 도 변경을 해보겠다.  

```java
@Getter
public class MemberSearchCriteria {

    private int age;
    private String name;

    public MemberSearchCriteria(int age, String name) {
        this.age = age;
        this.name = name;
    }

    public MemberSearchCriteria(int age) {
        this.age = age;
    }

    public MemberSearchCriteria(String name) {
        this.name = name;
    }
}
```

```java
public class MemberSpecification {

    public static Specification<Member> byName(MemberSearchCriteria criteria) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("name"), criteria.getName());
    }

    public static Specification<Member> byAgeGreaterThan(MemberSearchCriteria criteria) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.greaterThan(root.get("age"), criteria.getAge());
    }
}
```

```java
//MemberService
public List<Member> findAllByName(MemberSearchCriteria criteria) {
  return memberRepository.findAll(MemberSpecification.byName(criteria));
}

public List<Member> findAllByAgeGreaterThan(MemberSearchCriteria criteria) {
  return memberRepository.findAll(MemberSpecification.byAgeGreaterThan(criteria));
}
```

```java
//MemberServiceTest
@BeforeEach
void setUp() {
  memberService.createMember("kang", 19, LocalDate.of(2022, 2, 27));
  memberService.createMember("kang", 20, LocalDate.of(2022, 2, 27));
  memberService.createMember("kim", 21, LocalDate.of(2022, 2, 27));
  memberService.createMember("lee", 22, LocalDate.of(2022, 2, 27));
  memberService.createMember("min", 23, LocalDate.of(2022, 2, 27));
  em.clear();
}

@Test
public void findAll() {
  assertThat(memberService.findAllByAgeGreaterThan(new MemberSearchCriteria(21))).hasSize(2);
  assertThat(memberService.findAllByName(new MemberSearchCriteria("min"))).hasSize(1);
}
```

Service의 두 메서드 모두 MemberSearchCriteria를 파라미터로 받는 상황으로 변경하였다.  
MemberSpecification 에서는 정적 메서드로 여러 경우의 조건들을 리턴해주고 있다. 따로 클래스 구분하기가 싫어서 이렇게 변경한 것이다.  

<br/>

이번에는 Service의 저 메서드를 합쳐서 재밌는 형식을 만들어보겠다.  

```java
//MemberService
public List<Member> findAllByCustomCondition(MemberSearchCriteria criteria) {
  return memberRepository.findAll(new MemberSpecification(criteria));
}
```

```java
@RequiredArgsConstructor
public class MemberSpecification implements Specification<Member> {

    private final MemberSearchCriteria criteria;

    @Override
    public Predicate toPredicate(Root<Member> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
        List<Predicate> predicates = new ArrayList<>();

        if (criteria.getAge() != 0) {
            predicates.add(criteriaBuilder.greaterThan(root.get("age"), criteria.getAge()));
        }

        if (criteria.getName() != null) {
            predicates.add(criteriaBuilder.equal(root.get("name"), criteria.getName()));
        }

        final Predicate[] predicateArray = new Predicate[predicates.size()];
        return query.where(criteriaBuilder.and(predicates.toArray(predicateArray)))
                    .distinct(true)
                    .getRestriction();
    }

    public static Specification<Member> byName(MemberSearchCriteria criteria) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("name"), criteria.getName());
    }

    public static Specification<Member> byAgeGreaterThan(MemberSearchCriteria criteria) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.greaterThan(root.get("age"), criteria.getAge());
    }
}
```

```java
// MemberServiceTest
@BeforeEach
void setUp() {
  memberService.createMember("kang", 19, LocalDate.of(2022, 2, 27));
  memberService.createMember("kang", 20, LocalDate.of(2022, 2, 27));
  memberService.createMember("kim", 21, LocalDate.of(2022, 2, 27));
  memberService.createMember("lee", 22, LocalDate.of(2022, 2, 27));
  memberService.createMember("min", 23, LocalDate.of(2022, 2, 27));
  em.clear();
}

@Test
public void findAll() {
  assertThat(memberService.findAllByAgeGreaterThan(new MemberSearchCriteria(21))).hasSize(2);
  assertThat(memberService.findAllByName(new MemberSearchCriteria("min"))).hasSize(1);

  assertThat(memberService.findAllByCustomCondition(new MemberSearchCriteria(20))).hasSize(3);
  assertThat(memberService.findAllByCustomCondition(new MemberSearchCriteria("kang"))).hasSize(2);
  assertThat(memberService.findAllByCustomCondition(new MemberSearchCriteria(19, "kang"))).hasSize(1);
}
```

클라이언트한테 입력받은 MemberSearchCriteria의 필드 여부에 따라서 Predicate 들을 조건으로 추가하고 안하고를 결정하는 Specification을 만들어보았다.  
조건에 따른 동적인 조회 쿼리가 가능하다는 것이다.  

잘만 활용하면 이렇게 재밌는 형식으로 만들어 볼 수도 있다. 아직 많은 사용 경험은 없지만, JpaRepository에서 기본적으로 제공해주는 메서드인데 추가적인 조건을 주어야할 때에 사용하기에 굉장히 좋을 것 같다고 생각한다.  

---

### REFERENCE

https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/domain/Specification.html  

https://www.baeldung.com/rest-api-search-language-spring-data-specifications  

https://groti.tistory.com/49
