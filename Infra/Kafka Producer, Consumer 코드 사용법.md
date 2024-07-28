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

서버는 yaml에 기입해주었고, 기본 템플릿을 주입받아 사용하였다. ``send`` 메서드는 오버로드된 여러 시그니처들이 있다. 위의 코드에서는 토픽 명과 데이터만 넣었지만, 키, 파티션, 타임스탬프를 받을 수 있게끔 되어있고, ``ProducerRecord``를 받는 메서드도 있다.  

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

<br/>

## Spring Kafka를 이용한 컨슈머

