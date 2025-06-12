# Spring Kafka 트랜잭션 적용하기

producer를 사용할 때에 정확히 한 번 전송하기 위해 트랜잭션을 사용해야 하는 경우가 있을 수 있다.  
apache가 제공하는 ``KafkaProducer``가 아닌 spring이 제공하는 ``KafkaTemplate``을 이용해보겠다.  
전자에 대한 정보는 [baeldung](https://www.baeldung.com/kafka-exactly-once)을 참고하자.  

<br/>

## 필요한 설정

```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: test
```

``transaction-id-prefix`` 값을 설정해주면 된다. 해당 설정이 존재하면 아래와 같은 흐름을 통해 ``enable.idempotence`` 설정값이 true가 된다.  

```
# DefaultKafkaProducerFactory
String txId = (String)this.configs.get("transactional.id");
if (StringUtils.hasText(txId)) {
    this.setTransactionIdPrefix(txId);
    this.configs.remove("transactional.id");
}

...

public final void setTransactionIdPrefix(String transactionIdPrefix) {
    Assert.notNull(transactionIdPrefix, "'transactionIdPrefix' cannot be null");
    this.transactionIdPrefix = transactionIdPrefix;
    this.enableIdempotentBehaviour();
}

...

private void enableIdempotentBehaviour() {
    Object previousValue = this.configs.putIfAbsent("enable.idempotence", true);
    if (Boolean.FALSE.equals(previousValue)) {
        LOGGER.debug(() -> "The 'enable.idempotence' is set to false, may result in duplicate messages");
    }

}
```

spring이 제공하는 kafka에서는 boot 2.3 부터는 ``enable.idempotence``가 기본 설정이므로 따로 설정을 해주지 않아도 된다.  

apache에서 제공하는 ``KafkaProducer``를 이용할 때에는 ``enable.idempotence``값과 ``transactional.id``값을 설정해주어야 한다.  
streamBridge 또한 내부적으로 ``KafkaProducer``를 사용하므로 설정해주어야 한다. ``ProducerConfig``를 참고하자.  

<br/>

## 사용 예시

```kotlin
@GetMapping("/send/{isFailed}")
fun sendDataRecord(@PathVariable isFailed: Boolean): String {
    return kafkaTemplate.executeInTransaction { template ->
        template.send(topicName, "transaction data")
        if(isFailed) {
            throw RuntimeException("Send Failed")
        }
        "Success"
    }
}

@GetMapping("/sendWithAnnotation/{isFailed}")
@Transactional
fun sendDataRecordWithAnnotation(@PathVariable isFailed: Boolean): String {
    ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG
    kafkaTemplate.send(topicName, "transaction data")
    if(isFailed) {
        throw RuntimeException("Send Failed")
    }
    return "Success"
}
```

``KafkaTemplate``이 제공하는 ``executeInTransaction`` 메서드를 사용하거나 단순 ``@Transactional`` 어노테이션을 사용하면 된다.  
조금 더 상세히 트랜잭션을 다루고 싶다면 ``KafkaTransactionManager``을 이용하면 된다.  

<br/>

---

