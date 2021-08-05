# MySql Order By 쿼리 ONLY_FULL_GROUP_BY 문제 해결

MySql 에서는 ``Group By`` 사용 시에 필드에 집계함수만을 사용할 수 있게끔 하기 때문에 생긴 문제다.  

``SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));``

입력해주자.  

***

### REFERENCE

https://stackoverflow.com/questions/23921117/disable-only-full-group-by