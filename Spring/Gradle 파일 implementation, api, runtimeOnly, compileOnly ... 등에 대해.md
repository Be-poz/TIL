# Gradle 파일 implementation, api, runtimeOnly, compileOnly ... 등에 대해

아무 생각없이 gradle을 사용해왔는데 우테코 미션에 대한 리뷰 중 다음과 같은 것이 달렸다.  

![image](https://user-images.githubusercontent.com/45073750/120920862-eec65b80-c6fb-11eb-80d1-4d1abc05e729.png)

그래서 공부해보았다!  

먼저, ``Classpath``에 대해서 알아야 할 것 같다. ``Classpath``는 클래스나 jar 파일이 존재하는 위치라고 볼 수 있다.  
``Classpath``는 ``Compile-time classpath`` 와 ``Run-time classpath``로 나누어진다.  

``Compile-time classpath``는 에러없이 컴파일을 하기위해 필요한 클래스와 jar들의 위치를 나타낸다. 이렇게 컴파일을 하고 난 뒤에 애플리케이션의 실행이 당연히 될 것이라고 생각할 수 있는데 그것은 오산이다. 런타임에서 필요한 다른 클래스와 jar 가 필요할 수도 있기 때문이다.  

``Run-time classpath``는 애플리케이션이 정상적으로 실행하기위해 필요한 클래스들과 jar들의 경로다.  
Intellij에서 우측의 gradle 탭을 누르면 다음과 같이 나오는 것을 확인할 수가 있다.  

![image](https://user-images.githubusercontent.com/45073750/120924790-5ab2bf00-c710-11eb-8ca9-d6e09bb215f7.png)

선수 지식은 이정도면 될 것 같다. 이제 gradle의 dependency configurations 들을 알아보자.  

<br/>

![image](https://user-images.githubusercontent.com/45073750/120924556-43270680-c70f-11eb-9af3-ac5444ab2807.png)

test를 제외한 대표적인 종류는 다음과 같다. gradle 3.0 이후 위와 같이 변한 것인데 무엇이 변했는지 간략히 말하겠다.  
``compile``이 사라지고 ``implementation``과 ``api``로 나뉘었다.  
``testCompile`` -> ``testImplementation``, ``debugCompile`` -> ``debugImplementation``, ``provided`` -> ``compileOnly``  
우선 ``api``와 ``implementation``의 차이점을 설명해보겠다.  

```java
class Me{
  public static void main(String[] args){
    B b = new B();
  }
}

class B {
  public int asdf() {
    return new C().sum();
  }
}

class C {
  public int sum(){
    return 5;
  }
}
```

위와 같이 클래스 A,B,C 가 있다고 하자. 내가 정의한 클래스는 ``Me``이고, 외부 라이브러리 ``B``를 호출한 상황이라고 가정하자.  
Me -> B -> C 의 상황이라고 볼 수 있다. 컴파일을 위해서는(컴파일 타임) 나는 ``Me``와 ``B``에 대한 경로만 가지고 있으면 될 것이다. 그러나 런타임을 위해서는 ``C``에 대한 정보도 가지고 있어야 할 것이다. ``Me``와 ``B``에 대한 컴파일 타임 의존성을 가지고 있는 것이며, ``Me``, ``B``, ``C``에 대해 런타임 의존성을 가지고 있는 것이다.  

컴파일 타임 의존성과 런타임 의존성에 대한 추가적인 예시를 들어보겠다.  
위 코드의 예시는 A가 B를 호출하고 B가 C를 호출할 때에, 컴파일 타임 의존성은 A, B이고, 런타임 의존성은 A, B, C 인 것이다.  
만약, A가 ``Class.forName()``으로 ``B``를 호출하는 경우에는 컴파일 시에는 ``B``에 대한 정보가 필요하지 않으므로(컴파일 타임에서는 ``B``의 존재를 모름), 컴파일 타임 의존성은 A 인 것이고, 런타임 의존성은 A, B라고 볼 수 있다.  

만약 ``B``에 대한 dependency를 ``implementation``으로 설정하게 되면 ``compile path``에는 ``B``만 들어가게 된다.  
즉, 컴파일 타임에서 사용자(``Me``를 정의한 이용자)는 ``C``를 알 수가 없다. 그리고 ``C``가 수정되었다고 하더라도 ``Me``를 다시 컴파일 할 필요가 없다. 오직 ``B``만 컴파일을 하면 된다.  

그렇다면 ``api``로 설정하면 어떻게 될까?? ``implementation``과는 다르게 ``compile path``에 ``C``도 들어가게 된다.  
즉, 컴파일 타임에서 사용자는 ``C``를 알고 있는 상태이고, ``C``가 수정이 돼서 재빌드 해야되는 상황이 오면 ``A`` 또한 해주어야 하게 된다.  

그렇다면 ``compileOnly``와 ``runtimeOnly``의 차이점은 무엇일까? 간단히 말하자면 전자는 ``compileClassPath``에만 추가하겠다는 뜻이고, 후자는 ``runtimeClassPath``에만 추가하겠다는 뜻이다.  

이렇게 나누게되면 ``compileOnly``의 경우에는 빌드 결과물의 사이즈가 줄어들 것이고, ``runtimeOnly``의 경우에는 해당 클래스에서 코드 변경이 일어나도 컴파일을 다시 할 필요가 없게될 것이다.  

``compileOnly``의 대표적인 예는 ``lombok``이라고 할 수 있다. 컴파일 시에 ``getter``, ``setter`` 등 필요한 것을 생성시키고, 런타임 때에는 사용하지 않기 때문이다. ``runtimeOnly``는 DB관련이나 로그관련 dependency 이다. 이외에도 위의 코드 예시를 생각해보고 컴파일 타임에는 필요없는데 런타임에서는 의존하는 것들은 ``runtimeOnly``로 설정해줘야 겠다~ 라고 알아두면 된다.  

![image](https://user-images.githubusercontent.com/45073750/120927471-de25dd80-c71b-11eb-9785-88fac945ab8e.png)

정리하자면 위의 표와 같다. test에 관련한 것도 동일한 개념을 띈다.  

![image](https://user-images.githubusercontent.com/45073750/120927511-0281ba00-c71c-11eb-82ad-b5c42be83f36.png)

애매하면 ``implementation``으로 만사해결이다!!  
그렇다면 다시 한 번 첫 번째 사진을 살펴보자.  

![image](https://user-images.githubusercontent.com/45073750/120924556-43270680-c70f-11eb-9af3-ac5444ab2807.png)

이제 좀 이해가 갈 것이다. ``implementation`` 은 ``compile path``, ``runtime path`` 에 둘 다 들어가게 되니 저렇게 화살표를 다 받고 있는 것이고, ``compileOnly``와 ``runtimeOnly`` 는 각각 1개에만 속하니 화살표를 저렇게 받고 있는 것이다. 지금까지 배운 내용을 통해 ``compileOnlyApi``와 ``compileOnly``의 차이점 또한 유추할 수 있을 것이다.  

***

### Reference

https://docs.gradle.org/current/userguide/java_library_plugin.html#sec:java_library_configurations_graph  

https://www.geeksforgeeks.org/how-is-compiletime-classpath-different-from-runtime-classpath-in-java/  

https://tomgregory.com/gradle-implementation-vs-compile-dependencies/  

https://stackoverflow.com/questions/4270950/compile-time-vs-run-time-dependency-java/4271013  

https://ichi.pro/ko/gradle-jongsogseong-guseong-i-naebueseo-jagdonghaneun-bangsig-256411970261733  

https://stackoverflow.com/questions/44493378/whats-the-difference-between-implementation-and-compile-in-gradle