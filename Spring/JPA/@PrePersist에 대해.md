# @PrePersist에 대해

``@PrePersist`` 어노테이션은 JPA 엔티티 라이프 싸이클의 콜백을 조종할 수 있게하는 어노테이션이다.  

![image](https://user-images.githubusercontent.com/45073750/128022560-780de87d-e7a5-4d59-b8dd-ba96be7052ed.png)

다음과 같은 어노테이션들이 있다. 이런 콜백 이벤트들을 사용하는 방법은 엔티티 내부에 직접 메서드를 작성해주는 것과 ``EntityListener``를 만들어 주는 방법이 있다.  

![image](https://user-images.githubusercontent.com/45073750/128023802-7157953a-8ebe-4fe7-9bf6-088c3767d86d.png)

일반적으로 Auditing을 위해서 ``EntityLIstener``를 사용해본 적이 있을 것이다. 유의할 점은 콜백 메서드들은 void 타입을 리턴해야 한다는 것이다.  

내가 엔티티를 만들고 repository의 ``save``메서드를 호출할 때에 ``@PrePersist``가 달린 메서드가 호출이 되고 DB에 insert된 후에 ``@PostPersist`` 메서드가 호출이 된다. 만약 내가 ``@GeneratedValue``로 pk를 자동생성한다면 해당 pk는 ``@PostPersist``에서 사용할 수 있을 것이다. 그렇다면 코드로 확인해보자.  

![image](https://user-images.githubusercontent.com/45073750/128026568-01258dda-6b72-42c3-b49c-72ca71a619f5.png)

![image](https://user-images.githubusercontent.com/45073750/128026670-06f07143-2dc9-441f-a854-8351e51d34d1.png)

다음과 같이 코드를 작성했다.  
``@PrePersist``가 달린 메서드에 ``name``을 넣어주고 ``id``를 출력했다.  
``@PostPersist``가 달린 메서드에는 id를 출력하는 코드를 넣어주었다.  

출력결과를 살펴보자.  

![image](https://user-images.githubusercontent.com/45073750/128026867-60af101e-e015-468d-9db5-027e6e5a47cb.png)

``em.persist(positive)``라인에서 ``null, 쿼리문, 1``이 출력된 것을 확인할 수가 있었다.  
``persist``메서드가 호출이되자 ``@PrePersist``가 먼저 콜백되었고 해당 코드내에서 ``name``을 set하고 ``id``를 출력해준 것을 알 수가 있다. ``id`` 값은 ``null``로 출력이 되었다. ``@GeneratedValue``를 이용했기 때문이다.  

이후 쿼리문이 호출되었는데, ``name``에는 코드에서 설정해둔 bepoz가 쿼리문에 들어가는 것을 확인할 수가 있었다.  
``@PostPersist``가 호출이 되어 ``id``값이 1이 출력되는 것 또한 확인할 수가 있었다.  

  

내가 한 가지 우려했던 점은 기본 키 생성 전략이 ``IDENTITY``이기 때문에 영속성 컨텍스트에 집어넣기 이전에 pk를 얻어오기 위해 insert 쿼리를 날리게 되는데, 이 순서가 ``@PrePersist``보다 먼저 일어날까봐 걱정을 했다.  

그렇다면, 내가 의도한 ``name``을 set해주는 코드가 의미가 없어질테니 말이다.(물론, insert가 먼저 일어난 후에 set이 되었다 하더라도 dirty checking으로 db에 값을 넣어줄 수 있긴 했을거임. update 문을 한 번 더 실행해야겠지만)  

해당 어노테이션을 나는 엔티티 필드 중에 UUID를 만들어줄 일이 있어서 사용을 해봤는데 이후에 생성자에서 그냥 만들어주는 것으로 바꿔주었다. 엔티티 라이프 싸이클 주기를 컨트롤 할 때에 유용하게 사용할 수 있는 어노테이션인 것 같다.  

***

### REFERENCE

https://www.baeldung.com/jpa-entity-lifecycle-events