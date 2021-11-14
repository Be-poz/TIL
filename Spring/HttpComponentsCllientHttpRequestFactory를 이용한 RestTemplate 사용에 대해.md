# HttpComponentsCllientHttpRequestFactory를 이용한 RestTemplate 사용에 대해

RestTemplate을 선언할 때에 ``new RestTemplate()`` 다음과 같이 선언해서 이용할 수가 있다.  
그런데 이용법에 대한 구글링을 찾다보니 ``HttpComponentsClientHttpRequestFactory``를 생성자 파라미터로 넘겨주어 RestTemplate을 생성하는 코드를 찾아볼 수가 있었다. 다음과 같이 말이다.  

```java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
HttpClient client = HttpClientBuilder.create()
  									.setMaxConnTotal(50)
  									.setMaxConnPerRoute(20)
  									.build();
factory.setHttpClient(client);
factory.setConnectTimeout(2000);
factory.setReadTimeout(5000);

return new RestTemplate(factory);
```

그렇다면 대체 무슨 차이가 있고, 단순히 ``new RestTemplate()``을 이용해서 생성했을 때에는 어떻게 돌아간 것일까?  
이것에 대해 한 번 정리해보겠다.  

<br/>

RestTemplate은 http 서버와의 통신을 위해서 사용된다. 그리고 HttpClient를 추상화하여 제공해준다.  
동작원리는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/141672488-94a56e45-d464-4cac-bfcd-0ccc9541f739.png)

1. 어플리케이션이 RestTemplate를 생성하고, URI, HTTP메소드 등의 헤더를 담아 요청한다.

2. RestTemplate 는 HttpMessageConverter 를 사용하여 requestEntity 를 요청메세지로 변환한다.

3. RestTemplate 는 ClientHttpRequestFactory 로 부터 ClientHttpRequest 를 가져와서 요청을 보낸다.

4. ClientHttpRequest 는 요청메세지를 만들어 HTTP 프로토콜을 통해 서버와 통신한다.

5. RestTemplate 는 ResponseErrorHandler 로 오류를 확인하고 있다면 처리로직을 태운다.

6. ResponseErrorHandler 는 오류가 있다면 ClientHttpResponse 에서 응답데이터를 가져와서 처리한다.

7. RestTemplate 는 HttpMessageConverter 를 이용해서 응답메세지를 java object(Class responseType) 로 변환한다.

8. 어플리케이션에 반환된다.

그림을 보면 처음에 의문을 가졌던 ``HttpComponentsClientHttpRequestFactory`` 와 이름이 유사한 ``ClientHttpRequestFactory``의 존재를 확인할 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/141672581-ce783f26-2549-40b2-81be-fd06bba3cd6b.png)

코드를 살펴보니 ``HttpComponentsClientHttpRequestFactory`` 상위에 존재하는 인터페이스인 것을 확인할 수가 있었다.  
그렇다면 ``RestTemplate rt = new RestTemplate();``은 대체 어떤 식으로 돌아가고 있는걸까?  
factory를 받고있는 생성자에서부터 차근히 올라가보자.  

![image](https://user-images.githubusercontent.com/45073750/141672645-811ce85a-ab43-444f-8ec0-ded072c416a8.png)

RestTemplate.java  

![image](https://user-images.githubusercontent.com/45073750/141672665-adf066f5-66e3-4d78-bfc4-21dc22f030f9.png)

RestTemplate의 상위에 존재하는 ``InterceptingHttpAccessor`` 추상 클래스의 ``setRequestFactory`` (오버라이딩) 메서드  

![image](https://user-images.githubusercontent.com/45073750/141672712-ad3a9052-817f-415f-8306-ffd1db74466d.png)

``InterceptingHttpAcceptor`` 의 상위에 존재하는 ``HttpAccessor`` 추상 클래스의 ``setRequestFactory``메서드  

바로 이곳에서 ``this.requestFactory = requestFactory;`` 를 해주고 있었다. 그렇다면 기존의 factory는 무엇일까?  

![image](https://user-images.githubusercontent.com/45073750/141672758-e23b40a1-3422-4ed9-bb80-81ea8d7e302b.png)

``SimpleClientHttpRequestFactory()`` 였다.  

![image](https://user-images.githubusercontent.com/45073750/141672777-f82cceca-375f-4c07-83a8-04b2897d9ea6.png)

![image](https://user-images.githubusercontent.com/45073750/141672940-bae70ac1-9e34-460a-8e23-6a974874f393.png)

timeout에 대한 설명을 보면 시스템의 default 값으로 돌아간다고 한다. JVM에서는 그 값이 infinite이므로  

```
-Dsun.net.client.defaultConnectTimeout=<TimeoutInMiliSec>
-Dsun.net.client.defaultReadTimeout=<TimeoutInMiliSec>
```

다음과 같은 방법으로 override 시켜줄 수가 있다고 한다.  

``HttpComponentsClientHttpRequestFactory`` 의 코드를 보면 ``HttpClient``를 set 해줄 수가 있다.(글 도입부의 코드처럼)  
RestTemplate은 기본적으로 connection pool을 사용하지 않기 때문에 연결할 때 마다 tcp connection을 맺게 된다.  
이를 ``HttpClient``를 만들어서 connection pool을 사용하는 것으로 해결할 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/141673172-652af52c-6cb1-4a3c-a470-87b2405363da.png)

``HttpComponentsClientHttpRequestFactory``의 내부 코드다. ``HttpClient``를 따로 설정해주지 않았을 경우에는 default로 system properties를 기반으로 ``HttpClient``를 생성해주고 있다는 것을 확인할 수가 있다.  

``HttpClient``를 생성해주기 위해서 ``HttpClientBuilder``를 사용할 수 있다.  
``.setMaxConnTotal()``, ``.setMaxConnPerRoute()`` 를 통해서 connection의 개수를 설정해 줄 수가 있다.  

> `setMaxConnTotal` is the total maximum connections available in the connection pool.  
>  `setMaxConnPerRoute` is the total number of connections limit to a single port or url.

``setMaxConnTotal``은 connection pool의 최대 connection 개수고,  
``setMaxConnPerRoute``는 단일 포트나 url에 대한 connection의 개수 제한이라고 한다.  

<br/>

이렇게 해서 단순히 ``new RestTemplate()``을 이용한 RestTemplate 생성과  
``HttpComponentsCllientHttpRequestFactory``를 이용한 RestTemplate 생성에 대해 알아보았다.  

요약하자면 다음과 같다.  

1. ``new RestTemplate()``으로 사용하면 여러 설정들이 default로 돌아간다.
2. ``HttpComponentsCllientHttpRequestFactory``를 이용해서 timeOut 등 세부 설정을 해줄 수가 있다.
3. ``HttpClient``를 이용해서 connection pool을 사용할 수가 있다.

그리고 RestTemplate이 deprecated 되었기 때문에 WebClient를 무조건적으로 사용해야 한다고 생각했는데 아니었다.  

> NOTE: As of 5.0, the non-blocking, reactive org.springframework.web.reactive.client.WebClient offers a modern alternative to the RestTemplate with efficient support for both sync and async, as well as streaming scenarios. The RestTemplate will be deprecated in a future version and will not have major new features added going forward.

아직 deprecated 된 것은 아니고, 미래에 될 예정이라고 한다.  
새로운 feature가 추가되지 않을 것이니 동기/비동기 두 방식으로 처리도 가능하고 계속해서 발전할 WebClient를 사용하라고 하는 것이다. 기존의 RestTemplate이 사용하는데에 문제가 없다면 그대로 들고가면 될 것 같다. 새로운 API 작성을 하는 경우나 기존의 RestTemplate로 구현한 로직이 RestTemplate가 제공하는 기능으로 동작하기 어려운 경우에는 WebClient를 사용하면 될 것 같다.

---

### REFERENCE

https://zepinos.tistory.com/34  

https://sjh836.tistory.com/141  

https://stackoverflow.com/questions/31869193/using-spring-rest-template-either-creating-too-many-connections-or-slow/  

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/client/ClientHttpRequestFactory.html  

https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/HttpClient.html  

https://stackoverflow.com/questions/56942445/what-is-the-difference-beetween-max-connections-per-route-and-max-connections-to  

https://howtodoinjava.com/spring-boot2/resttemplate/resttemplate-timeout-example/  

https://stackoverflow.com/questions/65221077/difference-between-httpclients-createsystem-vs-httpclients-createdefault  

https://stackoverflow.com/questions/47974757/webclient-vs-resttemplate