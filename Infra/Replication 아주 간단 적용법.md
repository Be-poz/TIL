# Replication 아주 간단 적용법

![image](https://user-images.githubusercontent.com/45073750/138593806-def141d7-4d37-4983-9fec-a7786b4776ff.png)
![image](https://user-images.githubusercontent.com/45073750/138593879-973e034a-bf55-4679-8091-7bf2fa637c4b.png)

주석 해제(bind-address = 0.0.0.0 도 필요에 따라)  

sudo service mysql restart  

use mysql;

create user '{userName}'@'{% or IP}' identified by '{userPassword}';  (%는 모든 ip에 대해서 열어두는 것)  
[링크](https://stackoverflow.com/questions/50169576/mysql-8-0-11-error-connect-to-caching-sha2-password-the-specified-module-could-n) mysql8 이상에서는 비번이 뭔가 강화되는 것 같음. 비번을 mysql_native_password 로 바꿔줘야함  

![image](https://user-images.githubusercontent.com/45073750/138594344-cf236310-a8f4-409b-b4da-d8bbae5369c7.png)

``GRANT REPLICATION SLAVE ON {schema}.{tableName} TO '{userName}'@'{% or IP}' ``  

![image](https://user-images.githubusercontent.com/45073750/138594488-01dd9353-4ae4-45cb-8279-2221d263ce45.png)

File: 바이너리 로그파일  
Position: 바이너리 로그 파일 내부의 위치  

slave db의 /etc/mysql/mysql.conf.d/mysqld.cnf 에 밑과 같이 기입 (server-id가 master와 달라야함)  

![image](https://user-images.githubusercontent.com/45073750/138594826-70e0b803-0c90-4e8d-84cd-d0b33ccade67.png)

slave mysql 들어감  

use mysql;

``CHANGE MASTER TO MASTER_HOST ='{Master DB IP}', MASTER_PORT = {Master DB Port}, MASTER_USER = '{마스터에 만들어준 slave 유저 명}', MASTER_PASSWORD = '{유저의 비밀번호}', MASTER_LOG_FILE = '{bianry log file name}', MASTER_LOG_POS = {position value}``

start slave;

show slave status\G 로 상태확인  

slave_io_state: waiting for master to send event  
slave_io_running: yes  
slave_sql_running: yes  

가 나와야함.  다르다면  

```mysql
$ mysql -h {Master 서버 IP} -u {replication 계정} -p
Enter password: {replication 비밀번호}
```

이걸로 확인해봐야함  

slave db에도 같은 스키마가 있어야 함. 따로 추가해주기 귀찮으니 master 쪽에서 덤프 떠서 scp로 보내주자  

![image](https://user-images.githubusercontent.com/45073750/138597376-3a6e22a1-84c0-4879-92c1-6d6f0e72b552.png)

-u {userName} -p {schema} > {fileName}.sql  

sudo scp -i {key 위치} {목적 파일} {원격지ip}@{원격지ip}:{가져다 둘 원격지 장소}  

sudo mysql -u {유저id} -p {schema Name} < {dump file Name}

sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 3306 (slave db 8080 -> 3306 포트포워딩)  

```java
@Profile({"prod", "performance"})
@Configuration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
@EnableConfigurationProperties(MasterDataSourceProperties.class)
public class DataSourceConfiguration {

    private final MasterDataSourceProperties dataSourceProperties;
    private final JpaProperties jpaProperties;

    public DataSourceConfiguration(
            MasterDataSourceProperties dataSourceProperties,
            JpaProperties jpaProperties
    ) {
        this.dataSourceProperties = dataSourceProperties;
        this.jpaProperties = jpaProperties;
    }

    @Bean
    public DataSource routingDataSource() {
        DataSource master = createDataSource(
                dataSourceProperties.getUrl(),
                dataSourceProperties.getUsername(),
                dataSourceProperties.getPassword()
        );

        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("master", master);
        dataSourceProperties.getSlave().forEach((key, value) ->
                dataSources.put(value.getName(), createDataSource(
                        value.getUrl(), value.getUsername(), value.getPassword()
                ))
        );

        ReplicationRoutingDataSource replicationRoutingDataSource = new ReplicationRoutingDataSource();
        replicationRoutingDataSource.setDefaultTargetDataSource(dataSources.get("master"));
        replicationRoutingDataSource.setTargetDataSources(dataSources);

        return replicationRoutingDataSource;
    }

    private DataSource createDataSource(String url, String username, String password) {
        return DataSourceBuilder.create()
                .type(HikariDataSource.class)
                .url(url)
                .driverClassName("com.mysql.cj.jdbc.Driver")
                .username(username)
                .password(password)
                .build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        EntityManagerFactoryBuilder entityManagerFactoryBuilder =
                createEntityManagerFactoryBuilder(jpaProperties);

        return entityManagerFactoryBuilder.dataSource(dataSource())
                .packages("com.example.tyfserver")
                .build();
    }

    private EntityManagerFactoryBuilder createEntityManagerFactoryBuilder(
            JpaProperties jpaProperties
    ) {
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        return new EntityManagerFactoryBuilder(vendorAdapter, jpaProperties.getProperties(), null);
    }

    private DataSource dataSource() {
        return new LazyConnectionDataSourceProxy(routingDataSource());
    }

    @Bean
    public PlatformTransactionManager transactionManager(
            EntityManagerFactory entityManagerFactory
    ) {
        JpaTransactionManager jpaTransactionManager = new JpaTransactionManager();
        jpaTransactionManager.setEntityManagerFactory(entityManagerFactory);
        return jpaTransactionManager;
    }
}

```

```java
@ConfigurationProperties(prefix = "datasource")
public class MasterDataSourceProperties {

    private final Map<String, Slave> slave = new HashMap<>();

    private String url;
    private String username;
    private String password;

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public Map<String, Slave> getSlave() {
        return slave;
    }

    public static class Slave {

        private String name;
        private String url;
        private String username;
        private String password;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getUrl() {
            return url;
        }

        public void setUrl(String url) {
            this.url = url;
        }

        public String getUsername() {
            return username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }
}
```

```java
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

    private static final Logger LOGGER = LoggerFactory.getLogger(ReplicationRoutingDataSource.class);

    private SlaveNames slaveNames;

    @Override
    public void setTargetDataSources(Map<Object, Object> targetDataSources) {
        super.setTargetDataSources(targetDataSources);

        List<String> replicas = targetDataSources.keySet().stream()
                .map(Object::toString)
                .filter(string -> string.contains("slave"))
                .collect(toList());

        this.slaveNames = new SlaveNames(replicas);
    }

    @Override
    protected String determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly(); //조회 쿼리인 경우
        if (isReadOnly) {
            String slaveName = slaveNames.getNextName(); //다음 slave 선택

            LOGGER.info("Slave DB name: {}", slaveName);

            return slaveName;
        }

        return "master";
    }


    public class SlaveNames {

        private final String[] value;
        private int counter = 0;

        public SlaveNames(List<String> slaveDataSourceProperties) {
            this(slaveDataSourceProperties.toArray(String[]::new));
        }

        public SlaveNames(String[] value) {
            this.value = value;
        }

        public String getNextName() {
            int index = counter;
            counter = (counter + 1) % value.length;
            return value[index];
        }
    }
}
```

```yaml
datasource:
  url: jdbc:mysql://{ip}:{port}/{schema}?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true&useSSL=false&useUnicode=true
  username: {username}
  password: {password}

  slave:
    slave1:
      name: slave-1
      url: jdbc:mysql://{ip}:{port}/{schema}?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true&useSSL=false&useUnicode=true
      username: {username}
      password: {password}
```

```yaml
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        show_sql: false
        generate-ddl: false
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect
        physical_naming_strategy: org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy
```

yaml에 jpa 부분 직접 명시해줘야함. autoconfiguration 뺏기 때문에  

### 마주한 이슈  

실행하면 aws 어쩌고 에러가 뜨길레 이걸로 해결함 [링크](https://github.com/spring-cloud/spring-cloud-aws/issues/556)  

어느 순간 slave -> master 접속이 안되길레 찾아봤더니 다음과 같은 [문제](https://sd23w.tistory.com/414)가 발생해서 세팅을 다시하고 해결해줌  

---

### REFERENCE

https://prolog.techcourse.co.kr/posts/1184  

https://hyeon9mak.github.io/mysql-mariadb-replication-with-jpa/