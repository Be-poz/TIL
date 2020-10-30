# 영속성 전이(CASCADE)와 고아 객체에 대해

본 내용은 자바 ORM 표준 JPA 프로그래밍 책을 토대로 정리했습니다.  

***

### 영속성 전이(CASCADE)

```java
            Child child1 = new Child();
            Child child2 = new Child();

            Parent parent = new Parent();
            parent.addChild(child1);
            parent.addChild(child2);

            em.persist(child1);
            em.persist(child2);
            em.persist(parent);
```

Parent 안에 Child를 갖고 있고 addChild를 통해서 집어넣어주었다. 하지만 이 3개를 다 persist 해줘야 하는데, 이것을 영속성 전이를 통해서 편리하게 할 수가 있다.  

```java
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> childList = new ArrayList<Child>();
```

이렇게 말이다. ``orphanRemoval``은 상관이 없다. 차후에 설명하겠다.  

CasccadType에는 여러 가지가 있다.  

ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH. ALL은 모두 적용이다. 여러 속성을 같이 사용하고 싶을 때에는  

``cascade = {CascadeType.PERSIST, CascadeType.REMOVE}`` 다음과 같이 사용할 수가 있다. 그리고 물론 parent를 remove하면 parent들도 다 삭제가 된다.  

***

### 고아 객체

JPA는 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하는 기능을 제공하는데 이것을 고아 객체 제거라 한다.  

``orpahnRemoval = true`` 가 여기서 사용된다.  

```java
Parent parent1 = em.find(Parent.class, id);
parent1.getChildren().remove(0);
```

이렇게 삭제된 Child는 자동으로 삭제가 되게 된다.  

고아 객체 제거는 **참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능** 이다.  



CascadeType.ALL + orphanRemoval =true 를 동시에 사용하게되면 엔티티 스스로 생명주기를 관리한다는 뜻이다.  

영속성 전이나 고아 객체나 항상 해당 엔티티가 다른 곳에서 사용되지 않을 때에만 써야 안전하다. 반드시 지향하자.  

***