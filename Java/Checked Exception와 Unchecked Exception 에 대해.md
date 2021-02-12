# Checked Exception와 Unchecked Exception 에 대해

![exceptionImage](https://user-images.githubusercontent.com/45073750/107757999-a0b2d400-6d69-11eb-91a5-f93b7ab9b265.PNG)

Error는 시스템이 비정상적인 상황이 생겼을 때 발생한다. 시스템 레벨에서 발생하기 때문에 심각한 수준의 오류이다. 개발자가 미리 예측할 수 없기 때문에 애플리케이션에 오류에 대한 처리를 하지 않아도 된다.  

Exception은 구현한 로직에서 발생한다. 예외는 발생할 상황을 미리 예측하고 처리할 수 있다.  

|                            | Checked Exception                                            | Unchecked Exception                                          |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 처리 여부                  | 반드시 예외를 처리해야 함                                    | 명시적인 처리를 강제하지 않음                                |
| 확인 시점                  | 컴파일 단계                                                  | 실행 단계                                                    |
| 예외 발생 시 트랜잭션 처리 | roll-back 하지 않음                                          | roll-back 함                                                 |
| 대표 예외                  | Exception을 상속받는 하위 클래스 중 RuntimeException을 제외한 모든 예외<br />IOException, SQLException | RuntimeException 하위 예외<br />NullPointerException, IllegalArgumentException, IndexOutOfBoundException,SystemException |

이 두 Exception의 가장 명확한 구분 기준은 '처리 여부'이다.  

``CheckedException``의 경우에는 반드시 로직을 ``try-catch``로 감싸거나 ``throw`` 해주어야 한다.  
반면, ``UncheckedException``은 개발자의 부주의성으로 인한 예외가 대부분이고, 미리 예측하지 못했던 상황에서 발생하는 예외가 아니기 때문에 굳이 로직으로 처리를 할 필요가 없도록 만들어져 있다.  

예외 발생 시점으로도 차이를 알 수가 있다. 컴파일 단계에서 명확히 Exception 체크가 가능한 Exception이 ``CheckedException`` 그렇지 않은 것이 ``UncheckedException`` 이다. 후자는 실행 과정 중 발견된다해서 ``RuntimeException``이라고 불리는 것이다.  

그리고 roll-back 여부의 차이도 있다.  

```java
@Service
@RequiredArgsConstructor
@Transactional
public class MemberService {

  private final MemberRepository memberRepository;

  // (1) RuntimeException 예외 발생
  public Member createUncheckedException() {
    final Member member = memberRepository.save(new Member("yun"));
    if (true) {
      throw new RuntimeException();
    }
    return member;
  }

  // (2) IOException 예외 발생
  public Member createCheckedException() throws IOException {
    final Member member = memberRepository.save(new Member("wan"));
    if (true) {
      throw new IOException();
    }
    return member;
  }
}

/*
--- Ynu Log
Hibernate: 
    /* insert yun.blog.exception.member.Member
        */ insert 
        into
            member
            (id, name) 
        values
            (null, ?)
2019-05-16 00:55:16.117 TRACE 49422 --- [nio-8080-exec-2] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [yun]
2019-05-16 00:55:16.120 ERROR 49422 --- [nio-8080-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException] with root cause

java.lang.RuntimeException: null

--- Wan Log
Hibernate: 
    /* insert yun.blog.exception.member.Member
        */ insert 
        into
            member
            (id, name) 
        values
            (null, ?)
2019-05-16 00:55:43.931 TRACE 49422 --- [nio-8080-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [wan]
2019-05-16 00:55:43.935 ERROR 49422 --- [nio-8080-exec-4] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception

java.io.IOException: null
	at yun.blog.exception.member.MemberService.createCheckedException(MemberService.java:27) ~[classes/:na]
                                                                       
코드 출처: https://cheese10yun.github.io/checked-exception/
```

yun의 경우에는 exception이 일어난 후 roll-back 하고 있지만, wan의 경우에는 exception이 일어나도 roll-back되지 않고 트랜잭션이 commit까지 완료된다.  

``CheckedException``은 기본적으로 복구가 가능한 메커니즘을 가지고 있기 때문에 roll-back 되지 않는 것 같다. 하지만 현실적으로 ``CheckedException`` 예외가 발생했을 경우 복구 전략을 갖고 그것을 복구할 수 있는 경우는 그렇게 많지가 않다고 한다.  

그렇기 때문에 ``CheckedException``이 발생하면 더 구체적인 ``UncheckedException``을 발생시키고 예외에 대한 메세지를 명확하게 전달하는 것이 효과적이다. 이 방법은 Exception 처리 방법 3가지(예외 복구, 예외 회피, 예외 전환) 중 전환에 해당하는 방법이다.  

***

#### Reference

https://sabarada.tistory.com/73

https://cheese10yun.github.io/checked-exception/

https://www.nextree.co.kr/p3239/