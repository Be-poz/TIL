# AttributeConverter를 이용하여 DB에 Entity의 컬렉션 필드 저장하기

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class TimeTravel {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Convert(converter = TravelListToJsonConverter.class)
    private List<Travel> travels;

    public TimeTravel(List<Travel> travels) {
        this.travels = travels;
    }

    @Getter
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    static public class Travel {
        private String name;
        private String duration;

        public Travel(String name, String duration) {
            this.name = name;
            this.duration = duration;
        }
    }
}
```

위와 같은 엔티티가 존재하고 DB에 저장을 하려고 한다.  
Travel 클래스와 연관관계가 있는 것이 아니고 저 상태로 DB에 저장하고자 할 때, 일반적으로 값 타입 컬렉션 저장을 위해 사용하는 ``@ElementCollection`` 을 사용하곤한다. 위와 같이 클래스 타입이 아니라 ``List<String>``  을 저장할 때에도 말이다. 이 방법은 테이블을 따로 두어서 진행하는 방식이다.  

오늘 정리할 TIL은 조금 다른 방식이다. 위의 코드의 경우에는 Travel을 JSON 으로 바꿔 String 타입으로 DB에 저장하려고 한다.  

```java
public class TravelListToJsonConverter implements AttributeConverter<List<Travel>, String> {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(List<Travel> attribute) {
        try {
            return objectMapper.writeValueAsString(attribute);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public List<Travel> convertToEntityAttribute(String dbData) {
        TypeReference<List<Travel>> typeReference = new TypeReference<List<Travel>>() {};
        try {
            return objectMapper.readValue(dbData, typeReference);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

``AttributeConverter`` 를 구현하는 클래스를 만들고 해당 엔티티 필드에 ``@Convert(converter = ... .class)`` 를 이용하여 적용해주면 된다.  
중요한 점은 ``TypeReference`` 를 이용해야 DB에서 엔티티의 필드로의 Deserializing이 원활하게 이루어진다는 점이다.  
``convertToDatabaseColumn`` 메서드는 이름 그대로 엔티티를 DB에 집어넣을 때 어떤 식으로 넣을지에 대한 구현이고,  
``convertToEntityAttribute`` 메서드는 DB에서 빼와서 엔티티 필드로 변환할 때의 구현 내용이다.  

```json
{
    "travels": [
        {
            "name": "seoul",
            "duration": "maybe 2weeks?"
        },
        {
            "name": "new york",
            "duration": "a month"
        }
    ]
}
```

위의 값을 Body로 하여 ``TimeTravel`` 생성 요청을 해보았다.  

![image](https://user-images.githubusercontent.com/45073750/182373363-9479ca8b-d363-4969-ba26-5b5af0526b55.png)

그 결과 DB에 위와 같이 JSON 형식으로 잘 저장된 것을 확인할 수가 있다.  

<img width="437" alt="image" src="https://user-images.githubusercontent.com/45073750/182374989-41f3cd55-e2f5-4a1f-ba25-859e989a650c.png">

조회 시에도 잘 조합되어 조회되는 것을 확인할 수가 있다.  

<Br/>

클래스를 가지는 컬렉션이 아니라 String 에 대한 것도 한 번 확인해보겠다.  

```java
public class TimeTravel {
...

    // 새로 추가된 필드
    @Convert(converter = DelimiterConverter.class)
    private List<String> countries;
...
}

-------------------------------------------------------------------------------------------------------------

public class DelimiterConverter implements AttributeConverter<List<String>, String> {

    private final String DELIMITER = ",";

    @Override
    public String convertToDatabaseColumn(List<String> attribute) {
        return String.join(DELIMITER, attribute);
    }

    @Override
    public List<String> convertToEntityAttribute(String dbData) {
        return new ArrayList<>(Arrays.asList(dbData.split(",")));
    }
}
```

``List<String>`` 타입을 가지는 필드를 추가하고 간략하게 converter 작성을 하여 결과를 확인해보겠다.  

![image](https://user-images.githubusercontent.com/45073750/182375600-23cf7447-452a-459a-9970-270cb5e1e70f.png)

``,`` 가 붙어 잘 들어갔다.  

<img width="427" alt="image" src="https://user-images.githubusercontent.com/45073750/182376185-68ea2e2c-26e2-483a-97b8-4094cec46dae.png">

조회 시에도 문제 없이 조회된다.  

아무래도 클래스를 가지는 List 보다는 String 을 가지는 List의 경우가 훨씬 유용할 것이라고 생각된다.  
컨버터를 이용하여 이렇게 List를 저장할 때 사용하는 것 뿐만 아니라 DB에 넣고 뺄 때에 암호화/복호화가 필요한 경우에도 ``convertToDatabaseColumn`` , ``convertToEntityAttribute`` 메서드를 구현하면서 이곳에서 암,복호화를 진행하는 방법으로 이용할 수도 있다.  

---

