# WHY HATEOAS??


## what is RESTful API?

* /employees/3 은 REST 가 아니다.
* 단순히, `GET`, `POST`,etc 또한 REST 가 아니다.
* CRUD 를 사용한다고 해서 REST 가 아니다.

위의 방법들로 빌드한 것은 RPC(Remote Procedure Call)이라고 보는 것이 맞다.  
왜냐하면, 이것들만 가지고서는 서비스와 어떤 상호작용을 하고 있는지 알 길이 없기 때문이다.  
만약 서비스를 개시하고 싶다면, 이런 정보에 대한 docs 나 portal를 작성해야 하는 것이 맞다.  

REST 창시자 Roy Fielding은 다음과 같이 말했다.  
>If the engine of application state (and hence the API) is not being driven by hyper text,  
>then it cannot be RESTful and cannot be a REST API  

출처 : https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven  
***  

## ABOUT HATEOAS

HATEOAS는 'Hypermedia As The Engine Of Application State'의 약자이다.  

Restful API를 사용하는 클라이언트가 전적으로 서버와 동적인 상호작용이 가능하도록 하는 것이 HATEOAS이다.  
이러한 방법은 클라이언트가 서버로부터 어떠한 요청을 할 때, 요청에 필요한(의존되는) URI를 응답에 포함시켜 반환하는 것으로 가능하다.  

예를 들면, 사용자 정보를 POST하는 요청 후 사용자를 조회(GET), 수정(PUT), 삭제(DELETE) 할 수 있는 URI를 동적으로 알려주게 되는 것이다.  
이를 통해 얻는 장점은 다음과 같다.  

1. 요청 URI 정보가 변경되어도 클라이언트에서 동적으로 생성된 URI를 사용한다면, 클라이언트 입장에서는 URI 수정에 따른 코드 변경이 불필요하다.
2. URI 정보를 통해 의존되는 요청을 예측가능하게 한다.
3. 기존 URI 정보가 아니라 resource까지 포함된 URI를 보여주기 때문에 resource에 대한 확신을 가질 수 있게 된다.  

예시를 들자면, 
```java
GET /accounts/12345 HTTP/1.1
Host: bank.example.com
Accept: application/vnd.acme.account+json

//다음과 같이 요청을 하였을 때

HTTP/1.1 200 OK
Content-Type: application/vnd.acme.account+json
Content-Length: ...

{
    "account": {
        "account_number": 12345,
        "balance": {
            "currency": "usd",
            "value": 100.00
        },
        "links": {
            "deposit": "/accounts/12345/deposit",
            "withdraw": "/accounts/12345/withdraw",
            "transfer": "/accounts/12345/transfer",
            "close": "/accounts/12345/close"
        }
    }
}

//account의 정보와 더불어서 link로 현재 요청(상황)에서 수행할 수 있는 것들을 링크로 정보(rel과 url)를 줄 수가 있다.

HTTP/1.1 200 OK
Content-Type: application/vnd.acme.account+json
Content-Length: ...

{
    "account": {
        "account_number": 12345,
        "balance": {
            "currency": "usd",
            "value": -25.00
        },
        "links": {
            "deposit": "/accounts/12345/deposit"
        }
    }
}

//만약 요청을 보냈는데 잔액이 마이너스인 경우, link 정보가 달라야 할 것이다.
//위의 예시에는 'deposit' 에 대해서만 link 정보가 주어져 있다.
```

이제 사용을 해보겠다.
Gradle 프로젝트 기준
`implementation 'org.springframework.boot:spring-boot-starter-hateoas'` 를 build에 추가해준다.(json 관련도 당연 있어야 한다.)  
Event라는 Entity를 기존에 가지고 있고, 해당 Event에 대해서 HATEOAS 작업을 한다고 가정하겠다.  
link를 추가해주기 위해서는 RepresentationModel를 상속받는 Resource 클래스가 필요하다.  
전체의 코드를 가지고 설명하자면,  
```java
public class EventResource extends RepresentationModel {

    @JsonUnwrapped
    private Event event;

    public EventResource(Event event) {
        this.event = event;
        add(linkTo(EventController.class).slash(event.getId()).withSelfRel());
    }

    public Event getEvent(){
        return event;
    }
}

//여기서부터 controller 내부 메서드 코드입니다.
Event event = modelMapper.map(eventDto, Event.class);
WebMvcLinkBuilder selfLinkBuilder = linkTo(EventController.class).slash(newEvent.getId());
EventResource eventResource = new EventResource(event);
eventResource.add(linkTo(EventController.class).withRel("query-events"));
eventResource.add(selfLinkBuilder.withRel("update-events"));
eventResource.add(Link.of("http://test"));
eventResource.add(Link.of("asdf").withRel("aa"));
eventResource.add(Link.of("dddd").withSelfRel());
return ResponseEntity.created(createdUri).body(eventResource);
```
EventResource 클래스는 RepresentationalModel를 상속받은 클래스이다.  
필드값으로 객체인 Event를 가지고 있으며, 생성자에서 해당 객체를 받아온다. @JsonUnwrapped 와 생성자 내의 add 코드는 차후 설명하겠다.  

컨트롤러 내부의 코드를 보면, 먼저 DTO를 통한 입력 값을 event에 modelMapper를 통해 주입을 받게된다.  
이제 resource 클래스에 link를 추가할 수 있는데 추가하는 방법은 여러가지가 있을 수 있는데 대표적인 방법이라고 생각되는 것을 적었다.  

**eventResource.add(Link.of("url"));**    
Link 클래스의 of 메서드를 사용한 방법이다. 내부에 String 으로 url을 기입한다. 위와 같이 끝내게 되면 Relation이 "self"가 defualt가 된다.  

**eventResource.add(Link.of("url").withRel("aa"));**  
Realation이 "aa" 로 나타나게 된다. Link.of("url","rel") 를 통해서 표현할 수도 있고, withRel(rel) 이 아닌 withSelfRel()로 "self"를
표현해 줄 수도 있다.  

**eventResource.add(linkTo(클래스).withRel(rel));**  
해당 클래스의 RequestMapping url을 넣고 Relation을 설정해준다.  
`import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.linkTo;` 해준것이고, WebMvcLinkBuilder 소속이다.
해당 문법 또한 withSelfRel()를 통해 "self"를 표현해 줄 수가 있다.  
위의 Link.of 와 달리 withRel() or withSelfRel()를 꼭 기입해주어야 한다.  

컨트롤러 메서드 내부 코드를 살펴보면  
`WebMvcLinkBuilder selfLinkBuilder = linkTo(EventController.class).slash(newEvent.getId());`를 볼 수가 있는데,  
반복적인 link를 위해서 다음과 같이 사용할 수도 있다는 것을 나타내준 것이다. slash는 앞의 link 뒤에 '/' 후 slash 내부의 값을 붙여주는 역할  


EventResource 클래스의 생성자를 보면 생성을 하면서 "self" relation을 생성해준 것을 알 수가 있다.  

@JsonUnwrapped는 다음과 같은 상황에 사용된다.  
![capture1](https://user-images.githubusercontent.com/45073750/94361298-251ff580-00ee-11eb-9883-b9cbe5297ca7.PNG)
![capture2](https://user-images.githubusercontent.com/45073750/94361300-26e9b900-00ee-11eb-9419-f573750705c7.PNG)  


위는 사용하지 않은 상황이고 밑은 사용한 경우이다.  
Event 객체에서 Unwrap 해준다.  
물론 EventResource 클래스의 필드를 Event 객체가 아니라 Event 객체 내부의 필드를 다 풀어서 사용해줄 수도 있지만 상당히 번거로운 일인데,  
@JsonUnwrapped 를 통해서 간단하게 해결해 줄수도 있다. 이 방법이 아닌 CollectionModel를 상속받아 하는 방법도 있지만 생략하겠다.  
***

HATEOAS 1.0.2 버전 이후  
`ResourceSupport` -> `RepresentaionModel`  
`Resource` -> `EntityModel`  
`Resources` -> `CollectionModel`  
`PagedResources` -> `PagedModel`  
를 사용해야 한다.  

위의 설명중에서 `Link.of()` 메서드는 `new Link(String)` 와 같다.  
후자는 spring에서 지양하라고 나와서 전자를 택하였다.

***

`참고 링크`  
https://en.wikipedia.org/wiki/HATEOAS 
https://spring.io/guides/gs/rest-hateoas/  
https://docs.spring.io/spring-hateoas/docs/1.1.2.RELEASE/reference/html/  
https://spring.io/guides/tutorials/rest/  
https://wallees.wordpress.com/2018/04/19/rest-api-hateoas/  

***
