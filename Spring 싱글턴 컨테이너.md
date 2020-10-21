# Spring 싱글턴 컨테이너
해당 내용은 김영한 님의 스프링 핵심 원리 강의 내용을 토대로 정리했습니다.  

빈을 등록한 DI컨테이너(AppConfig)가 있다.  
클라이언트 A가 memberService를 요청하면 new memberService를 해서 return 해준다.  
클라이언트 B가 memberService를 요청하면 new memberService를 해서 return 해준다.  
클라이언트 C가 memberService를 요청하면 new memberService를 해서 return 해준다.  

```java
    @Test
    @DisplayName("스프링 없는 순수한 DI 컨테이너")
    public void pureContainer() throws Exception{
        AppConfig appConfig = new AppConfig();

        MemberService memberService = appConfig.memberService();
        MemberService memberService2 = appConfig.memberService();

        System.out.println(memberService);
        System.out.println(memberService2);

        assertThat(memberService).isSameAs(memberService2);
    }
```

스프링이 없는 순수한 DI컨테이너인 AppConfig는 요청을 받을 때마다 객체를 새로 생성한다.  
만약 고객 트래픽이 초당 100이 나오면 초당 100개의 객체가 생성되고 소멸하게 된다.  
이는 메모리 낭비를 발생시키고, 이를 해결하기 위해서 싱글턴 패턴을 이용한다.  

***
싱글턴 패턴은 알다시피 클래스의 인스턴스가 딱 1개만 생성되는 것을 보장하는 디자인 패턴이다.  
싱글턴 패턴의 예시 코드는 생략하고 이전에 정리해둔 링크를 참조바란다. [싱글턴 패턴에 대해](https://github.com/Be-poz/Books/tree/master/Head%20First%20Design%20Patterns/Singleton%20Pattern)  

싱글턴 패턴의 문제점은 다음과 같다.  
1. 싱글턴 패턴을 구현하는 코드 자체가 많이 들어간다.
2. 의존관계상 클라이언트가 구체 클래스에 의존한다. -> DIP를 위반한다.
3. 클라이언트가 구체 클래스에 의존해서 OCP 원칙을 위반할 가능성이 높다.
4. 테스트하기 어렵다.
5. 내부 속성을 변경하거나 초기화 하기 어렵다.
6. private 생성자로 자식 클래스를 만들기 어렵다.
7. 결론적으로 유연성이 떨어진다.
8. 안티패턴으로 불리기도 한다.

스프링 컨테이너는 위의 패턴의 문제점을 해결하면서, 객체 인스턴스를 싱글턴으로 관리한다.  

싱글턴 컨테이너
* 스프링 컨테이너는 싱글턴 패턴을 적용하지 않아도, 객체 인스턴스를 싱글턴으로 관리한다.
* 스프링 컨테이너는 싱글턴 컨테이너 역할을 한다. 이렇게 싱글턴 객체를 생성하고 관리하는 기능을 싱글턴 레지스트리라 한다.
* 싱글턴 패턴의 모든 단점을 해결하면서 객체를 싱글턴으로 유지할 수 있다.
* 싱글턴 패턴을 위한 지저분한 코드가 필요하지 않다.
* DIP, OCP, 테스트, private 생성자로부터 자유롭게 싱글턴을 사용할 수 있다. 

```java
    @Test
    public void springContainer() throws Exception{
        ApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

        MemberService memberService = ac.getBean("memberService", MemberService.class);
        MemberService memberService2 = ac.getBean("memberService", MemberService.class);

        System.out.println(memberService);
        System.out.println(memberService2);

        assertThat(memberService).isSameAs(memberService2);
    }
```
스프링을 이용하면 두 MemberService의 값이 동일하게 나오게 된다. 싱글턴 패턴이 적용되었기 때문이다.  

ApplicationContext에 대해 간단히 말하자면,  

ApplicationContext를 스프링 컨테이너라고 한다.  
기존에 개발자가 AppConfig를 사용해서 직접 객체를 생성하고 DI를 해줬지만, 이제는 스프링 컨테이너를 통해서 사용한다.  
스프링 컨테이너는 @Configuration이 붙은 AppConfig 설정 정보를 사용한다. 여기서 @Bean 이라 적힌 메서드를 모두 호출해서 반환된 객체를 스프링 컨테이너에 등록한다. 이렇게 스프링 컨테이너에 등록된 객체를 스프링 빈이라고 한다.  
스프링 빈은 @Bean이 붙은 메서드의 명을 스프링 빈의 이름으로 사용한다. 따로 정해줄 수도 있다.  
이전에는 개발자가 필요한 객체를 AppConfig를 사용해서 직접 조회했지만, 이제부터는 스프링 컨테이너를 통해서 스프링 빈을 찾아야 한다. 스프링 빈은 applicationContext.getBean() 메서드를 사용해서 찾을 수 있다.  

싱글턴 패턴을 사용할 때에 다음 사항을 주의해야 한다.  
* 무상태(stateless)로 설계해야 한다.  
  * 특정 클라이언트에 의존적인 필드가 있으면 안된다.
  * 특정 클라이언트가 값을 변경할 수 있는 필드가 있으면 안된다.
  * 가급적 읽기만 가능해야 한다.
  * 필드 대신에 자바에서 공유되지 않는, 지역변수, 파라미터, ThreadLocal 등을 사용해야 한다.
* 스프링 빈의 필드에 공유 값을 설정하면 큰 장애가 발생할 수 있다.  

간단한 예시를 들어보겠다.  
```java
public class StatefulService {

    private int price; //상태를 유지하는 필드

    public void order(String name, int price) {
        System.out.println("name="+name+"price="+price);
        this.price=price;
    }

    public int getPrice(){
        return price;
    }
}
```
다음의 상황에서  
```java
class StatefulServiceTest {
    @Test
    public void statefulServiceSingleton() throws Exception{
        //given
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);
        //when
        StatefulService stateful1 = ac.getBean(StatefulService.class);
        StatefulService stateful2 = ac.getBean(StatefulService.class);

        //Thread A: A사용자 10000원 주문
        stateful1.order("userA", 10000);
        //Thread B: B사용자 20000원 주문
        stateful2.order("userB", 20000);

        //Thread A : 사용자 A 주문 금액 조회
        int price = stateful1.getPrice();
        System.out.println("price="+price);

        //then
        Assertions.assertThat(stateful1.getPrice()).isEqualTo(20000);

    }
```
userA는 금액이 변하는 일을 겪게 될 것이다.  
```java
public class StatefulService {
    public int order(String name, int price) {
        System.out.println("name="+name+"price="+price);
        return price;
    }

}
```
다음과 같이 바꾸게 된다면 stateless 할 것이다.  

다시 본론으로 돌아와서 AppConfig.class를 살펴보자.  
```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService(){
        System.out.println("AppConfig.memberService");
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public OrderService orderService() {
        System.out.println("AppConfig.orderService");
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public MemberRepository memberRepository() {
        System.out.println("AppConfig.memberRepository");
        return new MemoryMemberRepository();
    }

    @Bean
    public DiscountPolicy discountPolicy() {
//      return new FixedDiscountPolicy();
        return new RateDiscountPolicy();
    }

}
```
코드를 보면 memberService를 호출하면 memberRepository() 메서드가 호출이 된다.  
orderService를 호출할 때에도 마찬가지로 memberRepository() 메서드가 호출이 된다.  
memberRepository()메서드에는 new MemoryMemberRepository()를 return 해준다.  
코드상으로는 싱글턴으로 보여지지 않는다. 코드상으로만 따졌을 때에 AppConfig를 가져다 쓰게되면  
memberRepository()에 있는 print가 3번이 찍혀야 될 것이다. 그런데 결과는 그렇지 않다.  

그 이유는 바로 ``@Configuration`` 에 있다.  
스프링은 클래스의 바이트코드를 조작하는 라이브러리를 사용한다.  

```java
    @Test
    void configurationDeep(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

        AppConfig bean = ac.getBean(AppConfig.class);
        System.out.println(bean.getClass());
    }
```
``AnnotationConfigApplicationContext``에 파라미터로 넘긴 값은 마찬가지로 스프링 빈으로 등록된다.  
이를 출력하면 ``class hello.core.AppConfig$$EnhancerBySpringCGLIB$$9e1ae7f5``의 값이 나온다.  
순수한 클래스라면 ``class hello.core.AppConfig``가 나와야 할 것이다. 그런데 ``CGLIB``같은 값이 붙었다.  

이것은 내가 만든 클래스가 아니라 스프링이 CGLIB라는 바이트코드 조작 라이브러리를 사용해서 AppConfig 클래스를 상속받은 임의의 다른 클래스를 만들고,
그 다른 클래스를 스프링 빈으로 등록한 것이다.  

CGLIB 내부는 엄청나게 복잡하겠지만, 스프링 컨테이너에 찾는 값이 있으면 그것을 반환하고 그렇지 않다면 생성해서 등록하는 로직이 있을 것이다.   
만약 ``@Configuration``을 적용하지 않게된다면 출력은 ``class hello.core.AppConfig``가 나오게 될 것이고,  
memberRepository() 또한 싱글턴이 적용되지 않아 3번이 호출될 것이다.  

***
