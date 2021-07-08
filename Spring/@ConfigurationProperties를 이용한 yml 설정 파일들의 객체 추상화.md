# @ConfigurationProperties를 이용한 yml 설정 파일들의 객체 추상화

 프로젝트를 진행하면서 구글, 네이버, 카카오 등의 OAuth2를 사용하게 되었다. 우리 팀은 각 서비스의 secret_key 등을 application.yml에 노출하고 싶지 않았다. Jasypt 등으로 암호화도 가능하겠지만 그냥 gitignore를 활용하기로 했다.  

![image](https://user-images.githubusercontent.com/45073750/124920166-02821c00-e032-11eb-937c-177b55a135e5.png)
![image](https://user-images.githubusercontent.com/45073750/124920390-3c532280-e032-11eb-89c5-265f8b81919a.png)

다음과 설정파일을 나눴다. 그리고 application.yml 내부에 ``spring:profiles:include = oath2`` 를 사용해주었다.  
profile을 모듈처럼 사용하는 방법이다. profile에 대해서는 [이 링크](https://bepoz-study-diary.tistory.com/371)를 확인하자. ``application-oauth2.yml``은 profile이 oauth2인 application.yml 이라고 보면된다.  

이제 위의 값들을 객체에서 받아주려고 했다. 처음에는  [이 링크](https://bepoz-study-diary.tistory.com/362)에서 설명하고 있는 ``@Value``와 ``@PropertySource`` 등을 이용하여 다음과 같은 어설픈 추상화 값 주입 Config 코드가 나왔다.  

![image](https://user-images.githubusercontent.com/45073750/124921369-49244600-e033-11eb-928e-385ff351d76a.png)

그리고 서비스 계층에서도 기껏 추상화를 해놓고 모든 타입을 필드로 가지고 있는 말도 안되는 설계가 나왔었다. Enum 으로 만들자니 위의 설정값들을 넣는 작업이 힘들었던 것이다. 이 문제를 ``@ConfigurationProperties`` 를 이용해서 해결하였다.  

![image](https://user-images.githubusercontent.com/45073750/124922525-8b9a5280-e034-11eb-84ae-c72f77a214d1.png)
![image](https://user-images.githubusercontent.com/45073750/124922567-93f28d80-e034-11eb-912f-90bd04baff7c.png)
![image](https://user-images.githubusercontent.com/45073750/124922598-9bb23200-e034-11eb-993c-cb6c0363b583.png)
![image](https://user-images.githubusercontent.com/45073750/124922633-a53b9a00-e034-11eb-9245-be337bbaf6fa.png)

``@ConfigurationProperties``는 [이곳](https://bepoz-study-diary.tistory.com/370)에 정리해놓았다.  
한 줄로 말하자면 설정 파일을 객체화 시키는 어노테이션이다. 기본적으로 이 어노테이션을 사용할 때에는 ``@ConstructorBinding``이 아닌  ``@Configuration``을 붙인다. 이 방식은 Setter를 사용해서 값을 주입하게 되는데, 개발자들은 Setter 혐오가 있기 때문에... 생성자 주입방식을 사용하기 위해 ``@ConstructorBinding``을 이용한 것이다. 물론 그냥 필드에 ``@Value``를 붙여서 처리해줄 수도 있지만 위와 같이 필드가 많은 경우에 정말 지저분한 코드가 되어버린다.  

``@ConstructorBinding``의 단점은 위 설명 링크에도 나와있지만 해당 클래스 내에서 빈 등록을 해줄 수가 없다는 점이다. 그래서 따로 Config 파일을 생성해서 등록해주었다.  

![image](https://user-images.githubusercontent.com/45073750/124923758-b9cc6200-e035-11eb-80bf-2abd0f69aa9d.png)

``OAuth2Service``에서 ``OAuth2Types`` 가 있는데 내부에는 ``List<OAuth2Type>`` 형식의 변수가 있다. 스프링에서 알아서 해당 타입의 빈 들을 넣어준다. 그 덕분에 모든 ``OAuth2Type`` 타입의 빈 들을 가지고 올 수 있게 된 것이다. 만약 내가 Github에 대한 OAuth를 추가해야되는 상황이 생기게되면 그냥 설정파일에 설정값들을 기입하고  ``GithubOAuth2`` 클래스만 생성하면 그걸로 끝이다! 굉장히 Enum 처럼 동작하는 것 같은 느낌이 든다. 사실 위의 ``@ConfigurationProperties`` 방식을 사용해서 Enum 을 사용하는 방향으로 개발할 수도 있긴하다. 우리 팀에서도 둘 중 하나를 택해야 했고, 위의 방식이 더 적절하다는 판단을 내렸다!  

이 주제에 대한 리팩토링을 하면서 지금까지 공부하고 정리해온 학습들의 지식을 바로바로 써먹을 수 있어서 굉장히 뿌듯하고 좋았던 것 같다.

***

### REFERENCE

[내 깃허브 TIL ㅎ](https://github.com/Be-poz/TIL)