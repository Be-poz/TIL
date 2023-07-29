# Filter와 server.compression 설정을 통한 api 압축

API를 압축해서 return 하려고 한다. 먼저 가장 대표적인 ``server.compression`` 설정을 알아보고 사용해보고 이후 Filter를 이용하여 조금 더 응용해보려고 한다.  

## server.compression 설정을 통한 api 압축

### 설정 종류

```yaml
server:
  compression:
    enabled:  
    min-response-size:
    mime-types:
    excluded-user-agents:
```

``server.compression`` 설정에는 위의 4개 항목이 있다. 단순히 yaml 파일에 기입을 해두면 동작한다.  

* ``enabled``: 압축 여부
  * default: false
* ``min-response-size``: 압축을 수행할 최소 용량
  * default: 2KB
  * 아래에서 언급할 것이지만 현재 http에서는 의미가 없는 설정
* ``mime-types``: 압축을 수행할 MIME 타입
  * default: text/html, text/xml, text/plain, text/css, text/javascript, application/javascript, application/json, application/xml
* ``excluded-user-agents``: 압축을 하지않을 user agents 목록

<Br/>

### 압축 전 후 데이터 용량 확인

```java
@RestController
@RequestMapping("/compress")
public class CompressController {

    @GetMapping
    public List<PersonDto> create() {
        final List<PersonDto> dtos = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            dtos.add(new PersonDto("name" + i, i));
        }

        return dtos;
    }

    @Getter
    @AllArgsConstructor
    class PersonDto {
        private String name;
        private int age;
    }
}
```

위의 api를 가지고 압축 전 후 용량을 확인해보았다.  
압축하길 바란다면 헤더에 ``Accept-Encoding: gzip`` 이 추가되어야 한다.

* 압축 X : 3.13MB
* 압축 O: 471KB

포스트맨을 사용해서 압축을 요청하면 포스트맨에서 response를 바로 압축해제 하여 보여주기 때문에 표시되어있는 용량으로는 압축된 것을 확인할 수 없으니 curl 요청을 해서 보는 방법으로 진행하였다.  

```sh
curl -v --location 'localhost:8000/compress' \
--header 'Accept-Encoding: gzip'> 1

ls -alh 1
```

<br/>

### min-response-size 설정 동작 확인

``min-response-size`` 설정을 해두면 헤더의 ``Content-Length`` 값과 비교하여 동작을 할 것인지 동작을 하지 않을 것인지 확인한다.  

압축 설정과 관련한 클래스는 ``CompressionConfig`` 클래스인데 해당 클래스의 useCompression 메서드에 해당 내용이 나와있다.  

```java
public boolean useCompression(Request request, Response response) {
        ...

        // If force mode, the length and MIME type checks are skipped
        if (compressionLevel != 2) {
            // Check if the response is of sufficient length to trigger the compression
            long contentLength = response.getContentLengthLong();
            if (contentLength != -1 && contentLength < compressionMinSize) {
                return false;
            }

            // Check for compatible MIME-TYPE
            String[] compressibleMimeTypes = getCompressibleMimeTypes();
            if (compressibleMimeTypes != null &&
                    !startsWithStringArray(compressibleMimeTypes, response.getContentType())) {
                return false;
            }
        }
  			...
    }
```

contentLength가 설정값보다 작으면 압축 X, 설정값보다 크거나 contentLength 값이 -1이면 MIME 타입으로 체크한다고 나와있다.  

현재의 HTTP는 ``Content-Length`` 정보가 response 헤더에 포함되어있지 않고 ``Transfer-Encoding: chunked`` 값으로 대처하기 때문에 ``Content-Length`` 값이 -1 으로 취급된다. 따라서 ``min-response-size`` 설정은 현재로서는 의미가 없다.  

<Br/>

### mime-types 설정 동작 확인

```java
  @GetMapping(path = "/text")
  public String create2() {
      StringBuilder result = new StringBuilder();
      for (int i = 0; i < 100000; i++) {
          result.append("name" + i);
      }
      return result.toString();
  }

---
  
server:
  compression:
    enabled: true
    mime-types: text/plain
```

``mime-types`` 설정을 ``text/plain``으로 설정해주었고 확인을 위해 새로운 api를 추가해주었다.  



#### Accept: application/json 으로 api 호출

```sh
curl -v --location 'localhost:8000/compress' \
--header 'Accept-Encoding: gzip' \
--header 'Accept: application/json' > 1
```

위와 같이 맨 처음 추가해준 api를 압축요청을 한채로 요청을 해보았으나 압축을 하지 않았을 때의 용량인 3.1MB를 돌려받았다.  
압축이 되지 않은 것을 알 수가 있다.

#### Accept: text/plain 으로 api 호출

```sh
curl -v --location 'localhost:8000/compress/text' \
--header 'Accept-Encoding: gzip' \
--header 'Accept: text/plain' > 1
```

이번에는 새롭게 추가한 api에 ``text/plain`` 으로 압축요청을 하니 218KB를 반환했다.  
압축을 요청하지 않은채로 호출을 해보니 868KB를 반환받았다. 압축이 된 것을 확인할 수가 있었다.  

``mime-types`` 설정은 의미가 있다는 것을 확인할 수가 있었다.  

<Br/>

## Filter를 통한 api 압축

``server.compression`` 설정은  특정  end point만 압축을 하는 기능을 지원하지 않는다.  
Filter를 이용해서 이것을 구현해보고자 한다.  

```java
@WebFilter("/compress")
@Component
public class CompressFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        // Response 객체가 HttpServletResponse 형인지 확인
        if (response instanceof HttpServletResponse) {
            // 압축하기 위한 ByteArrayOutputStream 생성
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            GZIPOutputStream gzipOutputStream = new GZIPOutputStream(byteArrayOutputStream);

            // 압축된 데이터를 HttpServletResponse로 전송할 수 있도록 필터 체인 내부에서 사용되는 응답 객체를 변경
            HttpServletResponse httpResponse = (HttpServletResponse) response;

            CompressedResponseWrapper compressedResponseWrapper = new CompressedResponseWrapper(httpResponse, gzipOutputStream);

            // Filter 체인을 통해 컨트롤러의 응답 처리
            chain.doFilter(request, compressedResponseWrapper);

            // 압축 스트림 닫기
            gzipOutputStream.close();

            // Content-Encoding 설정
            byte[] compressedData = byteArrayOutputStream.toByteArray();
            httpResponse.setHeader("Content-Encoding", "gzip");

            // 압축된 데이터를 실제 응답에 기록
            response.getOutputStream().write(compressedData);
        }
    }
}
```

```java
public class CompressedResponseWrapper extends HttpServletResponseWrapper {

    private final GZIPOutputStream gzipOutputStream;

    public CompressedResponseWrapper(HttpServletResponse response, GZIPOutputStream gzipOutputStream) {
        super(response);
        this.gzipOutputStream = gzipOutputStream;
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        return new GzipServletOutputStream(gzipOutputStream);
    }

    @Override
    public void setContentLength(int len) {
    }
}
```

```java
public class GzipServletOutputStream extends ServletOutputStream {

    private final GZIPOutputStream gzipOutputStream;

    public GzipServletOutputStream(GZIPOutputStream gzipOutputStream) {
        this.gzipOutputStream = gzipOutputStream;
    }

    @Override
    public void write(int b) throws IOException {
        gzipOutputStream.write(b);
    }

    @Override
    public boolean isReady() {
        return true;
    }

    @Override
    public void setWriteListener(WriteListener listener) {

    }
}
```

``/compress`` url에만 해당 필터를 적용시킨다. ``server.compression`` 설정을 하지 않았다는 가정 하에 진행을 한 것이고 만약 해당 설정을 하고 있다면 2중으로 압축하려하면서 오류가 날 수도 있으니 헤더에 ``Accept-Encoding: gzip`` 있는지 확인하고 압축 흐름을 타지않고 그냥 ``chain.doFilter(request, response)``만 호출하는 방식의 코드를 추가하면 될 것이다.  

---

### REFERENCE

https://www.springcloud.io/post/2022-05/resttemplate-gzip/#gsc.tab=0

