# 클래스의 ToString에 대해

클래스에는 Object 클래스의 .toString() 메서드를 기본적으로 사용하기 때문에 따로 오버라이드를 해주지 않으면 해당 클래스의 해시코드가 나오게 된다. 클래스 내의 필드들을 toString 하고 싶다면 내 입맛대로 구현해줄 수도 있지만, 이를 도와주는 여러 메서드들이 있다.  

***

### ToStringBuilder

``org.apache.commons.commons-lang3`` 의 ``ToStringBuilder``가 있다.  

``new ToStringBuilder(this).append("id",id)....toString()`` 뭐 이런식으로 append 하면서 쭉 붙여서 사용할 수도 있지만 해당 클래스의 필드 값이 변하면 계속 변경을 해주어야 하는 단점이 있다.  

``ToStringBuilder.reflectionToString(this);`` 를 사용하면 해당 클래스의 필드들을 전부 ToString 해준다. ``transient`` 키워드가 붙여진 필드는 자동으로 필터링 한다.  

``ToStringBuilder.reflectionToString(this, ToStringStyle.JSON_STYLE)`` 등 여러 형태로 출력할 수가 있다.  

* 아무것도 붙이지 않았을 경우: ``com.ticket.captain.TestEntity@79fc4e99[age=25,name=testName]``

* ``DEFAULT_STYLE``: ``com.ticket.captain.TestEntity@578d5d02[age=25,name=testName]``

* ``JSON_STYLE``: ``{"age":25,"name":"testName"}``

* ``SIMPLE_STYLE``: ``25,testName``

* ``MULTI_LINE_STYLE``: 

  ```JAVA
  com.ticket.captain.TestEntity@178c6261[
    age=25
    name=testName
  ]
  ```

* ``SHORT_PREFIX_STYLE``: ``TestEntity[age=25,name=testName]``

* ``NO_CLASS_NAME_STYLE``: ``[age=25,name=testName]``

* ``NO_FIELD_NAMES_STYLE``: ``com.ticket.captain.TestEntity@3791af[25,testName]``

***

### LOMBOK의 ``@ToString``

LOMBOK이 가진 어노테이션들은 정말 유용하다. ``@ToString`` 또한 그 중 하나다.  

클래스 위에 해당 어노테이션을 붙여줌으로써 사용할 수가 있다.  

출력형식은 ``TestEntity(name=testName, age=25)`` 다음과 같이 출력이 된다.  

``@ToString.Exclude``를 필드위에 선언해서 제외시킬 수도 있다.  
``@ToString(exclude ="필드명")`` 을 통해서 제외시키는 방법도 있다.  

``@ToString(onlyExplicitlyIncluded = true)``를 클래스에 선언하고 필드에 ``@Tostring.Include``를 적어서 지정된 것만 포함시킬 수도 있다.  
``@ToString(of = {"name","age"})`` 다음과 같이 of를 이용해서 출력할 필드명을 미리 정할 수도 있다.  

***

Reference  

https://projectlombok.org/features/ToString

