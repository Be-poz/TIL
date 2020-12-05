# String 객체와 String 리터럴에 대해

```java
String str1 = new String("abc");
String str2 = "abc";
```

String은 다음과 같은 방법으로 선언할 수 있다. 그렇다면 이 두 방법의 차이점은 과연 무엇일까??
``String str1 = new String("abc")`` 는 new 연산자를 이용한 생성법이고,  
``String str2 = "abc";`` 는 문자열 리터럴 생성법이다.  

new 연산자는 메모리의 ``heap`` 영역에 할당되고 리터럴 방식은 ``String Constant Pool`` 영역에 할당된다.  

> String Constant Pool 영역은 Java Heap Memory 내에 문자열 리터럴을 저장한 공간이며 HashMap으로 구현되어 있다. 한 번 생성된 문자열 리터럴은 변경될 수 없으며 리터럴로 문자열을 생성하면 내부적으로 String.intern() 이 호출된다. 
>
> String Pool에 같은 값이 있는지 찾고 있다면 그 참조값을 반환, 없다면 pool에 문자열을 등록 후 해당 참조값이 반환되는 형태이다.  
>
> 원래 이 pool 은 heap 영역이 아니라 Perm 영역에 존재했었는데 Perm 영역은 실행 시간(Runtime)에 가변적으로 변경할 수 없는 고정된 사이즈이기 때문에 intern 메서드의 호출 시에 저장 공간이 부족할 수 있어 OOM(Out Of Memory)가 발생할 수 있어 변경되었다. (tmi. Java 8부터 Perm 영역은 사라졌고 MetaSpace 영역이 그 자리를 대신하게 되었다)
>
> 어쨌거나 heap 영역으로 변경되었기 때문에 pool 의 모든 문자열도 GC의 대상이 되었다.

![stringpool](https://user-images.githubusercontent.com/45073750/101246499-22c5c180-3757-11eb-938a-61ccca6e0c38.png)

위의 그림과 같이 동작한다.  

```java
        String str = "bepoz";
        String str2 = "bepoz";
        String str3 = new String("bepoz");
        System.out.println(str==str2);	//true
        System.out.println(str==str3);	//false
```

그렇기에 주소 값을 비교하면 당연히 위와 같은 결과가 나오게 된다.  
str 와 str2는 ``String Constant Pool``에서 같은 참조 값을 가지고 오기 때문에!  

리터럴은 내부적으로 ``inter()`` 메서드를 호출한다.  
``intern()`` 메서드는 해당 문자열을 pool 에서 찾고 있으면 해당 참조 값을 가지고오고 그렇지 않다면 새로 집어넣고 해당 주소 값을 반환해준다.  

```java
        String str = "bepoz";
        String str2 = "bepoz";
        String str3 = new String("bepoz").intern();
        System.out.println(str==str2);	//true
        System.out.println(str==str3);	//true
```

아까 전의 코드에서 new 연산자 뒤에 ``intern()`` 메서드만 붙여주었다. 그랬더니 false의 결과를 가진 값이 true로 바뀐 것을 확인할 수가 있다. 왜냐하면 ``intern()`` 메서드로 인해 ``String Constant Pool`` 에서 해당 문자열을 찾고 그 참조 값을 가지고 왔기 때문이다.  

이렇게 String 객체와 String 리터럴에 대해 알아보았다. 이 내용을 통해서 String 이 왜 불변 객체로 만들어 졌는지에 대해 조금 더 이해할 수 있었던 것 같다. ``String Constant Pool`` 에서 같은 문자열 리터럴을 참조하기 위해서는 불변해야 서로 영향이 없을 테니까 말이다.  

불변 객체에 대한 개념이 흐릿하다면! [Immutable Object(불변 객체) 에 대해](https://github.com/Be-poz/TIL/blob/master/Java/Immutable%20Object%20(%EB%B6%88%EB%B3%80%20%EA%B0%9D%EC%B2%B4)%20%EC%97%90%20%EB%8C%80%ED%95%B4.md)

***