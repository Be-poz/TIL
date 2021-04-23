# Spring CORE 기초 정리

```java
@Component
public class LineDao {
}
```

``@Component``가 붙으면 해당 클래스를 빈 등록해준다는 뜻이다. ``@Controller``, ``@Service``, ``@Repository`` 어노테이션 내부에는 모두 ``@Component``를 포함하고 있다.  
``@ComponentScan``은 ``@Component``가 붙은 클래스들을 읽어들여 빈 등록을 해준다. 보통 main 함수를 가지고 있는 클래스 위에 ``@SpringBootApplication``를 달고있는데, 해당 어노테이션 내부에 ``@ComponentScan``이 존재한다. 현재 디렉토리 위치에서부터 아래로 내려가면서 스캔을 하기 때문에 보통 ``@ComponentScan``의 위치가 최상단에 존재하는 이유가 바로 이것이다.  

<br/>

의존관계 주입 방식에는 크게 3가지 방법이 있다.  

1. 필드 주입
2. 생성자 주입
3. 수정자 주입(setter)

### 필드 주입

```java
@Service
public class StationFieldService {
    @Autowired
    private StationRepository stationRepository;
}
```

필드 주입은 코드가 간결하지만 외부에서 변경이 불가능해서 테스트하기가 힘들고, DI 프레임워크가 없으면 아무것도 할 수 없다.  
따라서, 테스트 코드나 스프링 설정을 목적으로 하는 ``@Configuration`` 같은 곳에서만 특별한 용도로 사용된다.  

<br/>

### 생성자 주입

```java
@Service
public class StationConstructorService {
    private StationRepository stationRepository;

    public StationConstructorService(StationRepository stationRepository) {
        this.stationRepository = stationRepository;
    }
}
```

생성자 호출시점에 딱 1번만 호출하는 것이 보장되는 방법이고, **불변, 필수** 의존관계에서 사용된다.  
생성자가 1개 뿐이라면 ``@Autowired``를 생략해도 자동으로 주입된다. 이를 이용해서 lombok의 ``@RequiredArgsConstructor``를 이용하면 생성자 또한 생략이 가능하다.  

<br/>

### 수정자 주입(setter)

```java
@Service
public class StationSetterService {
    private StationRepository stationRepository;

    @Autowired
    public void setStationRepository(StationRepository stationRepository) {
        this.stationRepository = stationRepository;
    }
}
```

수정자를 통한 주입 방법이며, **선택, 변경 가능성**이 있는 경우에 사용된다.  

<br/>

주입 방법은 생성자 주입을 택하는 것이 좋다. 왜냐하면 **불변**의 성질 때문이다.  

* 대부분의 의존관계 주입은 한 번 일어나면 애플리케이션 종료시점까지 의존관계를 변경할 일이 없다. 오히려 대부분의 의존관계는 애플리케이션 종료 전까지 변하면 안된다.(불변해야 한다.)
* 수정자 주입을 사용하면, setter 메서드를 public으로 열어두어야 한다.
* 누군가 실수로 변경할 수도 있고, 변경하면 안되는 메서드를 열어두는 것은 좋은 설계 방법이 아니다.
* 생성자 주입은 객체 생성 시에 딱 1번만 호출되므로 불변하게 설계할 수 있다.

생성자 주입을 이용하고 `final`키워드를 이용한다면 생성자에서 값 설정이 안된 오류를 **컴파일 시점에서 막아준다.**

<br/>

그리고 일반 메서드에서 주입받는 방식 또한 존재하지만, 잘 쓰이지 않는다.  

``@Autowired``에 관해 더 자세한 내용은 [이곳](https://github.com/Be-poz/TIL/blob/master/Spring/%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84%20%EC%A3%BC%EC%9E%85%EC%97%90%20%EB%8C%80%ED%95%B4.md) 에서 확인할 수 있다.  

***