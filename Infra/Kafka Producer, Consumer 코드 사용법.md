# Kafka Producer, Consumer 코드 사용법

## Kafka Client를 이용한 프로듀서

```java
final Properties configs = new Properties();
configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

final KafkaProducer<String, String> producer = new KafkaProducer(configs);
final String key = "key5";
final String message = "this is value11";
final ProducerRecord<String, String> producerRecord = new ProducerRecord<>(TOPIC_NAME, message);
producer.send(producerRecord);
producer.flush();
producer.close();
```

``org.apache.kafka:kafka-clients`` 의존성을 추가한 상태다.  
``ProducerRecord``를 통해 데이터 레코드를 입력할 수 있는데 파티션 지정까지 가능하다. 존재하지 않은 파티션 번호를 입력하니 동작이 아예 멈췄다.  

``flush()``를 하는 이유는 단순히 ``send`` 메서드를 호출한다고 해서 바로 브로커에 데이터를 넣는 것이 아니라 프로듀서 내부적으로 배치 전송을 하기 위해서 버퍼 메모리 영역에 레코드들을 대기시킨 후 배치의 수 또는 일정시간이 지나면 전송을 하게 되는데 이 덕분에 카프카는 차별화된 전송 속도를 가지게 된다. 아무튼 버퍼에서 대기를 하기 때문에 바로 전송하기 위해 호출을 해준 것이다.  

``close()``는 애플리케이션을 종료하기 전에 프로듀서 인스턴스의 리소스들을 안전하게 종료하기 위함이다.  

<br/>

프로듀서는 데이터 레코드를 위에서 언급한 버퍼로 쌓기 전에 ``Partitioner``를 이용해 어떤 파티션에 전송될 것인지를 정한다. 그 후 ``Accumulator``에 쌓아두다가 sender가 전송한다.  

이 파티셔너는 별다른 설정이 없으면 ``DefaultPartitioner``가 파티셔너로 설정되어 동작하는데, 버전마다 이 동작 방식이 살짝 다르다. https://www.confluent.io/ko-kr/blog/apache-kafka-producer-improvements-sticky-partitioner/ 이 글에 따르면 key 값이 null인 경우 2.3 버전 이전에는 라운드 로빈 방식, 2.4 이후에는 스티키 파티셔닝 방식이 default 방식이라고 나온다. key 값이 존재할 때에는 버전 관계없이 hash 방식이 default이다.  

```java
public class CustomPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        if(keyBytes == null) {
            return 0;
        }

        if("bepoz".equals(key.toString())) {
            return 1;
        }

        final List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(topic);
        return Utils.toPositive(Utils.murmur2(keyBytes)) % partitionInfos.size();
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

-------
  
final Properties configs = new Properties();
configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

configs.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class);

final KafkaProducer<String, String> producer = new KafkaProducer(configs);
final String key = "bepoz";
final String message = "bepoz value2";
final ProducerRecord<String, String> producerRecord = new ProducerRecord<>(TOPIC_NAME, key, message);
producer.send(producerRecord);
producer.flush();
producer.close();
```

위와 같이 ``Partitioner`` 인터페이스를 구현하는 커스텀 파티셔너를 생성하고 프로퍼티에 추가하여 동작시켰다.  
리턴되는 int가 바로 데이터가 들어갈 파티션 번호에 해당된다. key가 ``bepoz`` 라면 1번, key가 없다면 0번, key가 존재한다면 해시값을 이용해 들어가게끔 한 것이다.  

```java
Future<RecordMetadata> send = producer.send(producerRecord);
RecordMetadata recordMetadata = send.get();
System.out.println("recordMetadata = " + recordMetadata);
//recordMetadata = ksy-topic-0630-0@3
```

``send`` 함수 호출 후에 Future 타입을 받고 이를 통해 응답결과를 동기적으로 알 수 있다.  
결과를 보면 0번째 파티션의 오프셋 3에 저장되었다는 것을 알 수가 있다.  

하지만, 동기이기 때문에 빠른 전송에 방해가 될 수 있다. 이를 위해 Callback 클래스를 생성해서 레코드의 전송 결과를  비동기 적으로 받아볼 수가 있다.  

```java
public class CustomProducerCallback implements Callback {

    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            System.out.println("Failed to send message with exception: " + exception);
        } else {
            System.out.println("Successfully sent message with metadata: " + metadata);
        }
    }
}

---
  
producer.send(producerRecord, new CustomProducerCallback());
```

``send`` 메서드에 같이 콜백 클래스를 넣으면 된다. 

<br/>

## Kafka Client를 이용한 컨슈머

```java
final Properties configs = new Properties();
configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
configs.put(ConsumerConfig.GROUP_ID_CONFIG, "ksy-group");
configs.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

final KafkaConsumer<String, String> consumer = new KafkaConsumer(configs);
consumer.subscribe(List.of(TOPIC_NAME));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1000));
    for (ConsumerRecord<String, String> record : records) {
        System.out.println("Received message: " + record);
    }
}

/*

Received message: ConsumerRecord(topic = ksy-topic-0630, partition = 2, leaderEpoch = 0, offset = 4, CreateTime = 1719748493611, serialized key size = -1, serialized value size = 15, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = this is value10)
Received message: ConsumerRecord(topic = ksy-topic-0630, partition = 2, leaderEpoch = 0, offset = 5, CreateTime = 1719748621546, serialized key size = 4, serialized value size = 15, headers = RecordHeaders(headers = [], isReadOnly = false), key = key3, value = this is value12)
Received message: ConsumerRecord(topic = ksy-topic-0630, partition = 2, leaderEpoch = 0, offset = 6, CreateTime = 1719750368124, serialized key size = 5, serialized value size = 11, headers = RecordHeaders(headers = [], isReadOnly = false), key = bepoz, value = bepoz value)
```

``poll`` 메서드에 들어가는 시간은 브로커로부터 데이터를 가져올 때 컨슈머 버퍼에 데이터를 기다리기 위한 타임아웃 간격을 뜻한다. 위의 프로듀서가 넣어준 데이터를 그대로 읽기 위해 ``auto.offset.reset`` 값을 ``earliest``로 설정하였다.  

위의 코드는 기본값인 auto sync를 이용한 것이지만, 컨슈머 강제종료 발생과 같은 경우에 데이터가 중복으로 들어오거나 유실될 수 있다. 만약 중복이나 유실을 허용하지 않는 서비스라면 자동 커밋이 아닌 명시적으로 오프셋을 커밋해야 한다.  

``commitSync()`` 메서드를 호출하면 된다. 하지만 브로커에 커밋 요청을 하고 커밋이 정상적으로 처리되었는지 응답하기까지 기다려야하기 때문에 컨슈머의 처리량에 영향을 끼친다. 동기가 아닌 비동기를 이용한 ``commitAsync()`` 메서드를 사용할 수도 있는데, 커밋 요청을 전송하고 응답이 오기 전까지 데이터 처리를 수행할 수 있다. 하지만 커밋 요청이 실패했을 경우 현재 처리 중인 데이터의 순서를 보장하지 않으며 데이터의 중복 처리가 발생할 수 있다.  

```
commitAsync(1)
commitAsync(2)
commitAsync(3)   1 성공
commitAsync(4)   2 실패
3 성공
4 성공
```

가령 위와 같은 케이스에서 2빼고 1,2,4는 성공했다. 하지만 2는 실패했기 때문에 해당 커밋을 다시 보낸다. 그러고 라스트 오프셋이 2가 되기 때문에 결국에는 3,4의 데이터를 다시 컨슘해서 커밋을 하게될 것이다.  

```java
final Properties configs = new Properties();
configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
configs.put(ConsumerConfig.GROUP_ID_CONFIG, "ksy-group3");
configs.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

final KafkaConsumer<String, String> consumer = new KafkaConsumer(configs);
consumer.subscribe(List.of(TOPIC_NAME));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1000));
    for (ConsumerRecord<String, String> record : records) {
        System.out.println("Received message: " + record);
    }
    consumer.commitAsync((offsets, exception) -> {
        if (exception != null) {
            System.out.println("Failed to commit offsets with exception: " + exception);
        } else {
            System.out.println("Successfully committed offsets: " + offsets);
        }
    });
}
```

이렇게 비동기 commit을 사용할 때에 콜백 함수를 정의해서 사용할 수도 있다.  
자동 커밋을 false로 두면 ``max.poll.records`` 설정이 조금 다르게 동작하는 것 같다. 1개씩 읽어오던데 이것과 관련한 것은 조금 깊어질 것 같아서 찾아보지 않았다. 그리고 ``max.poll.records``를 10으로 설정하고 파티션 0,1,2에 각각 3,3,4개씩 들어있는 상황이라면 이 3,3,4를 한 번에 읽는 것은 또 아닌 것 같다. 상황에 따라 파티션을 나눠서 읽어들이는 것을 확인했다.  

```java
private static class RebalanceListener implements ConsumerRebalanceListener {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Partitions are revoked.");
        consumer.commitSync(currentOffsets);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        System.out.println("Partitions are assigned.");
    }
}

---
  
consumer.subscribe(List.of(TOPIC_NAME), new RebalanceListener());
```

``ConsumerRebalanceListener``를 이용해서 리밸런스 발생 시의 동작도 제어할 수 있다.  
``onPartitionsRevoked``는 리밸런스가 시작되기 직전에 호출되는 메서드이고, ``onPartitionsAssigned``는 리밸런스가 끝난 뒤에 파티션이 할당 완료되면 호출되는 메서드이다.  

리밸런스 발생 시에 데이터 중복 처리를 막기 위해서 리밸런스 발생 직전에 커밋을 시켜준 코드이다.  



컨슈머를 토픽에 subscribe 하여 읽는 것 외에도 특정 토픽의 특정 파티션을 컨슈머에 명시적으로 할당하여 운영할 수도 있다.  

```java
consumer.assign(Collections.singleton(new TopicPartition(TOPIC_NAME, PARTITION_NUMBER)));
```

이렇게 ``subscribe()`` 메서드가 아닌 ``assign()`` 메서드를 사용해서 특정 토픽의 파티션을 할당시켰다.  



``KafkaConsumer`` 클래스는 ``wakeup()``이라는 컨슈머를 안전하게 종료할 수 있는 메서드를 지원한다. 해당 메서드가 호출된 후 ``poll()`` 메서드가 호출되면 ``WakeupException`` 예외가 발생한다. 

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
        for (ConsumerRecord<String, String> record : records) {
            System.out.println("Received message: " + record);
        }
    }
} catch (WakeupException e) {
    System.out.println("Wakeup consumer.");
} finally {
    consumer.close();
}

---
  
static class ShutdownThread extends Thread {
    @Override
    public void run() {
        consumer.wakeup();
    }
}
```

``poll()``메서드를 통해 지속적으로 레코드들을 받아 처리하다가 ``wakeup()``메서드가 호출되면, 다음 ``poll()``메서드가 호출될 때 ``WakeupException`` 예외가 발생한다. 이 ``wakeup()`` 메서드는 자바에서 셧다운 훅을 구현하여 명시적으로 구현할 수 있다. 셧다운 훅이란 사용자 또는 운영체제로부터 종료 요청을 받으면 실행하는 스레드를 뜻한다. 셧다운 훅을 사용하여 ``wakeup()`` 메서드를 호출하였다.  

<br/>

## Spring Kafka를 이용한 프로듀서

```gradle
implementation 'org.springframework.kafka:spring-kafka:3.2.1'
```

스프링 카프카 의존성을 설치하면 kafka-client 까지 같이 들어온다.  
스프링 카프카 프로듀서는 ``KafkaTemplate`` 이라는 클래스를 사용한다.  
스프링에서 제공하는 기본 템플릿을 사용하든가 아니면 ``ProducerFactory``를 사용해서 직접 템플릿을 생성하는 방법이 있다.  

<img width="578" alt="image" src="https://github.com/user-attachments/assets/e15c6419-bc5f-43e3-805c-2f8570cb12ee">

application.yaml 파일에 옵션을 설정하면 자동으로 오버라이드되어 설정된다. 카프카 클라이언트에서는 bootstrap-servers, key-serailizer, value-serializer를 선언하지 않으면 예외가 발생하지만 스프링 카프카에서는 localhost:9091, StringSerializer로 기본값이 세팅된다.  

```java
@SpringBootApplication
public class KafkaStudyApplication implements CommandLineRunner {

    private final static String TOPIC_NAME = "ksy-topic-0630";

    @Autowired
    private KafkaTemplate<String, String> template;

    public static void main(String[] args) {
        SpringApplication.run(KafkaStudyApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        template.send(TOPIC_NAME, "Hello, World!");
    }
}
```

서버는 yaml에 기입해주었고, 기본 템플릿을 주입받아 사용하였다. ``send`` 메서드는 오버로드된 여러 시그니처들이 있다. 위의 코드에서는 토픽 명과 데이터만 넣었지만, 키, 파티션, 타임스탬프를 받을 수 있게끔 되어있고, ``ProducerRecord``를 받는 메서드도 있다. 만약 ``Message<?>`` 파라미터가 들어간 메서드를 사용한다면 헤더의 값으로 ``KafakHeaders.TOPIC``,``KafkaHeaders.PARTITION``와 같은 값을 사용하면 된다.    

<Br/>

```java
@Configuration
public class CustomKafkaTemplateConfiguration {

    private final static String BOOTSTRAP_SERVERS =
            "";
    @Bean
    public KafkaTemplate<String, String> customKafkaTemplate() {
        final Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        final DefaultKafkaProducerFactory<String, String> factory = new DefaultKafkaProducerFactory<>(props);
        return new KafkaTemplate<>(factory);
    }
}

---
  
@Autowired
private KafkaTemplate<String, String> customKafkaTemplate;

public static void main(String[] args) {
    SpringApplication.run(KafkaStudyApplication.class, args);
}

@Override
public void run(String... args) throws Exception {
    customKafkaTemplate.send(TOPIC_NAME, "Hello, World!!!").handle((result, ex) -> {
        if (ex != null) {
            System.out.println("Failed to send message: " + ex);
        } else {
            System.out.println("Sent message: " + result);
        }
        return result;
    });
}
```

한 어플리케이션 내에서 프로듀서를 여러개 사용하고 싶을 때 커스텀한 카프카템플릿을 사용하면 된다.  
위와 같이 팩토리 클래스를 이용해 템플릿을 만들고 빈 등록을 해주었다. 그리고 해당 템플릿을 주입받아 사용하였다.  handle 메서드를 이용하여 성공, 실패에 대한 콜백을 지정하였다. 저 상태로 코드를 돌리면 전송하기 전에 어플리케이션이 종료가 되어 원활하게 토픽에 데이터가 안들어갈 수도 있으므로 테스트 용이니 뒤에 ``Thread.sleep()``이나 여타 다른 시간을 버는 코드를 추가하면 된다.  

스프링이 기본적으로 제공해주는 ``KafkaTemplate``은 yml 파일의 기본 설정을 따른다. 만약 이것도 설정되어 있지 않다면 진짜 default 값을 따른다. 위에서는 프로듀서를 여러개 사용하고 싶을 때라고 했지만 사실 세밀한 설정을 하고 싶으면 그냥 사용하면 된다.  

```java
public class CustomKafkaProducerListener implements ProducerListener {
    @Override
    public void onSuccess(ProducerRecord producerRecord, RecordMetadata recordMetadata) {
      ...
    }

    @Override
    public void onError(ProducerRecord producerRecord, RecordMetadata recordMetadata, Exception exception) {
      ...
    }
}

----------------------------------------------------------------------
  
KafkaTemplate template = new KafkaTemplate<>(factory);
template.setProducerListener(new CustomKafkaProducerListener());
```

``send().handle()``을 통해 성공/실패에 따른 비동기 처리를 할 수도 있지만 위와 같이  ``ProducerListener``를 구현하여 등록할 수도 있다. 여러 프로듀서에서 공통적으로 사용이 될 로직이면 리스너 클래스를 생성해두고 사용하는 식이 좋아 보인다. 프로듀서 개별적으로 성공/실패에 따른 처리 로직이 필요하다면 ``handle()``에서 처리하는 것이 좋아보인다.  

스프링에서  ``LoggingProducerListener``라는 로깅용 리스너 클래스를 따로 지원해주기도 한다.  

<br/>

## Spring Kafka를 이용한 컨슈머

스프링 카프카의 컨슈머는 기본 컨슈머를 2개의 타입으로 나누고 커밋을 7가지로 나누어 세분화되어있다.  
타입은 레코드 리스너(MessageListener)(SINGLE 이라고 말하기도 함)와 배치 리스너(BatchMessageListener)가 있다. 전자는 단 1개의 레코드를 처리하고, 후자는 ``poll()`` 메서드로 리턴받은 ``ComsumerRecords`` 처럼 한 번에 여러 개 레코드들을 처리할 수 있다.  

스프링 카프카에서는 사용자가 사용할 만한 커밋의 종류를 7가지로 세분화해놨고 커밋이라고 부르지 않고 'AckMode'라고 부른다. 프로듀서에서 사용하는 acks 옵션과는 다르다.  

| AcksMode         | 설명                                                         |
| ---------------- | ------------------------------------------------------------ |
| RECORD           | 레코드 단위로 프로세싱 이후 커밋                             |
| BATCH            | poll()메서드로 호출된 레코드가 모두 처리된 이후 커밋, default |
| TIME             | 특정 시간 이후 커밋, 시간 간격을 선언하는 AckTime 옵션 설정 필요 |
| COUNT            | 특정 개수만큼 처리된 이후에 커밋, 레코드 개수를 선언하는 AckCount 옵션 설정 필요 |
| COUNT_TIME       | TIME, COUNT 옵션 중 맞는 조건이 하나라도 나올 경우 커밋      |
| MANUAL           | acknowledge() 메서드가 호출되면 다음번 poll() 때 커밋을 한다. 매번 acknowledge() 메서드를 호출하면 BATCH 옵션과 동일하게 동작한다. 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 리스너로 사용해야 한다. |
| MANUAL_IMMEDIATE | acknowledge() 메서드를 호출한 즉시 커밋한다. 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 리스너로 사용해야 한다. |

```java
@KafkaListener(topics = TOPIC_NAME, groupId = "ksy-test")
public void recordListener(ConsumerRecord<String, String> record) {
    System.out.println("Received message: " + record);
}
```

기본적으로 위와 같이 사용하면 된다(String message 이렇게도 사용하려면 가능하긴 함). ``KafkaListener`` 어노테이션은 위의 사용법 이외에도 property 값을 따로 지정해주거나, 병렬 처리, 특정 토픽의 파티션만 읽게끔 하는 등의 설정을 할 수 있다. [KafkaListener annotaion docs](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html) 에서 확인해보자.  
SINGLE의 경우 Listener 타입 Default이기 때문에 따로 지정해줄 필요가 없다.  

```java
spring.kafka.listener.type: BATCH

----------------------------------------------------------------------------

@KafkaListener(topics = TOPIC_NAME, groupId = "ksy-test9")
public void batchListener(ConsumerRecords<String, String> records) {
    for (ConsumerRecord<String, String> record : records) {
        System.out.println("record = " + record);
    }
}
```

리스너의 타입을 ``BATCH``로 설정하고 여러 레코드들을 한 번에 가져올 수도 있다.  

수동 커밋을 사용하고 싶다면 ``BatchAcknowledgingMessageListener``,  ``BatchConsumerAwareMessageListener`` 를 사용하면 된다. 이 때, ``AckMode``는 ``MANUAL``, ``MANUAL_IMMEDIATE``로 설정하면된다.  

<br/>

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
    factory.setBatchListener(true);
    return factory;
}


-------------------------------------------------------------------------
  
@KafkaListener(topics = TOPIC_NAME, groupId = "custom-listener", containerFactory = "customContainerFactory")
public void customListener(ConsumerRecords<String, String> records) {
    for (ConsumerRecord<String, String> record : records) {
        System.out.println("custom listener's record = " + record);
    }
}
```

한 어플리케이션 내에서 여러개의 리스너를 사용해야 하는 상황이고 설정이 각기 다르다면 커스텀한 리스너 컨테이너를 사용하면 된다. 어노테이션에 위의 코드와 같이 커스텀하게 만들어준 팩토리를 지정해주면 된다. yml 파일에 ``auto.offset.reset`` 설정을 하지 않았다면 default가 ``latest`` 이기 때문에 팩토리 설정을 안해준 리스너로는 ``latest``로 동작을 하겠지만 커스텀하게 만들어준 곳에는 ``earliest``로 박아두었기 때문에 토픽에서 모든 레코드들을 읽어오는 방식으로 동작하게 된다.  

``factory.setBatchListener(true);`` 이 구문 때문에 listener의  타입이 BATCH가 되어서 배치로 읽어오게 된다.  

```java
@KafkaListener(topics = TOPIC_NAME, groupId = "original3", properties = "auto.offset.reset=earliest")
public void batchListener(ConsumerRecord<String, String> record) {
    System.out.println(record);
}
```

만약 위와 같이 따로  containerFactory를 사용하지 않은 상황에서  yml에 ``spring.kafka.listener.type: SINGLE``로 해두었다면 단건으로 읽어오게 될 것이다. 유의해야 할 점은 파라미터로  ``ConsumerRecords``로 두면 오류가 난다는 것이다. 배치로 읽어들이는 것이 아니기 때문! ``ConsumerRecord``로 두어야 한다.  

``@KafkaListener``을 사용하는 메서드에  ``@SentTo``를 사용해서 다른 토픽에 바로 프로듀싱을 할 수도 있다.  

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
    factory.setReplyTemplate(customKafkaTemplate());
    return factory;
}

---
  
@KafkaListener(topics = TOPIC_NAME, groupId = "sentToGroup", containerFactory = "customContainerFactory")
@SendTo("ksyTest-out")
public String sendToListener(ConsumerRecord<String, String> record) {
    System.out.println(record);
    return record.value();
}
```

위와 같이 factory의 `` setReplyTemplate``을 이용해서 reply 하기위한(토픽에 넣어주기 위한) 템플릿을 set 해주기만 하면된다.  
``KafkaTemplate``을 상속받은  ``ReplyingKafkaTemplate``을 선언해서 set 해주어도 된다.  

<br/>

## Spring Cloud Function, Stream을 이용한 프로듀서와 컨슈머

``spring-kafka:3.2.1``, ``spring-cloud-stream:4.1.3``, ``spring-cloud-stream-binder-kafka:4.1.3`` 기준 작성  

java의 functional interface인  ``Consumer``,  ``Function``, ``Supplier``을 이용해서 kafka에서 데이터를 읽고 쓰는 것을 할 수가 있다(spring-cloud-function). 정확히는 Bean 등록하여 사용하는 방식이다. 

```yaml
spring:
  application:
    name: kafka-study
  cloud:
    function:
      definition: testConsumer;testFunction
    stream:
      bindings:
        default:
          binder: kafka
          contentType: application/json
        testConsumer-in-0:
          binder: kafka
          destination: ksyTest
          contentType: application/json
          group: ksy-test-group3
          consumer:
            batch-mode: true
        testFunction-in-0:
          destination: ksyTest
          group: ksy-test-function
          consumer:
              batch-mode: true
        testFunction-out-0:
          destination: ksyTest-out

      kafka:
        binder:
          brokers: ""
        bindings:
          testConsumer-in-0:
              consumer:
                start-offset: earliest
                configuration:
                  key.deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
                  value.deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
          testFunction-in-0:
            consumer:
              start-offset: earliest
```

설정은 위와 같이  ``spring.cloud.stream.bindings.<functionName>+in/out+<index>`` 형태를 가지고 그 하위에 다른 설정들을 작성하는 식이다.  

in과 out은 말 그대로 input, output의 줄임이고, input은 읽어오는 토픽에 대한 정보를, output에는 write하는 토픽에 대한 정보를 입력하면 된다.  

``spring.cloud.stream.bindings``에 대한 설정 정보는 [해당  docs 페이지](https://docs.spring.io/spring-cloud-stream/reference/spring-cloud-stream/binding-properties.html)를 참고하자  
``spring.cloud.stream.kafka.binder`` 하위의 정보는 [해당  docs 페이지](https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-binder/config-options.html)를 참고하자  
``spring.cloud.stream.kafka.binder.consumer/producer`` 하위에 여러 옵션이 있는데  configuration 옵션도 있다. 이곳에 그냥 직접적으로 설정 값을 입력할 수도 있다. 위의 yaml에서는 ``key.deserializer``와 ``value.deserializer``를 따로 설정해두었다.   

spring-kafka를 사용할 때에 default deserializer가 StringDeserializer이지만, kafka binder를 사용할 때에는 내부적으로 kafka-client를 사용하기 때문에([ref](https://docs.spring.io/spring-cloud-stream/reference/kafka/kafka-binder/overview.html)) default로 ByteArrayDeserializer를 사용하기 때문에 사실 설정할 필요가 없긴하다.  

docs에 나와있는 것 처럼  여러 binding이 있어 한 번에 설정하기 위해 default 값을 설정하고 싶다면 ``spring.cloud.stream.kafka.default.consumer.<property>=<value>.`` 이런 식으로 둘 수도 있지만,  
spring-kakfa를 이용할 때 처럼 ``spring.kafka.consumer`` 하위에 두어도 설정은 된다.  
어찌되었던 간에 ``org.apache.kafka.clients.consumer.ConsumerConfig`` 이쪽으로 설정 값 들이 들어가고 이것으로 연결을 맺기 때문으로 보인다.  

```java
@Bean
public Consumer<List<String>> testConsumer() {
    return messages -> {
        for (String message : messages) {
            System.out.println("[testConsumer]message = " + message);
        }
    };
}

@Bean
public Function<List<String>, List<String>> testFunction() {
    return messages -> {
        System.out.println("[testFunction] message = " + messages);
        return messages;
    };
}
```

코드 자체는 위와 같이 함수명을 yaml에 선언했던 definition과 일치시키면 된다.  
위의 코드에서는  Function의 return을 List<String>으로 두었는데 이렇게 하면 해당 내부 요소가 하나씩 데이터로 나가는게 아니라 말 그대로 List 타입으로 들어가게 되니깐 조심하자.  

---

### REFERENCE

https://docs.spring.io/spring-kafka/reference/introduction.html

https://docs.spring.io/spring-cloud-stream/reference/index.html

https://docs.spring.io/spring-cloud-function/reference/spring-cloud-function/introduction.html

책 아파치 카프카  by 최원영
