``@PersistenceContext`` : 엔티티 매니저(``EntityManager``) 주입  
   스프링 데이터 JPA를 사용하면 ``EntityManager``도 주입 가능하다.  

``@PersistenceUnit`` : 엔티티 매니저 팩토리(``EntityManagerFacotry``) 주입  

``@Transactional`` : 트랜잭션, 영속성 컨텍스트  
  ``readonly = true`` : 데이터의 변경이 없는 읽기 전용 메서드에 사용, 영속성 컨텍스트를 플러시 하지 않으므로 약간의 성능이 향상된다. (읽기 전용에는 다 전용)  그리고 select 시에 스냅샷을 저장하지 않는다. (@QueryHint의 readOnly도 걸린다는 뜻이다)

``@SpringBootTest`` : 스프링 부트 띄우고 테스트 한다.(이게 없으면 ``@Autowired`` 실패 한다)  

***

``merge`` 의 동작 방식  

1. ``merge(member)`` 호출  

2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다.  

   2-1. 만약 1차 캐시에 엔티티가 없으면 DB에서 엔티티를 조회하고, 1차 캐시에 저장한다.  

3. 조회한 영속 엔티티(``mergeMember``)에 ``member``엔티티의 값을 채워 넣는다.

4. 영속 상태인 mergeMember를 반환한다. (처음 넘긴 member가 영속성 상태에 들어가게 되는 것이 아니라 이 반환 값이 들어가게 된다는 것을 주의)

> 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이 변경된다. 병합시 값이 없으면 ``null`` 로 업데이트 할 위험도 있다. (병합은 모든 필드를 교체한다)  
