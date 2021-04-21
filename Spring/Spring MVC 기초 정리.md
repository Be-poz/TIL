# Spring 기초 정리

### 매핑 요청

```java
@RestController
@RequestMapping("/http-method")
public class HttpMethodController {
    
    @PostMapping("/users")
    public ResponseEntity createUser(@RequestBody User user) {
        Long id = 1L;
        return ResponseEntity.created(URI.create("/users/" + id)).build();
    }

    @GetMapping("/users")
    public ResponseEntity<List<User>> showUser() {
        List<User> users = Arrays.asList(
                new User("이름", "email"),
                new User("이름", "email")
        );
        return ResponseEntity.ok().body(users);
    }
}
```

``@RequestMapping`` 을 통해서 매핑을 할 수가 있다. URL, HTTP 메서드, 매개 변수, 헤더 및 미디어 유형별 세부 속성을 설정할 수 있다. 위 코드에서 볼 수 있는 것 처럼 클래스 위에 ``@RequestMapping("/http-method")`` 를 선언해줌으로써, 공통적인 path에 대한 처리를 할 수가 있다.  

``@PostMapping``, ``@GetMapping``, ``@PutMapping``, ``@DeleteMapping``, ``@PatchMapping`` 등을 사용해서 HTTP 메서드 처리를 할 수가 있다. ``@RequestMapping(value = "/http-method", method = RequestMethod.GET)`` 다음과 같이 표현할 수도 있다.  

<br/>

### MediaType

```java
@RestController
@RequestMapping("/media-type")
public class MediaTypeController {

    @PostMapping(value = "/users", consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity createUser(@RequestBody User user) {
        Long id = 1L;
        return ResponseEntity.created(URI.create("/users/" + id)).build();
    }

    @GetMapping(value = "/users", produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<List<User>> showUser() {
        List<User> users = Arrays.asList(
                new User("이름", "email"),
                new User("이름", "email")
        );
        return ResponseEntity.ok().body(users);
    }

    @GetMapping(value = "/users", produces = MediaType.TEXT_HTML_VALUE)
    public String userPage() {
        return "user page";
    }
}
```

consumes 와 produces는 매핑 범위를 좁혀준다.  
위 코드의 createUser 메서드를 보면 ``consumes = MediaType.APPLICATION_JSON_VALUE`` 속성을 가지고 있다.  
이는 APPLICATION_JSON_VALUE의 타입의 요청만 받아들인다는 것을 의미한다.  
반대로 ``produces = MediaType.APPLICATION_JSON_VALUE`` 는 해당 형태의 유형으로 응답을 한다는 것을 의미한다.  

```java
    @DisplayName("Method Arguments - @RequestParam")
    @Test
    void requestParam() {
        String name = "hello";
        RestAssured.given().log().all()
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/method-argument/users?name={name}", name)
                .then().log().all()
                .statusCode(HttpStatus.OK.value())
                .body("size()", is(2))
                .body("[0].name", is(name));
    }
```

다음과 같은 테스트에서는 JSON_VALUE 만 accept 한다고 적혀져있다. 이 경우에 ``produces = MediaType.TEXT_HTML_VALUE`` 인 메서드는 무시될 것이고, ``produces = MediaType.APPLICATION_JSON_VALUE`` 인 메서드에만 반응을 할 것이다.  

```java
   @DisplayName("Method Arguments - @RequestBody")
    @Test
    void requestBody() {
        User user = new User("이름", "email@email.com");

        RestAssured.given().log().all()
                .contentType(MediaType.APPLICATION_JSON_VALUE)
                .body(user)
                .when().post("/method-argument/users/body")
                .then().log().all()
                .statusCode(HttpStatus.CREATED.value())
                .header("Location", "/users/1")
                .body("name", is("이름"))
                .body("email", is("email@email.com"));
    }
```

produces이 아니라 ``.contentType(Mediatype.APPLICATION_JSON_VALUE)``처럼 api를 날리는 타입이 JSON_VALUE 라고 명시한 경우에는 ``consumes = MediaType.APPLICATION_JSON_VALUE`` 가 붙어있는 메서드가 무리없이 실행될 수 있을 것이다.  

<br/>

### 매개변수와 헤더를 이용하여 매핑범위 좁혀주기

위의 MediaType과 비슷하게 매개 변수와 헤더를 이용해서 매핑범위를 좁혀줄 수가 있다.  

```java
    @DisplayName("Parameter Header - Params")
    @Test
    void messageForParam() {
        RestAssured.given().log().all()
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/param-header/message?name=hello")
                .then().log().all()
                .statusCode(HttpStatus.OK.value())
                .body(is("hello"));
    }

    @DisplayName("Parameter Header - Headers")
    @Test
    void messageForHeader() {
        RestAssured.given().log().all()
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .header("HEADER", "hi")
                .when().get("/param-header/message")
                .then().log().all()
                .statusCode(HttpStatus.OK.value())
                .body(is("hi"));
    }
```

다음과 같이 헤더와 매개변수에 특정 값을 넣은 상태로 api를 쏴줬다.  

```java
    @GetMapping(value = "/message")
    public ResponseEntity<String> message() {
        return ResponseEntity.ok().body("message");
    }

    @GetMapping(value = "/message", params = "name=hello")
    public ResponseEntity<String> messageForParam() {
        return ResponseEntity.ok().body("hello");
    }

    @GetMapping(value = "/message", headers = "HEADER=hi")
    public ResponseEntity<String> messageForHeader() {
        return ResponseEntity.ok().body("hi");
    }
```

분명 같은 Get 방식에 동일한 path 이지만, ``params = "name=hello"`` 와 ``headers = "HEADER=hi"`` 의 속성을 이용하여 매핑 범위를 좁혀준 것을 볼 수가 있다.  

<br>

### URI 패턴

- `"/resources/ima?e.png"` -경로 세그먼트에서 한 문자 일치
- `"/resources/*.png"` -경로 세그먼트에서 0 개 이상의 문자와 일치
- `"/resources/**"` -여러 경로 세그먼트 일치
- `"/projects/{project}/versions"` -경로 세그먼트를 일치시키고 변수로 캡처
- `"/projects/{project:[a-z]+}/versions"` -정규식과 일치하고 변수를 캡처

기본적으로 다음과 같다. 코드를 통해 알아보자.  

```java
    @GetMapping("/users/{id}")
    public ResponseEntity<User> pathVariable(@PathVariable Long id) {
        User user = new User(id, "이름", "email");
        return ResponseEntity.ok().body(user);
    }

    @GetMapping("/patterns/?")
    public ResponseEntity<String> pattern() {
        return ResponseEntity.ok().body("pattern");
    }

    @GetMapping("/patterns/**")
    public ResponseEntity<String> patternStars() {
        return ResponseEntity.ok().body("pattern-multi");
    }
```

```java
"/uri-pattern/patterns/multi"
"/uri-pattern/patterns/all"
"/uri-pattern/patterns/all/names"
  
"/uri-pattern/patterns/a"
"/uri-pattern/patterns/a"
  
"/uri-pattern/users/1"
```

첫 3개의 uri 그룹은 ``patternStars`` 메서드로 들어간다.  
두 번째 그룹은 ``pattern`` 메서드로 들어간다.  
세 번째 그룹은 ``pathVariable`` 메서드로 들어간다.  

<br/>

### Method Parameters

```java
    /**
     * MethodArgumentController > requestParam 메서드
     * > @RequestParam 활용하여 메서드 파라미터로 활용하기
     * > 생략이 가능하다는 점도 함께 알아보기
     * > @ModelAttribute와 차이를 알아보기
     */
    @DisplayName("Method Arguments - @RequestParam")
    @Test
    void requestParam() {
        String name = "hello";
        RestAssured.given().log().all()
                .accept(MediaType.APPLICATION_JSON_VALUE)
                .when().get("/method-argument/users?name={name}", name)
                .then().log().all()
                .statusCode(HttpStatus.OK.value())
                .body("size()", is(2))
                .body("[0].name", is(name));
    }

    /**
     * MethodArgumentController > requestParam 메서드
     * > @RequestBody 활용하여 메서드 파라미터로 활용하기
     * > 로그 창에서 http request의 body 값을 확인하기
     */
    @DisplayName("Method Arguments - @RequestBody")
    @Test
    void requestBody() {
        User user = new User("이름", "email@email.com");

        RestAssured.given().log().all()
                .contentType(MediaType.APPLICATION_JSON_VALUE)
                .body(user)
                .when().post("/method-argument/users/body")
                .then().log().all()
                .statusCode(HttpStatus.CREATED.value())
                .header("Location", "/users/1")
                .body("name", is("이름"))
                .body("email", is("email@email.com"));
    }
```

```java
    @GetMapping("/users")
    public ResponseEntity<List<User>> requestParam(@RequestParam("name") String userName) {
        List<User> users = Arrays.asList(
                new User(userName, "email"),
                new User(userName, "email")
        );
        return ResponseEntity.ok().body(users);
    }

    @PostMapping("/users/body")
    public ResponseEntity requestBody(@RequestBody User user) {
        User newUser = new User(1L, user.getName(), user.getEmail());
        return ResponseEntity.created(URI.create("/users/" + newUser.getId())).body(newUser);
    }
```

``@RequestBody`` 는 Http 요청의 Body 내용을 Java Object 로 변환시켜준다. Get 방식에서는 사용할 수 없고 Post 방식에서만 사용가능하다. MessageConverter 를 이용해서 변환시키기 때문에 따로 setter가 필요하지 않다.  

``@ModelAttribute`` 는 여러 파라미터들을 1대1로 객체에 바인딩하여 다시 View로 넘겨서 출력하기 위해 사용되는 오브젝트이다. MessageConverter를 사용하는 ``@RequestBody`` 와는 다르게 ``@ModelAttribute`` 는 여러 개의 파라미터를 바로 자바빈 객체로 매핑시킨다는 차이가 있다. 따라서 setter가 반드시 필요하다. 여러 개의 데이터가 날라와도 그 중 하나만 바인딩 시킬 수도 있다.  

```java
{
  writer : 'bepoz',
  title : 'about modelattribute'
}

@ModelAttribute("writer") String writer
```

다음과 같이 말이다.  

``@RequestParam`` 은 요청 파라미터를 메서드에서 1:1로 받기 위해서 사용한다. 위의 코드를 보면 ``@RequestParam("name") String userName`` 과 같이 String 타입을 받아오는 것을 볼 수가 있다.  

다시 말하지만, get 방식인 경우 ``@RequestBody`` 는 사용할 수 없다. ``@RequestParam`` 또는 ``@ModelAttribute`` 를 사용하자  

<br/>

### Return Value

```java
    @GetMapping("/message")
    @ResponseBody
    public String string() {
        return "message";
    }

    @GetMapping("/users")
    @ResponseBody
    public User responseBodyForUser() {
        return new User("name", "email");
    }

    @GetMapping("/users/{id}")
    public ResponseEntity responseEntity(@PathVariable Long id) {
        return ResponseEntity.ok(new User("name", "email"));
    }

    @GetMapping("/members")
    public ResponseEntity responseEntityFor400() {
        return ResponseEntity.badRequest().build();
    }
```

``@ResponseBody`` 를 붙이게되면 MessageConverter 를 통해 변환 되고 응답하게 된다.  
``@ResponseEntity`` 의 경우 개발자가 직접 header, body, status code 를 다룰 수 있게 해준다. 그리고 http 정보들이 json 형식으로 변환된다. 다른 형식으로 변환할 수도 있는데 default 가 json 이다.  그렇기 때문에 위의 코드를 보면 반환 타입이 ResponseEntity 인 메서드에서는 ``@ResponseBody`` 를 사용하지 않은 것을 확인할 수가 있다.  

더 많은 내용은 [[Spring] ResponseEntity 에 대해서](https://bepoz-study-diary.tistory.com/189) 에서 확인하자.  

<br/>

### Exception Handling

```java
    @GetMapping("/hello")
    public ResponseEntity exceptionHandler() {
        throw new CustomException();
    }

    @GetMapping("/hi")
    public ResponseEntity exceptionHandler2() {
        throw new HelloException();
    }

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<String> handle() {
        return ResponseEntity.badRequest().body("CustomException");
    }
```

``@ExceptionHandler`` 어노테이션을 사용하면 익셉션을 처리할 수 있는 메서드를 만들 수가 있다.  
위의 코드에서 ``exceptionHandler`` 메서드에서 ``CustomException`` 을 던지게 되면 ``@ExceptionHandler(CustomException.class)`` 가 붙은 메서드인 ``handle`` 메서드가 호출이 되고 ResponseEntity 를 반환하게 된다.  

```java
@ControllerAdvice()
public class HelloAdvice {
    @ExceptionHandler(HelloException.class)
    public ResponseEntity<String> handle() {
        return ResponseEntity.badRequest().body("HelloException");
    }
}
```

``@ControllerAdvice`` 는 모든 ``@Controller`` 에서 발생한 예외들을 잡아서 처리할 수 있게끔 하는 어노테이션이다.  
위 코드에서 볼 수 있듯이 ``HelloAdvice`` 라는 별도의 클래스를 만들고 ``@ControllerAdvice`` 를 붙여주고나니깐 외부의 Controller 에서 발생한 ``HelloException`` 을 ``@ExceptionHandler`` 를 통해 잡아준 것을 확인할 수가 있다.  

***

### Reference

https://docs.spring.io/spring-framework/docs/current/reference/html/web.html  

https://mangkyu.tistory.com/72

