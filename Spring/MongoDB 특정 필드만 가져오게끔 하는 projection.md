# MongoDB 특정 필드만 가져오게끔 하는 projection

## 개요

몽고 DB에서 document를 조회할 때 매치되는 doc의 모든 필드를 가져오지만 projection을 이용해 특정 필드만 가져올 수가 있다. 필드 수가 많은 collection에서 내가 원하는 데이터의 특정 필드값만 조회하고 싶은데, 전체 필드를 가져오는 것은 굉장히 비효율 적일 것이다. 그럴 때에 사용할 수 있다.

```java
public class Person {
    @Id
    private String id;
    private Name name; //Name 클래스는 firstName과 lastName 이렇게 2개의 필드가 존재
    @Positive
    private int age;
    @CreatedDate
    private LocalDateTime createdAt;
    @LastModifiedDate
    private LocalDateTime modifiedAt;
}
```

사용하려는 Person collection의 정보는 위와 같다.  

<br/>

## MongoDB Compass에서의 사용

<img width="465" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/21e33128-b586-4698-8cbf-bd9bd4765d14">

위와 같이 2건의 데이터를 집어넣은 상태이다.  
이 상황에서 ``age``가 20인 데이터를 검색하면 딱 저 데이터들이 리턴될 것이다. 하지만 내가 원하는 값은 ``name``만을 원한다면 project를 이용하면 된다.  

아래는 mongoDB Compass에서 project를 사용했을 때의 결과이다.  

<img width="371" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/47efcf42-de63-4209-8ec5-b4d4ed5e8168">

원하는 필드에 1을 기입하면 출력이 된다. Name은 Object 타입이고 내부에 여러 필드가 있다.  

<img width="462" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/ece5e18a-20c4-45e6-84ee-54bb8389d3e4">

이렇게 상세 필드를 기입하면 해당 필드만 뽑아올 수가 있다.  

<img width="400" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/d1af1194-94b8-44e2-9d26-bfc5de1f67ea">

이번에는 아예 0으로 두었더니 해당 필드만 빼고 가져온 것을 확인할 수가 있었다.  

1로 두었을 때에는 해당 필드만 가져오지만 0으로 두면 해당 필드를 제외하고 가져오게 된다.  

<img width="443" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/6b1e72e9-b783-4499-abc1-1a609fc5c12d">

1을 사용하게 되면 해당 필드말고도 ``_id`` 값을 가져왔었는데 이 또한 0으로 명시해서 가져오지 않을 수가 있다.  

<br/>

## @Query에서의 사용

이번에는 ``@Query`` 어노테이션을 사용하여 projections을 사용해보겠다.  

```java
public interface PersonRepository extends MongoRepository<Person, String>, PersonRepositoryCustom {

    @Query(value = "{'age': 20}", fields = "{'name': 1, 'age': 1}")
    List<Person> findAllWithProjections();
}

---------------------------------------------------------------------------------
  
@GetMapping("/projections")
public List<Person> getPersonWithProjections() {
    return personRepository.findAllWithProjections();
}
```

```json
[
    {
        "id": "64d8cb154360a130e3d5e422",
        "name": {
            "name": "negative Dont"
        },
        "age": 20,
        "createdAt": null,
        "modifiedAt": null
    },
    {
        "id": "64d8cb31f7736e129cf1fb9b",
        "name": {
            "name": "negative Dont"
        },
        "age": 20,
        "createdAt": null,
        "modifiedAt": null
    }
]
```

조회하면 필요한 값만 들고오는 것을 확인할 수 있다.  
 ``fields = "{'name': 1, '_id': 0}"`` 이런 식으로 변경했을 때에도 적절한 response를 돌려준다.  

<br/>

## MongoTemplate과 Query에서의 사용

```java
@PostMapping("/projections")
public List<Person> getPersonWithProjections(@RequestBody List<String> projections) {
    Query query = new Query();
    for (String projection : projections) {
        query.fields().include(projection);
    }
    return mongoTemplate.find(query, Person.class);
}
```

위와 같이 ``fields().include()`` 메서드를 이용할 수 있다. ``exclude()`` 를 통해 위에서 0을 설정한 결과값을 받을 수도 있다.  

<img width="499" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/699efec1-ac8f-4003-bf94-ee045d22e8c6">

include의 설명을 보면 ``projection``의 역할인 것을 확인할 수가 있다.  

---

### REFERENCE

https://www.mongodb.com/docs/manual/tutorial/project-fields-from-query-results/

