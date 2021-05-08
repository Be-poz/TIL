# @Transactional 에 대해

트랜잭션은 데이터베이스의 상태를 변환시키는 하나의 논리적 기능을 수행하기 위한 작업의 단위 또는 한꺼번에 모두 수행되어야 할 일련의 연산들을 의미한다.  

애플리케이션을 개발하다보면 여러 쿼리를 날려야 하는 로직을 맞닥뜨리게 된다.  
만약, 쇼핑몰에서 상품을 구매할 때 잔여 금액이 충분한지 확인하고 잔여 금액이 상품 가격보다 높을 때 구매 로직으로 넘어가야 하고 상품의 재고가 있는지 확인 후에 잔여 금액을 상품 가격만큼 감소시키고 로직을 종료해야 한다고 하자. 그런데 선택상품구매 단계에서 예외가 발생하여 상품이 없음에도 불구하고 있다고 판단하였거나 잔여 금액이 감소하는 찰나에 서버의 전원이 나가서 상품을 구매했는데도 회원의 잔여 금액이 감소하지 않을 수가 있다. 이를 위해 Transaction이 탄생하게 되었다.  

Transaction은 2개 이상의 쿼리를 하나의 커넥션으로 묶어 DB에 전송하고, 이 과정에서 에러가 발생할  경우 자동으로 모든 과정을 되돌려 놓는다. 이러한 과정을 구현하기 위해 Transaction은 하나 이상의 쿼리를 처리할 때 동일한 Connection 객체를 공유하도록 한다.  

<img width="659" alt="스크린샷 2021-05-08 오후 6 05 17" src="https://user-images.githubusercontent.com/45073750/117533482-10aebe80-b028-11eb-8048-7a41ab35db3a.png">

<br/>

트랜잭션의 성질은 다음과 같다.  

* 원자성(Atomicity) : 한 트랜잭션 내에서 실행한 작업들은 하나로 간주한다.
* 일관성(Consistency) : 트랜잭션은 일관성 있는 데이터베이스 상태를 유지한다.
* 격리성(Isolation) : 동시에 실행되는 트랜잭션들이 서로 영향을 미치지 않도록 격리해야한다.
* 지속성(Durability) : 트랜잭션을 성공적으로 마치면 결과가 항상 저장되어야 한다.

<br/>

스프링에서는 트랜잭션 처리를 지원하는데 ``@Transactional`` 어노테이션을 이용하여 사용할 수가 있다.  
이를 **선언적 트랜잭션**이라고 부른다.  

``@Transactional``이 추가되면, 트랜잭션 기능이 적용된 프록시 객체가 생성된다.  
이 프록시 객체는 해당 어노테이션이 포함된 메소드가 호출될 경우, PlatformTransactionManager를 사용하여 트랜잭션을 시작하고, 정상 여부에 따라 Commit 또는 Rollback 한다.  

<br/>

다수의 트랜잭션이 동시에 실행되는 상황에선 트랜잭션 처리방식을 좀 더 고려해야 한다.  

**Dirty Read**  

* 트랜잭션 A가 어떤 값을 1에서 2로 변경하고 아직 커밋하지 않은 상황에서 트랜잭션 B가 같은 값을 읽는 경우 트랜잭션 B는 2가 조회 된다.
* 트랜잭션 B가 2를 조회 한 후 혹시 A가 롤백되면 결국 트랜잭션 B는 잘못된 값을 읽게된 것이다. 즉, 아직 트랜잭션이 완료되지 않은 상황에서 데이터에 접근을 허용할 경우 발생할 수 있는 데이터 불일치이다.

**Non-Repeatable Read**  

* 트랜잭션 A가 어떤 값 1을 읽었다. 이후 A는 같은 쿼리를 또 실행할 예정인데, 그 사이에 트랜잭션 B가 값 1을 2로 바꾸고 커밋해버리면 A가 같은 쿼리 두 번을 날리는 사이 두 쿼리의 결과가 다르게 되어 버린다.
* 즉, 한 트랜잭션에서 같은 쿼리를 두 번 실행했을 때 발생할 수 있는 데이터 불일치이다. Dirty Read에 비해서는 발생 확률이 적다.

<img width="947" alt="스크린샷 2021-05-08 오후 5 13 49" src="https://user-images.githubusercontent.com/45073750/117532085-cb3ac300-b020-11eb-8e5d-b4250d93d132.png">

**Phantom Read**

* 트랜잭션 A가 어떤 조건을 사용하여 특정 범위의 값들을 읽었다. 이후 A는 같은 쿼리를 실행할 예정인데, 그 사이에 트랜잭션 B가 같은 테이블에 값을 추가해버리면 A가 같은 쿼리 두 번을 날리는 사이 두 쿼리의 결과가 다르게 되어 버린다.
* 한 트랜잭션 안에서 일정범위의 레코드를 두 번 이상 읽을 때, 첫 번째 쿼리에서 없던 유령 레코드가 두 번째 쿼리에서 나타는 현상을 말한다.
* 즉, 한 트랜잭션에서 일정 범위의 레코드를 두 번 이상 읽을 때 발생하는 데이터 불일치이다.

<img width="1062" alt="스크린샷 2021-05-08 오후 5 12 39" src="https://user-images.githubusercontent.com/45073750/117532059-a3e3f600-b020-11eb-9033-5a9afc591992.png">

<br/>

물론 위와 같은 상황을 방지할 수 있는 속성들이 있다.  
먼저 isolation(격리수준) 부터 살펴보자.  

**DEFAULT**  
기본 격리 수준(기본 설정, DB의 Isolation Level을 따른다)  
mysql은 repeatable read이고, oracle은 read committed다.  

**READ_UNCOMMITTED(level 0)**  
커밋되지 않는(트랜잭션 처리중인) 데이터에 대한 읽기를 허용한다. 이로인해 Dirty Read가 발생할 수 있다.  
보통의 개발 환경에서는 해당 레벨을 허용하지 않기 때문에 오류가 발생한다.  

**READ_COMMITTED(level 1)**  
트랜잭션이 커밋된 확정 데이터만 읽기를 허용한다. Dirty Read를 방지한다.  

**REPEADTABLE_READ(level 2)**  
트랜잭션이 완료될 때까지 SELECT 문장이 사용하는 모든 데이터에 shared lock이 걸리므로 다른 사용자는 그 영역에 해당되는 데이터에 대한 수정이 불가능하다.  

**SERIALIZABLE(level 3)**  
트랜잭션이 완료될 때까지 SELECT 문장이 사용하는 모든 데이터에 shared lock이 걸리므로 다른 사용자는 그 영역에 해당되는 데이터에 대한 수정 및 입력이 불가능하다. Phantom REad를 방지한다. 격리 수준이 올라갈 수록 성능 저하의 우려가 있다.  

``@Transactional(isolation = Isolation.DEFAULT)``와 같이 사용한다.  

<Br/>

전파 속성은 사진 파일로 대체하겠다.  
<img width="851" alt="스크린샷 2021-05-08 오후 5 15 23" src="https://user-images.githubusercontent.com/45073750/117532138-fd4c2500-b020-11eb-8e8e-88e258bd09c5.png">

<br/>

트랜잭션을 읽기 전용으로 설정할 수 있다. 성능을 최적화하기 위해 사용할 수도 있고 특정 트랜잭션 작업 안에서 쓰기 작업이 일어나는 것을 의도적으로 방지하기 위해 사용할 수도 있다. ``@Transactional(readOnly = true)``로 사용하면 되고, true인 경우 insert, update, delete 실행 시 예외 발생, 기본 설정은 false이다.  

<br/>

선언적 트랜잭션에서는 런타임 예외가 발생하면 롤백한다. 반면에 예외가 전혀 발생하지 않거나 체크 예외가 발생하면 커밋한다. 체크 예외를 커밋 대상으로 삼은 이유는 체크 에외가 예외적인 상황에서 사용되기보다는 리턴 값을 대신해서 비즈니스적인 의미를 담은 결과를 돌려주는 용도로 많이 사용되기 때문이다. 기본동작을 바꿀 수도 있다.  
``@Transactional``의 경우 ``rollbackFor`` 또는 ``rollbackForClassName``을 사용해서 바꾼다.  
``@Transactional(rollbackFor = Exception.class)``, ``@Transactional(rollbackForClassName = {"NullPointerException"})`` 와 같이 사용한다.  

<br/>

지정한 시간 내에 메소드 수행이 완료되지 않으 경우 rollback을 수행하도록 할 수 있다.  
``@Transactional(timeout = 10)`` 와 같이 사용하고, -1 일 경우 no timeout이며 기본 값이기도 하다.  

<br/>

``@Transactional``을 클래스 레벨에 사용하면 모든 메소드에 적용할 기본값이라는 것을 나타낸다. 물론 메소드마다 개별로 어노테이션을 달아도 된다. 클래스 레벨 어노테이션은 클래스 계층 구조 상위에 있는 클래스에는 적용되지 않는다.  

어노테이션이 달린 클래스를 빈으로 등록하면 ``@EnableTransactionManagement`` 을 달아서 빈 인스턴스에 트랜잭션을 적용할 수 있다. (스프링 부트에서는 디폴트로 지정되어 있으니 따로 달아줄 필요가 없다고 한다.)  

프록시를 사용할 때는 ``@Transactional``을 public 메소드에만 적용해야 한다. protected, private 메소드나 패키지에만 접근할 수 있는 메소드에 어노테이션을 선언한다고 해서 에러가 발생하는 것은 아니지만, 선언한다고해도 지정한 트랜잭션 설정을 활용하지 못한다. public이 아닌 메소드에 어노테이션을 달아야 한다면 AspectJ를 사용하는 것이 좋다.  

동일한 ``@Transactional``속성을 여러 메소드에서 반복하고 있다면 커스텀 어노테이션을 통해 정리할 수도 있다.  

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional(transactionManager = "order", label = "causal-consistency")
public @interface OrderTx {
}
//이런식으로 말이다. 여기서 label은 트랜잭션을 나타내는 설명을 추가한 String 이다. transactionManager는 단일 어플리케이션에 안에서 독립적인 트랜잭션 매니저가 여러 개 필요할 때에 식별자를 지정한 것이다.
```

<br/>

위에서 사진으로 설명을 대체했던 Propagation의 일부 예시를 사진으로 또 봐보자...  

<img width="834" alt="스크린샷 2021-05-08 오후 11 55 37" src="https://user-images.githubusercontent.com/45073750/117543712-e7f2ed00-b058-11eb-98c8-479ca0cb18fe.png">

더 큰 스코프에 정의된 '외부' 트랜잭션이 있다면 이 트랜잭션에 참여하는 물리적 트랜잭션을 강제한다.  
같은 스레드 내에 있는 일반적인 호출 스택에 알맞은 기본 설정이다.  

메소드마다 논리적인 트잭션 스코프를 생성하는데, 내부 트랜잭션 스코프는 외부 트랜잭션 스코프와 논리적으로 독립되기 때문에, 논리적인 트랜잭션 스코프는 개별적으로 롤백 only 상태를 결정할 수 있다. 이 설정에서는 모든 논리적 스코프는 동일한 물리적 트랜잭션에 매핑된다. 따라서 내부 트랜잭션 스코프의 롤백 only 마커에 따라 외부 트랜잭션은 실제로 커밋할 수도 있고 커밋하지 않을 수 있다.  

하지만 내부 트랜잭션 스코프에서 롤백 only 마커를 설정하면, 외부 트랜잭션은 스스로 롤백을 결정한게 아니기 때문에 이런 식의 롤백은 외부 트랜잭션 입장에서 예상치 못한 동작이 되어 ``UnexpectedRollbackException``이 발생한다. 내부 트랜잭션을 롤백 only로 마킹해도 외부에서 알 수 없으면 커밋을 호출할 것이다. 롤백했음을 명확히 알리려면 외부 호출자에게 해당 익셉션을 전달해야 한다.  

<img width="805" alt="스크린샷 2021-05-09 오전 12 26 53" src="https://user-images.githubusercontent.com/45073750/117544620-4752fc00-b05d-11eb-95fa-3a4f8ff89f10.png">

``PROPAGATION_REQUIRES_NEW``는 ``REQUIRED``와는 달리 트랜잭션 스코프마다 항상 독립적인 물리적 트랜잭션을 사용하며, 외부 스코프에 있는 기존 트랜잭션엔 참여하지 않는다. 이렇게 되면 리소스 트랜잭션이 다르기 때문에, 내부 트랜잭션의 롤백 상태와는 상관 없이 내부 트랜잭션이 완료돼 잠금이 해제되는 즉시 외부 트랜잭션을 독립적으로 커밋하거나 롤백할 수 있다. 이렇게 독립적인 내부 트랜잭션은 자체 격리수준, 타임아웃, 읽기 전용 설정을 선언할 수 있으며 외부 트랜잭션의 특성을 상속하지 않는다.  

<br/>

JPA를 사용하면 항상 트랜잭션 안에서 데이터를 변경해야 한다. 트랜잭션 없이 데이터를 변경하면 예외가 발생한다. 트랜잭션을 시작하려면 엔티티 매니저에서 트랜잭션 API를 받아와야 한다. JPA는 낙관적/비관적 락을 제공해주니 찾아서 공부해보자! 

***

### Reference

https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-declarative-annotations  

https://mangkyu.tistory.com/50  

http://wiki.gurubee.net/pages/viewpage.action?pageId=21200923  

https://goddaehee.tistory.com/167  

https://coding-factory.tistory.com/226

https://stackoverflow.com/questions/40724100/enabletransactionmanagement-in-spring-boot  

http://blog.breakingthat.com/2018/04/03/springboot-transaction-%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98/  

자바 ORM 표준 JPA 프로그래밍