## @ConfigurationProperties 에 대해

properties / yml 파일의 값 들을 가져와서 클래스에 바인딩해주는 어노테이션이다.  

![image](https://user-images.githubusercontent.com/45073750/119251822-a3805900-bbe3-11eb-92a4-c58c58fa4134.png)

위는 application.properties 이다. 이것을 다음과 같은 방법으로 가져와서 사용할 수가 있다.  
![image](https://user-images.githubusercontent.com/45073750/119251919-24d7eb80-bbe4-11eb-899d-5fdf38faa391.png)

``@ConfigurationProperteis("account")``를 이용해서 값을 바인딩 해주었다. prefix를 반드시 적어줘야 한다. 그리고 반드시 빈 등록이 되어야 하기 때문에 ``@Configuration``을 통해 빈 등록을 해준 것을 볼 수가 있다.  

``@Value("${account.name}")``을 주석처리 해놨는데, ``@ConfigurationProperties``를 사용하지 않고 ``@Value``를 이용해서 가져올 수도 있다.  

이 방법은 setter를 통한 바인딩을 하기 때문에 반드시 setter가 존재해야 한다. ``@Value``를 사용하는 경우에는 spEL를 사용할 수 있다는 단점이 있지만 타입 세이프하지 않기 때문에, 타입 세이프하길 원한다면 ``@ConfigurationProperties``를 사용하는 것이 낫다고 볼 수 있겠다(코드도 더 깔끔).  

하지만, setter를 사용한다는 점이 굉장히 찝찝하다. 이 경우에는 ``@ConstructorBinding``을 사용하면 된다.  
이 어노테이션은 생성자를 통한 바인딩을 하게끔 해준다. 그러면 setter를 삭제할 수 있다!  

![image](https://user-images.githubusercontent.com/45073750/119251885-f4904d00-bbe3-11eb-9e65-c3c4a086410d.png)

다음과 같이 말이다. 그런데 만약, 생성자에 필드가 빠져있다면 어떻게 될까? 그 경우에는 예외가 일어난다. 그런데 만약 다음과 같은 상황이라면??  

![image](https://user-images.githubusercontent.com/45073750/119251961-4933c800-bbe4-11eb-9ec9-cafad44139bc.png)

재밌게도 생성자로 바인딩하지 못했다면 setter를 찾아서 바인딩을 시켜주고 있다는 것을 확인할 수가 있었다.  
그러니, final을 붙여서 생성자에서 값이 들어가는지 확실하게 하자.  

위에서 빈 등록이 반드시 되어야 한다고 했다. 그래서 위에서는 ``@Configuration``을 사용해주었었다. ``@ConstructorBinding``을 사용하게되면 해당 클래스에서 빈 등록을 할 수가 없다. DI는 서로가 빈 등록이 되어있어야 하는데 ``name``, ``password``가 빈 등록이 되어있는 그런 필드들이 아니기 때문이라고 추측해본다. ``@ConfigurationProperties`` 클래스 들을 빈 등록하는 다른 방법이 있는데 이 방법을 이용하면 처리할 수 있다.  

![image](https://user-images.githubusercontent.com/45073750/119251836-b4c96580-bbe3-11eb-8816-060723f10c94.png)

다른 파일에서 ``@EnableConfigurationProperties``를 사용해서 등록을 해주는 방법이다. ``value = {등록하려는 클래스}`` 를 꼭 명시해주어야 한다.  

이렇게 ``@ConfigurationProperties`` 사용법에 대해서 간단히 알아보았다. Late Binding을 위해 ``@Value``를 사용해야 하는 경우도 있다고 한다. 모든 것은 해당 애플리케이션 상황에 맞추어서 하는게 역시나 베스트인 것 같다.  

***

### REFERENCE

https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConfigurationProperties.html  

https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConstructorBinding.html  

https://woowacourse.github.io/javable/post/2020-09-29-spring-properties-binding/  

https://programmer93.tistory.com/47