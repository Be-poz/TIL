# CORS와 처리 방법에 대해

CORS(Cross-Origin Resource Sharing, CORS, 교차 출처 리소스 공유)는 추가 HTTP 헤더를 사용하여, 한 출처(origin)에서 실행 중인 웹 어플리케이션이 다른 출처(corss-origin)의 선택한 자원에 접근할 수 있는 권한을 부여하도록 브라우저에 알려주는 체제를 말한다. 만약 내가 서비스하고 있지 않은 사이트에서 세션을 요청해서 세션을 획득할 수 있다면 해당 사이트는 악의적으로 내 세션을 탈취하거나 다른 행동을 할 수 있기에 브라우저에서는 이러한 요청을 막는다. 이 떄문에 CORS가 필요한 것이다.  

여기서 origin이란 특정 페이지에 접근할 때 사용되는 URL의 Schema(프로토콜), host(도메인), 포트를 말한다.  
만약 이 3가지 중 하나라도 다르면 다른 출처인 것이다. HTTP 요청에 대해서 HTML은 기본적으로 Cross-Origin 요청이 가능하다. 왜냐하면 HTML은 Cross-Origin 정책을 따르기 때문이다. 그러나 script 태그 내에 있는 HTTP요청에 대해서는 기본적으로 Same-Origin 정책을 따르고 있기 때문에 Cross-Origin 요청이 불가능하다.  

CORS 요청은 브라우저에 요청을 보낼 때 사전요청을 먼저 보낸다. 이를 ``preflight``라고 부른다. 서버 측에서 그 요청의 메서드와 헤더에 대해 인식하고 있는지를 체크하는 것이다.  
이때, ``Access-Control-Request-Method``(실제 요청에서 어떤 메서드를 사용할 것인지 서버에게 알리기 위해), ``Access-Control-Request-Headers``(실제 요청에서 어떤 header를 사용할 것인지 서버에게 알리기 위해), ``Origin`` 이렇게 3 가지의 헤더를 갖고있는 ``OPTIONS``요청을 보내게 된다.  

이제 이 요청에 대한 응답을 내려줌으로써 CORS가 이루어지게 되는 것이다. 대략적인 설명은 여기까지 하겠다.  

<br/>

이번에 미션을 진행하면서 CORS 에러가 났고, 브라운이 해당 코드를 추가하라고 했다.  

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**").allowedMethods("*").allowedOriginPatterns("*");
    }
}
```

``addMapping``은 CORS를 적용할(Response 헤더에 달아줌) URL 패턴을 정의하는 것이다.  
``allowedMethod``는 response의 메서드 요청을 설정해주는 것이다. 위의 코드에는 ``(*)``를 사용했으므로 모두 받아들이는 것이지만 내가 어떻게 설정하느냐에 따라 get은 되고, post는 안되게 할 수 있을 것이다. (eg. ``allowedMethod("GET")``)  
allowedOriginPatterns``는 허용 origin을 정해주는 것이다.(Response 헤더에 ``Access-Control-Allow-Origin : ...``을 붙임) 위의 경우에서 나는 프론트 서버 포트를 8081, 백엔드 서버 포트를 8080으로 두었기 때문에 ``(*)``이 아닌 ``http://localhost:8081``로 둘 수 있을 것이다.  

이외에도 특정 시간동안 ``preflight``를 캐싱해두기 위한 ``maxAge``설정 등을 할 수 있다.  

![image](https://user-images.githubusercontent.com/45073750/118636165-5aa95880-b80f-11eb-8ed7-d2ab93896c48.png)

![image](https://user-images.githubusercontent.com/45073750/118636274-790f5400-b80f-11eb-8071-8f77ce8d8151.png)

이런식으로 ``preflight``가 오고간다.  

현재 애플리케이션의 흐름은 ``Post /login/token``으로 토큰 값을 얻어오고 헤더에 토큰을 넣은 뒤에 ``Get /members/me``를 요청하는 식이다. 위의 설정을 했음에도 불구하고 에러가 났다. 그 이유는 다음과 같았다.  

``Get /members/me``를 하는 과정에서 해당 path에 인터셉터를 걸어놨었다. JWT를 이용하기 때문에 토큰 검증을 하기 위함이었다.  

![image](https://user-images.githubusercontent.com/45073750/118637267-8f69df80-b810-11eb-8ebf-456215e7d12c.png)

``AuthorizationExtractor.extract(request)``내부는 ``Authorization``헤더를 찾아서 토큰을 파싱하는 절차가 이루어지는데, ``preflight``에는 해당 헤더가 없기 때문에 파싱 로직으로 들어가지 못하고 ``null``을 리턴하여 ``not authorized``를 리턴해주고 있었던 것이다.  

그래서, 코드에 추가된 것 처럼 해당 request가 ``preflight``인지 검증을하고 맞다면 true를 반환해주게끔 수정해주었다. 이렇게 고치니 정상적으로 돌아갔다!  

```java
    private boolean isPreflightRequest(HttpServletRequest request) {
        return isOptionsMethod(request) && hasOrigin(request)
                && hasRequestMethods(request) && hasRequestHeaders(request);
    }

    public boolean isOptionsMethod(HttpServletRequest request) {
        return request.getMethod().equalsIgnoreCase(HttpMethod.OPTIONS.toString());
    }

    public boolean hasOrigin(HttpServletRequest request) {
        return Objects.nonNull(request.getHeader(ORIGIN));
    }

    public boolean hasRequestMethods(HttpServletRequest request) {
        return Objects.nonNull(request.getHeader(ACCESS_REQUEST_METHOD));
    }

    public boolean hasRequestHeaders(HttpServletRequest request) {
        return Objects.nonNull(request.getHeader(ACCESS_REQUEST_HEADERS));
    }
```

코드는 다음과 같이 구성하였다. 3 가지의 요청헤더를 확인하고 method를 확인하는 식으로 진행하였다.  

이전에는 발생하지 않았는데 어제 갑작스럽게 CORS가 발생한 덕분에 이렇게 배워갈 수 있어서 정말 다행이라고 생각한다.

***

### REFERENCES

https://developer.mozilla.org/ko/docs/Glossary/Preflight_request  

https://hannut91.github.io/blogs/infra/cors  

https://dev-pengun.tistory.com/entry/Spring-Boot-CORS-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0  

백엔드 크루 웨지 & 중간곰 & 수리