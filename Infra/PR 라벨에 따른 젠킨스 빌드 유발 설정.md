# PR 라벨에 따른 젠킨스 빌드 유발 설정

![image](https://user-images.githubusercontent.com/45073750/128036204-692f7e6a-67c9-44ca-9ab1-01f14c953374.png)

현재 진행중인 프로젝트의 main 브랜치는 다음과 같이 client와 server 디렉토리로 나뉘게 된다. 각각 프론트와 백이다.  

만약 단순히 해당 브랜치가 push 될 때에 빌드 유발을 하게끔 설정해놓는다면 client 디렉토리 내부의 코드가 변경이 생겨서 push 된다고 했을 때에 server를 담당하고 있는 젠킨스 아이템 또한 빌드 유발이 되어 불필요한 CI/CD가 일어나게 될 것이다.  

main 브랜치는 직접적인 푸쉬가 제한되고 오직 머지를 통해서만 변경이 가능하다는 가정하에, PR 라벨을 통한 효율적인 젠킨스 빌드 유발 설정에 대해 알아보겠다. 기본적인 젠킨스 사용법을 안다는 가정하에 진행하겠다.  

![image](https://user-images.githubusercontent.com/45073750/128036793-b00a6eb1-9a69-4bc0-b0df-50dda355c03f.png)

보통 빌드 유발 설정 시에 위의 사진에 해당하는 ``GitHub hook trigger for GITScm polling``을 많이 선택할 것이다.  
이 옵션은 깃허브로 부터 PUSH 훅을 받았을 때에 빌드가 유발이 되는 옵션이다. 즉, 위에서 언급한 문제점을 야기하는 옵션이다.  

![image](https://user-images.githubusercontent.com/45073750/128037133-9f217310-edf0-4774-8a50-49e6fcf3d2fd.png)

이 옵션을 대신 선택해주자.  
만약 이 옵션이 안보인다면 플러그인에 들어가서 ``Generic Webhook Trigger``를 설치해주자.  

![image](https://user-images.githubusercontent.com/45073750/128037811-a53e7789-0ba3-4994-905f-36358232a692.png)

![image](https://user-images.githubusercontent.com/45073750/128037927-394d377c-4c1f-4d97-8d3e-1c590f90ef4d.png)

설치해주면 옵션이 생겼을 것이다. 젠킨스를 사용했었다면 내가 사용하는 Repository의 setting/Webhooks에서 다음과 같이 설정을 해줬었을 것이다.  

![image](https://user-images.githubusercontent.com/45073750/128038448-c5d0e5cc-d72d-4bdf-a792-c7591f3332e4.png)

이제는 ``Generic Webhook Trigger``를 사용하게 되었으므로 설정이 조금 바뀐다.  
![image](https://user-images.githubusercontent.com/45073750/128038607-5496b3b6-bec7-4052-9314-824b63cb25ca.png)

젠킨스에도 잘 나와있는데 ``github-webhook/``대신 ``generic-webhook-trigger/invoke``로 바꿔주자.   

``Content Type``는 ``application/json``으로 설정해두자.  
``Generic Webhook Trigger``를 사용하는 이유를 보기 이전에 Webhook이 어떤 것들을 날려주는지 먼저 확인해보자.  
일단 깃허브의 웹훅 설정에서 ``Send me everything.``으로 바꿔주자. 이 설정은 웹훅을 날리는 기준을 정하는 것이다. ``Push``를 체크한다면 푸쉬에 대한 웹훅만 날리게 되는 반면, 밑의 설정은 모든 것을 날린다. PR의 open, closed, branch 로의 push, pr의 라벨 추가에 대한 이벤트까지 전부를 말이다.  ![image](https://user-images.githubusercontent.com/45073750/128038795-18acc465-322d-4ce8-87ce-9ce66045b88c.png)

현재 보고 있는 화면의 위쪽 탭을 보면 ``Recent Deliveries``라는 탭이 있을 것이다.  

![image](https://user-images.githubusercontent.com/45073750/128039677-7fa53b5e-a030-4544-b3ff-9d2fd8344e23.png)

이 곳을 보면 웹훅을 날린 정보들을 확인할 수가 있을 것이다. 우리는 바로 이 정보들을 가지고 조건을 걸어서 빌드 유발을 시켜줄 수가 있다. merged가 close되고, branch 명은 main이며 라벨 정보가 내가 선택한 라벨일 경우에 빌드 유발이 되게끔 해보겠다.  

<img width="626" alt="스크린샷 2021-08-04 오전 12 13 41" src="https://user-images.githubusercontent.com/45073750/128041038-cf24f773-db24-43b9-83e4-0a67bf8c7b01.png">

이 Payload에 적혀있는 값 들을 한 번 살펴보자.  
나는 jsonpath.com에 이 값 들을 가져다 놓은 후 필요한 필드들을 추출하는 작업을 해보았다.  

![image](https://user-images.githubusercontent.com/45073750/128041365-7aa90282-3a6e-4c58-b2b4-7e116554c9cf.png)

``$.pull_request.merged``에는 해당 PR이 merged 되었는지를 나타낸다. 결과는 true인 것을 확인할 수가 있었다.  
``$.pull_request.base.ref``는 내가 머지의 도착(?) 브랜치다. 나 같은 경우에는 main이다.  
``$.pull_request.labels``를 하게되면 해당 PR에 달아 놓은 라벨들이 모두 나오게 된다. JSONPath 문법에 따라서 ``$.pull_request.labels..name`` or ``$.pull_request.labels.[]name``을 하게되면 해당 라벨들의 ``name``필드들이 모두 나오게 된다. 이렇게 뽑아낸 필드들을 젠킨스에서 변수로 설정을 해주면된다.  

![image](https://user-images.githubusercontent.com/45073750/128041857-3ec8cd27-5a80-4877-b8e5-bc9d64cd2d72.png)

``Post content parameters`` 추가를 누르고 변수명을 기입하고 JSONPath Expression을 기입한다.  

![image-20210804002301255](/Users/kangseungyoon/Library/Application Support/typora-user-images/image-20210804002301255.png)

그리고 스크롤을 밑으로 내려서 나오는 ``Optional filter`` 탭에서 Expression에는 정규표현식을, Text에는 변수들을 작성한다. 나는 머지가 됐고, 브랜치가 main이며 라벨의 이름이 server이면 빌드가 유발되게끔 해놓은 것이다.  

regex.com에서 정규표현식을 작성하기가 수월해서 이곳에서 테스트해봤다.  

![image](https://user-images.githubusercontent.com/45073750/128042268-386e9349-053b-4b75-8cab-bde9ce810f32.png)

이렇게 말이다. 위의 상황에서는 라벨 값에 server, document 이렇게 2개가 들어온 상황을 연출한 것이고, 그 중 server가 있는지 체크를 한 것이다. 마지막으로 한 가지가 더 남았다.  

<img width="908" alt="스크린샷 2021-08-04 오전 12 25 52" src="https://user-images.githubusercontent.com/45073750/128042512-573acfda-099c-4198-9ca6-c61a5dd4577f.png">

``Generic Webhook Trigger``를 사용하기 위해서는 토큰을 설정해주어야 한다. 토큰을 활용한 무언가에 대해서는 나도 자세히는 잘 모르겠다. 먼저 Add를 눌러서 토큰을 만들어주자.   

![image](https://user-images.githubusercontent.com/45073750/128042870-8b919068-dcf2-4f70-9dca-b73ccc8d0531.png)

위의 ``Secret``에 입력할 값은 이전 캡처의 ``Token`` 쪽에 들어갈 값과 일치시켜야 한다. ``ID``는 본인의 해당 토큰에 대한 ``ID``니깐 ㄴ임의로 작성을 하고 Add를 눌러준다. 그리고 ``Token``필드에 이곳에서 만든 ``Secret``의 값을 적어준다.  

![image](https://user-images.githubusercontent.com/45073750/128043266-0c05cfb4-4a1f-4277-8d58-eedfef829b33.png)

깃허브 웹훅 URL에서 ``/generic-webhook-trigger/invoke``로 적어놨는데 이곳에 ``?token={각자의 토큰 값}``을 추가해주자. 이렇게하면 빌드 유발에 대한 설정이 끝난다.  

이제 PR을 여러 케이스로 날려서 내가 의도한대로 빌드 유발이 되는지 확인해보자.  
웹훅의 내용을 보고 본인이 원하는 조건으로 변경할 수도 있을 것이다. 이 글은 단순히 PR LABEL에 따른 조건을 가정한 것일 뿐이다.  

***