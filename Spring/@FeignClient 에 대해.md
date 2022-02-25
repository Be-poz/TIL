# @FeignClient 에 대해

Feign은 netflix에서 개발한 REST client 다. 현재는 스프링으로 포함된 것 같다.  
기존에 ``RestTemplate`` 을 사용한 API 호출을 코드양을 줄이고 굉장히 간편하게 호출할 수 있게끔 도와준다.  

```groovy
ext {
    set('springCloudVersion', "2021.0.0")
}

implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

다음과 같이 추가해서 사용할 수 있다. 그냥 https://start.spring.io/ 에서 ``OepnFeign`` 의존성을 추가하는 것이 쉬울 것이다.  
Intellij의 spring initializer를 이용해도된다.  

![image](https://user-images.githubusercontent.com/45073750/155686594-29088ed6-897a-4419-9dc8-125f83399a95.png)

<br/>

아주 간단한 사용법을 작성하겠다. ``@FeignClient`` 를 이용하면 data jpa를 사용하듯이 단순히 인터페이스만으로 API 호출이 가능하다.  

```java
//localhost:8000
@EnableFeignClients
@SpringBootApplication
public class BaseApplication {

    public static void main(String[] args) {
        SpringApplication.run(bepoz.workroom.feignClientExample.BaseApplication.class, args);
    }

}

@RestController
@RequestMapping("/feign")
@Slf4j
@RequiredArgsConstructor
public class BaseController {

    private final BaseService baseService;

    @GetMapping
    public String get() {
        return baseService.namesWithComma();
    }

}

@Service
@RequiredArgsConstructor
public class BaseService {

    private final BaseFeignClient client;

    public String namesWithComma() {
        return String.join(",", client.names());
    }
}

@FeignClient(name = "name-api", url = "localhost:8080/names")
public interface BaseFeignClient {

    @GetMapping(produces = "application/json")
    public List<String> names();
}
```

``localhost:8000/feign``을 호출하면 ``@FeignClient`` 에 정의해놓은 것 처럼 ``localhost:8080/names`` 로 API 요청이 나갈 것이다. 밑의 ``names()`` 메서드만 봐도 진짜 간단하다라는 것을 알 수 있을 것이다. Data Jpa 사용하던 것과 같다는 것을 느낄 것이다.  

```java
//localhost:8080
@RestController
public class MappingController {
  
    private final List<String> names = List.of("kang", "kim", "lee", "kwon");

    @GetMapping
    public List<String> names() {
        return names;
    }
}
```

<img src="https://user-images.githubusercontent.com/45073750/155687267-1a72b289-a1ba-40b9-bdcb-f4e090cc00d1.png" alt="image" style="zoom:10%;" />

이번에는 코드를 조금 더 추가해서 ``@RequestParam`` 과 ``@RequestBody`` 를 확인해보겠다.  

```java
//localhost:8000
@RestController
@RequestMapping("/feign")
@Slf4j
@RequiredArgsConstructor
public class BaseController {

    private final BaseService baseService;

    @GetMapping
    public String get() {
        return baseService.namesWithComma();
    }

    @GetMapping("/specific")
    public String name(String name) {
        return baseService.specificName(name);
    }

    @PostMapping
    public ResultDto isContainName(@RequestBody NameDto nameDto) {
        return baseService.isContainName(nameDto);
    }

}

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class NameDto {

    private String name;
}

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ResultDto {

    int statusCode;
    boolean result;

    private ResultDto(int statusCode, boolean result) {
        this.statusCode = statusCode;
        this.result = result;
    }

    public static ResultDto ok(boolean result) {
        return new ResultDto(200, result);
    }

    public static ResultDto error(boolean result) {
        return new ResultDto(500, result);
    }
}

@Service
@RequiredArgsConstructor
public class BaseService {

    private final BaseFeignClient client;

    public String namesWithComma() {
        return String.join(",", client.names());
    }

    public String specificName(String name) {
        return client.specificName(name);
    }

    public ResultDto isContainName(NameDto nameDto) {
        return client.isContainName(nameDto);
    }
}

@FeignClient(name = "name-api", url = "localhost:8080/names")
public interface BaseFeignClient {

    @GetMapping(produces = "application/json")
    public List<String> names();

    @GetMapping(path = "/specific", produces = "application/json")
    public String specificName(@RequestParam String name);

    @PostMapping(produces = "application/json")
    public ResultDto isContainName(@RequestBody NameDto nameDto);

}
```

```java
//localhost:8080
@RestController
@RequestMapping("/names")
public class MappingController {

    private final List<String> names = List.of("kang", "kim", "lee", "kwon");

    @GetMapping
    public List<String> names() {
        return names;
    }

    @GetMapping("/specific")
    public String specificName(@RequestParam String name) {
        if (names.contains(name)) {
            return name;
        }
        return "nope";
    }

    @PostMapping
    public ResultDto isContainName(@RequestBody NameDto nameDto) {
        if (names.contains(nameDto.getName())) {
            return ResultDto.ok(true);
        }
        return ResultDto.error(false);
    }

}
```

parameter와 body에 데이터를 넣어서 post 전송하는 것 또한 문제없이 동작하였다.  
``localhost:8080`` 에도 동일한 NameDto와 ResultDto가 정의되어 있다.  
ResultDto는 statusCode가 있는데 boolean result가 이상해보이긴하지만 예시의 코드니깐 무시하자. 말하고자 했던 것은 만약 MSA 같은 환경을 사용하는 경우에 response에 대한 컨벤션이 있을 것이고 제네릭을 사용하고 있을 것이다. 그러한 ResponseDto를 보여주기 위해서 ResultDto를 정의해서 보여준 것이다.  

공통적인 Dto 형식을 정의하고 주고받고 한다는 뜻이다.  
이런 API 호출을 ``RestTemplate``을 사용해서 구현했다면 꽤나 귀찮은 코드들이 들어가게되는데 feign 을 이용해서 간편하게 사용할 수가 있다. 추가적으로 필요한 정보들은 docs를 확인하면 좋을 것 같다.  

---

### REFERENCE

https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/  

https://github.com/OpenFeign/feign

