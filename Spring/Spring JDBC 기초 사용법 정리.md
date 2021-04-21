# Spring JDBC 기초 사용법 정리

### NamedParameterJdbcTemplate

```java
@Repository
public class NamedParamDAO {
    private NamedParameterJdbcTemplate namedParameterJdbcTemplate;

    public NamedParamDAO(NamedParameterJdbcTemplate namedParameterJdbcTemplate) {
        this.namedParameterJdbcTemplate = namedParameterJdbcTemplate;
    }

    /**
     * MapSqlParameterSource
     * public <T> T queryForObject(String sql, SqlParameterSource paramSource, Class<T> requiredType)
     */
    public int useMapSqlParameterSource(String firstName) {
        String sql = "select count(*) from customers where first_name = :first_name";

        SqlParameterSource namedParameters = new MapSqlParameterSource("first_name", firstName);

        return this.namedParameterJdbcTemplate.queryForObject(sql, namedParameters, Integer.class);
    }

    /**
     * BeanPropertySqlParameterSource
     * public <T> T queryForObject(String sql, SqlParameterSource paramSource, Class<T> requiredType)
     */
    public int useBeanPropertySqlParameterSource(Customer customer) {
        String sql = "select count(*) from customers where first_name = :firstName";

        SqlParameterSource namedParameters = new BeanPropertySqlParameterSource(customer);

        return this.namedParameterJdbcTemplate.queryForObject(sql, namedParameters, Integer.class);
    }
}
```

``:firstName`` 와 같이 두면 해당 값을 ``MapSqlParamewterSource(:다음에 적은 값)`` 을 통해 값을 넣어줄 수가 있다.  
``BeanPropertySqlParameterSource(객체)`` 를 이용하면 해당 객체 내부의 ``:적은 값``을 해당 객체의 필드에서 뽑아와서 알 수가 있다.  

<br/>

### Select 문

```java
@Repository
public class QueryingDAO {
    private JdbcTemplate jdbcTemplate;

    public QueryingDAO(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    private final RowMapper<Customer> actorRowMapper = (resultSet, rowNum) -> {
        Customer customer = new Customer(
                resultSet.getLong("id"),
                resultSet.getString("first_name"),
                resultSet.getString("last_name")
        );
        return customer;
    };

    /**
     * public <T> T queryForObject(String sql, Class<T> requiredType)
     */
    public int count() {
        String sql = "select count(*) from customers";
        return jdbcTemplate.queryForObject(sql, Integer.class);
    }

    /**
     * public <T> T queryForObject(String sql, Class<T> requiredType, @Nullable Object... args)
     */
    public String getLastName(Long id) {
        String sql = "select last_name from customers where id = ?";
        return jdbcTemplate.queryForObject(sql, String.class, id);
    }

    /**
     * public <T> T queryForObject(String sql, RowMapper<T> rowMapper, @Nullable Object... args)
     */
    public Customer findCustomerById(Long id) {
        String sql = "select id, first_name, last_name from customers where id = ?";
        return jdbcTemplate.queryForObject(
                sql,
                (resultSet, rowNum) -> {
                    Customer customer = new Customer(
                            resultSet.getLong("id"),
                            resultSet.getString("first_name"),
                            resultSet.getString("last_name")
                    );
                    return customer;
                }, id);
    }

    /**
     * public <T> List<T> query(String sql, RowMapper<T> rowMapper)
     */
    public List<Customer> findAllCustomers() {
        String sql = "select id, first_name, last_name from customers";
        return jdbcTemplate.query(
                sql,
                (resultSet, rowNum) -> {
                    Customer customer = new Customer(
                            resultSet.getLong("id"),
                            resultSet.getString("first_name"),
                            resultSet.getString("last_name")
                    );
                    return customer;
                });
    }

    /**
     * public <T> List<T> query(String sql, RowMapper<T> rowMapper, @Nullable Object... args)
     */
    public List<Customer> findCustomerByFirstName(String firstName) {
        String sql = "select id, first_name, last_name from customers where first_name = ?";
        return jdbcTemplate.query(sql, actorRowMapper, firstName);
    }
}
```

``queryForObject``는 결과 쿼리라인이 1개일 때 사용한다.  
``jdbcTemplate.queryForObject("select count(*) from customers", Integer.class);`` 에서는 결과값이 Integer 형태로 나오게되므로 ``Integer.class``를 인수로 넣어주었다.  

```java
        String sql = "select last_name from customers where id = ?";
        return jdbcTemplate.queryForObject(sql, String.class, id);
```

위 경우에서도 결과가 String 타입이므로 ``String.class``를 넣어준 것을 볼 수가 있다. ?  의 등장순서에 맞게끔 인수로 id를 넣어준 것도 확인할 수가 있다.  

```java
    public Customer findCustomerById(Long id) {
        String sql = "select id, first_name, last_name from customers where id = ?";
        return jdbcTemplate.queryForObject(
                sql,
                (resultSet, rowNum) -> {
                    Customer customer = new Customer(
                            resultSet.getLong("id"),
                            resultSet.getString("first_name"),
                            resultSet.getString("last_name")
                    );
                    return customer;
                }, id);
    }
```

위 코드에서는 여러 컬럼들이 나오게된다. 이를 ``resultSet.get타입("컬럼명")``과 같이 사용해서 원하는 값을 뽑아올 수 있고, 예제처럼 객체를 생성해서 반환해줄 수도 있다.  

```java
    public List<Customer> findAllCustomers() {
        String sql = "select id, first_name, last_name from customers";
        return jdbcTemplate.query(
                sql,
                (resultSet, rowNum) -> {
                    Customer customer = new Customer(
                            resultSet.getLong("id"),
                            resultSet.getString("first_name"),
                            resultSet.getString("last_name")
                    );
                    return customer;
                });
    }
```

where 절을 사용하게되면 쿼리의 결과 수가 1이 아닌 다수가 나오게 되는데, ``query``를 사용하게되면 List 형태로 뽑아올 수 있게 된다. 내부 컬럼을 뽑는 것은 이전 예제와 같다. 만약 first_name만 뽑아오고 싶다면 ``(resultSet, rowNum) -> resultSet.getString("first_name")``와 같이 사용하면 된다.  

```java
    private final RowMapper<Customer> actorRowMapper = (resultSet, rowNum) -> {
        Customer customer = new Customer(
                resultSet.getLong("id"),
                resultSet.getString("first_name"),
                resultSet.getString("last_name")
        );
        return customer;
    };
```

다음과 같이 메서드로 따로 뽑아내는 것도 가능하다.  

<br/>

### Insert 문, Delete 문, Update 문

```java
@Repository
public class UpdatingDAO {
    private JdbcTemplate jdbcTemplate;

    public UpdatingDAO(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    private final RowMapper<Customer> actorRowMapper = (resultSet, rowNum) -> {
        Customer customer = new Customer(
                resultSet.getLong("id"),
                resultSet.getString("first_name"),
                resultSet.getString("last_name")
        );
        return customer;
    };

    /**
     * public int update(String sql, @Nullable Object... args)
     */
    public void insert(Customer customer) {
        String sql = "insert into customers (first_name, last_name) values (?, ?)";
        jdbcTemplate.update(sql, customer.getFirstName(), customer.getLastName());
    }
    /**
     * public int update(String sql, @Nullable Object... args)
     */
    public int delete(Long id) {
        String sql = "delete from customers where id = ?";
        return jdbcTemplate.update(sql, Long.valueOf(id));
    }

    /**
     * public int update(final PreparedStatementCreator psc, final KeyHolder generatedKeyHolder)
     */
    public Long insertWithKeyHolder(Customer customer) {
        String sql = "insert into customers (first_name, last_name) values (?, ?)";

        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, customer.getFirstName());
            ps.setString(2, customer.getLastName());
            return ps;
        }, keyHolder);

        return keyHolder.getKey().longValue();
    }
}
```

``jdbcTemplate.update()``를 사용하면 Insert, Delete, Update 문을 처리할 수가 있다.  
insert를 실행한 후에 해당 id 값을 바로 얻을 수 있는 방법이 있다.  

```java
    public Long insertWithKeyHolder(Customer customer) {
        String sql = "insert into customers (first_name, last_name) values (?, ?)";

        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, customer.getFirstName());
            ps.setString(2, customer.getLastName());
            return ps;
        }, keyHolder);

        return keyHolder.getKey().longValue();
    }
```

뽑아내고자 하는 값을 ``new String[] {"id"}`` 로 뽑아서 바로 리턴하는 것을 확인할 수가 있다.  

<br/>

### SimpleJdbcInsert

```java
@Repository
public class SimpleInsertDao {
    private SimpleJdbcInsert insertActor;

    public SimpleInsertDao(DataSource dataSource) {
        this.insertActor = new SimpleJdbcInsert(dataSource)
                .withTableName("customers")
                .usingGeneratedKeyColumns("id");
    }

    /**
     * Map +
     * insertActor.executeAndReturnKey
     */
    public Customer insertWithMap(Customer customer) {
        Map<String, Object> parameters = new HashMap<String, Object>(3);
        parameters.put("first_name", customer.getFirstName());
        parameters.put("last_name", customer.getLastName());
        Long id = insertActor.executeAndReturnKey(parameters).longValue();
        return new Customer(id, customer.getFirstName(), customer.getLastName());
    }

    /**
     * SqlParameterSource +
     * insertActor.executeAndReturnKey
     */
    public Customer insertWithBeanPropertySqlParameterSource(Customer customer) {
        SqlParameterSource parameters = new BeanPropertySqlParameterSource(customer);
        Long id = insertActor.executeAndReturnKey(parameters).longValue();
        return new Customer(id, customer.getFirstName(), customer.getLastName());
    }
}
```

키 값을 바로 빼내는 방법으로 추가적으로 위와 같은 코드를 이용할 수도 있다.  

***