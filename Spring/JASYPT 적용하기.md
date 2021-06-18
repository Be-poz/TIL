# JASYPT 적용하기

JASYPT는 'Java Simplified Encryption'의 약어이다.  
개발자가 암호화 작동 방식에 대한 깊은 지식 없이도 최소한의 노력으로 프로젝트에 기본 암호화 기능을 추가할 수 있는 JAVA 라이브러리이다.  

* 단방향 및 양방향 암호화를 위한 보안 수준이 높은 표준 기반 암호화 기술이다.
* Hibernate과 transparent한 통합이 가능하다.
* Spring 기반 어플리케이션과의 통합에 적합 하며 Spring Security 와도 통합 가능하다.
* 어플리케이션 구성을 암호화하기 위한 통합 기능을 가지고 있다.

.properties 파일이나 .yml 파일 등에서 민감한 정보가 있고 암호화할 필요성이 있을 때에도 사용할 수 있다.  

이 글은 해당 상황일 때를 기반으로 작성하였다.  

`compile 'com.github.ulisesbocchio:jasypt-spring-boot-starter:2.1.2'`를 build.gradle에 추가해주자.  

```java
@Configuration
public class JasyptConfig {

    @Bean("jasyptStringEncryptor")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword(System.getProperty("jasypt.encryptor.password"));
        config.setAlgorithm("PBEWithMD5AndDES");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.NoIvGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```
그리고 다음 클래스를 추가해준다.  

setPassword는 안에 문자열을 넣어 정할 수도 있지만 중요한 정보이기 때문에 외부 환경변수에 저장해주는 방식을 이용했다.  

이제 암호화를 해보자  
[암호화해주는 싸이트](https://www.devglan.com/online-tools/jasypt-online-encryption-decryption)  
해당 링크에서  
1. 암호화 할 문자를 입력한다.  
2. Two Way Encryption을 선택한다.  
3. 시크릿 키를 입력한다. (해당 키는 위의 코드에서 setPassword에 들어갈 키이다.)
4. Encrypt를 누른다.  

해당 암호화 된 결과 값을 우측 칸에 적고 시크릿 키를 입력하면 복호화가 이루어진다.  
암호화 된 값을  
```java
  datasource:
    url: ENC(SHZ3Wn9WLIBcQXX3hG5/7WHoIvkR+UF0fYYerwztivYoE8ec6vM4wAKLxhjRjSMS)
    username: ENC(DoImz8/fXhvry9F7FEcOaA==)
```
다음과 같이 ENC를 붙이고 ()로 감싸주면 애플리케이션이 알아서 복호화를 해서 사용해주게 된다.  

시크릿 키 입력은 Intellij 기준 상단의 RUN > Edit Configurations 의 VM Option에  
``-Djasypt.encryptor.password=키`` 과 같이 입력하면 된다.  

![image](https://user-images.githubusercontent.com/45073750/122531070-1a313a80-d05a-11eb-8363-73d3f4b2e454.png)

Test코드를 실행할 때에 이것을 추가해야 된다는 번거로운 점이 있는게 단점이다.

***