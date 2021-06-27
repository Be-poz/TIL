# BindingResult와 Errors의 차이점 에 대해

에러처리를 할 때에 파라미터로 ``BindingResult``와 ``Errors``를 사용할 때가 있다. 이 둘의 차이점은 무엇일까?  

``BindingResult``는 인터페이스고, ``Errors`` 인터페이스를 상속받고 있다.  
실제 넘어오는 구현체는 ``BeanPropertyBindingResult`` 인데 ``BindingResult``대신 ``Errors``를 사용해도 된다.  

``Errors`` 인터페이스는 단순한 오류 저장과 조회 기능을 제공한다.  
``BindingResult``는 여기에 더해서 추가적인 기능들을 제공한다. (ex. ``addError()``)

***

### REFERENCE

https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-2/dashboard