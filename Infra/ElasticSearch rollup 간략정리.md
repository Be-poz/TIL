# ElasticSearch rollup 간략정리

특정 정보를 es에 계속해서 쌓고있는 상황이라고 하자, 정보를 계속해서 인덱싱하다보면 용량이 점점 늘어나게 될것이고 이는 리소스 비용의 증가로 이어질 것이다. 이 정보들을 summary/압축하여 가볍게 보관하면서 이전의 데이터를 사용하기 위해 롤업을 사용한다.  

예시로 사용할 인덱스와 도큐먼트 정보는 아래와 같다.

```json
PUT rollupexample-2022.01.01
{
  "mappings": {
    "properties": {
      "updateType": {"type": "keyword"},
      "age": {"type": "long"},
      "date": {"type": "date", "format": "epoch_millis"}
    }
  }
}

POST _bulk
{"index":{"_index":"rollupexample-2022.01.01", "_id":"1"}}
{"updateType":"UPDATE", "age": 10, "date": 1667228400000}			// 00:00:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"2"}}
{"updateType":"UPDATE", "age": 20, "date": 1667230200000}			// 00:30:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"3"}}
{"updateType":"INSERT", "age": 30, "date": 1667231100000}			// 00:45:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"4"}}
{"updateType":"DELETE", "age": 40, "date": 1667275200000}			// 13:00:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"5"}}
{"updateType":"INSERT", "age": 40, "date": 1667278800000}			// 14:00:00
```

이제 롤업을 생성해보겠다. 먼저 키바나에서 제공해주는 툴을 이용해 생성해보겠다.  
es용 오픈소스인 open distro에서 제공하는 롤업이다.  

키바나 탭의 Index Management를 탭을 누른다.  

<img width="393" alt="image" src="https://user-images.githubusercontent.com/45073750/192099862-8f3fe9fd-3f8f-44bb-a9cb-056e1b06171d.png">

누르고 Rollup Jobs 탭에서 Create rollup job 버튼을 누르면 다음과 같이 job name을 정하게 된다.  

<img width="1098" alt="image" src="https://user-images.githubusercontent.com/45073750/192099879-36c6fa6e-45d5-4fd6-ad10-48f380bb72ff.png">

* name: rollup job의 이름
* source index: source가 될 인덱스, 와일드카드 사용 가능
* target index: source index를 가지고 생성할 롤업 index

source index는 ``rollupexample-2022.01.01`` , target index는 ``example-summary`` 로 설정했다.  
target index에 입력한 인덱스가 존재하지 않다면 자동으로 생성해준다.  

<img width="1904" alt="image" src="https://user-images.githubusercontent.com/45073750/192099945-7c4325f9-45f7-4291-b891-d36b24842480.png">

next를 누르면 다음과 같은 화면을 마주하게된다.  

* timestamp field: date 타입의 필드로 선택가능하며, 해당 필드를 기준으로 시간계산을 하게된다.
* interval: 어느정도의 interval 단위로 summary를 해서 인덱싱할 것인지를 선택하는 것이다. 그렇기 때문에 만약 1시간으로 지정해둔다면 생성된 롤업 인덱스에 대해 1시간 이상 단위로의 데이터 조회를 할 수 있지만 그 미만의 단위로는 불가능하다.
* aggregation / metrics: 롤업을 진행하면서 source target이 가지고 있는 정보를 어떤 형태로 롤업할 것인지를 정하는 것이다. 이용하고자 하는 정보는 이곳에 다 추가해야 한다. 롤업이 생성된 후에 추가가 불가능하다. 그리고 cardinality가 작은 값 부터 추가해야 성능에 좋다고 한다. metrics는 numeric type만 추가할 수가 있다.

<img width="2052" alt="image" src="https://user-images.githubusercontent.com/45073750/200254414-1bda48c6-4fdf-44ba-ae9f-ded09d50dfbd.png">

일단 위와 같이 설정하고 넘어갔다.  

<img width="649" alt="image" src="https://user-images.githubusercontent.com/45073750/192100110-3ea955b7-8b47-4741-8a4c-b0bbbc5e9b36.png">

* Continuous: 생성한 롤업을 계속해서 돌릴 것인지 또는 단기성으로 돌릴 것인지를 선택
* Rollup interval: 어느 주기로 롤업을 돌릴 것인지를 정하는 것이다. 즉 스케줄링 주기를 의미한다. 내가 앞서 설정한 time stamp field의 interval보다 작은 것은 의미가 없으므로 그 이상 값으로 설정하자
* Page per execution: 설명에 나외있듯이 실행시에 처리되는 페이지의 값이고 이 값이 클수록 빠르지만 메모리를 비용이 높아진다고 한다
* Execution delay: 롤업이 실행된 뒤 딜레이 시간을 주는 것이다

<br/>

생성하게되면 target index에 명시해두었던 인덱스명으로 인덱스가 생성된다.  

```json
GET example-summary/_search
{
  "size": 0,
  "aggs": {
    "types": {
      "terms": {
        "field": "updateType",
        "size": 10
      }
    }
  }
}

// result
"buckets" : [
  {
    "key" : "INSERT",
    "doc_count" : 2
  },
  {
    "key" : "UPDATE",
    "doc_count" : 1
  },
  {
    "key" : "DELETE",
    "doc_count" : 1
  }
]
```

설정을 할 때에 updateType 필드에 대한 terms를 추가해두었기 때문에 해당 api 요청을 보낼 수 있었다.  
그런데 updateType UPDATE의 count가 2인 것을 확인할 수 있는데, 도큐먼트에서는 update가 2개였다. 그런데 1로 된 이유는 date interval을 1시간으로 잡아두었기 때문에 1시간 동안의 내용을 요약해서 롤업하기 때문이다.  

00:00:00 ~ 00:59:99 까지의 내용을 요약하면 update 가 있었고 insert가 있었기 때문에 각 1건씩으로 처리된 것이다.  
그리고 13:00:00 에 delete 1건, 14:00:00에 insert 1건 이렇게 insert가 2건으로 표기된 것이다.  

1시간 동안 updateType이 몇 건이 들어갔는지가 일반적으로 중요한데 위의 결과에서는 그렇지 않다. 이를 위한 방법은 뒤에서 설명하겠다.  

```json
GET example-summary/_search
{
  "size": 0,
  "aggs": {
    "types": {
      "terms": {
        "field": "updateType"
      },
      "aggs": {
        "sumAge": {
          "sum": {
            "field": "age"
          }
        }
      }
    }
  }
}

// result
"buckets" : [
  {
    "key" : "INSERT",
    "doc_count" : 2,
    "sumAge" : {
      "value" : 70.0
    }
  },
  {
    "key" : "DELETE",
    "doc_count" : 1,
    "sumAge" : {
      "value" : 40.0
    }
  },
  {
    "key" : "UPDATE",
    "doc_count" : 1,
    "sumAge" : {
      "value" : 30.0
    }
  }
]
```

위와 같이 metrics에 추가해둔 age를 이용할 수도 있다.  

<br/>

위에서 언급한 updateType이 1시간 동안 몇 건이 들어갔는지가 중요한 경우 확인할 수 있는 방법이 있다.  
롤업을 생성할 때에 metrics는 numeric type의 필드만 설정이 가능했었다. 

<img width="2374" alt="image" src="https://user-images.githubusercontent.com/45073750/200259513-1843ac30-d206-4df6-824d-d6edc2e3a6b2.png">

그런데 여기를 보면 value count라는 값이 있다. 바로 이 value count를 이용해 조회를 할 수가 있다. 문제는 updateType은 keyword type이기에 이곳에서 추가할 수가 없다. 하지만, api를 따로 호출해서 롤업을 생성하는 경우에는 추가해줄 수가 있다.  

```json
PUT _opendistro/_rollup/jobs/rollupExample
{
  "rollup": {
    "enabled": true,
    "schedule": {
      "interval": {
        "period": 1,
        "unit": "MINUTES",
        "start_time": 1665680400000
      }
    },
    "last_updated_time": 1665633600000,
    "description": "rollup example",
    "source_index": "rollupexample-2022.01.01",
    "target_index": "example-summary",
    "page_size": 1000,
    "delay": 0,
    "continuous": true,
    "dimensions": [
      {
        "date_histogram": {
          "source_field": "date",
          "target_field": "date",
          "fixed_interval": "60m"
        }
      },
      {
        "terms": {
          "source_field": "updateType",
          "target_field": "updateType"
        }
      }
    ],
    "metrics": [
      {
        "source_field": "updateType",
        "metrics": [
          {
            "value_count": {}
          }
        ]
      }
    ]
  }
}
```

open distro가 지원하는 롤업을 사용했기에 api 또한 open distro의 방식을 따랐다. [관련링크](https://opendistro.github.io/for-elasticsearch-docs/docs/im/index-rollups/)  

metrics에 updateType의 value_count를 추가했다. 그리고 아래와 같이 조회를 하면 원하는대로 조회를 할 수 있게된다.  

```json
GET example-summary/_search
{
  "size": 0,
  "aggs": {
    "types": {
      "terms": {
        "field": "updateType",
        "size": 10
      },
      "aggs": {
        "total_count": {
          "value_count": {
            "field": "updateType"
          }
        }
      }
    }
  }
}

// result
"buckets" : [
  {
    "key" : "INSERT",
    "doc_count" : 2,
    "total_count" : {
      "value" : 2
    }
  },
  {
    "key" : "DELETE",
    "doc_count" : 1,
    "total_count" : {
      "value" : 1
    }
  },
  {
    "key" : "UPDATE",
    "doc_count" : 1,
    "total_count" : {
      "value" : 2
    }
  }
]
```

UPDATE의 total_count가 2로 찍힌 것을 확인할 수가 있다. 이제 continuous 하게 돌아가는 롤업을 생성해보겠다.  
아래와 같은 예시를 사용하겠다. 

```json
PUT rollupexample-2022.01.01
{
  "mappings": {
    "properties": {
      "updateType": {"type": "keyword"},
      "date": {"type": "date", "format": "epoch_millis"}
    }
  }
}

POST _bulk
{"index":{"_index":"rollupexample-2022.01.01", "_id":"1"}}
{"updateType":"UPDATE", "date": 1667228400000}							//00:00:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"2"}}
{"updateType":"UPDATE", "date": 1667228460000}							//00:01:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"3"}}
{"updateType":"INSERT", "date": 1667228520000}							//00:02:00
{"index":{"_index":"rollupexample-2022.01.01", "_id":"4"}}
{"updateType":"DELETE", "date": 1667228530000}							//00:02:10
{"index":{"_index":"rollupexample-2022.01.01", "_id":"5"}}
{"updateType":"INSERT", "date": 1667228540000}							//00:02:20
```

롤업을 돌리게되면,  아래와 같이 current rollup window에 현재 시각이 찍히게 되는데, 롤업을 처음 돌리면 source index에서 도큐먼트들을 읽어 설정한 date와 aggregation에 맞게끔 정리를 하게된다.

<img width="2363" alt="image" src="https://user-images.githubusercontent.com/45073750/200263473-18286a75-e46b-428f-a3b5-485aebd86796.png">

이 상황에서 해당 window 시각보다 이전의 데이터를 집어넣게되면 반영이 안되고 미래의 시각을 넣게되면 해당 시각의 window가 될 때에 반영이 되어서 롤업인덱스에 인덱싱된다.  

<br/>

그렇다면 다른 롤업을 생성해서 source index는 다르지만 target index는 기존의 다른 롤업과 겹치게끔 생성하면 어떻게 될까?  

```json
PUT rollupexample2-2022.01.01
{
  "mappings": {
    "properties": {
      "updateType": {"type": "keyword"},
      "date": {"type": "date", "format": "epoch_millis"}
    }
  }
}

POST _bulk
{"index":{"_index":"rollupexample2-2022.01.01", "_id":"1"}}
{"updateType":"UPDATE", "date": 1667228580000}							// 00:03:00
{"index":{"_index":"rollupexample2-2022.01.01", "_id":"2"}}
{"updateType":"UPDATE", "date": 1667228640000}							// 00:04:00
{"index":{"_index":"rollupexample2-2022.01.01", "_id":"3"}}
{"updateType":"INSERT", "date": 1667228700000}							// 00:05:00
{"index":{"_index":"rollupexample2-2022.01.01", "_id":"4"}}
{"updateType":"DELETE", "date": 1667228760000}							// 00:06:00
{"index":{"_index":"rollupexample2-2022.01.01", "_id":"5"}}
{"updateType":"INSERT", "date": 1667228820000}							// 00:07:00
```

rollupexample2 라는 것을 만들고 example-summary를  target index로 가지는 롤업을 생성하고 돌려보겠다.  

```json
GET example-summary/_search
{
  "size":0, 
  "aggs": {
    "types": {
      "terms": {
        "field": "updateType",
        "size": 10
      }
    }
  }
}

// result
"buckets" : [
  {
    "key" : "INSERT",
    "doc_count" : 4
  },
  {
    "key" : "UPDATE",
    "doc_count" : 4
  },
  {
    "key" : "DELETE",
    "doc_count" : 3
  }
]


GET example-summary/_search
{
  "size":0, 
  "query": {
    "range": {
      "date": {
        "gte": 1667228400000,
        "lt": 1667228880000
      }
    }
  },
  "aggs": {
    "types": {
      "terms": {
        "field": "updateType",
        "size": 10
      }
    }
  }
}

// result
"buckets" : [
  {
    "key" : "INSERT",
    "doc_count" : 2
  },
  {
    "key" : "UPDATE",
    "doc_count" : 2
  },
  {
    "key" : "DELETE",
    "doc_count" : 1
  }
]
```

생성을 하고 날린 api를 보면 updateType의 terms 자체는 잘 분류되어서 들어간 것을 확인할 수가 있지만, date는 고려되지 않은 것을 알 수가 있다. 그러니 각 롤업 job마다 별도의 target index를 두는 것이 바람직할 것이다. 

<br/>

### REFERENCE

https://www.youtube.com/watch?v=I5-9x_pQ-Y0

https://opendistro.github.io/for-elasticsearch-docs/docs/im/index-rollups/  

https://www.elastic.co/guide/en/elasticsearch/reference/current/rollup-overview.html
