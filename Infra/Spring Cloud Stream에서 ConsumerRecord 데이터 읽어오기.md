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
위와 같은 방식의 문제는 batch-mode 일 때에는 ``BatchMessagingMessageConverter``를 사용해야 한다.  

```java
    @Bean
    BatchMessageConverter customBatchMessageConverter() {
        return new CustomBatchMessageConverter();
    }

    public static class CustomBatchMessageConverter extends BatchMessagingMessageConverter {

        @Override
        public Message<?> toMessage(List<ConsumerRecord<?, ?>> records, @Nullable Acknowledgment acknowledgment, org.apache.kafka.clients.consumer.Consumer<?, ?> consumer, Type type) {
            Message<?> message = super.toMessage(records, acknowledgment, consumer, type);
            return MessageBuilder.createMessage(records, message.getHeaders());
    }

---
  
    @Bean
    public Consumer<List<ConsumerRecord<String, NameDto>> testConsumer() {
        return messages -> {
            ...
        };
    }
```

~~``toMessage`` 메서드에서 ``NameDto``로 변환해서 넣어주었다. 그리고 Consumerd에서도 ``Message<List<NameDto>>``형태로 받아주었다. ``getHeaders()`` 내부를 살펴보면 list의 size 만큼 헤더의 각 요소가 list로 들어가있는 것을 확인할 수 있었다. 원래 그냥 ``List<ConsumerRecord>``로 받아서 사용을 하려고 했는데 이게 버전마다 조금씩 다른 것 같다. 위의 예제는 spring-cloud ``4.1.3`` 버전을 사용했는데 디버깅 하면서 내부 코드를 살펴보니 convert 하는 과정의 흐름과 ``List<ConsumerRecord>``를 그냥 사용하고 있는 ``3.2.4``버전에서의 흐름이 다른 것을 확인할 수 있었다. 그래서 위와 같은 형태로 변형해서 사용하게 되었다. 근데 이게 아주 100% 확실하게 버전의 이유 때문인지는 잘 모르겠다. 조금 더 살펴봐야 할 것 같다.~~

batch-mode default 설정을 잘못해줘서 생긴 문제였다 ㅜㅜ 정상적으로 작동되는 것을 확인할 수 있었다.

<br/>

---

## REFERENCE

https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-reactive-binder/consuming.html

