# Spring Data MongoDB 잊어버리는 것들 정리

### 기본적으로 ``CrudRepository`` 를 사용

![image](https://user-images.githubusercontent.com/45073750/217474216-34e8b254-53a2-46a4-ba2a-180e046f8685.png)

core conecept로 ``CrudRepository`` 를 사용한다. 하위 계층에 일반적으로 사용하는 ``MongoRepository`` 가 존재한다.

ref. https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#repositories.core-concepts

<br/>

### data jpa와 다르게 생성 시에 컬렉션 생성이 되지 않는다

data jpa에서 ``spring.jpa.hibernate.ddl-auto`` 옵션으로 create 과 같은 옵션을 주어 어플리케이션 생성 시에 테이블을 생성하고는 했는데, data mongodb 에서는 따로 컬렉션 생성 코드를 호출하거나 기존에 존재하지 않는 컬렉션에 document가 insert 될 때 생성이 된다.  

<br/>

### id를 따로 넣지 않으면 자동적으로 ObjectId를 채워넣는다

```java
@Document
public class Person {

    private String id;
    private String name;
    private int age;
    private LocalDateTime createdAt;
}
```



<img width="341" alt="image" src="https://user-images.githubusercontent.com/45073750/217484399-24762e34-c2cd-4c1d-bd95-9965c6fd6fe5.png">

첫 번째는 cli를 이용하여 insert 한 것이고 나머지는 spring application 내부에서 mongoRepository를 이용하여 insert 한 것이다.  
``_id`` 를 따로 명시하지 않으면 ObjectId로 알아서 채워넣어준다  

<br/>

### _id는 ``@Id`` 지정이 되어있지 않아도 매핑이 가능하다

```java
@Document
public class Person {

    private String id;
    private String name;
    private int age;
    private LocalDateTime createdAt;
}
```

위와 같이 ``@Id`` 지정이 되어있지 않아도 매핑이 가능하다. 클래스 필드 명이 ``id`` 면 알아서 ``_id`` 와 매핑 해준다. 필드명이 ``id``가 아니라면 매핑이 제대로 되지 않지만 ``@Id`` 를 붙이면 필드명이 ``id`` 가 아니더라도 mongo 컬렉션의 ``_id`` 와 매핑이 된다.  

ref. https://www.mongodb.com/docs/manual/reference/method/db.collection.insert/#behaviors

<br/>

### ``@AccessType(value = Type.PROPERTY)`` 를 사용할 수 있다

```java
@Document
public class Person {

    @AccessType(value = Type.PROPERTY)
    private String id;
    private String name;
    private int age;
    private LocalDateTime createdAt;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        this.createdAt = LocalDateTime.now();
    }

    public String getId() {
        return name+":"+age;
    }
}
```

mongo 에서도 ``@AccessType(value = Type.PROPERTY)`` 를 마찬가지로 사용할 수가 있다.  

<img width="360" alt="image" src="https://user-images.githubusercontent.com/45073750/217488799-414fcbe3-8e9e-4f49-bcb1-9fe2a6b4e872.png">

<br/>

### 쿼리 보고싶을 때의 yml 설정

```yaml
logging:
  level:
    org.springframework.data.mongodb.core.MongoTemplate: DEBUG
```

``MongoRepository``도 결국은 내부적으로 ``MongoOperations``를 거쳐 ``MongoTemplate``을 호출하기 때문에 위의 path를 DEBUG로 걸면 호출 쿼리를 로깅할 수 있다.  

ref. https://stackoverflow.com/questions/16209681/what-is-the-difference-between-save-and-insert-in-mongo-db

<Br/>

### 생성하려는 document에 id 유무에 따라 insert, save로 나뉜다

```java
	@Override
	public <S extends T> S save(S entity) {

		Assert.notNull(entity, "Entity must not be null");

		if (entityInformation.isNew(entity)) {
			return mongoOperations.insert(entity, entityInformation.getCollectionName());
		}

		return mongoOperations.save(entity, entityInformation.getCollectionName());
	}
```

``JpaRepository``의 경우에는 ``persist``와 ``merge`` 로 나뉘었다면(이때 merge 시에 select -> insert의 과정을 거쳐서 성능문제 이슈가 있을 수 있었음),  
``MongoRepository``의 경우 ``insert``와 ``save``로 나뉘게 된다.  

``save`` 는 upsert와 같이 동작한다. 해당 ``_id``를 가진 document가 있으면 update 쳐주고 그렇지 않다면 insert 동작을 한다.  
``insert``는 insert 동작을 그냥 한다.  

<br/>

### id를 채워준 document save 시에는 auditing 기능인 ``@CreatedDate`` 가 동작하지 않는다

id를 채워준 document를 save 시에는 ``@CreatedDate`` 가 동작하지 않아 해당 필드가 null로 들어간다.  

``@LastModifiedDate`` 은 id 여부에 상관없이 채워져 저장된다.  

<br/>

### multiple update, insert는 유의해서 사용하자

single update, insert 는 atomic이 보장이 되지만, multiple의 경우는 그렇지 않다(``findAndModify`` 는 atomic을 보장한다).  

docs에 따르면 이를 위해서 transaction을 특정 버전부터 지원을 한다고는 되어있지만, 이는 큰 비용이기 때문에 효율적인 스키마 디자인을 하는 편이 더 낫다고 나와있다.

```
You can also create a unique index on a field so that it can only have unique values. This prevents inserts and updates from creating duplicate data. You can create a unique index on multiple fields to ensure the combination of field values is unique. 
```

유니크한 필드 및 인덱스를 이용해서 사전에 방지하는 것 또한 좋은 방법이다.  

upsert 또한 유니크한 필드 또는 인덱스를 사용해야 위험을 피할 수 있다.

ref.  

https://www.mongodb.com/docs/manual/core/write-operations-atomicity/  

https://www.mongodb.com/docs/v3.2/reference/method/db.collection.update/#use-unique-indexes  
https://stackoverflow.com/questions/17008947/whats-the-difference-between-spring-datas-mongotemplate-and-mongorepository  

https://stackoverflow.com/questions/51366551/is-mongodb-update-with-query-atomic  
https://stackoverflow.com/questions/64785448/spring-data-mongo-repository-is-saveall-atomic  
https://stackoverflow.com/questions/41071617/atomically-update-multiple-documents-and-return-them

<br/>

### BulkMode ORDERED, UNORDERED 차이

말그대로 BulkOperations를 수행할 때에 등록한 작업을 순차적으로 할 것인지 아닌지에 대한 설정이다.  

```java
	/**
	 * Mode for bulk operation.
	 **/
	enum BulkMode {

		/** Perform bulk operations in sequence. The first error will cancel processing. */
		ORDERED,

		/** Perform bulk operations in parallel. Processing will continue on errors. */
		UNORDERED
	};
```

위의 코드 주석에도 나와있고 docs에도 나와있듯이 ORDERED 일 때에는 하나가 실패하면 진행중인 프로세스를 취소시키고, UNORDERED는 실행 순서가 상관이 없기 때문에 병렬적으로 진행된다고 나와있다. 그리고 에러가 발생해도 그대로 진행한다.  

ref. https://www.mongodb.com/docs/manual/reference/method/Bulk/  

---

