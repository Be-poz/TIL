# JacksonAnnotationsInside에 대해

```java
@JsonDeserialize(converter = 컨버터 클래스)
@JsonSerialize(converter = 컨버터 클래스)
@JacksonAnnotationsInside
@Retention(RetentionPolicy.RUNTIME)
public @interface JacksonBoxAnnotation {
}
```

jackson 관련 어노테이션들을 하나의 어노테이션에 한데모아 선언하고 ``@JacksonAnnotationsInside`` 어노테이션을 붙여서 완성시킨다. 그렇게 만들어진 어노테이션이 붙은 곳에는 해당 어노테이션 내부에 선언되어진 다른 어노테이션들에 적용받게 된다.

가령 위의 코드를 예시를 들면 dto 필드 위에 해당 어노테이션을 붙이면 serialize/deserialize 가 적용이된다.

