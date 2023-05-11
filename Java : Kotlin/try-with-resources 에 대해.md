# try-with-resources 에 대해

try-with-resources 는 손쉽게 자원 할당을 해제할 수 있는 방법이다.  

어떻게 사용하는지 바로 확인해보자.  

```java
private static void printFile () throws IOException {

    try (FileInputStream input = new FileInputStream ( "file.txt")) {

        int 데이터 = input.read ();
        while (데이터! = -1) {
            System.out.print ((char) 데이터);
            데이터 = input.read ();
        }
    }
}
```

다음과 같이 사용한다. 기존의 try-catch-finally 였다면 finally 블록을 따로 만들어주고 그 부분에 close 메서드들을 호출했어야 했다. 하지만, try-with-resources 를 사용하면 try 블록이 완료되는 순간에 자동으로 close 메서드를 호출하게 된다.  

자바 9에서는 위의 코드에서 ``FileInputStream input = new FileInputStream("file.txt")`` 를 try 블록 바깥으로 빼고 try 블록에는 input만 넣는 것으로 처리할 수 있다.  

<br/>

try-with-resources가 모든 객체에 대해 close 메서드를 호출하는 것은 아니다.  

```java
public interface AutoCloseable {
    void close() throws Exception;
}
```

바로 AutoCloseable 인터페이스를 구현하고 있어야만 close 메서드를 호출해준다.  

물론 try-with-resources 에도 catch로 예외를 잡을 수 있으며, finally 또한 사용할 수 있다.  

***

### Reference

http://tutorials.jenkov.com/java-exception-handling/try-with-resources.html