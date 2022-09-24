## ElasticSearch rollup으로 visualize 그리기 간략정리

목적: 로그를 계속해서 쌓다보면 용량을 계속해서 차지할테고 이는 리소스 비용의 증가로 이어짐. 이것을 summary/압축하여 가볍게 보관하면서 이전의 로그데이터를 사용하기 위함  

일반적인 인덱스는 필드에 값을 저장하는 형태라면 rollup은 metric 이나 aggregation 연산을 정의해두면 이를 background 에서 수행되면서 해당 값으로 조회를 할 수 있게된다.  

elastic search의 오픈소스인 open distro 를 이용하여 키바나에서 rollup을 생성하겠다.  

먼저 인덱스 정보는 다음과 같은 상태이다.  

```json
PUT bepoz
{
  "mappings": {
    "properties": {
      "name": {"type": "keyword"},
      "age": {"type": "short"},
      "gender": {"type": "keyword"},
      "date": {"type": "date", "format": "epoch_millis"}
    }
  }
}

POST _bulk
{"index":{"_index":"bepoz", "_id":"1"}}
{"name":"kang seung yoon", "age":20, "gender":"male", "date": 1662994800000}
{"index":{"_index":"bepoz", "_id":"2"}}
{"name":"kim tae hwan", "age":22, "gender":"male", "date": 1663038000000}
{"index":{"_index":"bepoz", "_id":"3"}}
{"name":"kim mi sun", "age":22, "gender":"female", "date": 1663059600000}
{"index":{"_index":"bepoz", "_id":"4"}}
{"name":"lee mi hyun", "age":32, "gender":"female", "date": 1663077600000}
{"index":{"_index":"bepoz", "_id":"5"}}
{"name":"lee su hyun", "age":33, "gender":"female", "date": 1663081199000}
{"index":{"_index":"bepoz", "_id":"6"}}
{"name":"kang gam chan", "age":20, "gender":"male", "date": 1663081200000}
{"index":{"_index":"bepoz", "_id":"7"}}
{"name":"kim hwan", "age":22, "gender":"male", "date": 1663092000000}
{"index":{"_index":"bepoz", "_id":"8"}}
{"name":"kim mari", "age":22, "gender":"female", "date": 1663124400000}
{"index":{"_index":"bepoz", "_id":"9"}}
{"name":"lee ju", "age":32, "gender":"female", "date": 1663164000000}
```

<img width="393" alt="image" src="https://user-images.githubusercontent.com/45073750/192099862-8f3fe9fd-3f8f-44bb-a9cb-056e1b06171d.png">

<img width="1098" alt="image" src="https://user-images.githubusercontent.com/45073750/192099879-36c6fa6e-45d5-4fd6-ad10-48f380bb72ff.png">

* name: rollup job의 이름
* source index: source가 될 인덱스, 와일드카드 사용 가능
* target index: source index를 가지고 만들 index 

source index 는 ``bepoz`` , ``target index`` 는 ``bepoz_rollup`` 으로 설정했다. target index 는 존재하지 않은 인덱스를 입력하면 아예 해당 인덱스로 만들어준다.  

<img width="1904" alt="image" src="https://user-images.githubusercontent.com/45073750/192099945-7c4325f9-45f7-4291-b891-d36b24842480.png">

* timestamp field: date 타입의 필드가 들어가야하고 해당 필드를 기준으로 date histogram을 사용하므로 date 타입의 필드가 반드시 있어야 한다.
* interval은 해당 histogram의 최소 단위를 지정하는 것인데, 1시간으로 해두면 1시간 이상의 단위로 date histogram이 가능하다. 더 작은 단위로는 불가
* 밑의 aggregation과 metrics는 rollup job이 실행되면서 source target을 가지고 해당 설정으로 background 에서 집계를 하게된다. 이곳에 설정해둔 값으로만 이용이 가능하므로 사용할 집계 필드들은 다 추가해두어야 한다. cardinality가 작은 값 부터 추가해야 성능에 좋다고 한다.  ``bepoz`` 인덱스의 모든 필드를 추가해 주었다.

source index에 scripted field가 정의되어 있다고해도 이것은 실제로 인덱싱된 필드가 아니기에 사용할 수가 없다.  

<img width="649" alt="image" src="https://user-images.githubusercontent.com/45073750/192100110-3ea955b7-8b47-4741-8a4c-b0bbbc5e9b36.png">

이건 해당 job의 스케줄 단위를 설정하는 것이다. continuous는 계속 job 돌릴 것인지 여부, rollup interval은 job의 스케줄 주기이다. 1분으로 설정해두었다면 1분마다 해당 job이 돌아갈 것이다. page와 execution은 사진 속 영어 설명을 참조하자.  

이제 만들어진 rollup을 이용해 visualization을 만들어보자. 먼저 인덱스 패턴이 추가가 되어야 한다.  
Stack Management > Index patterns > Create index pattern 에서 추가하자.  

<img width="1507" alt="image" src="https://user-images.githubusercontent.com/45073750/192100312-609a99fa-b706-4355-b6fe-2d12970068c4.png">

유의해야 할 점은 타임필드 정하는 곳에서 ``I dont' want...`` 으로 설정해야한다. 이렇게 설정해야 만들어진 rollup의 count를 읽을 수가 있다. 

```json
GET bepoz/_search
{
  "query": {
    "term": {
      "name": {
        "value": "kang seung yoon"
      }
    }
  }
}

GET bepoz_rollup/_search
{
  "query": {
    "term": {
      "name": {
        "value": "kang seung yoon"
      }
    }
  }
}
```

일반적인 인덱스 조회인 전자인 경우 ``name`` 필드에 대해 term 쿼리가 가능한데,  
후자는 size를 명시적으로 0으로 설정하라는 오류 메세지가 뜨게된다. 아마 내부적으로 field:value 방식이 아니라 background에서 집계를 하여 저장하기 때문이라고 추측하고 있다. 따라서 반드시 aggs 집계와 size: 0을 같이 사용해야만 한다.  

visualize에 들어가서 table 형식으로 하나 만들어보겠다.  

<img width="753" alt="image" src="https://user-images.githubusercontent.com/45073750/192100996-65ec6280-eb66-47dc-bf71-990ba0703b5a.png">

Add를 통해 sub aggregation 을 걸 수도 있다.  

<img width="1314" alt="image" src="https://user-images.githubusercontent.com/45073750/192100999-d8f56fb6-026f-4151-a902-120ca68f3758.png">

일반적인 인덱스로 visualize 를 그리는 거였으면 Buckets의 Advanced 탭에서 json input을 이용해 필드를 조작할 수 있었을 것이다.  

<img width="672" alt="image" src="https://user-images.githubusercontent.com/45073750/192101074-bba43e20-476d-4aa6-b490-6697c1d7bd9b.png">

위의 input은 일반 인덱스였으면 아래와 같이 동작했을 것이다.  

```json
GET bepoz/_search
{
  "size": 0,
  "aggs": {
    "names": {
      "terms": {
        //field, size 생략
        "script": "doc['name'].value + '-wow'"
      }
    }
  }
}

/*
      "buckets" : [
        {
          "key" : "kang gam chan-wow",
          "doc_count" : 1
        },
        {
          "key" : "kang seung yoon-wow",
          "doc_count" : 1
        },
...
```

하지만, rollup인 경우 이미 만들 때에 특정 필드로 aggregation 하게끔 설정해놓았기 때문에 aggregation 내부에 script를 사용할 수가 없다. 그리고 rollup 인덱스 내부에 scripted field 또한 정의할 수가 없다. 

이게 참 아쉬운 점인 만약 내가 name 필드를 공백마다 나눠서 해당 count를 재고 싶다고 해보자  

<img width="621" alt="image" src="https://user-images.githubusercontent.com/45073750/192101527-42036515-bdfd-4af3-b97c-76e98edb33db.png">

이렇게 말이다. 이렇게 하려면 name의 term aggregation 내부에서 script 를 이용해서 할 수가 있는데,

```json
GET bepoz/_search
{
  "size": 0,
  "aggs": {
    "names": {
      "terms": {
        "script": "doc['name'].value.splitOnToken(' ')"
      }
    }
  }
}
```

이렇게 말이다. 근데 이런 field의 value 조작을 할 수가 없다는 단점이 있다.  

visualize 내에서 query 를 하려면  

<img width="522" alt="image" src="https://user-images.githubusercontent.com/45073750/192101569-9a02cdf8-deea-429e-9676-b9cfa39c6609.png">

이쪽에서 주어진 필드로 필터를 걸거나 ``Edit as Query DSL`` 로 상세하게 내가 짤 수도 있다.  

<img width="377" alt="image" src="https://user-images.githubusercontent.com/45073750/192101605-514be581-92a5-4f73-9714-64180e9d3530.png">

간단히 위와 같이 해보았다. 그러면 해당 name 만 값이 나오게 된다.  

아쉬운 점은 source index 에서 rollup을 하는 과정에서 나는 모든 source index의 document를 읽는 것이 아니라 일부만 읽고 싶은데 아직 해당 기능이 없는 것 같다. 기존 index 양이 너무 크면 클러스터 성능에 영향을 끼쳐서 date 타입으로 특정 age 이하인 document만 rollup하여 인덱싱 하고 싶은데 말이다.. ㅜ  

그리고 unique count, match 등등 지원을 하지 않은 집계와 query가 꽤나 많다. 엘라스틱서치 내에서도 아직 experimental 한 feature 라고 소개를 할 정도여서 ... 내가 사용하고자 하는 것이 지원해주는 기능이면 쓰겠는데 그런 것이 아니면 어쩔 수 없이 그냥 자바에서 따로 etl 할 수 밖에 없을 것 같다...

---

###  REFERENCE

https://opendistro.github.io/for-elasticsearch-docs/docs/im/index-rollups/rollup-api/

https://www.elastic.co/guide/en/elasticsearch/reference/current/rollup-put-job.html
