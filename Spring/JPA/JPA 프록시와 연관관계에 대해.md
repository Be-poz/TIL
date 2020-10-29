# JPA 프록시와 연관관계에 대해

본 내용은 자바 ORM 표준 JPA 프로그래밍 책을 토대로 정리했습니다.  

```java
Member findMember1 = em.find(Member.class, member.getId());
Member findMember2 = em.getReference(Member.class, member.getId());
System.out.println("findMember.id=" + findMember.getId());
System.out.println("findMember.username=" + findMember.getUsername());
```

첫째 줄의 코드는 바로 query를 실행하는 반면, 두번째 줄의 코드는 실행을 하지않다가 print 에서 getId와 username을 만나고 나서야 실행을 했다. findMember2.getClass() 를 print 해보니 다음과 같이 나왔다.``class hellojpa.Member$HibernateProxy$odcVHpjy``  

이 녀석은 무엇일까???  

***

* em.find(): DB를 통해서 실제 엔티티 객체 조회
* em.getReference(): DB 조회를 미루는 가짜(프록시) 엔티티 객체 조회

이 프록시는 껍데기는 같고 내부는 텅텅 비어있다. ``Entity target = null`` 라는 값 또한 가지고 있다.  

사용하는 입장에서는 진짜 객체인지 프록시 객체인지 구분하지 않고 사용하면 되겠다.  

프록시 객체는 실제 객체의 참조(target)를 보관한다. 프록시 객체를 호출하면 프록시 객체는 실제 객체의 메서드를 호출하게 된다.  

```java
Membermember = em.getReference(Member.class, "id1");
member.getname();
```

다음의 초기 상황에서 MemberProxy 객체는 target 값이 null 이다.  

이때 getName()이 호출이 되면 target이 null 이기 때문에 영속성 컨텍스트에 초기화를 요청하게 되고 DB를 조회하게 된다.  

DB조회를 통해 실제 Entity를 생성하게 되고 target은 이 Entity를 가리키게 된다. target.getName()을 실행하면 진짜 Entity에 가서 해당 메서드를 실행하게 되는 것이다.  



### 프록시 특징

* 프록시 객체는 처음 사용할 때 한 번만 초기화한다.
* 프록시 객체를 초기화 할 때, 프록시 객체가 실제 엔티티로 바뀌는 것이 아니다. 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능한 것이다.
* 프록시 객체는 원본 엔티티를 상속 받는다. 따라서 타입 체크 시 주의해야한다.(==비교 대신 instance of 를 사용)

```java
Member member1 = new Member();
member1.setUsername("member1");
em.persist(member1);

Member member2 = new Member();
member2.setUsername("member2");
em.persist(member2);

em.flush();
em.clear();

Member m1 = em.find(Member.class, member1.getId());
Member m2 = em.find(Member.class, member2.getId());

System.out.println("m1==m2: " + (m1.getClass() == m2.getClass()));
```

위의 경우에는 true 로 찍힌다. 하지만, m2 를 em.getReference 를 하게되면?? false 가 뜬다. 이게 실무에서 해당 엔티티가 프록시가 올지 진짜가 올지 알 수 없기 때문에 == 비교를 하면 안된다는 것이다.  

* 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.getReference()를 호출해도 실제 엔티티를 반환한다.

```java
Member member1 = new Member();
member1.setUsername("member1");
em.persist(member1);

Member member2 = new Member();
member2.setUsername("member2");
em.persist(member2);

em.flush();
em.clear();

Member m1 = em.find(Member.class, member1.getId());
System.out.println("m1= "+m1.getClass());

Member reference = em.getReference(Member.class, member1.getId());
System.out.println("reference= "+reference.getClass());

System.out.println(m1 == reference);
```

위의 경우에는 둘 다 실제 엔티티를 print 해준다. 그 이유는 이미 멤버를 1차 캐쉬에 있는데 프록시로 가져올 이점이 없기 때문이다.  

위의 상황에서 둘 다 em.getReference() 를 사용해도 true가 나온다.  

**JPA는 한 영속성 컨텍스트이고 같은 pk 값이면 무조건 true를 반환해주어야 하기 때문이다.** 이게 사실 진짜 이유다.  

그렇다면,  

```java
Member refMember = em.getReference(Member.class, member1.getId());
System.out.println("refMember= "+refMember.getClass());

Member findMember = em.find(Member.class, member1.getId());
System.out.println("findMember= "+findMember.getClass());

System.out.println(refMember == findMember);
```

이 경우는 어떨까?? 처음에 프록시 객체가 있고, 이후에 진짜 엔티티를 가지고 올 것이다. 그렇다면 false 값이 나올텐데, JPA는 true를 보장해주어야 한다. 놀랍게도 findMember도 프록시를 반환한다. 그 결과 true를 반환하게 된다.  

그러니깐, **em.find() 를 사용해도 프록시가 나올 수 있다는 것이다.** 우리는 개발을 할 때에 어차피 해당 객체가 프록시 객체인지 모르고, 알 필요도 없다.  

* 영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일 때, 프록시를 초기화하면 문제가 발생한다.

```java
Member refMember = em.getReference(Member.class, member1.getId());
System.out.println("refMember= "+refMember.getClass());

refMember.getUsername();
```

이 경우에는 프록시 객체가 생성되었다가 getUsername() 때에 영속성 컨텍스트에 초기화를 요청하고 DB를 조회한다.  

```JAVA
Member refMember = em.getReference(Member.class, member1.getId());
System.out.println("refMember= "+refMember.getClass());

em.detach(refMember);

refMember.getUsername();
```

이러면 오류가 발생한다. 준영속 상태가 되었기 때문이다. getReference를 호출하는 시점에서 id 값은 넘기기 때문에 프록시 객체에서 최소한 id 값은 내부에 가지게 된다. 그래서 refMember.getId()는 오류가 나지 않는다.  

추가로 덧붙이자면, 이전같으면 em.close() 를 호출하고 getUsername를 호출했을 때에 오류가 떴을 것이다. 그러나, hibernate가 업데이트가 되면서, 트랜잭션이 살아있으면 em.close()를 호출해도 완전히 닫히지 않은 읽기 가능 상태로 남아있게 된다. 그래서 close 한다해도 select을 쏜다. 하지만, 실무에서는 트랜잭션을 먼저 닫고 em.close()를 하기 때문에 크게 걱정안해도 된다고 한다.  



### 프록시 확인

* 프록시 인스턴스의 초기화 여부 확인 (PersistenceUnitUtil.isLoaded(Object entity))
* 프록시 클래스 확인 방법 (entity.getClass())
* 프록시 강제 초기화 (org.hibernate.Hibernate.initialize(entity);)  (hibernate 에서 하는 것이고, JPA 표준에는 강제 초기화가 없다.)

```java
Member refMember = em.getReference(Member.class, member1.getId());
//프록시 클래스 확인 함
System.out.println("refMember= "+refMember.getClass());
//프록시 인스턴스의 초기화 여부 확인함
//여기에서는 false 가 뜨겠지만, getUsername() 같은걸로 초기화 하면 true가 뜰 것임
System.out.println("isLoaded= "+emf.getPersistenceUnitUtil.isLoaded(refMember));

//강제 초기화, getUsername() 뭐 이런걸로 초기화 안해두 된다.
Hibernate.initialize(refMember);
```

***

