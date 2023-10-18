# RedisTemplate, RedisCacheManager 설정에 대해

레디스를 캐시서버로 이용하려고 하는 상황이다.  

```java
@Configuration
@RequiredArgsConstructor
public class RedisConfig {

// spring.data.redis yaml에 정의하고 이걸 토대로 자동으로 생성되는 RedisConnectionFactory 빈을 사용 추천
    private final RedisProperties redisProperties;

    @Bean
    public RedisTemplate<Object, Object> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());

        return redisTemplate;
    }

  // 수정) 아래 부분은 yaml 파일의 spring.data.redis 내부에 집어넣고 자동으로 생성되는 RedisConnectionFactory 빈을 주입받아서 사용하는 것이 더 편하다.
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(redisProperties.getHost(), redisProperties.getPort());
//        config.setUsername("bepoz");
//        config.setPassword("pwd");
        return new LettuceConnectionFactory(config);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class CacheService {

    private final RedisTemplate<Object, Object> redisTemplate;

    public void setCacheString() {
        redisTemplate.opsForValue()
                     .set("String", "StringValue");
    }

    public void setCacheInteger() {
        redisTemplate.opsForValue()
                .set("Integer", 1);
    }

    public void setCacheObject() {
        redisTemplate.opsForValue()
                .set("Object", new ObjectDto("bepoz", 100));
    }

    public Object getCache(String key) {
        return redisTemplate.opsForValue()
                                .get(key);
    }

    public void deleteAll() {
        redisTemplate.opsForValue().getAndDelete("String");
        redisTemplate.opsForValue().getAndDelete("Integer");
        redisTemplate.opsForValue().getAndDelete("Object");
    }

    static class ObjectDto {

        private final  String name;
        private final int age;
      
        public ObjectDto() {}
      
        public ObjectDto(String name, int age) {
            this.name = name;
            this.age = age;
        }
      
        public String getName() {
          return name;
        }
      
        public int getAge() {
          return age;
        }
    }
}
```

```java
@RestController
@RequestMapping
@RequiredArgsConstructor
public class CacheController {

    private final CacheService cacheService;

    @GetMapping
    public Object getCache(@RequestParam String key) {
        return cacheService.getCache(key);
    }

    @PostMapping
    public void setCache() {
        cacheService.setCacheString();
        cacheService.setCacheInteger();
        cacheService.setCacheObject();
    }

    @DeleteMapping
    public void deleteCache() {
        cacheService.deleteAll();
    }
}
```

초기 코드는 다음과 같다. ``RedisConnectionFactory`` 가 먼저 생성이 되어야 하는데, jedis 와 lettuce 중 lettuce 커넥션 팩토리를 사용하였다. 사실 그냥 ``new LettuceConnectionFactory({host}, {port})`` 로 바로 return 해줘도 되는데, 어차피 내부적으로 ``RedisStandaloneConfiguration`` 을 호출하고 있어서 풀어서 썼다.  

username/password 를 설정할 수도 있지만 현재 사용하고 있지 않으므로 주석처리했다. 레디스 클러스터를 사용하는 경우에는 ``new RedisClusterConfiguration()`` 를 이용하여야 한다.  

```yaml
spring:
  data:
    redis:
      cluster:
        max-redirects: 5
        nodes: ***
      password: ***
      lettuce:
        pool:
          max-idle: 8
          min-idle: 1
          max-active: 8
          max-wait: 5s
          time-between-eviction-runs: 10m
```

대충 위와 같이 yaml 파일에 정의를 해두고 따로 properties를 사용하지 않고 기본적으로 factory 빈이 생성되게끔 하고 이것을 주입받아서 사용하는 것이 더 편하다.  
RedisConnectionFactory 내부의 RedisClusterConfiguration에 해당 값들이 들어가게 된다.

```java
@Bean
public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory);

    return redisTemplate;
}
```



아무튼 생성한 팩토리로 ``RedisTemplate`` 에 set 해주고 이를 서비스에서 이용해주었다. ``opsForValue()`` 는 레디스가 지원하는 여러 컬렉션 타입 중에 key-value 형태를 사용하기 위해 사용하였다.  

이대로 api를 호출해서 캐시를 저장하고 이를 cli로 확인해보면 오류가 마주하게된다. Object를 serializing 할 수 없다는 것이다. Object를 set 하는건 잠시 주석처리하고 이외의 값들을 넣어보겠다.  

<img width="504" alt="image" src="https://user-images.githubusercontent.com/45073750/173631778-1fb09844-dd01-4e1f-9ed5-ef635e3b5c08.png">

이런 출력을 보지 않기위해 ``RestTemplate`` 을 선언하는 과정에서 serializer를 설정해주어야 한다.(기본 serializer는 ``JdkSerializationRedisSerializer`` 를 사용한다)  

```java
    @Bean
    public RedisTemplate<Object, Object> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(StringRedisSerializer.UTF_8);
				
        return redisTemplate;
    }

//new StringRedisSerializer() == StringRedisSerializer.UTF_8 == RedisSerializer.string()
//편한 것으로 사용하자.
```

``StringRedisSerializer`` 를 적용해주었고 set api를 보내면 이번에는 Integer를 String 으로 변환 불가 어쩌고 하는 예외가 나온다. 그래서 그냥 key에만 적용하고 돌려보았다.  

<img width="943" alt="image" src="https://user-images.githubusercontent.com/45073750/173632275-5f8de813-a40e-4d77-946e-4600440bf869.png">

key 값이 정상적으로 string 으로만 보이게 되었다. 하지만 여전히 value는 이상하다.  
이제 다른 serializer를 이용해보겠다.  

```java
    @Bean
    public RedisTemplate<Object, Object> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        return redisTemplate;
    }
//new GenericJackson2JsonRedisSerializer() == RedisSerializer.json()
```

결과는 다음과 같았다.  

<img width="848" alt="image" src="https://user-images.githubusercontent.com/45073750/173634563-35c810fa-ccbd-4ce0-9379-ba97f94d3923.png">

json 형식으로 변경을 해준다. 그리고 Object의 경우 class의 path 또한 나오게 된다.  
이 방식은 이후에 get 해올 때에 동일한 class path 여야만 가져올 수 있다는 단점이 있다.  
여러 어플리케이션 서버에서 해당 레디스에 접근해서 사용하는 것이 아니고 한 곳에서만 넣었다 뺐다하는 식이면 상관은 없을 것 같다.  

이번에는 ``Jackson2JsonRedisSerializer`` 를 사용해보겠다.  

```java
    @Bean
    public RedisTemplate<Object, Object> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(ObjectDto.class));

        return redisTemplate;
    }
```

<img width="723" alt="image" src="https://user-images.githubusercontent.com/45073750/173641030-ad340d1e-1b6d-439b-8660-c43fe19e8715.png">

이번에는 class path 없이 들어간 것을 확인할 수가 있었다.  

![image](https://user-images.githubusercontent.com/45073750/173656276-c400446c-ce9a-43cb-a09d-eab50cb1e8a1.png)

캐시를 get 해올 때에 지정한 클래스 타입과 다르면 binding 하는 과정에서 예외를 발생시킨다.(다른 Dto 클래스를 하나 생성하고 해당 클래스로 설정해서 get하는 방식으로 확인을 해봄) 그리고 전체 template에 대한 serializer를 하나의 target에 맞춘 serializer로 설정한다는 것이 옳다고 생각들지는 않는다. 물론 해당 레디스 서버에서 하나의 object type에 대해서만 사용한다고 하면 상관없을 것 같다. 타입을 Object.class로 두면 binding도 잘되고... 하나의 클래스에 맞춘 것이 아니니깐 괜찮을 것 같기도 하지만 이 부분에 대해서는 확답을 하지는 못할 것 같다.  

<br/>

이제 RedisCacheManager에 대해 살펴보겠다. 레디스를 캐시 서버로 사용할 때에 이용하는 클래스다. 잔말말고 코드로 살펴보자.  

```java
//RedisConfig
		@Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfig = RedisCacheConfiguration.defaultCacheConfig()
                                                                                 .disableCachingNullValues()
                                                                                 .entryTtl(Duration.ZERO)
                                                                                 .serializeKeysWith(SerializationPair.fromSerializer(RedisSerializer.string()))
                                                                                 .serializeValuesWith(SerializationPair.fromSerializer(RedisSerializer.json()));
        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory)
                                                         .cacheDefaults(cacheConfig)
                                                         .disableCreateOnMissingCache()
                                                         .withInitialCacheConfigurations(getRedisConfigMap())
                                                         .build();
    }

    private Map<String, RedisCacheConfiguration> getRedisConfigMap() {
        final SerializationPair<String> keySerializationPair = SerializationPair.fromSerializer(StringRedisSerializer.UTF_8);
        final SerializationPair<Object> valueSerializationPair = SerializationPair.fromSerializer(RedisSerializer.json());

        return Map.of("BEPOZ", RedisCacheConfiguration.defaultCacheConfig()
                                                               .disableCachingNullValues()
                                                               .computePrefixWith(cacheName -> "COOL_PREFIX::" + cacheName + "::")
                                                               .serializeKeysWith(keySerializationPair)
                                                               .serializeValuesWith(valueSerializationPair)
        );
    }
```

```java
//CacheService
    public void setCacheUsingCacheManager() {
        Cache cache = Optional.ofNullable(redisCacheManager.getCache("BEPOZ"))
                              .orElseThrow(IllegalArgumentException::new);
        cache.put("manager", new ObjectDto("kang", 100));
    }

    public Object getCacheUsingCacheManager() {
        Cache cache = Optional.ofNullable(redisCacheManager.getCache("BEPOZ"))
                              .orElseThrow(IllegalArgumentException::new);
        return cache.get("manager").get();
    }
```

``CacheManager`` 를 사용하게되면 ``Cache`` 라는 클래스를 이용하게된다. 이것을 통해서 값을 넣고, 가져오고, 삭제하고 할 수 있다.  
``redisCacheManager`` 메서드부터 살펴보자. ``RedisCacheManager``에서 사용할 defaultConfig를 내가 따로 지정해 준 것이다.  

지정해주지 않아도 내부 구현을 살펴보면 default가 존재한다는 것을 확인할 수가 있다.

![image](https://user-images.githubusercontent.com/45073750/173781760-5afb5739-2725-464a-b510-359425ddfe22.png)

그렇지만 따로 별도로 캐시서버의 default한 config를 설정해줄 때에 ``.cacheDefaults()`` 를 이용해 설정해줄 수가 있다.  
이전에 ``RedisTemplate`` 을 설정해주었듯이 설정해주었다. ttl의 ``Duration.ZERO`` 는 바로 삭제한다는 것은 아니고 삭제시간을 두지 않는다는 설정이다. 위 이미지에서도 볼 수 있듯이 기존의 default 설정 중 하나이다. ``.disableCachingNullValue()`` 는 value로 null을 허용안하겠다는 뜻이고, serializer 설정은 ``RedisTemplate`` 때와 동일하다.  

``.disableCreateOnMissingCache()`` 는 캐시를 찾지못하면 바로 생성해주지 않겠다는 옵션인데 추후에 다시 말하겠다.  
``withInitialCacheConfigurations`` 는 특정 캐시와 해당 캐시의 configuration 을 초기화 해두는 옵션이다.  

``BEPOZ`` 라는 캐시의 이름으로 해당 캐시의 config를 설정해주고 이를 initiate 한 것이다. prefix 부분을 기억해두자. 이제 값을 위의 ``CacheService`` 코드 그대로 삽입해보겠다.  

![image](https://user-images.githubusercontent.com/45073750/173783031-1310c9bf-e2d4-418a-8464-1808c21ee453.png)

결과는 다음과 같이 나오게된다. prefix가 붙어서 cacheName과 함께 key로 저장된 것을 확인할 수가 있다.  
코드에서도 먼저 ``redisCacheManager.getCache()`` 를 이용하여 캐시이름을 찾고 그곳에서 내가 저장할 때 넣었던 key 값인 manager를 이용해서 value를 갖고오게된다. redis-cli를 통해 확인해보면 사실 prefix 까지 붙은 값을 모두 합친 것이 key 이긴하다.  
아마 ``CacheManager.getCache()``  를 하는 과정에서 prefix 이후의 값을 key로 보고 값을 가져오는 것 같다. (여기에서는 ``"COOL_PREFIX::" + cacheName + "::"`` 이후의 값인 manager)  

존재하지 않은 캐시를 get 해오려하면 null을 return하게 된다. 정확히는 ``disableCreateOnMissingCache`` 설정을 해주었기 때문이다. 만약 이것을 설정하지 않는다면 즉석으로 캐시를 만들고 이를 return 해준다.  

![image](https://user-images.githubusercontent.com/45073750/173784112-b3149cae-8d17-4ad3-92e7-d2f3cb574a7c.png)
![image](https://user-images.githubusercontent.com/45073750/173784147-c547ed67-391d-4c31-b625-7da3b5bc402f.png)

``disableCreateOnMissingCache`` 는 한 마디로 위 코드 내용에서 ``allowInFlightCacheCreation`` 값을 false로 해주는 옵션인 것이다.  캐시를 자동으로 만들 때에 configuration은 ``RedisCacheManager`` 를 등록할 때 만든 default한 configuration 을 따라가게된다. 따라서 웬만하면 ``withInitialCacheConfigurations`` 를 이용해 캐시와 캐시설정을 미리 등록해두는 것이 바람직 할 것이다.  

```java
    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfig = RedisCacheConfiguration.defaultCacheConfig()
                                                                     .disableCachingNullValues()
                                                                     .entryTtl(Duration.ZERO)
                                                                     .computePrefixWith(cacheName -> "DEFAULT_PREFIX::" + cacheName + "::")
                                                                     .serializeKeysWith(SerializationPair.fromSerializer(RedisSerializer.string()))
                                                                     .serializeValuesWith(SerializationPair.fromSerializer(RedisSerializer.json()));
        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory)
                                                         .cacheDefaults(cacheConfig)
//                                                         .disableCreateOnMissingCache()
//                                                         .withInitialCacheConfigurations(getRedisConfigMap())
                                                         .build();
    }
```

![image](https://user-images.githubusercontent.com/45073750/173785765-29d0b194-be6d-4e14-a158-051565ab97c6.png)

default configuration의 prefix를 설정하고 캐시를 set 한 모습이다. ``disableCreateOnMissingCache`` 옵션을 주석처리 하였기에 캐시를 즉석으로 바로 만들면서 default configuration을 이용하게되었고, prefix도 설정한 대로 나온 것을 확인할 수가 있다.  

``RedisCacheManager``를 사용하는 상태에서 ``RedisTemplate`` 을 통한 호출도 자유롭다(물론 ``RedisCacheManager`` 의 configuration을 따라가지는 않는다). 그리고 ``RedisTemplate`` 선언 없이 ``RedisCacheManager`` 만 선언해서 사용할 수도 있다.  

``RedisCacheManager`` 를 사용하게되면 캐시 종류에 따른 configuration 분리도 할 수 있으며, prefix를 통한 key 중복 또한 방지할 수 있다는 이점이 있다. 이것을 이용하면 ``Jackson2JsonRedisSerializer`` 을 사용할 때에 하나의 클래스 타입에 지정되는 문제 또한 해결할 수 있을 것 같다(하나의 캐시네임에 한 종류의 클래스 타입만 들어간다는 가정하에).  

***

### REFERENCE

https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/#redis:template  

https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/#redis:support:cache-abstraction

