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

```yaml
spring:
  cloud:
    stream:
      bindings:
        consumer:
          enable-dlq: true
          dlq-name: {dlqTopicName}
```

spring cloud stream을 사용하는 경우 dlq 사용 및 토픽명 등록을 위의 yaml에 작성하는 것으로 할 수 있다.  

