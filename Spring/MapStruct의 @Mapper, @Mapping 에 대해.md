# MapStruct의 @Mapper, @Mapping 에 대해

MapStruct은 엔티티와 DTO간의 매핑을 손쉽게 하게끔 도와준다.  

```groovy
    implementation 'org.mapstruct:mapstruct:1.4.2.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.4.2.Final'
```

사용법에 대해 바로 들어가보겠다.  

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Chicken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    private BigDecimal price;

    private String sourceName;

    public Chicken(long id, String name, BigDecimal price, String sourceName) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.sourceName = sourceName;
    }
}

---
  
@NoArgsConstructor
@Getter
public class ChickenDto {

    public String name;
    public BigDecimal price;
    public String sourceName;
}

---
  
@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR, componentModel = "spring")
public abstract class ChickenMapper {

    public abstract ChickenDto toChickenDto(Chicken chicken);
}
```

이게 끝이다. 너무나도 간단하다. ``@Mapper`` 속성의 ``componentModel`` 에 spring 으로 설정을 해서 빈 등록이 되게끔 한다.  
Chicken에서 필드를 뽑아 ChickenDto에 생성하는 클래스를 자동으로 만들어 준다 아래와 같이 말이다.  

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-16T17:27:46+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.4.jar, environment: Java 11.0.11 (AdoptOpenJDK)"
)
@Component
public class ChickenMapperImpl extends ChickenMapper {

    @Override
    public ChickenDto toChickenDto(Chicken chicken) {
        if ( chicken == null ) {
            return null;
        }

        ChickenDto chickenDto = new ChickenDto();

        chickenDto.name = chicken.getName();
        chickenDto.price = chicken.getPrice();
        chickenDto.sourceName = chicken.getSourceName();

        return chickenDto;
    }
}
```

엔티티와 DTO의 필드명이 다른 경우에는 아래와 같이 처리해준다.  

```java
@NoArgsConstructor
@Getter
public class ChickenDto {

    public String name;
    public BigDecimal price;
    public String source;  //sourceName 에서 source 로 변경
}

---
  
@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR, componentModel = "spring")
public abstract class ChickenMapper {

    @Mapping(source = "sourceName", target = "source")
    public abstract ChickenDto toChickenDto(Chicken chicken);
}
```

source는 말그대로 소스, 파라미터로 들어온 Chicken을 ChickenDto로 변환하는 작업이니 Chicken의 sourceName이 source가 된다. target은 ChickenDto의 타겟이 될 필드를 일컫는다. 따라서 source로 지정해준 것이다.  

소스의 sourceName을 타겟의 source에 매핑해준다는 뜻이다.  

이번에는 Dto의 특정 필드에는 매핑이 안되었으면 하는 경우를 살펴보자.  

```java
public void createChicken() {
  Chicken chicken = new Chicken(id.getAndIncrement(), "chicken", BigDecimal.valueOf(18000), "honey");
  chickenBox.put(chicken.getId(), chicken);
}

---

@NoArgsConstructor
@Getter
public class ChickenDto {

    public String name;
    public BigDecimal price;
    public String sourceName = "dto default source name";
}
```

Chicken을 생성할 때에 ``sourceName`` 필드에 "honey" 으로 집어넣었다. 만약 이대로 해당 Chicken을 ChickenDto로 변환하게된다면 Dto의 sourceName은 "honey" 으로 나올 것이다.  

그런데 만약 해당필드는 임의의 값을 내가 넣어야 해서 매핑이 안되기를 원한다면 ??(위 코드에서는 Dto에서 초기부터 할당되어있는 값을 갖고와야한다는 가정을 두고 초기화를 해준 코드를 적었다)  

```java
@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR, componentModel = "spring")
public abstract class ChickenMapper {

    @Mapping(target = "sourceName", ignore = true)
    public abstract ChickenDto toChickenDto(Chicken chicken);
}
```

target(ChickenDto)의 "sourceName" 은 무시한다는 설정을 주었다. 이제 Chicken의 sourceName이 ChickenDto에 매핑되지 않는다. 그 결과 Dto의 sourceName을 조회 시에 ``dto default source name`` 으로 조회되는 것을 확인할 수가 있었다.  

이번에는 DTO에 여러 엔티티가 들어가는 경우를 살펴보자.  

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Bill {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private BigDecimal price;

    public Bill(long id, BigDecimal price) {
        this.id = id;
        this.price = price;
    }
}

---
  
@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR, componentModel = "spring")
public abstract class ChickenMapper {

    @Mapping(target = "price", source = "bill.price")
    public abstract ChickenDto toChickenDto(Chicken chicken, Bill bill);
}

```

``Bill`` 이라는 새로운 엔티티가 추가되었고, 필드에는 price가 있다.  
``Chicken`` 에도 price 필드가 존재하기 때문에 Mapper한테 Dto의 price에 어떤 엔티티의 price 필드로 매핑할지 알려줘야 한다.  

그래서 source 란에 ``bill.price`` 와 같이 입력해서 지정을 해준 것이다. ChickenDto의 price 필드를 조회해보니 Bill의 price 값과 같은 것을 확인할 수가 있었다.  

</br>

엔티티와 Dto 간의 매핑 스트레스를 줄여주는 MapStruct에 대해서 알아보았다.  

---

### REFERENCE

https://mapstruct.org/
