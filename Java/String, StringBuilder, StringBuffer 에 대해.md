# String, StringBuilder, StringBuffer 에 대해

 String 은 불변 객체이다. 불변 객체에 대해 잘 모른다면 이 포스팅부터 보자 [(불변 객체)](https://github.com/Be-poz/TIL/blob/master/Java/Immutable%20Object%20(%EB%B6%88%EB%B3%80%20%EA%B0%9D%EC%B2%B4)%20%EC%97%90%20%EB%8C%80%ED%95%B4.md)  

```java
String str = "a";
str = str + "b";
System.out.println(str);	// "ab"
```

위의 String 연산은 기존의 str에 "b"가 붙어서 "ab"가 된 것처럼 보이지만 사실은 새로운 메모리 영역을 할당한 것이다. 이 말인 즉슨, String을 통한 문자열 추가 삭제를 빈번히 하게된다면 힙 메모리 영역에 많은 가비지가 생기게 된다는 것이다.  

이럴 때에 사용하는 것이 **StringBuilder, StringBuffer** 이다.  
이 둘은 가변성을 가지기 때문에 동일 메모리에서 문자열 변경이 가능하다.  

```java
        StringBuilder sb = new StringBuilder();
        sb.append("abc");

        StringBuffer sbf = new StringBuffer();
        sbf.append("def");

        System.out.println(sb.toString());
        System.out.println(sbf.toString());
```

그렇다면 이 둘의 차이점은 무엇일까 ? ?  

바로 **동기화의 유무** 이다. **StringBuffer는 동기화 키워드를 지원해서 멀티쓰레드 환경에서 안전하다(thread-safe)**  

**StringBuilder 는 동기화를 지원하지는 않지만 단일 쓰레드 환경에서 성능이 더 뛰어나다.**  

```java
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }

    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }
```

StringBuffer의 append 메서드에는 ``synchronized``가 붙어있는 것을 확인할 수가 있다.  



``String ``: 멀티 쓰레드 환경, 문자열 연산 적을 때  
``StringBuffer`` : 멀티 쓰레드 환경, 문자열 연산 많을 때  
``StringBuilder`` : 단일 쓰레드 환경, 문자열 연산 많을 때  

<br>

String 으로 연산 시에 계속해서 할당을 하고 그렇기 때문에 효율이 떨어졌다고 말했지만 JDK 1.5 버전 부터는 String 연산 과정이 StringBuilder를 이용하도록 변경되었다.  

```java
String str = "abc";
str = str + "d" + "e" + "f";

String s = (new StringBuilder()).append(str).append("d").append("e").append("f").toString();
```

내부 연산이 다음과 같이 동작한다.  

**하지만, 연산 때에 StringBuilder가 생성되기 때문에 만약 반복문 안에서 문자열을 더하게 된다면 반복문 횟수만큼 StringBuilder가 생성이 되어 상대적으로 속도가 느릴 수 밖에 없다.**  

```java
        String str = "abc";
        String str2 = "abc";
        for (int i = 0; i < 100000; i++) {
            str = "abc" + i;
        }

        for (int i = 0; i < 1000000; i++) {
            str2 += "abc" + i;
        }
```

위 코드의 2개의 반복문 중에서 첫 반복문과 두 번째 반복문의 수행시간은 크게 차이가 난다.  
왜 일까 ???  

```java
String s1 = (new StringBuilder()).append("abc").append(i);
String s2 = (new StringBuilder()).append(str2).append("abc").append(i);
```

내부적으로 다음과 같이 돌아가게 되는데, 밑의 경우에 기존의 문자열을 포함하면 계속해서 더하기 때문에 문자열의 길이가 길어질 수 밖에 없다. 그 결과 반복문 안의 ``new StringBuilder()``는 길어진 만큼의 문자열에 맞추어 공간을 만들고 할당해야 한다. 그렇기 때문에 시간 차이가 나게 되는 것이다.  

**즉, 반복문 안에서 문자열 연산을 할 때에는 아무리 JDK 1.5 이후에 String 이 StringBuilder를 통한 연산을 지원한다고 하더라도 String을 사용하기에는 굉장히 불리하다.**  

```java
final String str = "abc";
String result = abc + "d" + "e" + "f";

//
String s = "abcdef";
```

추가적으로, 문자열 상수를 이용하는 경우에는 컴파일 과정에서 하나의 문자열로 변경된다.  
그러나, 그냥 여러 개의 변수를 더하는 경우에는 위와 같은 최적화는 발생하지 않는다.  

배보다 배꼽이 더 큰 정리가 된 것 같은데 몰랐던 점을 배울 수 있는 기회가 된 것 같다.



***

Reference

https://ifuwanna.tistory.com/221  

https://12bme.tistory.com/42  

https://madplay.github.io/post/difference-between-string-stringbuilder-and-stringbuffer-in-java