# Spring Batch Multi Thread Processing 정리

## AsyncItemProcessor, AsyncItemWriter

* Step 안에서 ``ItemProcessor``가 비동기적으로 동작하는 구조
* ``AsyncItemProcessor``와 ``AsyncItemWriter``가 함께 구성이 되어야 함
* ``AsyncItemProcessor``로 부터 ``AsyncItemWriter``가 받는 최종 결과값은 ``List<Future<T>>`` 타입이며 비동기 실행이 완료될 때까지 대기한다
* spring-batch-integration 의존성 필요

<img width="1405" alt="image" src="https://user-images.githubusercontent.com/45073750/210568276-8663c5b7-9386-43a3-9bd6-ddda06bdab00.png">

