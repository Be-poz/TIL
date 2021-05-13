# [JDBC] 단일 Select을 위한 방법들

```sql
Line
id bigint auto_increment not null,
name varchar(255) not null unique,
color varchar(20) not null,
primary key(id)
```

다음과 같은 ``Line`` 테이블이 있다.  

``select * from line where name = ?`` 다음과 같은 쿼리문의 결과값을 받아오려는 메소드를 만든다고 한다.  

```java
public Line findLineByName(String name) {
  String sql = "select * from line where name = ?";
  return jdbcTemplate.queryForObject(sql, lineRowMapper(), "1호선");
}

private RowMapper<Line> lineRowMapper() {
  return (rs, rowNum) -> new Line(
  					rs.getLong("id"),
  					rs.getString("name"),
  					rs.gwetSTring("color")
  );
}
```

다음과 같이 확인할 수 있을 것이다.  

이번에 내가 글을 쓴 이유는 다음과 같다.  
만약 위의 경우에서 결과값이 0개거나 2개 이상이면 예외가 발생할 것이다. 각각, ``EmptyResultDataAccessException``과 ``IncorrectResultSizeDataAccessException``이 발생하게 된다(전자의 예외는 후자의 예외를 상속받고 있다).  

해당 예외를 잡아줘서 ``null``을 던져주던가 해야하는데 그게 싫어서 ``Optional``을 사용한다고 가정하자.  

```java
public Optional<Line> findLineByName(String name) {
  String sql = "select * from line where name = ?";
  try{
    return Optional.ofNullable(jdbcTemplate.queryForObject(sql, lineRowMapper(), "1호선"));
  } catch(IncorrectResultSizeDataAccessException e) {
    return Optional.empty();
  }
}

public Optional<Line> findLineByName(String name) {
  String sql = "select * from line where name = ?";
  return jdbcTemplate.query(sql, lineRowMapper(), "1호선").stream().findAny();
}
```

다음과 같은 방식으로 사용할 수 있을 것이다.  

전자의 방법은 ``queryForObject``를 사용한 방식이다. 값을 제대로 찾으면 ``Optional.ofNullable()``로 감싸서 리턴하지만 그 값이 0개나 2개 이상인 경우 따로 예외를 잡아서 ``Optional.empty()``를 반환해주고 있다.  

``query`` 방식은 ``List``로 반환을 받는다. 이를 스트림으로 돌려서 아무거나 찾은 후에 리턴을 한다. 만약 찾지 못한 경우에는 ``Optional.empty()``를 반환하게 된다.  

가독성 측면에서는 후자가 훨 좋아보인다. 전자의 방법은 결과값의 개수가 딱 1개일 때에는 제대로 리턴하고 그 외에는 예외가 터지기 때문에 따로 잡아줘야 한다. 후자의 방법은 0,1개 일 때에는 적절하지만 2개 이상일 때에는 어떤 값을 찾아서 리턴해줄지 알 수 없다. 그리고 애초에 단일 건을 조회하기 위한 메소드가 아니기도 하다.  

이렇게 장단점이 있다. 어떤 것을 쓰든 취향껏 사용하면 될 것 같다. 추가로 다음과 같은 방법도 있다.  

```java
public Line findLineByName(String name) {
  String sql = "select * from line where name = ?";
  List<Line> lines = jdbcTemplate.queryForObject(sql, lineRowMapper(), "1호선");
  Line line = DataAccessUtils.singleResult(lines);
  return line;
}
```

``DataAccessUtils.singleResult(Collection)``이 바로 그 방법이다.  

``query``메소드를 사용했는데 단일 행이 나올 것이라고 예상되는 구문을 ``Collection`` 타입을 파라미터로 받는 ``singleResult``메소드에 넣어주었다. 파라미터의 사이즈를 체크해서 0이면 ``null``을 리턴하고 2개 이상이면 ``IncorrectResultSizeDataAccessException``를 리턴한다. ``Optional``형식을 사용하지 않고 단일 객체를 리턴하는데 ``query``를 사용했을 경우에 이용할 수 있겠다.  

여러가지 방법을 살펴봤는데 상황에 맞게끔 적절히 사용하는 것이 좋다고 생각한다.  

***