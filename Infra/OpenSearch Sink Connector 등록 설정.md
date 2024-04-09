# OpenSearch Sink Connector 등록 설정

## OpenSearch Sink Connector 란

말 그대로 open search를 위한 sink connector 입니다. 카프카 토픽의 내용을 바로 open search의 index로 인덱싱하기 위해서 사용합니다. 

elastic search에서 fork 되어서 나온 open search 라고 elastic search sink connector를 사용하면 동작하지 않습니다. 
( 관련해서 es 직원이 답변해둔 [링크](https://forum.confluent.io/t/elasticsearch-sink-connector-version-compatibility-table/4145) )  

es sink connector의 경우 confluent 사에서 공식적으로 plugin을 제공한다(ex. confluentinc/kafka-connect-elasticsearch:14.0.12). 

open search sink connector의 경우 찾아보기론 confluent의 공식 plugin은 없으며 aiven이라는 곳에서 제공하는 plugin이 있다. 일부 설정은 es sink connector와 다르지만(ex. es sink connector에는 'topic.index.map'의 설정으로 토픽명과 인덱스명 매핑이 가능하지만 open search sink에는 없다), 대부분의 설정은 같다.  

<br/>

## 사용하기 위해 필요한 사항

모든 커넥터들이 그렇지만 open search sink connector 또한 plugin이 설치되어있어야 한다.  
사용 중인 커넥터의 'plugin.path' 설정에 필요한 plugin 내용이 존재해야 한다.  
어떤 방식을 사용해도 상관없지만 나는 아래의 방식을 사용했다.

```sh
wget https://github.com/Aiven-Open/opensearch-connector-for-apache-kafka/releases/download/v3.0.0/opensearch-connector-for-apache-kafka-3.0.0.tar

tar -xf opensearch-connector-for-apache-kafka-3.0.0.tar
```

이렇게 plugin을 준비하고 커넥트 서버를 재시작한 후 커넥트 서버에 '/connector-plugins' get api 를 호출하였을 때 플러그인이 추가가 되어있으면 이제 준비가 다 된 것이다. 

<Br/>

## 커넥터 등록 api 설정

다른 여러 커넥터 등록 때와 동일하게 '/{connector-name}/config' 의 put 요청을 날리면 됩니다.  
내가 로컬에서 테스트할 때에 돌렸던 body 값은 아래와 같다.  

```json
{
    "tasks.max":1,
    "connector.class":"io.aiven.kafka.connect.opensearch.OpensearchSinkConnector",
    "topics": "test-topic",
    "connection.url": "{opensearch url}",
    "connection.username": "${file:/etc/secret-volume/connection_info.properties:username}",
    "connection.password": "${file:/etc/secret-volume/connection_info.properties:password}",
    "type.name": "_doc",
    "key.ignore": "true",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "schema.ignore": "true"
}
```

configuration의 정보는 [confluent es sink connector configuration 공식 문서](https://docs.confluent.io/kafka-connectors/elasticsearch/current/configuration_options.html)와 open search sink connector를 제공한 [aiven의 공식 문서](https://aiven.io/docs/products/kafka/kafka-connect/howto/opensearch-sink)를 참고했다. 

위의 body 내용은 'test-topic' 이라는 토픽에서 컨슘하여 open search에 인덱싱하는 커넥터를 등록한 것이다. open search sink connector는 topic명 그대로 index 명으로 생성이 된다. 따라서 이것을 원하지 않는다면 alias를 이용하자.  

username과 password 같은 값 들은 하드코딩해서 그대로 입력하면 안된다. 왜냐하면 해당 커넥터의 config 조회를 하면 그대로 들어나기 때문이다. 따라서 properties 파일 안에 넣어두고 그것을 읽어오는 방식을 취해야 한다. [confluent에서도 권장하는 방법이다.](https://docs.confluent.io/platform/current/connect/security.html#fileconfigprovider) 

이를 사용하기 위해서는 커넥트 설정이나 커넥터 설정에 아래와 같은 설정이 추가되어있어야 한다. 

```yaml
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
```

<br/>

## Transforms 이용하여 토픽 데이터 변경하기

Confluent사가 지원하는 Transforms 기능을 이용하여 토픽의 데이터를 조금 변형하여 인덱싱되게끔 할 수 있다. 

공식문서는 https://docs.confluent.io/platform/current/connect/transforms/overview.html 이며, 일부 기능의 경우에는 [connect-transformations](https://www.confluent.io/hub/confluentinc/connect-transforms) 이라는 plugin 설치가 추가로 필요합니다.

공식문서에 워낙 자세하게 나와있어서 자세한 설명은 필요없을 것 같고 일부 자주 사용될 것 같은 기능만 간략하게 말하겠다.

커넥터를 등록할 때 사용했던 api를 그대로 사용을 하고 body 값에 추가적인 설정을 하면 됩니다.  

### 제외하고 싶은 필드

```yaml
{
    ...
    "transforms": "DropField",
    "transforms.DropField.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.DropField.exclude": "{제외하고 싶은 필드}"
}
```

transforms 에는 그냥 임의의 이름을 두고, type에는 어떤 transforms를 사용할 것인지를 적어두면 된다. 위의 예시에서는 ReplaceFields$Value 를 택했고 value를 replace 하게끔 지원해주는 type이다. 

그리고 exclude 로 버리고자 하는 필드를 입력한다. exclude는 ReplaceField 의 properties 값이고 이건 transforms 종류마다 다르다.  

### 카프카 헤더값 추가

```yaml
{
    ...
    "transforms": "InsertField",
     "transforms.InsertField.type": "org.apache.kafka.connect.transforms.InsertField$Value",
     "transforms.InsertField.offset.field": "{offset 정보가 들어갈 필드의 이름}",
     "transforms.InsertField.partition.field": "{patition 정보가 들어갈 필드의 이름}",
     "transforms.InsertField.timestamp.field": "{topic record가 생성된 시각정보가 들어갈 필드의 이름}",
     "transforms.InsertField.topic.field": "{topic의 이름이 들어갈 필드의 이름}"
}
```

일반적으로 카프카 메타데이터의 값이 색인되지 않기 때문에 위의 값으로 색인을 시킬 수가 있다. 
아예 새로운 필드를 추가할 수도 있긴 하지만 하드코딩된 값을 추가하는 것만 가능하고 다른 필드의 값을 빼서 넣는 형식은 불가능하다.  

여기서 생성되는 timestamp.field는 long type으로 자동생성되기 때문에 date 타입을 원한다면 미리 템플릿 등록을 해두어야 한다.  

```json
PUT _template/{index name}
{
  "index_patterns": [
    "{index pattern}"
    ], 
  "mappings": {
    "properties" : {
      "kafka" : {
        "properties" : {
          "kafkaCreateDateTime" : {
            "format" : "epoch_millis",
            "type" : "date"
          }
        }
      }
    }
  }
}
```



### 특정 조건에 맞을 때에만 record 색인하기

```yaml
{
    ...
    "transforms": "FilterExample",
    "transforms.FilterExample.type": "io.confluent.connect.transforms.Filter$Value",
    "transforms.FilterExample.filter.condition": "$[?(@.updatedFields.price)]",
    "transforms.FilterExample.filter.type": "include",
    "transforms.FilterExample.missing.or.null.behavior": "exclude",
}
```

필터를 줘서 해당 조건에 맞는 record를 include 할지 exclude 할지 결정하는 transforms 이다.  
condition에는 jsonPath 형식으로 들어가게 된다. jsonPath의 filter operator 문법을 이용해서 진행하면 된다.  

<br/>

### 일자별 index 생성

```yaml
{
    ...
    "transforms": "TimestampRouter",    
    "transforms.TimestampRouter.type": "org.apache.kafka.connect.transforms.TimestampRouter",
    "transforms.TimestampRouter.topic.format": "${topic}-${timestamp}",
    "transforms.TimestampRouter.timestamp.format": "yyyyMMdd"
}
```

해당 topic을 indexing 하는데 일자별로 index가 생기길 원한다면 위와 같이 진행하면 된다.  

### 여러 transforms를 한 번에 사용하기

```yaml
{
    ...
    "transforms": "FilterExample, InsertField",
    "transforms.FilterExample.type": "io.confluent.connect.transforms.Filter$Value",
    "transforms.FilterExample.filter.condition": "$[?(@.updatedFields.displayPrice)]",
    "transforms.FilterExample.filter.type": "include",
    "transforms.FilterExample.missing.or.null.behavior": "exclude",
    "transforms.InsertField.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.InsertField.offset.field": "{offset 정보가 들어갈 필드의 이름}",
    "transforms.InsertField.partition.field": "{patition 정보가 들어갈 필드의 이름}",
    "transforms.InsertField.timestamp.field": "{topic record가 생성된 시각정보가 들어갈 필드의 이름}",
    "transforms.InsertField.topic.field": "{topic의 이름이 들어갈 필드의 이름}"
}
```

여러 transforms를 한 번에 사용할 때에는 그냥 위와 같이 transforms안에 여러개를 정의해두면 된다.  
동작하는 순서는 정의한 순서대로 동작한다. 
따라서 만약 특정 필드를 exclude 시켰거나 필드의 이름을 변경시켰는데 그 다음 transforms 단계에서 해당 필드를 이용하려고 한다면 오류가 발생할 수도 있다.  

<br/>

## 주의해야 할 점

상품과 같이 id를 key로 가지고 있는 topic을 consume 하여 indexing 하는 경우 오류가 발생할 수 있다. 

동일한 document 대상으로 여러 task가 계속해서 update를 치게되는 과정에서 version 어쩌고하면서 문제가 발생할 수 있다. 이 문제는 elastic/open search가 낙관적 락을 사용하기 때문에 발생한다. 애초에 빈번한 update가 일어나는 경우 es는 적절하지 않다고 한다.

이럴 떄에는 'key.ignore' configuration 값을 true로 주면 해결이 된다. true로 주면 '{토픽명}+{파티션 번호}+{오프셋 번호}' 의 형식으로 id를 가지게 된다. 

---

### REFERENCE

https://docs.confluent.io/platform/current/installation/configuration/connect/index.html#bootstrap-servers

https://aiven.io/docs/products/kafka/kafka-connect/howto/opensearch-sink

https://docs.confluent.io/platform/current/connect/transforms/overview.html

https://discuss.elastic.co/t/version-conflict-409-question/311335

https://discuss.elastic.co/t/version-conflict-issue-while-updating-data-continously/344065

