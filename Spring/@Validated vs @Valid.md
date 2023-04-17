# @Validated vs @Valid

@Valid는 이전에도 자주썼었기에 어떤 역할을 갖고 있는지 알고있었지만, @Validated는 생소했다.  
특히 object가 아닌 파라미터에 대한 valid 작업이 인상깊었는데 이 글을 통해 정리해보도록 하겠다.  

```java
@PostMapping("/check")
public String check(@RequestBody @Valid CheckRequest request) {
  return "ok";
}

---
  
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CheckRequest {

    @Positive
    private int num;
}
```

@Valid는 ``javax.validation.constraints`` 에서 지원하는 valid 조건들을 검증한다.  
문제가 발생 시에 ``MethodArgumentNotValidException`` 이 발생하게 된다. 위의 코드에서 파라미터로 ``Errors`` 나 ``BindingResult`` 를 두게되면 따로 예외가 발생하지 않고, ``hasErrors()`` 등의 메서드로 입맛에 맞게 처리를 할 수가 있다.  

@Validated는 검증작업을 하도록 하는 어노테이션이다. 위에서 @Valid를 사용하는데 무슨뜻인가 싶을 것이다.  
@Valid는 컨트롤러에서만 동작한다. 그 이유는 @Valid가 동작하는 시점과 관련이 있다.  

![image](https://user-images.githubusercontent.com/45073750/121858410-8eee3700-cd31-11eb-80cf-45711e1deed1.png)

![image](https://user-images.githubusercontent.com/45073750/121858680-d1b00f00-cd31-11eb-8ca8-6fc2f7003a4c.png)

@Valid는 ArgumentResolver 과정에서 처리되기 때문이다. 핸들러 어댑터 처리 과정 중에 말이다.  
따라서 컨트롤러(핸들러)에서만 동작하는 것이다.  

@Validated는 컨트롤러 계층이 아닌 다른 계층에서도 유효성 검사를 할 수 있게끔 해준다. (@Valid는 자바 스펙인 반면, @Validated는 Spring에서 지원하는 어노테이션이다)  

코드를 통해 확인을 해보자.  

```java
@RestController
@RequiredArgsConstructor
public class CheckController {

    private final CheckService checkService;

    @PostMapping("/check")
    public String check(@RequestBody CheckRequest request) { //CheckRequest는 맨 위 코드와 동일
        checkService.check(request);
        return "ok";
    }
}

---
  
@Service
public class CheckService {

    public void check(@Valid CheckRequest request) {}
}
```

이 상황에서 CheckRequest의 num에 음수값을 넣은 후에 API 요청을 했더니 Exception 발생없이 정상적으로 동작하는 것을 확인할 수가 있었다.  

```java
@Service
@Validated
public class CheckService {

    public void check(@Valid CheckRequest request) {}
}
```

이번에는 @Validated 를 붙인 후 돌려보았다.  

![image](https://user-images.githubusercontent.com/45073750/154293320-1a7f8a7f-476f-465b-94ba-32af5643f1a5.png)

정상적으로 validation 작업이 이루어졌다.  
그런데 기존에 컨트롤러 계층에서 valid 문제가 발생하면 ``MethodArgumentNotValidException``  이 발생했었지만 이번에는 ``ConstraintViolationException`` 이 발생한 것을 볼 수가 있다. 그 이유는 컨트롤러 계층이 아닌 곳에서  ``@Validated`` 를 이용한 경우에는 위의 에러코드에서도 확인할 수 있듯이 aop로 동작하고 이 과정에서 문제가 생긴 것이기 때문이다.  

그리고 유용하게 사용할 수 있는 또 다른 방법은 다음과 같다.  

```java
@RestController
@RequiredArgsConstructor
@Validated
public class CheckController {

    private final CheckService checkService;

    @PostMapping("/check")
    public String check(@RequestBody @Valid CheckRequest request) {
        checkService.check(request);
        return "ok";
    }

    @GetMapping("/check")
    public String check(@RequestParam @Positive int num) {
        return "ok";
    }
}
```

object 가 아닌 일반적인 파라미터에 대해서도 체크가 가능하다는 것이다.  
RequestParam으로 num 값을 음수로 보내주었더니 마찬가지로 ConstraintViolationException이 발생했다.  
@Validated는 컨트롤러에서만 한정적인 것이 아니라고 했으니 서비스 계층에서도 이를 응용할 수 있다.  

```java
@RestController
@RequiredArgsConstructor
public class CheckController {

    private final CheckService checkService;

    @PostMapping("/check")
    public String check(@RequestBody @Valid CheckRequest request) {
        checkService.check(request);
        return "ok";
    }

    @GetMapping("/check")
    public String check(@RequestParam int num) {
        checkService.checkNum(num);
        return "ok";
    }

    @GetMapping("/checkList")
    public String checkList() {
        checkService.checkListCount(
                Arrays.<Integer>asList(1, 2, 3)
        );
        return "ok";
    }
}

---
  
@Service
@Validated
public class CheckService {

    public void check(@Valid CheckRequest request) {}

    public void checkNum(@Positive int num) {}

    public void checkListCount(@Size(min = 4) List<Integer> list) {}
}
```

컨트롤러의 checkList에서 3개의 인자를 가진 List 컬렉션을 서비스 계층에 넘기고 해당 서비스에서 valid 작업을 한다.  
적절하게 ConstraintViolationException이 발생했다.  

이렇게 타 계층에 보내는 파라미터에 대한 검증을 하기에도 적절하다.  

@Validated는 이외에도 valid 그룹 지정이라는 기능이 있다고 하는데, 이 글을 통해 정리하고자 했던 지식은 바로 위의 내용이었기 때문에 생략하도록 하겠다.  

---
