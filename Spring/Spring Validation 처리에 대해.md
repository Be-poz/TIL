# Spring Validation 처리에 대해

지금까지 Validation이거나 연관된 주제에 대한 내용을 여러번 정리했었다.  


[@Valid를 이용한 Exception처리와 ThymeLeaf 처리](https://bepoz-study-diary.tistory.com/187)  
[Validator 생성 시 주의해야 할 점, Invalid target 오류](https://github.com/Be-poz/TIL/blob/master/Spring/Validator%20%EC%83%9D%EC%84%B1%20%EC%8B%9C%20%EC%A3%BC%EC%9D%98%ED%95%B4%EC%95%BC%20%ED%95%A0%20%EC%A0%90%2C%20Invalid%20target%20%EC%98%A4%EB%A5%98.md)  
[ErrorSerializer에 대해](https://github.com/Be-poz/TIL/blob/master/Spring/ErrorSerializer%EC%97%90%20%EB%8C%80%ED%95%B4.md)  
[Custom Validator 적용하기](https://github.com/Be-poz/TIL/blob/master/Spring/Custom%20Validator%20%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0.md)  
[@NotNull, @NotEmpty, @NotBlank](https://github.com/Be-poz/TIL/blob/master/Spring/%40NotNull%2C%20%40NotEmpty%2C%20%40NotBlank%20%EC%97%90%20%EB%8C%80%ED%95%B4.md)  

그런데 읽다보니 ``ResponseEntityHandlerException``를 통해서 처리하는 방법, ``BindingResult``, ``Errors``로 처리하는 방법 등 여러 선택사항이 있어 대체 이것들이 뭐길레 방법이 이렇게 있지 헷갈렸다. 그래서 정리해본다.  

<br/>

<img width="402" alt="스크린샷 2021-05-09 오후 7 21 40" src="https://user-images.githubusercontent.com/45073750/117568564-efb79d80-b0fb-11eb-98f5-09808d803384.png">
<img width="771" alt="스크린샷 2021-05-09 오후 7 21 49" src="https://user-images.githubusercontent.com/45073750/117568566-f2b28e00-b0fb-11eb-9127-dbeb8cb868ea.png">

먼저 다음과 같이 진행해보았다. ``@NotBlank``를 달아주었고 default message가 아닌 message 속성값을 줘서 내가 원하는 문구를 지정해놨다. 그리고 ``@Valid``를 걸어서 검증작업을 해주었다. 위의 결과는 400 bad_request 가 나왔다.  

<img width="749" alt="스크린샷 2021-05-09 오후 7 34 54" src="https://user-images.githubusercontent.com/45073750/117568843-a5cfb700-b0fd-11eb-91a1-652cbcd8745c.png">

그렇다면 ``Errors errors``를 추가한 현재는 어떤 값이 나왔을까? 결과는 정상적으로 리턴이 되었다.  
``BindingResult result``일 때에도 정상적으로 리턴이 되었다.  

``.hasErrors()``로 에러가 존재하는지 확인 후 return 하려는 값을 내가 원하는대로 처리해줄 수 있다.  

왜 맨 처음 케이스는 400에러가 났을까?? 왜냐하면 ``@Valid``를 어기면 ``MethodArgumentNotValidException``이 발생하기 때문이다. 이것을 ``@ExceptionHandler``를 이용해서 처리해줄 수가 있다.  

<img width="1031" alt="스크린샷 2021-05-09 오후 7 38 36" src="https://user-images.githubusercontent.com/45073750/117568957-2e4e5780-b0fe-11eb-8423-ed86691f3854.png">

다음과 같은 방법으로 말이다. 위의 코드를 보면 ``@ExceptionHandler(MethodArgumentNotValidException.class)``로 잡은 것을 볼 수 있다. 그리고 밑에 오버라이딩된 메소드도 볼 수 있다.  

둘 중 하나의 방법을 사용하면 되는데, 오버라이딩 메소드는 ``ResponseEntityExceptionHandler``를 상속받아 사용한 메소드이다. ``ResponseEntityExceptionHandler``는 Spring MVC가 제공하는 Exception들의 처리를 간단하게 등록하도록 도와주는 클래스다. 반환값이 ``ResponseEntity``인 것을 사용해보면 알 수 있다.  

<img width="1056" alt="스크린샷 2021-05-09 오후 7 43 39" src="https://user-images.githubusercontent.com/45073750/117569093-df54f200-b0fe-11eb-8605-67c0b60aed23.png">

메소드들의 일부이다. 이 곳에 ``MethodArgumentNotValidException`` 맞춤 메소드인 ``handleMethodArgumentNotValid`` 메소드를 오버라이딩해서 사용한 것이다. 이곳에서 예외를 핸들링 해버리면 파라미터에 ``BindingResult``, ``Errors``를 추가하지 않아도 내가 원하는 행위를 처리할 수가 있다.(물론 자신이 어떤 식으로 수행할 것인지에 따라 다를 수 있다)  

<br/>

그렇다면 ``BindingResult``와 ``Errors``는 무슨 차이점이 있는걸까?  
`Errors`는 유효성 검증 결과를 저장할 때 사용하는 인터페이스다.  
``BindingResult``도 인터페이스다. ``Errors``의 하위 인터페이스다. 폼 값을 커맨드 객체에 바인딩한 결과를 저장하고 에러 코드로부터 에러 메세지를 가져온다.  

<img width="990" alt="스크린샷 2021-05-09 오후 8 00 32" src="https://user-images.githubusercontent.com/45073750/117569564-3bb91100-b101-11eb-8867-3ae562ddb796.png">

(Spring 3.0 프로그래밍 - 최범균)  

***

### Reference

https://docs.spring.io/spring-framework/docs/3.1.3.RELEASE_to_3.2.0.RC2/Spring%20Framework%203.2.0.RC2/index.html?org/springframework/web/servlet/mvc/method/annotation/ResponseEntityExceptionHandler.html  

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/validation/BindingResult.html  

