# DATA JPA 사용 시 조심해야할 클래스 명에 대해

![image](https://user-images.githubusercontent.com/45073750/125097037-ab9f4400-e110-11eb-8d72-2f4df5786fde.png)

``Member``라는 엔티티 내부에 다음과 같은 Enum을 가지고 있다.  
![image](https://user-images.githubusercontent.com/45073750/125097138-c376c800-e110-11eb-8139-e66863c2b1cd.png)

그리고 위와 같이 Data Jpa를 사용해 메서드 이름으로 쿼리를 생성하고 있다. 결론부터 말하자면 위의 코드는 컴파일 에러가 발생한다.(Unable to locate Attribute with the the given name)    

![image](https://user-images.githubusercontent.com/45073750/125101359-181c4200-e115-11eb-94b4-c1b162f45889.png)

그 이유는 다음과 같다.  

엔티티가 DDL을 이용하여 테이블을 생성할 때에 따로 설정하지 않은 상태라면 필드명을 통해 컬럼명을 정하게 된다.  
처음에 맞이하는 대문자들은 소문자로 바꾸고 이후에 맞이하는 대문자들은 언더바를 사용해 표시한다.  

```java
private String nickName -> nick_name 으로 생성
private String Nickname -> nickname 으로 생성
private String NickName -> nick_name 으로 생성
private String NIckName -> nick_name 으로 생성
```

보통 필드명은 카멜케이스를 사용하니 ``nickName``과 같이 사용하게 될 것이다.  
그렇다면 위의 ``private OAUth2Type oAuth2Type``은 어떤 값이 컬럼명이 될까? ``o_auth2type``이 컬럼명이 된다.  

대문자를 맞이하는 경우 언더바로 구분을 지어주지만, 만약 숫자가 들어온다면 구분이 된다고 취급해서 언더바를 사용하지 않고 ``2``를 그냥 붙여준 것으로 추측 가능하다. 여기까지는 문제가 없다. 문제는 바로 메서드 이름으로 쿼리를 생성하는 과정에 있다.  

``findByEmailAndOAuth2Type`` 해당 메서드의 쿼리가 어떤식으로 읽히는지 한 번 살펴보겠다. ``find``는 ``select``이 되겠고, ``By``를 붙여 그 조건을 입력해준다. 위 메서드에서는 ``Email``, ``OAuth2Type``을 조건으로 두었다. ``String email`` 필드를 ``Email``이라고 표현한 것을 보아 조건을 입력할 때에 앞글자를 대문자로 둔다는 추측이 가능하다.  

그렇다면 쿼리의 조건이 아마 다음과 같이 나가게 될 것이다.  ``where email =? and oauth2type = ?``  
문제점을 깨달았나?! ``Member``에서 필드로 ``private OAuth2Type oAuth2Type``은 DB 컬럼명으로 ``o_auth2type``을 갖고 있는데, 조건에서는 다른 값으로 그것을 찾고 있는 것이다!!  

![image](https://user-images.githubusercontent.com/45073750/125099331-df7b6900-e112-11eb-9bb1-f844689bbde7.png)

이렇게 바꾸면 되는거 아닌가? 그러면 ``o_auth2type``으로 찾지 않을까?? 이 방법은 위에서 언급한대로 메서드 이름으로 쿼리를 생성할 때에 필드의 값은 앞 글자가 대문자가 되어야 하므로 불가능하다.  
인텔리제이에서도 ``Cannot resolve property``를 보여준다.  

![image](https://user-images.githubusercontent.com/45073750/125099621-2cf7d600-e113-11eb-8afb-0c3f53212ea5.png)

이렇게 중간에 언더바가 들어가도 property를 제대로 읽지 못한다.  

![image](https://user-images.githubusercontent.com/45073750/125099716-47ca4a80-e113-11eb-8c57-d4788e1ef3f9.png)

![image](https://user-images.githubusercontent.com/45073750/125099763-557fd000-e113-11eb-9682-d050651a7b38.png)

엔티티 내부 필드명을 ``OAuth2Type``으로 바꾸면 DB 컬럼명이 ``oauth2type``이 될 것이니 위와 같이 변경하면 컴파일 에러가 사라진다. 하지만, 이 방법은 일반적으로 변수명을 카멜케이스로 쓰기 때문에 상당한 혼동을 줄 수 있다고 생각한다.  

가장 납득할만한 방법은 이것이라고 생각해서 다음과 같이 변경해주었다.  

![image](https://user-images.githubusercontent.com/45073750/125100382-fd959900-e113-11eb-907a-823ca2db3e20.png)
![image](https://user-images.githubusercontent.com/45073750/125100395-01292000-e114-11eb-9b7c-78302f1ec6bd.png)

클래스의 명의 두 번째 글자에 대문자를 넣어주지 않는 것이다. 정말 예상하지도 못한 오류였다. 개인적으로 충분히 겪을 수 있는 문제라고 생각하고 조심해야 될 것 같다!  

***