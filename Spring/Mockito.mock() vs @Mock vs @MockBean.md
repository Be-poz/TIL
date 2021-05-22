## Mockito.mock() vs @Mock vs @MockBean

## Mockito.mock()

```java
@Test
public void givenCountMethodMocked_WhenCountInvoked_ThenMockedValueReturned() {
    UserRepository localMockRepository = Mockito.mock(UserRepository.class);
    Mockito.when(localMockRepository.count()).thenReturn(111L);

    long userCount = localMockRepository.count();

    Assert.assertEquals(111L, userCount);
    Mockito.verify(localMockRepository).count();
}
```

``Mockito.mock()``은 클래스나 인터페이스의 mock 객체를 만들어준다.  
우리는 이 mock을 이용해서 [stub](https://ko.wikipedia.org/wiki/%EB%A9%94%EC%86%8C%EB%93%9C_%EC%8A%A4%ED%85%81)을 사용할 수 있다.  

<br/>

## @Mock

이 어노테이션은 ``Mockito.mock()``을 줄인 것이다. ``mock()`` 메서드를 사용하는 대신 어노테이션을 사용하는 것으로 대체할 수가 있다.  

```java
@RunWith(MockitoJUnitRunner.class)
public class MockAnnotationUnitTest {
    
    @Mock
    UserRepository mockRepository;
    
    @Test
    public void givenCountMethodMocked_WhenCountInvoked_ThenMockValueReturned() {
        Mockito.when(mockRepository.count()).thenReturn(123L);

        long userCount = mockRepository.count();

        Assert.assertEquals(123L, userCount);
        Mockito.verify(mockRepository).count();
    }
}
```

``MockitoJUnitRunner``을 사용해서 이를 이용할 수 있다.(``@RunWith(MockitoJUnitRunner.class)``는 Junit 버전 5 미만일 때에 사용하고, 버전이 5 이상이라면  ``@ExtendWith(MockitoExtension.class)``를 사용하면 된다)  

``@Mock``은 실패하는 경우에 에러 메세지를 통해 문제를 찾기 쉽게끔 해준다.  

```
Wanted but not invoked:
mockRepository.count();
-> at org.baeldung.MockAnnotationTest.testMockAnnotation(MockAnnotationTest.java:22)
Actually, there were zero interactions with this mock.

  at org.baeldung.MockAnnotationTest.testMockAnnotation(MockAnnotationTest.java:22)
```

다음과 같이 말이다. 이렇게 ``@Mock``을 통해 mock 객체를 만든 다음에 이를 사용하는 곳에 ``@InjectMocks``를 이용해 주입해서 사용할 수도 있다.  

```java
@ExtendWith(MockitoExtension.class)
class PathServiceTest {

    @Mock
    private StationDao stationDao;
    @Mock
    private SectionDao sectionDao;
    @InjectMocks
    private PathService pathService;
```

위의 코드는 내가 우테코 미션을 수행할 때 사용했던 코드이다. ``PathService``내부에서 ``StationDao``와 ``SectionDao``를 사용하고 있기 때문에 2개의 객체를 mock으로 생성한 다음에 ``PathService``에 ``@InjectMocks``를 이용해 주입해 주었다.  

<br/>

## @MockBean

``@MockBean``을 이용해서 Spring application context에 mock 객체를 추가해줄 수 있다. 만약 application context에 같은 타입의 빈이 존재한다면 해당 빈을 mock으로 교체한다. 같은 타입의 빈이 존재하지 않는다면, 해당 mock 객체를 빈으로 추가해준다. 이 어노테이션은 통합 테스트를 진행할 때에 mock을 사용해야하는 상황의 특정 빈의 경우에 유용하다.  

```java
@RunWith(SpringRunner.class)
public class MockBeanAnnotationIntegrationTest {
    
    @MockBean
    UserRepository mockRepository;
    
    @Autowired
    ApplicationContext context;
    
    @Test
    public void givenCountMethodMocked_WhenCountInvoked_ThenMockValueReturned() {
        Mockito.when(mockRepository.count()).thenReturn(123L);

        UserRepository userRepoFromContext = context.getBean(UserRepository.class);
        long userCount = userRepoFromContext.count();

        Assert.assertEquals(123L, userCount);
        Mockito.verify(mockRepository).count();
    }
}
```

해당 빈을 사용하기 위해서는 ``MockitoJUnitRunner``를 사용했던(코드 기준, Junit 버전 5 이상부터는``MockitoExtension``을 사용해야함) ``@Mock``와는 달리 ``SpringRunner``를 사용해야 한다.(이것 또한 Junit 버전5 이상이라면 ``@ExtendWith(SpringExtension.class)``를 사용해야한다) . 또는 ``@WebMvcTest``같은 어노테이션을 선언하고 mock 객체를 추가해준다는 뜻으로 사용할 수 있다. 그러니깐 쉽게 말하자면 mock 객체로 만들어주기 위한 어노테이션이다.  

이제 해당 어노테이션을 사용한 객체는 mock으로 객체된 것을 위의 검증하는 코드를 보면 확인할 수 있다.  

<br/>

만약 통합 테스트라면 ``@MockBean``을 사용하면 될 것이고, 여타 다른 spring 빈들이 필요가 없고 특정 빈들만 mock으로 가지고 있으면 된다면 ``@Mock``을 이용한 테스트를 진행하면 될 것 같다.

***

### REFERENCE

https://www.baeldung.com/java-spring-mockito-mock-mockbean