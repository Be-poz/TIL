# Jpa Auditing에서 OffsetDateTime 사용하기

``@EnableJpaAuditing``을 사용한 jpa의 auditing에서 ``@CreateDate`` 과 같은 Date 관련 기능은 기본적으로 LocalDateTime 타입이 할당된다.  

```java
/**
 * Default {@link DateTimeProvider} simply creating new {@link LocalDateTime} instances for each method call.
 *
 * @author Oliver Gierke
 * @author Christoph Strobl
 * @since 1.5
 */
public enum CurrentDateTimeProvider implements DateTimeProvider {

	INSTANCE;

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.auditing.DateTimeProvider#getNow()
	 */
	@Override
	public Optional<TemporalAccessor> getNow() {
		return Optional.of(LocalDateTime.now());
	}
}
```

default로 위의 ``getNow`` 메서드가 실행이 되는데 다른 세부 타입의 TemporalAccessor을 이용하고 싶다면 그냥 새롭게 빈 등록을 해주면 된다.  

```java
@Configuration
public class DateTimeProviderConfig {

    @Bean("offSetDateTimeProvider")
    public DateTimeProvider dateTimeProvider() {
        return () -> Optional.of(OffsetDateTime.now());
    }
}
```

``DateTimeProvider`` 을 구현한 빈을 등록한 후,  
``@EnableJpaAuditing(dateTimeProviderRef = "offSetDateTimeProvider")``  좌측과 같이 넣어주면 완료~!  

---

