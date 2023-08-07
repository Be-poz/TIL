# Spring Cloud Stream Kafka Binder 정리

```gradle
implementation 'org.springframework.cloud:spring-cloud-stream-binder-kafka'
```

```yaml
spring:
  cloud:
    stream:
      function:
        definition: feederDownload
      bindings:
        feederDownload-in-0:
          group: com.test.downloader
          destination: feeder.request.download
          contentType: application/json
          consumer:
            concurrency: 1
            maxAttempts: 3
            backOffInitialInterval: 30000
            backOffMaxInterval: 30000

  kafka:
    bootstrap-servers: ${kafka.brokers}
    consumer:
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      properties:
        max.poll.interval.ms: 7200000 # 2시간
        heartbeat.interval.ms:  60000 # 1분
        session.timeout.ms: 300000 #5분
        max.poll.records: 1
```

