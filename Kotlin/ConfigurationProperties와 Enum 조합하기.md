# ConfigurationProperties와 Enum 조합하기

```kotlin
enum class Person(
    val lastName: String,
    val age: Int,
) {
    A("kang", 20),
    B("kim", 30),
    C("lee", 40),
}
```

```kotlin
@ConfigurationProperties("")
class UrlConfigurationProperties(
    val info: Map<Person, UrlInfo>
) {
    data class UrlInfo(
        val url: String
    )
}
```

```yaml
info:
  A:
    url: a.com
  B:
    url: b.com
  C:
    url: c.com
```

위와 같이 yaml 파일을 ``ConfigurationProperties``를 이용하여 읽을 때에 enum을 key로 map 형식으로 받아 활용할 수 있다.  

---

