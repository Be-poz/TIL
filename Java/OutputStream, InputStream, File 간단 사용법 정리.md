# OutputStream, InputStream, File 간단 사용법 정리

어느 한쪽에서 다른 쪽으로 데이터를 전달하기 위해서는 두 대상을 연결하고 데이터를 전송할 수 있는 통로가 필요한데 이것을 스트림이라고 생각하면 된다. 스트림은 단방향통신만 가능하므로 입력과 출력을 위한 스트림이 각각 존재해야한다.  

입력 스트림이 InputStream, 출력 스트림이 OutputStream이다. 스트림은 FIFO 다.  

## OutputStream

OutputStream은 다른 매체에 바이트로 데이터를 쓸 때 사용한다. 데이터를 쓰기위해 ``write(int a)`` 메서드를 이용한다.    

```java
        @Test
        void OutputStream은_데이터를_바이트로_처리한다() throws Exception {
            byte[] bytes = {110, 101, 120, 116, 115, 116, 101, 112};
            final OutputStream outputStream = new ByteArrayOutputStream(bytes.length);
            outputStream.write(bytes);

            final String actual = outputStream.toString();

            assertThat(actual).isEqualTo("nextstep");
            outputStream.close();
        }
```

``write(int a)`` 는 하나씩 쓰기 때문에 굉장히 비효율적이다.  
위 코드에서 사용한 ``write(byte[] data)`` 나 ``write(byte[] data, int off, int let)`` 을 이용하면 효율적으로 1바이트 이상을 한 번에 전송할 수 있다.  
``BufferedOutputStream`` 을 이용하면 버퍼링을 사용할 수 있다.  

***

## InputStream

InputStream은 다른 매체로부터 바이트로 데이터를 읽을 때 사용한다. 데이터를 읽을 때 ``read()`` 를 사용한다.  
``read()`` 메서드는 매체로부터 단일 바이트를 읽고, 0부터 255 사이의 값을 int 타입으로 반환한다.  
Stream 끝에 도달하면 -1 을 반환한다.  

```java
        @Test
        void InputStream은_데이터를_바이트로_읽는다() throws IOException {
            byte[] bytes = {-16, -97, -92, -87};
            final InputStream inputStream = new ByteArrayInputStream(bytes);
            String actual = new String(inputStream.readAllBytes());

            assertThat(actual).isEqualTo("🤩");
            assertThat(inputStream.read()).isEqualTo(-1);
            inputStream.close();
        }
```

### FilterStream

필터는 필터 스트림, reader, writer로 나뉜다. 필터는 바이트를 다른 데이터 형식으로 변환할 때 사용한다.  
reader, writer는 UTF-8, ISO 8859-1 같은 형식으로 인코딩한 텍스트를 처리하는데 사용된다.  

```java
        @Test
        void 필터인_BufferedInputStream를_사용해보자() throws IOException {
            final String text = "필터에 연결해보자.";
            final InputStream inputStream = new ByteArrayInputStream(text.getBytes());
            final InputStream bufferedInputStream = new BufferedInputStream(inputStream);

            byte[] actual = inputStream.readAllBytes();

            assertThat(bufferedInputStream).isInstanceOf(FilterInputStream.class);
            assertThat(actual).isEqualTo("필터에 연결해보자.".getBytes());
        }
```

필터 생성자에 객체를 전달하면 필터가 연결된다. 데코레이션 패턴의 대표적인 예이다.  

### InputStreamReader, BufferedReader

자바의 기본 문자열은 UTF-16 유니코드 인코딩을 사용한다. 방티를 문자로 처리하려면 인코딩을 신경 써야 한다.  
InputStreamReader를 사용하면 지정된 인코딩에 따라 유니코드 문자로 변환할 수 있다.  
reader, writer를 사용하면 입출력 스트림을 바이트가 아닌 문자 단위로 데이터를 처리하게된다.  

```java
        @Test
        void BufferedReader를_사용하여_문자열을_읽어온다() throws IOException {
            final String emoji = String.join("\r\n",
                    "😀😃😄😁😆😅😂🤣🥲☺️😊",
                    "😇🙂🙃😉😌😍🥰😘😗😙😚",
                    "😋😛😝😜🤪🤨🧐🤓😎🥸🤩",
                    "");
            final InputStream inputStream = new ByteArrayInputStream(emoji.getBytes());
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

            final StringBuilder actual = new StringBuilder();
            while (reader.ready()) {
                actual.append(reader.readLine() + "\r\n");
            }

            assertThat(actual).hasToString(emoji);
        }
```

``InputStreamReader`` 로 감싸주고 ``BufferedReader`` 로 다시 한 번 감싸준 것을 확인할 수가 있다.  
``\r\n`` 을 확인하고 잘라준다.  

***

## File

File 객체를 생성하려면 파일의 경로를 알아야 한다.  
![image](https://user-images.githubusercontent.com/45073750/132122362-786f80b1-8743-4fc3-bf7c-b17a89019933.png)  
현재 파일의 경로는 다음과 같다.  

![image](https://user-images.githubusercontent.com/45073750/132122420-9c1913dd-b6f1-4cbc-99da-8bf393cfd94e.png)

```java
    @Test
    void resource_디렉터리에_있는_파일의_경로를_찾는다() {
        final String fileName = "nextstep.txt";
        URL resourceUrl = getClass().getClassLoader().getResource(fileName);
        File file = new File(resourceUrl.getPath());
        //File file = new File(resourceUrl.toURI());
        String actual = file.getName();

        assertThat(actual).endsWith(fileName);
    }
```

만약 앞서서 다른 폴더가 있다면 ``fileName = "{folderName}/nextstep.txt";``  가 되어야 할 것이다.  

이제 해당 File의 내용을 읽어서 Stream을 이용해 전달해야 한다.  

```java
    @Test
    void 파일의_내용을_읽는다() throws Exception {
        final String fileName = "nextstep.txt";
        URL resource = getClass().getClassLoader().getResource(fileName);
        File file = new File(resource.getPath());
        Path path = file.toPath();
        List<String> actual = Files.readAllLines(path);
        assertThat(actual).containsOnly("nextstep");
    }
```

``Files.readAllLines(Path path)``  를 이용해서 해결하였다. 라인별로 읽는 것이 아니라 통채로 읽고 싶다면 ``Files.readAllBytes(Path path)`` 를 이용할 수도 있다. 반환타입은 ``byte[] ``  이며 이것을 ``new String(byte[] b)`` 를 이용해서 String으로 변환할 수도 있다.  

***

### REFERENCE

https://github.com/woowacourse/jwp-dashboard-http  

https://smujihoon.tistory.com/163