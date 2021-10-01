# application.yml의 Profile에 대한 테스트 이것저것

Profile에 대한 개념이 있다는 가정하에 진행하겠다. 모른다면 [!링크](https://bepoz-study-diary.tistory.com/371)를 참고하자.  

``application-{프로파일}.yml`` 은 특정 Profile에 맞는 yml 파일이다. Profile을 따로 지정하지 않고 실행하게 된다면 ``application.yml``이 실행이 된다. 그렇다면 이때의 Profile은 무엇으로 실행되는 것일까??  

![image](https://user-images.githubusercontent.com/45073750/135568239-5e406e9a-1865-4372-9887-f62dedcc8f23.png)

[Baeldung 싸이트](https://www.baeldung.com/spring-yaml) 피셜로 'default' Profile 이라고 한다.  
그렇다면 Profile을 지정하지 않고 돌렸는데 ``application.yml`` 이 없다면 어떻게 될까??  

```yaml
//application.yml
person:
  name: defaultBepoz
  age: 100
  
//application-bepoz.yml
person:
  name: bepozBepoz
  weight: 1000
```

yml 파일은 다음과 같은 2 종류를 세팅해놨고 일단 ``application.yml``을 지운채 진행해보겠다.  

![image](https://user-images.githubusercontent.com/45073750/135566829-8e4698db-fea7-4d40-bc2c-0aa91311daea.png)

yml 파일에서 person.name을 읽어오는 config 클래스를 추가하고 밑의 테스트를 돌려보았다.  

![image](https://user-images.githubusercontent.com/45073750/135566914-80b9cf79-c482-4a65-982c-ddb6a5383174.png)

![image](https://user-images.githubusercontent.com/45073750/135566973-3d3f8a07-a6b9-4f00-a571-4ebc5bf14748.png)

결과는 예상한대로 나왔다. yml 파일을 제대로 못읽어와서 빈 생성에서 실패한 것을 확인할 수가 있었다.  
``application.yml`` 를 추가하고 돌리니 에러 없이 제대로 실행이 되는 것 또한 확인할 수가 있었다.  

이번에는 config 클래스를 다음과 같이 변경해보았다.  
![image](https://user-images.githubusercontent.com/45073750/135567170-913c04d0-1db1-4c96-94db-a39d1c85629a.png)

테스트 코드는 다음과 같이 변경했다.  
![image](https://user-images.githubusercontent.com/45073750/135567294-70520f41-4c07-4d11-92a2-e97dd77716b6.png)

결과는 위에서 나왔던 동일한 에러가 나왔다. ``application.yml``에는 ``weight``에 대한 정보가 없기 때문에 빈 생성에 문제가 생기는 것이다. 그렇다면 테스트의 프로파일을 bepoz로 준다면 어떻게 될까??  
![image](https://user-images.githubusercontent.com/45073750/135567519-0c784ac1-b516-4f95-bdd3-bbb259f1edc5.png)

테스트가 성공적으로 통과했다! ``application-bepoz.yml`` 에는 분명 ``age``에 대한 정보가 없는데도 말이다.  
이를 통해, ``application.yml``은 profile이 ``default``인데, 정말 default 처럼 작동한다는 것을 확인할 수가 있었다.  
``application-bepoz.yml``에서 값을 찾지 못하자 ``application.yml``에서 값을 뒤져서 갖고오는 것이다. 정말 Default 그 자체다!!  

모든 Profile에 공통적으로 들어가는 내용은 따로 profile yml에 넣지말고 ``application.yml``에 넣어줘도 의도한대로 작동할 것이다.  

***