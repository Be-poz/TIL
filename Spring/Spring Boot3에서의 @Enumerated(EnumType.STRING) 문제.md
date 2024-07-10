# Spring Boot3에서의 @Enumerated(EnumType.STRING) 문제

Spring Boot 3 부터는 Hibernate 6 버전을 default로 사용하고 여기서는 ``@Enumerated(EnumType.STRING)`` 을 enum 필드에 붙여도 db에 enum 타입으로 들어간다. 따라서 추가적인 조치를 취해주어야 한다.  

```java
@Column(name = "enum_name", nullable = false, columnDefinition = "varchar")
@Enumerated(EnumType.STRING)
private EnumName enumName = EnumName.BEPOZ;
```

---

