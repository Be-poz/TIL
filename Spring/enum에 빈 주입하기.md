# enum에 빈 주입하기

Enum에는 빈을 어떤식으로 주입할까?? 처음에는 다음과 같은 착각을 했었다.  

![image](https://user-images.githubusercontent.com/45073750/125093425-3e3de400-e10d-11eb-97b8-f91edd8e736a.png)

처음에는 위와 같이 착각을 했다. 빈 타입을 필드로 가지고 있으면 무조건 생성자에 적어줘야 된다는 착각 말이다. Enum을 위와 같은 방식으로만 사용을 해왔기 때문에 일어난 착각이었다. 굳이 생성자에 넣지 않아도 되고 위와 같이 넣는다면 빈으로 등록되어있는 ``GoogleOAuth2``를 가져와서 주입해주는 것이 아니라 새로 생성하는 것이 될 것이다.  

![image](https://user-images.githubusercontent.com/45073750/125093998-dcca4500-e10d-11eb-89e0-f8086a8cb630.png)

결국 필드로만 갖게하고 생성자에는 넣지 않았다. 그렇다면 어떻게 이 값을 주입해주어야 할까?  
그 방법은 바로 주입을 위한 클래스를 만들어주는 것이다. Enum 내부에 만들어줘도 되지만 나는 따로 클래스를 생성해주었다.  

![image-20210709233324739](/Users/kangseungyoon/Library/Application Support/typora-user-images/image-20210709233324739.png)

위와 같은 클래스를 정의해 주었다. Enum에서 공통의 빈을 사용한다면 필드도 하나만 주입받고 그것을 Enum에 set 해주면 되었을 것이지만, 나의 상황은 각기 다른 추상화 되어있는 빈 들을 주입해주어야 하기 때문에 위와 같은 방식을 취하게 되었다.  

이제 ``OAuth2Type`` Enum에는 내가 원하는 빈이 필드로 주입되었다. 위 방식의 단점은 해당 주입받은 빈 필드에 ``final`` 키워드를 사용하지 못한다는 점이다.  

***

### REFERENCE

https://stackoverflow.com/questions/16318454/inject-bean-into-enum/39097926