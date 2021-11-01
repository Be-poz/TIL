# @Async 를 이용한 비동기 처리에 대해

``@Async`` 는 비동기적으로 처리를 할 수 있게끔 스프링에서 제공하는 어노테이션이다.  
해당 어노테이션을 붙이게 되면 각기 다른 쓰레드로 실행이 된다. 즉, 호출자는 해당 메서드가 완료되는 것을 기다릴 필요가 없다.  

이 어노테이션을 사용하기 위해서는 ``@EnableAsync`` 가 달려있는 configuration 클래스가 우선적으로 필요하다.  

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

> By default, both Spring's @Async annotation and the EJB 3.1 @javax.ejb.Asynchronous annotation will be detected.  -@EnableAsync 어노테이션 내부 설명- 

``@EnableAsync`` 는 스프링의 ``@Async`` 어노테이션과 EJB 3.1 javax.ejb.Asynchronous 를 감지한다고 한다.  

``@Async`` 어노테이션을 사용하기 위해서는 2가지 제약조건이 있다.  

1. public 메서드일 것
2. 동일 클래스에서 호출하는 Self-invocation 이어서는 안된다는 것

프록시를 사용하기 위해서 메서드는 public이어야 하고, Self-invocation를 사용하게되면 프록시를 무시하고 바로 메서드를 호출하기 때문이다.  
[관련링크](https://dzone.com/articles/effective-advice-on-spring-async-part-1)  

기본적으로, 스프링은 비동기적으로 메서드를 실행하기 위해서 ``SimpleAsyncTaskExecutor``를 사용한다.  
(``SimpleAsyncTaskExecutor``는 요청이 오는대로 계속해서 쓰레드를 생성한다.)  
이것을 어플리케이션 레벨 또는 각 메서드 레벨에서 override 함으로써 default를 변경할 수 있다.  

1. 메서드 레벨

   ```java
   @Configuration
   @EnableAsync
   public class AsyncConfig {
   
       @Bean("customAsyncExecutor")
       public Executor customAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(5);
           executor.setMaxPoolSize(5);
           executor.setThreadNamePrefix("bepoz");
           executor.initialize(); // 꼭 써줘야 한다.
           return executor;
       }
   }
   ```

   ```java
   @Service
   @Slf4j
   public class AsyncService {
   
       @Async("customAsyncExecutor")
       public void call() {
   				log.info("async Test");
       }
   }
   ```

   ``@Async`` value로 등록한 Executor의 이름을 적는 것으로 사용  

2. 어플리케이션 레벨

   ```java
   @Configuration
   @EnableAsync
   public class AsyncConfig implements AsyncConfigurer {
   
       @Override
       public Executor getAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(5);
           executor.setMaxPoolSize(5);
           executor.setThreadNamePrefix("bepoz");
           executor.initialize(); // 꼭 써줘야 한다.
           return executor;
       }
   }
   ```

위의 방법은 AsyncConfigurer 인터페이스를 구현하여 ``getAsyncExecutor()`` 를 오버라이딩 함으로써 default Executor가 내가 설정해둔 Executor가 된다. 애플리케이션에서 ``@Async`` 를 사용했을 때 해당 Executor를 사용하게 된다.  

리턴 타입이 Future이 아닌 Void인 경우 예외가 발생해도 메서드 호출 쓰레드까지 전파가 되지 않아 ``AsyncUncaughtExceptionHandler``를 구현한 클래스를 생성하고 ``AsyncConfigurer`` 인터페이스의 ``getAsyncUncaughtExceptionHandler`` 메서드를 오버라이딩 해주어야 한다. 자세한 것은 생략하겠다.  

위 코드에 나온 [ThreadPoolTaskExecutor vs ThreadPoolExecutor 차이 간단 정리](https://github.com/Be-poz/TIL/blob/master/Spring/ThreadPoolTaskExecutor%20vs%20ThreadPoolExecutor%20%EC%97%90%20%EB%8C%80%ED%95%B4.md)  

어플리케이션 레벨에서의 구현 코드를 보면 ``AsyncConfigurer`` 인터페이스를 구현해서 사용을 하고 있는데,  
``AsyncConfigurerSupport`` 를 상속받아서 구현할 수도 있다.  

![image](https://user-images.githubusercontent.com/45073750/139533674-6210b02b-586c-4fd8-9963-7807f48af514.png)

그러나 클래스 내부를 살펴보면 상위 인터페이스로 ``AsyncConfigurer`` 가 있다는 것을 확인할 수가 있다.  
그리고, 자바 8 버전부터 default 메서드가 생기고 적용되었기 때문에 인터페이스인  ``AsyncConfigurer`` 를 사용해서 진행할 수 있다.  
개인적으로 class 상속보다는 interface 구현이 더 낫다고 생각하기 때문에 ``AsyncConfigurer``를 이용해서 진행하는 것이 더 좋다고 생각한다.  

<br/>

이제 직접 사용하면서 해보겠다.  

1. 아무 것도 없이 ``@EnableAsync`` 만 사용하는 경우

   ```java
   @Configuration	
   @EnableAsync
   public class AsyncConfig {
   }
   ```

   ```java
   @Service
   @Slf4j
   public class AsyncService {
       @Async
       public void call() {
           log.info("async Test");
       }
   }
   ```

   ```java
       @Test
       public void test() throws InterruptedException {
           for (int i = 0; i < 10000; i++) {
               asyncService.call();
           }
           Thread.sleep(3);
       }
   ```

   ![image](https://user-images.githubusercontent.com/45073750/139541055-a41b9b57-1d1a-4e59-9ab7-f22b9dc14224.png)  

   Thread 명이 아니라 task로 나온다... 위에서 설명한대로라면 ``SimpleAsyncTaskExecutor`` 로 돌아갔을 것이다.  
   하지만, task로 찍히니 뭔가 못미덥다. 이 부분은 뒤에서 더 살펴보겠다.  

2. AsyncConfigurer를 구현하는 경우

   ```java
   @Configuration
   @EnableAsync
   public class AsyncConfig implements AsyncConfigurer {
       @Override
       public Executor getAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(5);
           executor.setMaxPoolSize(5);
           executor.setThreadNamePrefix("5bepoz");
           executor.initialize();
           return executor;
       }
   }
   ```

   서비스 코드와 테스트 코드는 동일  
   ![image](https://user-images.githubusercontent.com/45073750/139626668-116804f0-6694-41b7-a60a-e5110b0490d0.png)

   커스텀하게 지정해준 executor로 돌아가는 것을 확인할 수가 있다.  

   

3. AsyncConfigurer 구현없이 Bean 등록을 해준 경우

   1. 1개의 Bean만 등록해준 경우

      ```java
      @Configuration
      @EnableAsync
      public class AsyncConfig {
      
          @Bean
          public Executor customExecutor() {
              ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
              executor.setCorePoolSize(5);
              executor.setMaxPoolSize(5);
              executor.setThreadNamePrefix("5bepoz");
              executor.initialize();
              return executor;
          }
      }
      ```

      서비스 코드와 테스트 코드는 동일하다.  
      ![image](https://user-images.githubusercontent.com/45073750/139626978-f061a5a3-4f0f-4500-a3a1-f3e286ea4dab.png)

      이 경우에도 내가 따로 지정해준 executor가 돌아가는 것을 확인할 수가 있었다.  
      기본적으로 default가 ``SimpleAsyncTaskExecutor``로 돌아가지만, Executor 타입의 Bean이 유니크하게(1개만) 등록되어있으면 해당 Executor로 실행하는 것 같다.(뒤쪽에서 이에 대해 더 추측해본다)  

      

   2. 여러 개의 Bean을 등록한 경우

      ```java
      @Configuration
      @EnableAsync
      public class AsyncConfig {
      
          @Bean
          public Executor customExecutor() {
              ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
              executor.setCorePoolSize(5);
              executor.setMaxPoolSize(5);
              executor.setThreadNamePrefix("5bepoz");
              executor.initialize();
              return executor;
          }
      
          @Bean
          public Executor customExecutor2() {
              ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
              executor.setCorePoolSize(10);
              executor.setMaxPoolSize(10);
              executor.setThreadNamePrefix("10bepoz");
              executor.initialize();
              return executor;
          }
      }
      ```

      서비스 코드와 테스트 코드는 동일하다.  

      ![image](https://user-images.githubusercontent.com/45073750/139627179-d7638e08-ea0a-486a-a1e8-81ab8c1678b1.png)

      이번에는 ``skExecutor`` 라는 이름의 쓰레드로 돌아간 것을 확인할 수가 있었다.  
      ``skExecutor``는  ``SimpleAsyncTaskExecutor`` 의 약어이다. 3-1 케이스와 달리 이 케이스에는 Executor 타입으로 여러 Bean이 등록되어있고 서비스코드의 ``@Async`` 에서 따로 지정을 안해주었기 때문에 default executor인 ``SimpleAsyncTaskExecutor``로 돌아간 것으로 보인다. 
      아까전에는 ``skExecutor`` 로 표기되지않고 그냥 ``task-n`` 으로 표기되었다. 그렇다면 왜 이번에는 그렇지 않았을까?  
      아마 그 이유는 이 경우에는 여러  Executor 들이 존재하기 때문에 명시를 분명히 해야하기 때문이라고 추측한다.  

      본론으로 돌아와서, 그렇다면 ``@Async`` 어노테이션에 Bean 이름을 지정을 해주면 어떻게 될까?  

      ```java
      @Service
      @Slf4j
      public class AsyncService {
      
          @Async("customExecutor")
          public void call() {
              log.info("async Test " + Thread.currentThread());
          }
      }
      ```

      서비스 코드의 ``@Async``어노테이션에 첫 번째 Bean 이름을 명시해주었다. 그러자 다음과 같은 결과를 확인할 수가 있었다.  
      ![image](https://user-images.githubusercontent.com/45073750/139627845-867e9b93-113c-42b0-8c57-b219c96e789a.png)

      Bean 이름을 따로 명시를 하지 않으면 필드이름이나 메서드명으로 등록되기 때문에 cucstomExecutor 라는 이름으로 Bean을 찾을 수 있었다. 확실히 명시해주기 위해서는 다음과 같이 Bean 이름을 정해주는 것도 좋다고 생각한다.  

      ```java
      @Configuration
      @EnableAsync
      public class AsyncConfig {
      
          @Bean("customExecutor")
          public Executor customExecutor() {
              ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
              executor.setCorePoolSize(5);
              executor.setMaxPoolSize(5);
              executor.setThreadNamePrefix("5bepoz");
              executor.initialize();
              return executor;
          }
      
          @Bean("customExecutor2")
          public Executor customExecutor2() {
              ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
              executor.setCorePoolSize(10);
              executor.setMaxPoolSize(10);
              executor.setThreadNamePrefix("10bepoz");
              executor.initialize();
              return executor;
          }
      }
      ```

      

4. AsyncConfigurer를 구현하면서 다른 Executor를 Bean으로 선언한 경우

   ```java
   @Configuration
   @EnableAsync
   public class AsyncConfig implements AsyncConfigurer {
   
       @Override
       public Executor getAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(5);
           executor.setMaxPoolSize(5);
           executor.setThreadNamePrefix("5bepoz");
           executor.initialize();
           return executor;
       }
   
       @Bean("customExecutor")
       public Executor customExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(20);
           executor.setMaxPoolSize(20);
           executor.setThreadNamePrefix("custom");
           executor.initialize();
           return executor;
       }
   }
   ```

   ```java
   @Service
   @Slf4j
   public class AsyncService {
   
       @Async
       public void call() {
           log.info("async Test " + Thread.currentThread());
       }
   }
   ```

   ![image](https://user-images.githubusercontent.com/45073750/139628183-0341bf2e-372e-42d0-8ee7-a33cc2d26a9b.png)

   ``AsyncConfigurer``를 구현하여 default Executor를 바꿔준 후, 또 다른 Executor를 등록해준 코드다.  
   ``@Async`` 어노테이션에 별 다른 Bean 이름을 지정해주지 않고 돌렸더니 default로 돌아가서 5bepoz{n} 이 찍힌 것을 확인할 수가 있다. 이 상황에서 다른 Executor 호출도 가능한지 확인해보자. 코드를 다음과 같이 변경해주었다.  

   ```java
   @Service
   @Slf4j
   public class AsyncService {
   
       @Async("customExecutor")
       public void call() {
           log.info("async Test " + Thread.currentThread());
       }
   }
   ```

   ![image](https://user-images.githubusercontent.com/45073750/139628473-71fff219-3965-49e7-a7b4-f7691034cc23.png)

   예상한대로 돌아가는 것이 확인되었다.  

번외) Executor 선언을 Executors가 지원하는 정적 팩토리 메서드로 선언하는 경우  

![image](https://user-images.githubusercontent.com/45073750/139628853-06011a99-b8ac-444b-81f9-2766cb6cda88.png)

``Executor``를 자바에서 제공하는 ``Executors.newFixedThreadPool(n)``로 받은 다음 형변환을 시켜서 사용되는지 확인해보겠다.  
내부적으로 ``ThreadPoolExecutor``를 리턴해주니깐 되지 않을까? ``ThreadPoolTaskExecutor`` 내부에서도 ``ThreadPoolExecutor``로 돌아가니깐!

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        return (ThreadPoolExecutor) Executors.newFixedThreadPool(5);
    }
}
```

```java
@Service
@Slf4j
public class AsyncService {

    @Async
    public void call() {
        log.info("async Test " + Thread.currentThread());
    }
}
```

![image](https://user-images.githubusercontent.com/45073750/139629538-a0df6138-53f9-4523-ac5e-8904423ad239.png)

되는 걸로 보인다. 그렇다면 다른 방식에서는 어떨까?  

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor customExecutor() {
        return (ThreadPoolExecutor) Executors.newFixedThreadPool(5);
    }
}
```

앞서 실험해본 결과 위와 같은 코드에서도 한 개의 ``Executor`` Bean이 등록되어있으면 해당 Executor를 default Executor로 사용하는 것을 확인할 수가 있었다. 그러나 위 코드에서는 그렇지 않았다.  
![image](https://user-images.githubusercontent.com/45073750/139629780-9c458151-ffd4-47a4-b18d-b1279e594e29.png)

희안하다. 제대로 못읽는 것 같다. ``@Async`` 어노테이션에 Bean 이름을 따로명시하면 또 인식해서 돌아간다.  

```java
@Service
@Slf4j
public class AsyncService {

    @Async("customExecutor")
    public void call() {
        log.info("async Test " + Thread.currentThread());
    }
}
```

![image](https://user-images.githubusercontent.com/45073750/139629927-29f12169-896f-42aa-a7df-53c7ad2e6abc.png)

개인적으로 추측해보자면, ``ThreadPoolTaskExecutor`` 와 ``@Async`` 는 스프링 프레임워크에서 지원을 해주기 때문에 ``AsyncConfigurer`` 를 구현하지 않은 상태로 Executor 타입의 단일 Bean을 ``ThreadPoolTaskExecutor`` 로 선언을 해도 알아서 그것을 default executor 로 읽는 것 같다. ``ThreadPoolTaskExecutor`` 가 사용하기도 더 편하므로 이 클래스를 사용하는 것이 여러모로 더 낫다고 주관적인 의견을 제시해본다.  

 <br/>

이렇게 ``@Async`` 어노테이션에 대해서 살펴보았다.  
사용법에 대한 정리를 하자면,  

1. 모든 코드에서 적용되는  default Executor를 변경하고 싶다면 ``@AsyncConfigurer`` 를 구현하여 변경해주자.(물론 위 글에서 나온 것 처럼 ``ThreadPoolTaskExecutor``로 단일 Bean을 만들어 변경해줄 수도 있다. 하지만 목적에 맞지 않는 구현방법이라고 생각한다.)
2. ``@Async`` 어노테이션에 별도의  Bean 이름을 지정해주지 않으면 default executor 로 돌아간다.
3. 일부 상황에서 다른 Executor로 돌리고 싶다면 커스텀한 Executor를 Bean으로 정의해주자.
4. ``@Async("{beanName}")`` 을 이용해서 내가 사용하고 싶은 Executor를 명시해주자.

---

### REFERENCE

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableAsync.html  

https://www.baeldung.com/spring-async#the-async-annotation  

https://spring.io/guides/gs/async-method/

https://kwonnam.pe.kr/wiki/springframework/async  

https://dzone.com/articles/effective-advice-on-spring-async-part-1

