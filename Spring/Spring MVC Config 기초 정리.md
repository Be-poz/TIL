# Spring MVC Config 기초 정리

### addViewControllers

```java
    /**
     * 사용자가 "/"로 요청을 보냈을 때 hello.html이 응답되어야 함
     * <p>
     * WebMvcConfiguration의 addViewControllers 메서드로 설정하기
     */
		@Test
    void addViewControllers() {
        // when
        ExtractableResponse<Response> response = RestAssured
                .given().log().all()
                .when().get("/")
                .then().log().all().extract();

        // then
        Assertions.assertThat(response.statusCode()).isEqualTo(HttpStatus.OK.value());
    }

@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {
    /**
     * https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-view-controller
     *
     * "/" 요청 시 hello.html 페이지 응답하기
     */
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("hello");
    }
}
```

``addViewControllers`` 메서드를 오버라이드하여 위의 코드와 같이 입력하게되면 간단하게 view로 이동시킬 수 있다.  
Model을 넘길 필요가 없고 view 컨트롤에 대한 처리가 적을 경우 유용하게 쓰일 수 있다.  

<br/>

### addInterceptors

```java
    /**
     * 비인가 사용자가 회원 목록 조회를 했을 때 권한이 없다는 응답이 나와야 함
     * 현재는 모든 사용자가 접근이 가능하지만 LoginInterceptor를 통해 인증 과정을 거치도록 하기
     * <p>
     * 디버깅 해보기!
     * <p>
     * WebMvcConfiguration의 addInterceptors 메서드로 설정하기
     */
    @Test
    void addInterceptors() {
        RestAssured
                .given().log().all()
                .when().get("/admin/members")
                .then().log().all()
                .statusCode(HttpStatus.UNAUTHORIZED.value())
                .extract();
    }

// Config 클래스 내부
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/admin/**");
    }

//
public class LoginInterceptor extends HandlerInterceptorAdapter {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String accessToken = request.getHeader("Authorization");
        if (accessToken == null) {
            throw new AuthorizationException();
        }

        return super.preHandle(request, response, handler);
    }
}
```

``HandlerInterceptorAdapter``를 상속받아서 인터셉터 클래스를 생성하고 ``addInterceptors`` 메서드를 오버라이딩하여 해당 인터셉터를 등록할 수가 있다. 아 그 전에, 인터셉터란 컨트롤러에 들어오는 요청을 가로채는 역할을 한다.  

필터는 DispatcherServlet이 실행 되기 전에 작동하고, 인터셉터는 실행 후에 작동한다.  

``preHandle``은 컨트롤러 실행 전에 작동, ``postHandle``은 view를 렌더링 하기 전에 작동, ``afterCompletion``은 요청 처리 완료 후, 뷰를 렌더링 한 후의 콜백 핸들러 실행 결과에 따라 호출된다.  

각 메서드의 반환값이 true라면 핸들러의 다음 체인이 실행되지만 false 라면 중단하고 남은 인터셉터와 컨트롤러가 실행되지 않는다.  

![interceptor원리](https://user-images.githubusercontent.com/45073750/115983827-9a996900-a5de-11eb-9190-eb2f23555c37.png)

 <br/>

### HandlerMethodArgumentResolver

```java
    /**
     * MvcConfigController에서 @AuthenticationPrincipal LoginMember loginMember 파라미터에 값을 셋팅할 수 있게 설정하기
     * AuthenticationPrincipalArgumentResolver를 활용하기
     * <p>
     * 디버깅 해보기!
     * <p>
     * WebMvcConfiguration의 addInterceptors 메서드로 설정하기
     */
    @Test
    void addArgumentResolvers() {
        RestAssured
                .given().log().all()
                .when().get("/favorites")
                .then().log().all()
                .statusCode(HttpStatus.OK.value())
                .extract();
    }

    /**
     * https://www.baeldung.com/spring-mvc-custom-data-binder#1-custom-argument-resolver
     *
     * AuthenticationPrincipalArgumentResolver 등록하기
     */
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new AuthenticationPrincipalArgumentResolver());
    }


public class AuthenticationPrincipalArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(LoginMember.class);
//        return parameter.hasParameterAnnotation(AuthenticationPrincipal.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        return new LoginMember(1L, "email", 120);
    }
}


//
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthenticationPrincipal {
    boolean required() default true;
}


//
    @GetMapping("/favorites")
    public ResponseEntity<List<FavoriteResponse>> showFavorites(LoginMember loginMember) {
        if (loginMember == null || loginMember.getId() == null) {
            throw new AuthorizationException();
        }

        List<FavoriteResponse> favoriteResponses = Arrays.asList(
                new FavoriteResponse(),
                new FavoriteResponse(),
                new FavoriteResponse(),
                new FavoriteResponse()
        );
        return ResponseEntity.ok().body(favoriteResponses);
    }
```

``HandlerMethodArgumentResolver``는 특정 조건에 맞는 파라미터가 들어왔을 때 원하는 값을 바인딩하게끔 해주는 인터페이스다.  ``supportsParameter``메서드는 해당 파라미터를 resolver가 지원하는지에 대한 boolean을 리턴한다.  
``resolveArgument``는 내가 원하는 바인딩하는 객체를 리턴한다.  

컨트롤러에서 내가 설정한 파라미터가 들어오면 바인딩이 시작되는데, ``AuthenticationPrincipalArgumentResolver``의 ``supportsParameter``를 확인해보면 파라미터 타입으로 확인한 것이 있고, 주석 처리된 코드를 보면 어노테이션으로 확인하는 것이 있다.  

``@AuthenticationPrincipal``이 붙어있으면 실행이 되는 것이다. ``HandlerMethodArgumentResolver``를 구현한 후에 ``WebMvcConfigurer``를 구현하고 있는 Config 클래스에 ``addArgumentResolvers``메서드를 오버라이딩 해주고 등록을 해주면 작동된다.  

***

### References

https://www.baeldung.com/spring-mvc-custom-data-binder#1-custom-argument-resolver  

https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-interceptors  

https://velog.io/@kingcjy/Spring-HandlerMethodArgumentResolver%EC%9D%98-%EC%82%AC%EC%9A%A9%EB%B2%95%EA%B3%BC-%EB%8F%99%EC%9E%91%EC%9B%90%EB%A6%AC