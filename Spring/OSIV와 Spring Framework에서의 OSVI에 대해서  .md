# OSIV와 Spring Framework에서의 OSIV에 대해서

스프링 컨테이너는 **트랜잭션 범위의 영속성 컨텍스트** 전략을 기본으로 사용한다.  
트랜잭션이 시작할 때 영속성 컨텍스트를 생성하고 트랜잭션ㄴ이 끝날 때 영속성 컨텍스트를 종료한다.  

![image](https://user-images.githubusercontent.com/45073750/141610903-843ccefe-e8c7-41da-8948-0c2e4d5f6cb6.png)

보통 비즈니스 로직을 시작하는 서비스 계층에 ``@Transactional`` 어노테이션을 선언해서 트랜잭션을 시작한다.  
이 어노테이션으로 인해 트랜잭션 AOP가 동작을 하고, 트랜잭션을 커밋하면 JPA는 flush로 변경 내용을 데이터베이스에 반영한 후에 데이터베이스 트랜잭션을 커밋한다. 예외 발생 시에는 트랜잭션을 롤백하고 종료한다. 이 때에는 flush를 호출하지 않는다.  

**트랜잭션이 같으면 같은 영속성 컨텍스트를 사용한다.**  
**트랜잭션이 다르면 다른 영속성 컨텍스트를 사용한다.**  

위의 그림대로라면 트랜잭션이 종료되면서 영속성 컨텍스트도 함께 종료되고 관리되던 엔티티들은 준영속 상태가 된다.  

```java
class OrderController {
  public String view(Long id) {
    Order order = orderService.find(id);
    Member member = order.getMember();
    order.getName();
  }
}
```

위 코드에서 Bepoz는 준영속 상태인 것이다. 즉, 변경감지와 지연로딩이 동작하지 않는다.  
따라서 ``bepoz.getName()`` 으로 지연 로딩이 불가능해서 예외가 발생하게 된다.  

뷰 레이어에서 엔티티를 수정할 일은 거의 없지만, 지연 로딩을 사용할 경우는 생길 수 있다. 이 때 문제가 발생할 수 있다.  
간단한 해결법은 1. 페치전략 수정  2. JPQL 페치 조인  3. 강제 초기화 를 들 수 있겠다.  

또는 FACADE 계층을 도입해서 서비스 계층과 프레젠테이션 계층 사이에 논리적인 의존성을 분리할 수 있다.  

```java
class MemberFacade {
  @Autowired
  MemberService bepozService;
  
  public Order find(Long id) {
    Order order = orderService.find(id);
    Member member = order.getMember();
    return order;
  }
}
```

하지만, 이런 방법은 화면별로 최적화된 엔티티를 위해 여러 조회 메서드가 필요하므로 굉장히 번거롭다.  
**이 모든 문제는 엔티티가 프레젠테이션 계층에서 준영속 상태이기 때문에 발생한다.**  

<br/>

## OSIV

OSIV(Open Session In View) : 영속성 컨텍스트를 뷰까지 열어둔다는 뜻이다.  
영속성 컨텍스트가 살아있으면 엔티티는 영속 상태로 유지된다. 따라서 뷰에서도 지연 로딩을 사용할 수 있다.  

<br/>

## 과거 OSIV: 요청 당 트랜잭션

OSIV의 핵심은 뷰에서도 지연 로딩이 가능하도록 하는 것이다.  
요청 당 트랜잭션은 클라이언트의 요청이 들어오자마자 서블릿 필터나 스프링 인터셉터에서 트랜잭션을 시작하고 요청이 끝날 때 트랜잭션도 끝내는 것이다.  

![image](https://user-images.githubusercontent.com/45073750/141610826-3d0c3b37-4599-4858-b6c2-9adad562508a.png)

하지만, 이 방식에는 문제점이 있다.  

```java
class MemberController {
	public String viewMember(Long id) {
    Member member = memberService.getMember(id);
    member.setName("Bepoz");
    model.addAttribute("member", member);
    ...
	}
}
```

뷰를 렌더링한 후에 트랜잭션 커밋 시에 flush가 일어나면서 member의 이름이 Bepoz로 변경이 될 것이다.  
이 문제를 해결하기 위해서 여러 방법이 있다.  

1. 엔티티를 읽기 전용 인터페이스로 제공

   ```java
   interface MemberView {
     public String getName();
   }
   
   @Entity
   class Member implements MemberView {
     ...
   }
   
   class MemberService {
     public MemberView getMember(Long id) {
       return memberRepository.findById(id);
     }
   }
   ```

   프레젠테이션 계층은 읽기 전용 메서드만 있는 인터페이스를 사용하므로 엔티티를 수정할 수 없다.  

2. 엔티티 래핑

   ```java
   class MemberWrapper {
     private Member member;
     
     public MemberWrapper(Member member) {
       this.member = member;
     }
     
     public String getName() {
       return member.getName();
     }
   }
   ```

3. DTO만 반환

   ```java
   class MemberDTO {
     private String name;
     
     //getter, setter
   }
   ...
   MemberDTO memberDTO = new MemberDTO();
   memberDTO.setName(member.getName());
   return membeDTO;
   ```

   단순히 데이터만 전달하는 객체를 넘긴다.

이 3가지 방법 모두 코드량이 증가한다는 단점이 있다. 개발자들끼리 프레젠테이션 계층에서 엔티티를 수정하면 안 된다고 컨벤션을 정하는 방법이 나을 수도 있지만, 쉽지 않다.  

이런 문제점들로 인해서 요청 당 트랜잭션 방식의 OSIV는 최근에는 거의 사용하지 않는다.  

이런 문제점들을 보완해서 비즈니스 계층에서만 트랜잭션을 유지하는 방식의 OSIV를 사용한다.  
스프링 프레임워크가 제공하는 OSIV가 바로 이 방식을 사용하는 OSIV다.  

<br/>

## 스프링 OSIV: 비즈니스 계층 트랜잭션

스프링 프레임워크가 제공하는 OSIV는 "비즈니스 계층에서 트랜잭션을 사용하는 OSIV"다.  
OSIV를 사용하기는 하지만 트랜잭션은 비즈니스 계층에서만 사용한다는 뜻이다.  

![image](https://user-images.githubusercontent.com/45073750/141611991-3e547a15-3a08-41d1-a1c6-a191a4b9361d.png)

1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성한다. 단 이때 트랜잭션은 시작하지 않는다.
2. 서비스 계층에서 ``@Transactional``로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다.
3. 서비스 계층이 끝나면 트랜잭션을 커밋하고 영속성 컨텍스트를 플러시한다. 이떄 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않는다.
4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 조회한 엔티티는 영속 상태를 유지한다.
5. 서블릿 필터나, 스프링 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료한다. 이때 플러시를 호출하지 않고 바로 종료한다.

### 트랜잭션 없이 읽기

영속성 컨텍스트를통한 모든 변경은 트랜잭션 안에서 이루어져야 하고, 트랜잭션 없이 flush 호출 시 예외가 발생한다.  
엔티티를 변경하지 않고 단순히 조회만 할 때는 트랜잭션이 없어도 되는데 이것을 트랜잭션 없이 읽기(Nontransactional reads) 라고 한다. 지연 로딩도 조회 기능이므로 트랜잭션 없이 읽기가 가능하다.  

스프링이 제공하는 비즈니스 계층 트랜잭션 OSIV를 정리하자면 다음과 같다.  

* 영속성 컨텍스트를 프레젠테이션 계층까지 유지한다.
* 프레젠테이션 계층에는 트랜잭션이 없으므로 엔티티를 수정할 수 없다.
* 프레젠테이션 계층에는 트랜잭션이 없지만 트랜잭션 없이 읽기를 사용해서 지연로딩을 할 수 있다.

위에서 접했던 코드를 다시 한 번 살펴보겠다.  

```java
class MemberController {
	public String viewMember(Long id) {
    Member member = memberService.getMember(id);
    member.setName("Bepoz");
    model.addAttribute("member", member);
    ...
	}
}
```

스프링이 제공하는 OSIV를 적용하면 ``setName``을 호출해서 이름을 변경시켜도 flush가 발생하지 않아 변경이 일어나지 않을 것이다.  

* 트랜잭션을 사용하는 서비스 계층이 끝날 때 트랜잭션이 커밋되면서 이미 flush를 호출했다. 스프링이 제공하는 OSIV 서블릿 필터나 OSIV 스프링 인터셉터는 요청이 끝나면 플러시를 호출하지 않고 ``em.close()``로 영속성 컨텍스트만 종료시킨다.
* 프레젠테이션 계층에서 flush를 호출해도 트랜잭션 범위 바깥이므로 예외가 발생한다.

<br/>

하지만, 주의할 점이 있다.  

```java
class MemberController {
  public String viewMember(Long id) {
    Member member = memberService.getMember(id);
    member.setName("bepoz");
    
    memberService.business();
    return "bepoz";
  }
}

@Transactional
class MemberService {
  public void business() {
    ...
  }
}
```

위 코드와 같은 경우 Member를 조회한 후에 이름을 변경시키고 ``business()`` 메서드를 호출해서 해당 영속성 컨텍스트에서 트랜잭션을 시작해버린다. 그 결과 ``business()`` 의 트랜잭션이 끝나면서 flush를 호출시켜 이름이 변경되는 쿼리가 나가게 될 것이다.  

이런 문제를 해결하는 단순한 방법은 트랜잭션이 있는 비즈니스 로직을 모두 호출하고 나서 엔티티를 변경하면 된다.  

---

### REFERENCE

자바 ORM 표준 JPA 프로그래밍 13장