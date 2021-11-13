# RestTemplate과 WebClient의 Thread 이용

![image](https://user-images.githubusercontent.com/45073750/141648437-fbb5754b-63ea-45a1-909a-5e66184644dc.png)

RestTemplate은 caller thread(메서드를 호출한 쓰레드)를 이용한다.  
WebClient는 task를 큐에 넣는데, task와 thread는 조금 다른 개념이다.  

WebClient는 Spring WebFlux에서 지원하는 기술이고 내부적으로 이벤트 루프를 이용한다.  

![image](https://user-images.githubusercontent.com/45073750/141648929-8ca3d83e-865f-435c-8901-69ee119cb36d.png)

단일 쓰레드가 아닌 fix된 쓰레드 사이즈만큼의 개수로 돌아가게 된다.  

---

### REFERENCE

https://www.baeldung.com/spring-webclient-resttemplate?utm_campaign=rest&utm_content=webclient&utm_medium=email&utm_source=email  

https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-concurrency-model