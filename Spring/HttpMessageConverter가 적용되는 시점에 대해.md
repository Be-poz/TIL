# HttpMessageConverter가 적용되는 시점에 대해

스프링을 사용할 때에 직렬화와 역직렬화 시에 메세지 컨버터를 사용하게 된다. 먼저 클라이언트의 요청을 반환하는 여러 예시 코드를 살펴보겠다.  

```java
@Controller
public class RequestBodyController {


    @GetMapping("/response-body-json-v1")
    public ResponseEntity<BepozObject> responseBodyJsonV1() {
        BepozObject bepozObject = new BepozObject();
        bepozObject.setUsername("userA");
        bepozObject.setAge(20);
        return new ResponseEntity<>(bepozObject, HttpStatus.OK);
    }

    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    @GetMapping("/response-body-json-v2")
    public BepozObject responseBodyJsonV2() {
        BepozObject bepozObject = new BepozObject();
        bepozObject.setUsername("userA");
        bepozObject.setAge(20);
        return bepozObject;
    }
}
```

우리는 위와 같은 상황에서 객체를 JSON 형태로 바꿔주기 위해 메세지 컨버터를 사용하게 된다.  

![image](https://user-images.githubusercontent.com/45073750/121855148-fdc99100-cd2d-11eb-8dab-37fc703d6e00.png)

스프링 부트 내부에서는 위의 그림과 같이 돌아간다는 것을 간략히 생각하고 있자.  
``@ResponseBody``를 사용하게되면 ``ViewResolver``대신 ``HttpMessageConverter``가 동작하게 된다.  
``String``과 같은 기본 문자처리는 ``StringHttpMessageConverter``를 이용하게되고, 객체의 경우에는 ``MappingJackson2HttpMessageConverter``를 이용하게된다. 이 외에도 여러 케이스에 맞는 컨버터들이 등록되어있어서 그것을 사용하게된다.  

``@ResponseBody``의 경우에 위의 컨버터를 사용한다고 말했지만 정확히는 다음과 같을 때 사용된다.  

* HTTP 요청 : ``@RequestBody``, ``HttpEntity(ResponseEntity)`` (``ResponseEntity``는 ``HttpEntity``를 상속받았음)  
* HTTP 응답 : ``@ResponseBody``, ``HttpEntity(ResponseEntity)``

<br/>

그렇다면, ``HttpMessageConverter``는 어떤식으로 동작할까?? 간단히 코드로 살펴보겠다.  

```java
public interface HttpMessageConverter<T> {
  
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

	List<MediaType> getSupportedMediaTypes();

	default List<MediaType> getSupportedMediaTypes(Class<?> clazz) {
		return (canRead(clazz, null) || canWrite(clazz, null) ?
				getSupportedMediaTypes() : Collections.emptyList());
	}

	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```

주석은 모두 제거하였다.  
간략히 설명하자면, ``canRead``, ``canWrite``를 통해 읽고 쓸 수 있느지에 대한 여부를 확인한다. 컨버터는 요청과 응답 두 경우 모두 사용되기 때문이다.  
이후 ``read``, ``write`` 메서드를 통해 읽고 쓰는 동작을 실행하게 된다.  

``canRead``, ``canWrite`` 메서드의 인수 타입을 보면 클래스와 미디어타입을 체크하는 것을 확인할 수가 있다.  
대표적인 컨버터는 다음과 같다. ``ByteArrayHttpMessageConverter``, ``StringHttpMessageConverter``, ``MappingJackson2HttpMessageConverter``. 우선순위도 앞에서 부터 오름차순 순으로 적었다.  

``ByteArrayHttpMessageConverter``는 ``byte[]`` 데이터를 처리한다.  
클래스 타입은 ``byte[]``이며 미디어 타입은 ``*/*`` 이다.  

``StringHttpMessageConverter``는 ``String`` 문자로 데이터를 처리한다.  
클래스 타입은 ``String``이며 미디어 타입은 ``*/*``이다.  

``MappingJackson2HttpMessageConverter``는 json 으로 데이터를 처리한다.  
클래스 타입은 객체 또는 ``HashMap``이며 미디어 타입은 ``application/json`` 관련이다.  

이게 무슨 뜻인지 한 번 예시로 살펴보자.  

```java
content-type : application/json
  
@RequestMapping
void hello(@RequestBody String data) {}
```

위 코드의 경우 일단 String 타입이니 ``ByteArrayHttpMessageConverter``는 스킵되고 그 다음 컨버터인 ``StringHttpMessageConverter``가 적합하다고 판단할 것이다. 그 후에 미디어 타입을 체크하게 되는데 해당 컨버터는 모든 미디어 타입을 받아들이니 이것 또한 통과되어 ``StringHttpMessageConverter``를 사용하게 된다.  

```java
content-type : application/json
  
@RequestMapping
void hello(@RequestBody BepozObject data) {}
```

위의 경우에는 객체타입이니 ``MappingJackson2HttpMessageConverter``가 적합하다고 판단할 것이고 미디어 타입 또한 일치하므로 해당 컨버터를 사용하게 될 것이다.  

```java
content-type : text/html
  
@RequestMapping
void hello(@RequestBody BepozObject data) {}
```

그렇다면 위의 코드는 어떨까? 객체이니 ``MappingJackson2HttpMessageConverter``가 적합하다고 판단하겠지만, 미디어 타입 체크를 하는 과정에서 탈락될 것이다. 만약 올바른 컨버터를 찾지 못한다면 이에 대한 예외가 발생하게된다.  

위와 같이 ``canRead`` 메서드를 통해 컨버팅 여부를 확인한 후에 ``read`` 메서드를 사용해 객체를 생성 후 반환한다.  
응답의 경우도 마찬가지로 ``canWrite``를 통해 판단한 후 ``write``메서드를 호출하여 HTTP 응답 메세지 바디에 데이터를 생성하게 된다. 조금 다른점은 request의 경우에는 ``content-type``의 미디어 타입을 확인했다면 response를 하는 이 경우에는 ``accept``의 미디어 타입을 확인한다는 점이 차이가 있다.(정확히는 ``@RequestMapping``의 ``produces`` 속성)  

![image](https://user-images.githubusercontent.com/45073750/121858176-4b93c880-cd31-11eb-9ad0-89b2bf1f62fd.png)

스프링은 ``HttpMessageConverter``를 구현하는 정말 많은 종류의 컨버터를 지원하는 것을 알 수가 있다.  

<br/>

간략히 ``HttpMessageConverter``에 대해 알아봤다. 그렇다면 이 컨버터는 어디서 적용되는 것일까??  

![image](https://user-images.githubusercontent.com/45073750/121858410-8eee3700-cd31-11eb-80cf-45711e1deed1.png)

``DispatcherServlet``은 위와 같은 형태로 동작을 실행한다.(인터셉터와 예외처리를 제외한 그림)  
``@RequestMapping`` 어노테이션이 핸들러 매핑과 핸들러 어댑터에 대한 정보를 주게되는데 컨버터는 이 핸들러 어댑터를 수행하는 과정에서 이루어진다.  

![image](https://user-images.githubusercontent.com/45073750/121858680-d1b00f00-cd31-11eb-8ca8-6fc2f7003a4c.png)

어댑터에서 이 핸들러를 실행할 때에 ``ArgumentResolver``를 이용해서 파라미터를 넘겨준다.  
우리가 컨트롤러 메서드를 작성할 때에 아무생각 없이 사용하던 ``@RequestBody``,  ``Model``, ``@ModelAttribute`` 등이 모두 ``ArgumentResolver``라는 것을 통해서 생성한 후에 해당 파라미터를 넘겨주었던 것이다. 이 내용은 [링크](https://github.com/Be-poz/TIL/blob/master/Spring/Spring%20MVC%20Config%20%EA%B8%B0%EC%B4%88%20%EC%A0%95%EB%A6%AC.md) 이 곳에서 한 번 다뤘었던 내용인데, 이때 사용했던 것은 추가적인 ``ArgumentResolver``를 추가하기 위해서 ``WebMvcConfigurer``의 ``addArgumentResolvers``메서드를 오버라이딩 했었다.  

스프링 코드를 보면 메세지 컨버터와 같이 기본적으로 많은 리졸버들을 제공하고 있다는 것을 확인할 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/121859396-9d891e00-cd32-11eb-887d-1f01ec75549f.png)

![image](https://user-images.githubusercontent.com/45073750/121859557-ca3d3580-cd32-11eb-99c0-e019b8c5f61c.png)

위의 코드는 ``RequestResponseBodyMethodProcessor``의 코드이다. ``RequestBody``와 ``ResponseBody`` 어노테이션이 있는지 확인하고 있는 것을 볼 수가 있고 그에 따른 argument를 생성하는 코드도 볼 수가 있다. 기본적으로 제공해주는 파라미터에 대한 내용은 [spring docs](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-arguments) 에 나와있다.  

그림에 나와있는 ``ReturnValueHandler`` 또한 스프링에서 확인할 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/121861222-9cf18700-cd34-11eb-98c1-0c19be732b20.png)

다음과 같은 인터페이스가 있고 이것을 구현하는 수많은 클래스들이 있다. 이를 구현하는 클래스의 목록도 마찬가지로 [spring-docs]( https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-return-types) 에서 확인할 수가 있다.  

위에 사진으로 있는 ``RequestResponseBodyMethodProcessor`` 코드를 자세히 살펴보자. ``resolveArguments``메서드를 보면 컨버터 어쩌고 하는 코드가 있다는 것을 확인할 수가 있다. 상세 코드는 직접 봐보도록 하자 꽤나 코드가 길다!  

결국 컨버터는 ``HandlerMethodArgumentResolver``와 ``HandlerMethodReturnValueHandler``의 과정에서 이용을 하는 것이었다.  

![image](https://user-images.githubusercontent.com/45073750/121862081-7253fe00-cd35-11eb-8769-49515eb43dc6.png)

결론적으로 위와 같은 그림의 형태를 띄게 된다.  

***

### REFERENCE

https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1