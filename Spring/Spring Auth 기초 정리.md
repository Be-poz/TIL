# Spring Auth 기초 정리

## 세션이란? 

세션이란 클라이언트 별로 서버에 저장되는 정보다.  
웹 클라이언트가 서버측에 요청을 보내게 되면 서버는 클라이언트를 식별하는 session id를 생성한다.  
서버는 session id를 이용해서 key와 value를 이용한 저장소인 HttpSession을 생성한다.  
서버는 session id를 저장하고 있는 쿠키를 생성하여 클라이언트에 전송한다.  
클라이언트는 서버측에 요청을 보낼 때 session id를 가지고 있는 쿠키를 전송한다.  
서버는 쿠키에 있는 session id를 이용해서 그 전 요청에서 생성한 HttpSession을 찾고 사용한다.  

![session](https://user-images.githubusercontent.com/45073750/116081657-7ec4be80-a6d5-11eb-958e-a5727c8f8e31.png)
![session2](https://user-images.githubusercontent.com/45073750/116081668-81271880-a6d5-11eb-9b1b-641feade0d5f.png)

세션을 얻기 위해서는 request로 부터 ``getSession()`` 메서드를 호출해야 하지만, 스프링은 알아서 얻어와주기 때문에 그냥 ``HttpSession``을 써서 사용하면 된다. 이정도로 설명을 대략하고 코드를 살펴보자.  

## 테스트 코드 살펴보기

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AuthControllerTest {
    private static final String USERNAME_FIELD = "email";
    private static final String PASSWORD_FIELD = "password";
    private static final String EMAIL = "email@email.com";
    private static final String PASSWORD = "1234";

    @LocalServerPort
    int port;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
    }

    @Test
    void sessionLogin() {
        MemberResponse member = RestAssured
                .given().log().all()
                .auth().form(EMAIL, PASSWORD, new FormAuthConfig("/login/session", USERNAME_FIELD, PASSWORD_FIELD))
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/members/me")
                .then().log().all()
                .statusCode(HttpStatus.OK.value()).extract().as(MemberResponse.class);

        assertThat(member.getEmail()).isEqualTo(EMAIL);
    }
}
```

이번 테스트는 RestAssured를 사용하면서 ``auth().form``을 이용해 권한을 먼저 확인한다.  
``form()``의 선언은 다음과 같다.  
``RequestSpecification form(String userName, String password, FormAuthConfig config);``  
1,2 번째 인수에 userName과 password를 건내주고 세 번째 인수로 FormAuthConfig 타입을 보낸다.  

``FormAuthConfig``의 생성자는 다음과 같다.  
``public FormAuthConfig(String formAction, String userNameInputTagName, String passwordInputTagName)`` 첫 번째에는 action 즉, 매핑 url 값. 2, 3번째로는 userName과 password의 key 값 설정이라고 보면 되겠다.  

이제 ``/login/session``이 어떤 메서드와 이어져있는지 살펴보자.  

```java
    /**
     * ex) request sample
     * <p>
     * POST /login/session HTTP/1.1
     * content-type: application/x-www-form-urlencoded; charset=ISO-8859-1
     * host: localhost:55477
     * <p>
     * email=email@email.com&password=1234
     */
    @PostMapping("/login/session")
    public ResponseEntity sessionLogin(@RequestParam("email") String email,
                                       @RequestParam("password") String password, HttpSession session) {
        // TODO: email과 password 값 추출하기
        if (authService.checkInvalidLogin(email, password)) {
            throw new AuthorizationException();
        }

        // TODO: Session에 인증 정보 저장 (key: SESSION_KEY, value: email값)
        session.setAttribute("value", email);

        return ResponseEntity.ok().build();
    }
```

다음과 같다. 파라미터를 받아오고 해당 값이 유효한지 확인 후 유효하다면 세션에 저장을 한다.  

``authService.checkInvalidLogin``의 구현은 따로 생성해놨는데, 다음과 같다.  

```java
    public boolean checkInvalidLogin(String principal, String credentials) {
        return !EMAIL.equals(principal) || !PASSWORD.equals(credentials);
    }
/*
    private static final String EMAIL = "email@email.com";
    private static final String PASSWORD = "1234";
```

이제 ``/member/me``로 api를 날려보자.  

```java
    /**
     * ex) request sample
     * <p>
     * GET /members/me HTTP/1.1
     * cookie: JSESSIONID=E7263AC9557EF658C888F02EEF840A19
     * accept: application/json
     */
    @GetMapping("/members/me")
    public ResponseEntity findMyInfo(HttpSession session) {
        // TODO: Session을 통해 인증 정보 조회하기 (key: SESSION_KEY)
        String email = (String) session.getAttribute("value");
        MemberResponse member = authService.findMember(email);
      /*
          public MemberResponse findMember(String principal) {
       			 return new MemberResponse(1L, principal, 10);
   			 }
    */
        return ResponseEntity.ok().body(member);
    }
```

아까 세션에다가 email 값을 저장해뒀었는데 그것을 가지고와서 MemberResponse를 반환해주고 ResponseEntity의 body안에 넣어준 후 리턴한다.  

<br/>

```java
//    private static final String EMAIL = "email@email.com";
//    private static final String PASSWORD = "1234";

		@Test
    void tokenLogin() {
        String accessToken = RestAssured
                .given().log().all()
                .body(new TokenRequest(EMAIL, PASSWORD))	//email, password 필드를 가진 dto
                .contentType(MediaType.APPLICATION_JSON_VALUE)
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().post("/login/token")
                .then().log().all().extract().as(TokenResponse.class).getAccessToken();

        MemberResponse member = RestAssured
                .given().log().all()
                .auth().oauth2(accessToken)
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/members/you")
                .then().log().all()
                .statusCode(HttpStatus.OK.value()).extract().as(MemberResponse.class);

        assertThat(member.getEmail()).isEqualTo(EMAIL);
    }
```

그 다음 테스트는 JWT이다. 먼저 ``/login/token``으로 post 요청을 보내 토큰을 얻어온다.  

```java
    /**
     * ex) request sample
     * <p>
     * POST /login/token HTTP/1.1
     * accept: application/json
     * content-type: application/json; charset=UTF-8
     * <p>
     * {
     * "email": "email@email.com",
     * "password": "1234"
     * }
     */
    @PostMapping("/login/token")
    public ResponseEntity tokenLogin(@RequestBody TokenRequest tokenRequest) {
        // TODO: TokenRequest 값을 메서드 파라미터로 받아오기 (hint: @RequestBody)
      	//TokenResponse 는 String accessToken을 필드로 가진 dto
        TokenResponse tokenResponse = authService.createToken(tokenRequest);
        return ResponseEntity.ok().body(tokenResponse);
    }
```

``authService.createToken(tokenRequest);``를 통해 TokenResponse를 반환받아 body에 넣어 반환한다. JWT 부분은 뒤에서 다뤄보겠다.  

```java
        MemberResponse member = RestAssured
                .given().log().all()
                .auth().oauth2(accessToken)
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/members/you")
                .then().log().all()
                .statusCode(HttpStatus.OK.value()).extract().as(MemberResponse.class);
```

테스트 다음 부분을 보면 ``auth().oauth2(accessToken)``의 형식으로 받아온 토큰을 보내주고 ``/members/you``로 날렸다.  

```java
    /**
     * ex) request sample
     * <p>
     * GET /members/you HTTP/1.1
     * authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJlbWFpbEBlbWFpbC5jb20iLCJpYXQiOjE2MTAzNzY2NzIsImV4cCI6MTYxMDM4MDI3Mn0.Gy4g5RwK1Nr7bKT1TOFS4Da6wxWh8l97gmMQDgF8c1E
     * accept: application/json
     */
    @GetMapping("/members/you")
    public ResponseEntity findYourInfo(HttpServletRequest request) {
        // TODO: authorization 헤더의 Bearer 값을 추출하기
        String token = request.getHeader("authorization").split(" ")[1];
        MemberResponse member = authService.findMemberByToken(token);
        return ResponseEntity.ok().body(member);
    }
```

매핑된 메서드를 살펴보면 토큰을 통해 MemberResponse를 반환하고 body에 넣어 반환해준 것을 볼 수가 있다.  

<br/>

## JWT란 무엇인가??  

쿠키 인증은 쿠키에 아이디나 암호와 같은 사용자 정보를 담아 서버로 보내게 되는데, HTTP 방식의 통신을 사용할 경우 제 3자가 해당 정보를 염탐할 수 있다. 세션의 경우 세션ID를 보내므로 쿠키에 비해 보안성이 높다고 볼 수 있지만 서버에 추가적인 데이터베이스 공간이 필요하다는 단점이 있다. 이러한 단점들을 해결할 수 있는 방법이 바로 토큰 기반 인증이다. 데이터가 인코딩이 되어있긴 하지만 누구나 디코딩을 할 수 있어서 데이터 유출에 대한 피해가 있을 수 있지만 서명 부분은 헤더와 페이로드를 통해 만들어지기 때문에 데이터 변조 후 재전송하는 것을 막을 수 있다.

JWT는 Json Web Token의 줄임말이며, 두 개체에서 JSON 객체를 사용하여 가볍고 자가수용적인 방식으로 정보를 안전성 있게 전달해준다. JWT는 필요한 모든 정보를 자체적으로 지니고 있다. 기본정보, 전달 할 정보 그리고 검증됐다는 것을 증명해주는 signature를 포함하고 있다. 따라서 자가수용적이라고 부른다.  

JWT는 '.' 을 구분자로 3가지의 문자열로(헤더, 내용, 서명) 나눠져있다.  

헤더는 토큰의 타입과, 알고리즘 종류를 알려준다. 타입은 JWT이며, 알고리즘은 보통 SHA256 or RSA를 사용한다.  

내용(payload) 부분에는 토큰에 담을 정보가 들어가 있다. 여기에 담는 정보의 한 조각을 클레임(claim)이라고 부르고, 이것은 name/value 의 한 쌍으로 이뤄져있다. 토큰에는 여러개의 클레임들을 넣을 수 있다. 클레임의 종류는 크게 등록된(registered) 클레임, 공개(public) 클레임, 비공개(private) 클레임으로 나뉜다.  

서명(signature) 부분은 헤더의 인코딩 값과, 정보의 인코딩 값을 합친 후 주어진 비밀키로 해쉬를 하여 생성된다.  

JWT의 흐름은 다음과 같다.  

![jwt](https://user-images.githubusercontent.com/45073750/116086855-3b6d4e80-a6db-11eb-99f5-3a8a00296969.png)

애플리케이션이 실행될 때, JWT를 static 변수와 로컬 스토리지에 저장하게 된다. static 변수에 저장되는 이유는 HTTP 통신을 할 때마다 JWT를 HTTP 헤더에 담아서 보내야 하는데, 이를 로컬 스토리지에서 계속 불러오면 오버헤드가 발생하기 때문이다. 클라이언트에서 JWT를 포함해 요청을 보내면 서버는 허가된 JWT인지를 검사한다. 또한 로그아웃을 할 경우 로컬 스토리지에 저장된 JWT 데이터를 제거한다. (실제 서비스의 경우에는 로그아웃 시, 사용했던 토큰을 blacklist라는 DB 테이블에 넣어 해당 토큰의 접근을 막는 작업을 해주어야 한다)  

생성된 토큰은 HTTP 통신을 할 때 Authorization이라는 key의 value로 사용된다.(위의 코드에서 본 것과 같이)  
``Authorization : <type> <credentials>``  

인증타입은 다음과 같다.  

* Basic : 사용자 아이디와 암호를 Base64로 인코딩한 값을 토큰으로 사용한다. (RFC 7617)
* Bearer : JWT 혹은 OAuth에 대한 토큰을 사용한다. (RFC 6750)
* Digest : 서버에서 난수 데이터 문자열을 클라이언트에 보낸다. 클라이언트는 사용자 정보와 nonce를 포함하는 해시값을 사용하여 응답한다. (RFC 7616)
* HOBA : 전자 서명 기반 인증 (RFC 7486)
* Mutual : 암호를 이용한 클라리언트-서버 상호 인증
* AWS4-HMAC-SHA256 : AWS 전자 서명 기반 인증

현재 우리는 JWT를 사용하기 때문에 Bearer 타입을 사용한 것이다.  

### **[ JWT 단점 및 고려사항 ]** 

- Self-contained: 토큰 자체에 정보를 담고 있으므로 양날의 검이 될 수 있다.
- 토큰 길이: 토큰의 페이로드(Payload)에 3종류의 클레임을 저장하기 때문에, 정보가 많아질수록 토큰의 길이가 늘어나 네트워크에 부하를 줄 수 있다.
- Payload 인코딩: 페이로드(Payload) 자체는 암호화 된 것이 아니라, BASE64로 인코딩 된 것이다. 중간에 Payload를 탈취하여 디코딩하면 데이터를 볼 수 있으므로, JWE로 암호화하거나 Payload에 중요 데이터를 넣지 않아야 한다.
- Stateless: JWT는 상태를 저장하지 않기 때문에 한번 만들어지면 제어가 불가능하다. 즉, 토큰을 임의로 삭제하는 것이 불가능하므로 토큰 만료 시간을 꼭 넣어주어야 한다.
- Tore Token: 토큰은 클라이언트 측에서 관리해야 하기 때문에, 토큰을 저장해야 한다.

JWT에 대해 이제 알아봤으니 위에서 생략했던 코드들을 살펴보자.  
먼저 build.gradle에 ``implementation 'io.jsonwebtoken:jjwt:0.9.1'``가 있어야 한다.  

```java
//AuthService
		public MemberResponse findMemberByToken(String token) {
        String payload = jwtTokenProvider.getPayload(token);
        return findMember(payload);
    }

    public TokenResponse createToken(TokenRequest tokenRequest) {
        if (checkInvalidLogin(tokenRequest.getEmail(), tokenRequest.getPassword())) {
            throw new AuthorizationException();
        }

        String accessToken = jwtTokenProvider.createToken(tokenRequest.getEmail());
        return new TokenResponse(accessToken);
    }

//JwtTokenProvider
@Component
public class JwtTokenProvider {
    @Value("${security.jwt.token.secret-key}")
    private String secretKey;
    @Value("${security.jwt.token.expire-length}")
    private long validityInMilliseconds;

    public String createToken(String payload) {
        Claims claims = Jwts.claims().setSubject(payload);
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMilliseconds);

        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(validity)
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .compact();
    }

    public String getPayload(String token) {
        return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);

            return !claims.getBody().getExpiration().before(new Date());
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

``Claims claims = Jwts.claims().setSubject(payload);`` subject에 대한 내용을 가진 Claim을 만들어 주었다.  
이 Claim은 ``{ sub : "..." }`` 형태로 나오게된다. 밑의 ``Jwts.builder()``들도 정보들을 하나씩 달아주는 거라고 보면된다. 위 코드의 빌더를 통해 만들어진 토큰의 Claim은 다음과 같다.  
``{sub=email@email.com, iat=1619448551, exp=1619452151}``  
내가 원하는 key와 value를 추가해줄 수도 있다.  

```java
Jwts.builder()
  .claim("password", password)
```

``Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().getSubject();``  
이 코드를 이용하여 Claims의 subject 정보를 가지고 올 수 있다.  
``getBody()`` 까지만 쓰면 Claims가 반환이 된다. ``getSubject()``, ``getIssuedAt()``, ``getExpiration()``과 같은 메서드들이 존재한다. 내가 만약에 ``.claim("password", password)`` 이런 형식으로 추가했고 이 값을 갖고 오고 싶다면, ``.getOrDefault(key, defulatValue)``를 통해 가져올 수 있다.  

참고로, ``@Value(${...})`` 이 부분은 application.yml에 설정한 값을 가지고 온 것을 나타낸 것이다.  

<img width="475" alt="스크린샷 2021-04-26 오후 10 56 45" src="https://user-images.githubusercontent.com/45073750/116094716-bc7c1400-a6e2-11eb-91bf-03c7f397fc30.png">

```java
    public String createToken(TokenRequest tokenRequest) {
        Claims claims = Jwts.claims().setSubject(tokenRequest.getEmail());
        Claims claims2 = Jwts.claims().setSubject(tokenRequest.getPassword());
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMilliseconds);

        return Jwts.builder()
                .setClaims(claims)
                .setClaims(claims2)
```

다음과 같이 수정해보니깐 ``getPayload``메서드의 값이 1234 가 나왔다. 마지막에 설정한 claim으로 덮히는 것 같다.  

***

### Reference

https://m.blog.naver.com/PostView.nhn?blogId=good_ray&logNo=221360993022&proxyReferer=https:%2F%2Fwww.google.com%2F  

https://velog.io/@cada/%ED%86%A0%EA%B7%BC-%EA%B8%B0%EB%B0%98-%EC%9D%B8%EC%A6%9D%EC%97%90%EC%84%9C-bearer%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C  

https://www.baeldung.com/spring-security-session#2-injecting-the-raw-session-into-a-controller  

https://velopert.com/2389  

https://www.baeldung.com/java-json-web-tokens-jjwt