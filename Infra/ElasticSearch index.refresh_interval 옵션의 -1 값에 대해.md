# ES index.refresh_interval 옵션의 -1 값에 대해

``GET {index_name}/_settings``을 조회해보면 index의 설정이 나오는데, 이때 ``refresh_interval`` 값이 나오는 경우도 욌고 없는 경우도 있다. 없다면 기본값인 1s 으로 돌아가는 것이다.  

ES는 near-realtime으로 돌아간다. 만약 문서를 색인했으면 검색에서 조회가 되기 위해서는 refresh가 되는 것을 기다려야 한다.  이 때 사용되는 옵션이``refresh_interval`` 다.  

refresh 작업 자체가 비싼 작업이기 때문에 색인 성능을 높이기 위해서는 이 값을 올리는 것이 좋다.  

이 값을 -1로 둔다는 것은 refresh를 비활성화한다는 뜻이기 때문에 추가된 문서가 검색이 되지 않을 수 있다.  

ES는 색인된 문서를 translog에 저장하고 조건이 충족되면 이 데이터가 lucene 세그먼트에 커밋되고 이때 문서가 검색 가능 상태로 전환된다. 예로들면 translog의 크기가 너무 커지거나, 일정 시간이 지나거나, 노드가 재시작될 때 등의 경우 자동으로 커밋이 발생하여 refresh 작업 없이도 문서가 검색 가능 상태가 될 수 있다.  

어쨋든 높은 색인 성능이 필요해서 이 값을 -1로 두었다면 색인이 끝난 뒤 -1 해제를 해주거나 검색을 위해 refresh api를 따로 호출을 하던가 해야한다.  

---

### REFERENCE

https://sematext.com/blog/elasticsearch-refresh-interval-vs-indexing-performance/

https://opster.com/guides/elasticsearch/glossary/elasticsearch-refresh-interval/

