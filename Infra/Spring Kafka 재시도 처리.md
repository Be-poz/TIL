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

첫 번째 타입은 ConsumerRecord와 Exception을 가진 BiConsumer 타입의 함수형 인터페이스다. 위의 코드에서는 해당 값을 사용해서 로깅을 했다. 그리고 ``BackOff``를 구현하고 있는 ``FixedBackOff`` 클래스를 정의했는데 1000ms 마다 재시도를 하고 maxAttempt 값을 2로 두었다.  

