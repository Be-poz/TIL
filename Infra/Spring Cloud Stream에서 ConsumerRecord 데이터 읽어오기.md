# Spring Cloud Stream에서 ConsumerRecord 데이터 읽어오기

spring-cloud-function-context 4.1.3 버전 기준으로 spring-cloud-stream을 사용할 때에 consume 하는 타입으로 ``ConsumerRecord``를 설정하면 에러가 발생한다.  Converter를 등록해주어야 한다. 아래의 코드는 ``RecordMessageConverter``를 구현하고 있는 ``MessagingMessageConverter`` 클래스를 만들어 빈 등록 해준 코드다. 

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
// yaml 파일
spring.cloud.stream.kafka.bindings.{binding-name}.consumer.converter-bean-name: customMesageConverter
```

이렇게 하면 ``ConsumerRecord``를 받아서 사용할 수 있다.  
<br/>

batch-mode가 true일 때에는 조금 달라진다.  
일단 ``BatchMessageConverter``를 구현하는 클래스인 ``BatchMessagingMessageConverter`` 클래스를 만들어주어야 한다.  

```java
    @Bean
    public Consumer<List<ConsumerRecord<String, NameDto>>> testConsumer() {
        return messages -> {
            System.out.println("messages = " + messages);
        };
    }

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
// yaml 파일
spring.cloud.stream.kafka.bindings.{binding-name}.consumer.converter-bean-name: BatchMessagingMessageConverter
```

batch-mode를 사용하기 전과 다르게 사용을 한다면 위의 코드를 돌려보면 messages 리스트가 비어있는채로 들어오게 된다.  
토픽에서 컨슘하고 consumer로 메세지를 보내주는 과정에서 [SmartCompositieMessageConverter의 fromMessage](https://github.com/spring-cloud/spring-cloud-function/blob/e2836b22180b444395157693335b6d2c615f540b/spring-cloud-function-context/src/main/java/org/springframework/cloud/function/context/config/SmartCompositeMessageConverter.java#L78-L124) 메서드를 타게 되는데, batch가 아닐 때에는 첫 번째 조건문에서 바로 ``message.getPayload()``를 리턴해주지만 batch 일 때에는 이곳을 타지 않고 ``converter.fromMessage``쪽 구문을 실행하게 되고 이곳에서 컨버팅 되지 않아 빈 list를 최종적으로 return 하게 된다.  

``ConsumerRecord``를 사용한 것이 아닌  ``List<NameDto>``이런 식으로 사용을 했다면 기본적으로 등록되어있는 컨버터 중 하나인 ``JsonMessageConverter``에서 컨버팅을 해서 정상적으로 메세지들이 읽혀진다.  

하지만 ``ConsumerRecord``인 경우에는 ``JsonMessageConverter``의 흐름을 타면서 ``convertFromInternal``메서드를 호출, 이곳에서 ``JsonMapper``의 흐름을 타게된다. 

<img width="1785" alt="image" src="https://github.com/user-attachments/assets/97235186-5d9c-4cc5-a364-f1212f1246a6">

결국 위의 메서드에서 예외가 발생하고 바깥 scope에서 예외 발생시 null을 발생시키고 또 이 바깥 scope에서 null이면 resultList 변수에 아무 것도 추가하지 않아 최종적으로 Consumer에서 빈 리스트를 보게되는 것이다.  

<br/>

그러면 어떻게 해야할까? 이것에 대한 해결책을 2가지 생각해보았다.  

1. ``BatchMessagingMessageConverter`` 에서 다른 타입으로 변환해주기

   ```java
   public static class CustomBatchMessageConverter extends BatchMessagingMessageConverter {
   
       @Override
       public Message<?> toMessage(List<ConsumerRecord<?, ?>> records, @Nullable Acknowledgment acknowledgment, org.apache.kafka.clients.consumer.Consumer<?, ?> consumer, Type type) {
         
           final Message<?> message = super.toMessage(records, acknowledgment, consumer, type);
           final ObjectMapper objectMapper = new ObjectMapper();
           final List<NameDto> collect = records.stream()
                                          .map(record -> {
                                              try {
                                                  return objectMapper.readValue((String) record.value(), NameDto.class);
                                              } catch (IOException e) {
                                                  throw new RuntimeException(e);
                                              }
                                          })
                                          .collect(Collectors.toList());
           final Message<List<NameDto>> result = MessageBuilder.createMessage(collect, message.getHeaders());
           return result;
       }
   }
   
   ---
     
   @Bean
   public Consumer<Message<List<NameDto>>> testConsumer() {
       return message -> {
           System.out.println("message = " + message);
       };
   }
   ```

   컨버터에서 컨슘한 ``List<ConsumerRecord>``를 내가 컨슈머에서 읽으려고 하는 ``Message`` 타입으로 변경해주는 것이다.  
   ``Message``에도 헤더에 대한 정보가 들어있기 때문에 사용하는데 이상이 없다.  

2. 커스텀한 ``AbstractMessageConverter``추가하기

   위에서 언급한 [SmartCompositieMessageConverter의 fromMessage](https://github.com/spring-cloud/spring-cloud-function/blob/e2836b22180b444395157693335b6d2c615f540b/spring-cloud-function-context/src/main/java/org/springframework/cloud/function/context/config/SmartCompositeMessageConverter.java#L78-L124) 이 로직에서 ``List<ConsumerRecord>``를 컨버팅할 적절한 컨버터가 없어서 결국 아무것도 없는 list가 return이 되었다. 이것을 위해서 따로 컨버터를 만들고 빈등록을 해줌으로써 ``ConsumerRecord``를 이용하는 것이다.  

   ```java
   public class AssignableTypeMessageConverter extends AbstractMessageConverter {
   
       public AssignableTypeMessageConverter() {
           super(List.of(MimeTypeUtils.ALL));
       }
   
       @Override
       protected boolean supports(Class<?> clazz) {
           return true;
       }
   
       @Override
       protected boolean canConvertFrom(Message<?> message, Class<?> targetClass) {
           return ClassUtils.isAssignableValue(targetClass, message.getPayload());
       }
   
       @Override
       protected Object convertFromInternal(Message<?> message, Class<?> targetClass, Object conversionHint) {
           return message.getPayload();
       }
   
       @Override
       protected boolean canConvertTo(Object payload, MessageHeaders headers) {
           return false;
       }
   }
   
   ---
     
   @Bean
   public AbstractMessageConverter assignableTypeMessageConverter() {
       return new AssignableTypeMessageConverter();
   }
   ```

   이 상태로 수행하게 되면 아래와 같이 등록된 컨버터의 목록에 생성해준 메세지 컨버터가 보이게 된다.  
   <img width="572" alt="image" src="https://github.com/user-attachments/assets/5cfdfde4-4eef-4204-925b-e0e72af70aa3">

   위의 코드로 진행을 하게되면 ``convertFromInternal``에서 따로 내부적으로 json 변환이라던지 그런 단계가 없기 때문에 ``ConsumerRecord``의 value를 확인해보면 ``NameDto``가 아닌 읽어들인 그대로가 있는 것을 확인할 수 있다. 만약 value.deserializer가 StringDeserializer라면 jsonString 형태 그대로 찍힐 것이고, ByteArrayDeserializer 라면 바이트 배열이 찍히게 될 것이다. 일단 이렇게 받은 다음에 컨슈머 코드 내부에서 dto로 변환하는 로직을 추가하는 방향으로 진행하면 된다.

개인적으로는 ``BatchMessagingMessageConverter``에서 미리 변환을 해서 ``Message`` 타입으로 받아서 사용하는 것이 조금 더 직관적이고 깔끔하지 않나 싶다. ``ConsumerRecord``를 사용하는 것 자체가 메세지의 메타데이터들을 사용하고 싶다는 뜻일텐데, ``Mesasge``에서도 충분히 해당 데이터들을 가져올 수 있기 때문이다.  

``Message<List<*>>`` 이런 형태라고 해서 n개의 데이터 중에서 첫 번째에 해당하는 메세지의 헤더 정보만 있을 것이라고 생각할 수도 있는데,  
<img width="717" alt="image" src="https://github.com/user-attachments/assets/417e574b-2784-43dc-93e0-942cdb6afedf">

그렇지 않기 때문에 그냥 ``Message`` 타입을 이용해서 헤더 정보를 이용하면 된다.


---

## REFERENCE

https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-reactive-binder/consuming.html

