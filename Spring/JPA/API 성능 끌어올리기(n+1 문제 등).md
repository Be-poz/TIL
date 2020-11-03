# API 성능 끌어올리기(n+1 문제 등)
본 내용은 김영한 님의 스프링 부트와 JPA 활용2 강의 내용을 토대로 정리했습니다.


### **지연로딩과 컬렉션이 없는 경우**  

```java
@GetMapping("/api/v1/members")
public List<Member> membersV1(){
    return memberService.findMembers();
}
```

**응답 값으로 엔티티를 직접 외부에 노출한 경우의 문제점**  

* 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
* 기본적으로 엔티티의 모든 값이 노출된다.
* 응답 스펙을 맞추기 위해 로직이 추가된다.(``@JsonIgnore``, 별도의 뷰 로직 등등)
* 실무에서는 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어렵다.
* 엔티티가 변경되면 API 스펙이 변한다.
* 추가로 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어렵다.(별도의 Result 클래스 생성으로 해결)

**결론** : API 응답 스펙에 맞추어 별도의 DTO를 반환한다.  



```JAVA
@GetMapping("/api/v2/members")
public Result membersV2(){
    List<Member> findMembers = memberService.findMembers();
    
    List<MemberDto> collect = findMembers.stream()
        .map(m-> new MemberDto(m.getName()))
        .collect(Collectors.toList());
    
    return new Result(collect);
}

@Data
@AllArgsConstructor
class Result<T>{
    private T data;
}

@Data
@AllArgsConstructor
class MemberDto{
    private String name;
}
```

**DTO로 반환한 경우**  

* 엔티티가 변해도 API 스펙이 변경되지 않는다.
* 추가로 ``Result`` 클래스로 컬렉션을 감싸서 향후 필요한 필드를 추가할 수 있다.

***

### **지연 로딩이 포함된 경우**  

엔티티로 직접 반환하는 경우는 건너뛰겠다.  

```java
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2(){
    List<Order> orders = orderRepository.findAll();
    List<SimpleOrderDto> result = orders.stream()
        .map(o-> new SimpleOrderDto(o))
        .collect(toList());
    
    return result;
}

@Data
static class SimpleOrderDto{
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    
    public SimpleOrderDto(Order order){
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
    }
}
```

엔티티를 DTO로 변환하는 일반적인 방법이다.  

쿼리가 총 1 + n + n 번 실행된다.  

* ``order`` 조회 1번 (order 조회 결과 수가 n이 된다)
* ``order -> member`` 지연 로딩 조회 n 번
* ``order -> delivery`` 지연 로딩 조회 n번



```java
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> ordersV3(){
    List<Order> orders = orderRepository.findAllWithMemberDelivery();
    List<SimpleOrderDto> result = orders.stream()
        .map(o -> new SimpleOrderDto(o))
        .collect(toList());
    
    return result;
}

public List<Order> findAllWithMemberDelivery(){
    return em.createQuery(
    	"select o from Order o"+
    		" join fetch o.member m"+
    		" join fetch o.delivery d", Order.class)
        .getResultList();
}
```

페치 조인하는 경우이다.  

* 엔티티를 페치 조인(fetch join)을 사용해서 쿼리 1번에 조회하였다.  
* 페치 조인으로 ``order -> member``, ``order -> delivery``는 이미 조회된 상태이므로 지연로딩 X

***

### **컬렉션이 포함된 경우**  

**엔티티를 직접 노출하는 경우**  

```java
@GetMapping("/api/v1/orders")
public List<Order> ordersV1(){
    List<Order> all = orderRepository.findAll();
    for(Order order : all){
        order.getMember().getName();	//LAZY 강제 초기화
        order.getDelivery().getAddress();//LAZY 강제 초기화
        List<OrderItem> orderItems = order.getOrderItems();
        orderItems.stream()
            .forEach(o->o.getItem().getName());	//LAZY 강제 초기화
    }
    return all;
}
```

* ``orderItem``, ``item`` 관계를 직접 초기화하면 ``Hibernate5Module`` 설정에 의해 엔티티를 JSON으로 생성한다.
* 양방향 연관관계면 무한 루프에 걸리지 않게 한곳에 ``@JsonIgnore``를 추가해야 한다.
* 엔티티를 직접 노출하므로 좋은 방법은 아니다.



**엔티티를 DTO로 변환하는 경우**  

```JAVA
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2(){
    List<Order> orders = orderRepository.findAll();
    List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
    
    return result;
}

@Data
static class OrderDto{
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemDto> orderItems;
    
    public OrderDto(Order order){
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
        orderItems = order.getOrderItems().stream()
            .map(orderItem -> new OrderItemDto(orderItem))
            .collect(toList());
    }
}

@Data
static class OrderItemDto{
    private String itemName;
    private int orderPrice;
    private int count;
    
    public OrderItemDto(OrderItem orderItem){
        itemName = orderItem.getItem().getName();
        orderPrice = orderITem.getOrderPrice();
        count = orderItem.getCount();
    }
}
```

이 방법은 지연로딩을 이용하기 때문에 SQL 실행이 너무나도 많다.  

``order`` 1번, ``member``,``address`` N번(order 조회 수 만큼), ``orderItem`` N번(order 조회 수 만큼),  

``item`` N번(orderItem 조회 수 만큼)  



**페치 조인을 이용한 경우**  

```java
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3(){
    List<Order> orders = orderRepository.findAllWithItem();
    List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
    return result;
}

public List<Order> findAllWithItem(){
    return em.createQuery(
    	"select distinct o from Order o"+
    		" join fetch o.member m"+
    		" join fetch o.delivery d"+
    		" join fetch o.orderItems oi"+
    		" join fetch oi.item i", Order.class)
        .getResultList();
}
```

* 페치 조인으로 SQL이 1번만 실행된다.
* ``distinct``를 사용한 이유는 일대다 조인에서 row수가 증가하게 되므로, 중복을 걸러주기 위해 사용했다. 
* 단점은 **페이징이 불가능**하다는 것이다.



**페이징 처리를 가능하게 하는 경우**  

컬렉션을 페치 조인하면 페이징이 불가능하다. row 수가 예측 불가능하게 증가하기 때문이다.  
따라서, ToOne 관계를 먼저 모두 페치조인을 한다. ToOne 관계는 row 수를 증가시키지 않기 때문에 페이징 쿼리에 영향을 주지 않기 때문이다. 이후 컬렉션은 지연로딩으로 조회한다.  
지연 로딩 성능 최적화를 위해 ``hibernate.default_fetch_size``, ``@BatchSize``를 적용한다.  
이 옵션을 사용하면 컬렉션이나, 프록시 객체를 한꺼번에 설정한 size 만큼 IN 쿼리로 조회하게 된다.  

```java
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(
    		@RequestParam(value = "offset",defaultValue="0")int offset,
			@RequestParam(value="limit",defaultValue="100") int limit){
    
    List<Order> orders = orderRepository.findAllWithMemberDelivery(offset,limit);
    List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
    
    return result;
}

spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 1000
```

개별 설정은 ``@BatchSize``를 적용하면 된다.  

만약에, order 가 2개고 내부 orderItem이 2개였다면 order와 delivery를 가지고 오는 호출 1번, 그리고 order가 2개니깐 2번에 각각 orderItem이 2개씩 있으니 orderItem만 4번 호출이 됐을 것이다.  
그러나 ``@BatchSize``를 이용하면 처음 order와 delivery 땡겨오는 1번, 그리고 나머지 order 루프와 orderitem 루프도 1번에 해결이 된다. 설정한 BatchSize 만큼 미리 땡겨와서 IN 쿼리를 수행하기 때문이다.  

모두 다 fetch join을 하면 1번의 호출로 가능하지만, 위의 상황은 3번의 호출이 이루어진다. 하지만, DB 데이터 전송량이 최적화 되고 무엇보다도 페이징이 가능하다. 그리고 fetch를 여러 테이블 한 번에 하게되면 중복데이터가 많아진다.  

* 장점
  * 쿼리 호출 수가 1+N 에서 1+1 로 최적화 된다.
  * 조인보다 DB 데이터 전송량이 최적화 된다. 
  * 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만, DB 데이터 전송량이 감소한다.
  * 컬렉션 페치 조인은 페이징이 불가능 하지만 이 방법은 페이징이 가능하다.
* 결론
  * ToOne 관계는 페치 조인해도 페이징에 영향을 주지 않는다. 따라서 ToOne 관계는 페치조인으로 쿼리 수를 줄이고 해결하고, 나머지는 ``hibernate.default_batch_fetch_size``로 최적화 하자.

***

DTO로 바로 받아오는 작업은 생략하겠다.  
