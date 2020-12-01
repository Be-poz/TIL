# @BeforeEach @BeforeAll @AfterEach @AfterAll 에 대해

테스트를 실행 시에 반복되는 세팅 로직 등이 있을 때에 이 어노테이션들을 사용할 수가 있다.  코드로 바로 확인해보자  

```java
@SpringBootTest
public class eachAllTest {

    @BeforeEach
    public void beforeEach() {
        System.out.println("BeforeEach");
    }

    @BeforeAll
    static void beforeAll(){
        System.out.println("BeforeAll");
    }

    @AfterEach
    public void afterEach() {
        System.out.println("AfterEach");
    }

    @AfterAll
    static void afterAll() {
        System.out.println("AfterAll");
    }

    @Test
    public void test1() throws Exception{
        System.out.println("test1");
    }

    @Test
    public void test2() throws Exception{
        System.out.println("test2");
    }
```

특이한 점은 ~All 어노테이션이 붙은 메서드들은 static이 붙었다는 것이다.  

실제로 실행시점도 static이 참조될 때 인지 스프링 로고가 뜨기전에 BeforeAll 가 print 되는 것을 알 수가 있었다.  

```
BeforeEach
test1
AfterEach
```

Each는 각 테스트들 마다 적용이 된다. 말그대로 each 각각 인 셈이다.  

1. 전체 테스트를 돌렸을 때

```
BeforeAll

BeforeEach
test1
AfterEach

BeforeEach
test2
AfterEach

AfterAll
```

2. 1개의 테스트만 돌렸을 때  

```
BeforeAll

BeforeEach
test1
AfterEach

AfterAll
```

전체 테스트 돌렸을 때에만 All 이 호출되는 것은 아닐까? 라고 생각을 해봤지만 그것은 아니었다.  
이 어노테이션들은 테스트 코드 운영 시에 굉장히 유요하다는 것을 사용해보면 알 수 있을 것이다.  

굳이 주의할 점을 뽑는다면, ``@BeforeEach``로 특정 엔티티를 생성해주고 save 시켰는데 ``@AfterEach``정의하는 것을 놓쳐서 그것을 삭제시켜주지 못해서 ``@BeforeEach`` 만 여러번 호출이 되어 문제가 생길 수 있다는 점을 유의해야 할 것이다.  

***

Reference  

[BaelDung Docs](https://www.baeldung.com/junit-before-beforeclass-beforeeach-beforeall)