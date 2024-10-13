# Spring Cloud Stream에서 ConsumerRecord 데이터 읽어오기

```java
    @Bean
    public Consumer<ConsumerRecord<String, NameDto>> testConsumer() {
        return message -> {
            System.out.println("message = " + message);
        };
    }

    @Bean
    RecordMessageConverter customMessageConverter() {
        return new MessagingMessageConverter() {
            @Override
            public Message<?> toMessage(ConsumerRecord<?, ?> record, Acknowledgment acknowledgment,
                                        org.apache.kafka.clients.consumer.Consumer<?, ?> consumer, Type payloadType) {
                return MessageBuilder.withPayload(record).build();
            }
        };
    }

---
  
spring.cloud.stream.kafka.bindings.testConsumer-in-0.consumer.converter-bean-name: customMessageConverter
```

위와 같이 설정하여 spring cloud stream을 이용한 consumer에서 ``ConsumerRecord``를 사용할 수 있게끔 하였다.  
위와 같은 방식의 문제는 batch-mode 일 때, ``List<ConsumerRecord>``나 ``ConsumerRecords<>`` 인 경우 받지를 못한다는 것이다.  

관련해서 계속 찾아봤는데 내용이 딱히 없다. ``ConsumerRecord``를 사용해야 하는데 batch가 필요할 때에는 그냥 ``@KafkaListener``를 이용해서 처리하는 방식으로 진행하는 방향으로 해야할 것 같다.  

<br/>

---

## REFERENCE

https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-reactive-binder/consuming.html

