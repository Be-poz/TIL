# @Profile, @ActiveProfiles 에 대해

로컬로 애플리케이션을 돌릴 때와 테스트를 돌릴 때 그리고 실제 운영을 하기위한 배포를 할 때, 각기 다른 설정을 주고 싶을 수가 있다. 또는 각 설정에 맞는 빈을 가져와서 사용하고 싶을 수가 있다.  
예를 들면, 배포를 할 때에는 실제 실무에서 사용하는 DB에다가 연결하고 싶을 것이고 테스트나 로컬 시에는 인메모리 DB를 사용하고 싶을 수도 있다.  

이때, ``@Profile``을 이용하게 된다.  어노테이션을 알아보기 이전에 properties / yml 설정 파일을 살펴보면서 ``profiles``이 무엇인가에 대해 간략히 알아보자.  

```
spring:
  profiles:
    active: local
    
---

spring:
	config:
		activate:
			on-profile: local
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:testdb
    username: sa
    password:
    
---

spring:
	config:
		activate:
			on-profile: prod
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:8080/db_name?serverTimezone=UTC&characterEncoding=UTF-8
    username: sa
    password:
```

위의 파일은 ``application.yml`` 파일이다. 하나의 파일에 여러 ``profiles``에 대한 설정을 해주었고 맨 위에 ``local``을 ``active``한다고 적어놓았다.  
이 경우에는 ``profiles: local``인 h2 DB가 돌아갈 것이다. 이 값을 ``prod``로 변경해준다면 mysql DB가 돌아갈 것이라는 것을 유추할 수 있을 것이다. 이런게 ``profiles``다. 이제 ``@Profile``을 알아보자.  

![image](https://user-images.githubusercontent.com/45073750/119254229-1b548080-bbf0-11eb-847b-662856be5718.png)

``Station``은 내부에 ``String name``필드가 있고 생성자로 그 값을 받는 객체다.  
흔한 ``@Configuration``에 ``Station``타입을 빈 등록 하기위해 ``@Bean``을 이용하고 있다. 그리고 그 밑에 ``@Profile``가 있는 것을 볼 수가 있다. 그리고 내부에 ``"test"``, ``"local"`` 을 명시해주었다. 이 뜻은 현재의 ``profile``에 따라 해당 어노테이션이 붙은 것을 실행시킨다는 뜻이다. 위의 코드는 한 눈에 알아볼 수 있게끔 클래스 안의 메서드에 붙인 것이고, 클래스 위에 붙여서 사용할 수도 있다. ``TestConfiguration.class``에는 ``Profile("test")`` 를 붙이고, ``LocalConfiguration.class``를 만들고 이 클래스에는 ``Profile("local")``를 붙여서 사용할 수 있을 것이다.  

그렇다면 ``@ActiveProfiles``는 무엇일까? 바로 테스트 수행 시에 어떤 ``profile``을 사용할 것인지 정해주는 어노테이션이다.  

![image](https://user-images.githubusercontent.com/45073750/119255390-8903ab00-bbf6-11eb-8cc5-c178d95600da.png)

이렇게 말이다. ``@ActiveProfiles("test")``라고 붙여줬으니 ``test`` ``profiles``로 돌리게 된다.(바깥에서 주입한 profile보다 우선순위가 더 높음)  

위의 yml 파일에서는 한 파일 안에 여러 ``profiles``들을 선언해주었다. 이것을 다음과 같이 나눌 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/119255626-c74d9a00-bbf7-11eb-86b4-9dded29bb683.png)

이렇게 ``application-{profiles}.properties``로 파일명을 붙여서 구분해줄 수가 있다.(yml도 가능)  
이 상태로 그냥 돌리게 되면 어떤 ``profiles``를 사용해야할지 모르기 때문에 런타임 에러가 날 수 있다. 그래서 ``application.properties``를 추가한 후에 ``spring.profiles.active = local`` 를 추가해서 정해줘야 한다. 테스트 코드에서 ``@ActiveProfiles``를 사용한 경우에는 상관 없다. 

만약 ``spring.profiles.active = local`` 을 설정하지않고 그냥 테스트를 돌리게되면 ``application.properties`` 의 값을 참조하게 된다. 왜냐하면 해당 파일은 default profile 이기 떄문이다.  

또는 터미널에서 실행할 때 ``$ java -jar -Dspring.profiles.active=prod [jar파일명] `` 이렇게 활성화 시킬 ``profiles`` 값을 설정해서 돌릴 수 있다. 따라서 ``application.properties``에 ``local``로 잡아두고 배포를 하기 위해 터미널 작업을 할 때에 ``-Dspring.profiles.active=prod``를 사용해 ``prod`` ``profiles``를 사용하면 된다.  

<br/>

<img width="311" alt="image" src="https://user-images.githubusercontent.com/45073750/192697743-408f22b8-48c8-4cde-88dc-e30546721d1f.png">

멀티모듈을 사용하고 있고 api모듈은 core 모듈을 가져와서 사용하는 경우라고 가정하자.  
core 내부에 yml 파일이 있는데 스프링부트 2.4 이전에는 별다른 설정이 없어도 해당 파일을 가져와서 사용할 수 있었지만,  
2.4 부터는 다른 설정을 해주어야 한다.  

```yaml
spring:
  config:
    import: classpath:application-core.yml
```

위와 같이 해주어야 제대로 읽어들일 수가 있다. 

***

### REFERENCE

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/ActiveProfiles.html  

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Profile.html  

https://galid1.tistory.com/664
