# Spring Kafka 재시도 처리

## @KafkaListener를 사용하는 경우

```java
@Bean
public CommonErrorHandler customErrorHandler() {
    final FixedBackOff fixedBackOff = new FixedBackOff(1000, 2);
    return new DefaultErrorHandler((rec, ex) -> {
        log.error("Error message is " + rec.value() + ' ' + ex.getMessage());
    }, fixedBackOff);
}
```

```java
@Bean
public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> customContainerFactory() {
    final Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

    final DefaultKafkaConsumerFactory<String, String> cf = new DefaultKafkaConsumerFactory<>(props);
    final ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(cf);
    factory.setCommonErrorHandler(customErrorHandler());
    return factory;
}
```

``CommonErrorHandler``를 정의해서 ``KafkaConsumerFactory``에 set 해주면 된다.  

``CommonErrorHandler`` 생성자에는 2가지 타입의 파라미터가 있다.

1. ``ConsumerRecordRecoverer``
2. ``BackOff``

첫 번째 타입은 ``ConsumerRecord``와 ``Exception``을 가진 BiConsumer 타입의 함수형 인터페이스다. 위의 코드에서는 해당 값을 사용해서 로깅을 했다. 그리고 ``BackOff``를 구현하고 있는 ``FixedBackOff`` 클래스를 정의했는데 1000ms 마다 재시도를 하고 maxAttempt 값을 2로 두었다. 첫 시도 후 실패한 뒤에 한 번 더 실행을 하여 총 2번의 시도를 하게 될 것이다.  

``ConsumerRecordRecoverer`` 같은 경우에는 maxAttempt를 모두 채운 다음에 동작을 한다. 위의 코드의 경우 error 로그를 결국 한 번만 찍게 되는 것이다.  

```java
@Bean
public CommonErrorHandler customErrorHandler() {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(customKafkaTemplate(), (rec, ex) -> new TopicPartition("dlq topic name", rec.partition()));
    final FixedBackOff fixedBackOff = new FixedBackOff(1000, 3);
    return new DefaultErrorHandler(recoverer, fixedBackOff);
}
```

에러가 난 record에 대해서 DLQ에 넣는 경우에는 위와 같이 ``DeadLetterPublishingRecoverer``을 이용해서 정의하면 된다. 생성자의 2번째 파라미터의 경우 ``BiFunction<ConsumerRecord<?, ?>, Exception, TopicPartition>`` 타입을 가지고 있는데, 문제가 발생한 record의 partition 그대로 넣어주고 싶어서 rec.partition 으로 코드를 작성하였다.  

<br/>

## Spring Cloud Stream을 사용하는 경우

```java
@Bean
public ListenerContainerCustomizer<AbstractMessageListenerContainer<byte[], byte[]>> customizer() {
    return (container, dest, group) -> {
        container.setCommonErrorHandler(customErrorHandler());
    };
}

@Bean
public CommonErrorHandler customErrorHandler() {
    return new DefaultErrorHandler((rec, ex) -> {
        log.error("Error!!!");
    }, new FixedBackOff(1000, 2));
}
```

Spring cloud stream을 사용하는 경우 기본적으로 위와 같이 ``ListenerContainerCustomizer``를 이용하여 ``CommonErrorHandler``를 등록해줄 수가 있다. [Customizer docs](https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/spring-cloud-stream.html#_advanced_consumer_configuration)  
굳이 customizer를 사용하고 싶지 않다면, ``spring.cloud.stream.kafka.bindings.consumer.{channelName}.consumer.common-error-handler-bean-name``에 설정해주면 된다. [Kafka Consumer Properties docs](https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-binder/config-options.html#kafka-consumer-properties)

dlq에 넣고 싶다면 위에서 한 번 다뤘던 ``DeadLetterPublishingRecoverer``을 이용한 Error handler를 정의하면 된다.  

```yaml
spring:
  cloud:
    stream:
      kafka:
        bindings:
          consumer:
            enable-dlq: true
            dlq-name: {dlqTopicName}
```

단순히 dlq에만 전송하고 싶고 크게 커스터마이징 할 일이 없다면 단순히 위와 같이 yaml 설정을 통해 dlq 사용 및 토픽명 등록을 할 수도 있다.  

``@StreamRetryTemplate``을 이용하는 방법도 있는데 이것은 생략하겠다.  

---

## REFERENCE

https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka_tips.html#simple-dlq-with-kafka

https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/spring-cloud-stream.html#_advanced_consumer_configuration

https://docs.spring.io/spring-cloud-stream/reference/spring-cloud-stream/overview-error-handling.html

