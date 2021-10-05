# 간략히 알아보는 FlyWay 적용법

## FlyWay?

* DB Migration Tool

<br/>

## 도입하게된 이유는? 

* 진행 중인 프로젝트에서 ``spring.jpa.hibernate.ddl-auto`` 속성을 ``update``을 이용했었는데 운영서버와 개발서버 간의 문제가 간간히 발생했고, ``create`` 속성으로 두기에도 데이터가 계속해서 날라가는 단점이 있었다. ``validate`` 으로 두고 진행하니, 운영서버와 개발서버를 나누고 있는 상황에서 한 쪽에만 DDL, DML을 적용되는 실수가 발생할 수 있기에 이를 쉽게 관리하기 위해 사용했다. 또한, 버전 기록이 남아있어 변경 내역을 확인할 수가 있다는 장점이 있다. 

<br/>

## 적용법

1. build.gradle 의존성 추가

   ```yaml
   dependencies {
     implementation 'org.flywaydb:flyway-core:6.4.2'
   }
   ```

2. application.yml / properties에 설정값 추가

   ```yaml
   spring:
     flyway:
       baseline-on-migrate: true
       baseline-version: 0
   ```

   ``baseline-on-migrate``  

   default 값은 false 이다. true일 경우 ``flyway_schema_history`` 테이블이 없는 경우 생성해준다.  
   false는 해당 테이블이 기존에 생성되 있어야 한다.  

   ``baseline-version``  

   default 값은 1이다. 밑에서 언급할 version의 시작 기준 값을 나타낸다.  

3. sql 파일 작성

   ![image](https://user-images.githubusercontent.com/45073750/135987215-2175ad46-de41-4c7d-9915-b91dca30243f.png)

   ``V1__init.sql`` 에는 테이블 생성 쿼리가, ``V2... .sql`` 에는 컬럼 이름 변경에 대한 쿼리가 적혀져 있는 상황이다.  

   resources/db/migration/prod 디렉터리 내부에 sql 파일을 작성한다.  
   파일 이름 형식은 ``V{버전 숫자}__{name}.sql`` 이다. 언더바는 반드시 2개가 들어가야한다.  
   위의 설정에서 ``baseline-version`` 을 0으로 두어야 V1 부터 읽는다.  

   DB 내의 ``flyway_schema_history`` 에 적힌 버전을 보고 반영될 버전 오름차순으로 차례대로 적용해준다.  

<br/>

DB 별로 스크립트가 다를 수 있으니 syntax error에 유의해야 한다.  
``spring.jpa.hibernate.ddl-auto`` 는 ``validate`` 으로 두자.  

***