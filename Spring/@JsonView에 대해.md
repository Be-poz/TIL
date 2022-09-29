# @JsonView에 대해

```java
public class Member {

    private String name;
    private int age;
    private String importantInformation;
}
```

다음과 같은 Member 클래스에 중요한 정보가 있고 특정 api 요청에 대해서 노출이 되면 안된다고 할 때,  ``@JsonView`` 를 이용하는 방법이 있다.  

```java
public class AccessLevel {

    public static class Normal {
    }

    public static class Secret extends Normal {
    }
}

---------------------------------------------------------------------
  public class Member {

    @JsonView(value = AccessLevel.Normal.class)
    private String name;
    
    @JsonView(value = AccessLevel.Normal.class)
    private int age;

    @JsonView(value = AccessLevel.Secret.class)
    private String importantInformation;
}

---------------------------------------------------------------------
@RestController
@RequestMapping("/jsonView")
public class JsonViewController {

    private final Member member = new Member("kang", 100, "top secret");

    @GetMapping
    @JsonView(value = AccessLevel.Normal.class)
//  @JsonView(value = AccessLevel.Secret.class)
    public Member get() {
        return member;
    }
}
```

다음과 같이 노출하고 싶은 필드와 그렇지 않은 필드에 ``@JsonView`` 를 달아두고 api를 호출하는 곳에서도 마찬가지로 ``@JsonView`` 를 이용해 필드 노출에 대한 선택을 할 수가 있다.  

---

### REFERENCE

https://www.baeldung.com/jackson-json-view-annotation
