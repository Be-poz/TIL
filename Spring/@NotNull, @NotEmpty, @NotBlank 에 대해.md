# @NotNull, @NotEmpty, @NotBlank

``import javax.validation.constraints`` 에 들어가있는 세 어노테이션에 대해 알아보겠다.  



***

### @NotNull

```java
@Setter @Getter
public class BoardDto {
    @NotNull
    String value;
}

@BeforeAll
public static void setupValidatorInstance(){
    validator = Validation.buildDefaultValidatorFactory().getValidator();
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set("wow");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(0);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set("");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(0);
}
```

``@NotNull``은 null 값만 걸러낸다. ``""`` 는 null이 아니니깐 통과된다. 첫 번째 케이스에서 해당 violations를 출력해보면 다음과 같이 출력이 된다.  

> ConstraintViolationImpl{interpolatedMessage='널이어서는 안됩니다', propertyPath=value, rootBeanClass=class com.helloasean.manage.domain.dto.BoardDto, messageTemplate='{javax.validation.constraints.NotNull.message}'}

String 뿐만 아니라 int, long 등등 다 검사가 가능하다. int 로 두고 set 해주지 않았을 때에 기본값이 0이니깐 통과가 되지만 primitive type 이 아닌 Integer 를 사용하고 set 해주지 않으면 null 이 들어가 걸리게 된다.  CharSequence, Collection, Map, Array를 넣을 수 있다.  



 

***

### @NotEmpty

```java
@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set("wow");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(0);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set("");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}
```

``@NotEmpty`` 는 ``@NotNull``의 valid 를 implementation 하면서 추가로 해당 오브젝트의 size/length 를 검사해서 0 보다 큰지를 확인한다.  

```java
	@Override
	public boolean isValid(int[] array, ConstraintValidatorContext constraintValidatorContext) {
		if ( array == null ) {
			return false;
		}
		return array.length > 0;
	}
// int[] array 말고 정말 많은 종류의 파라미터 형태가 정의되어 있다.
```

이 곳에  ``@NotNull`` 때와 같이 int, long, Integer 등을 넣으면 다음과 같은 에러가 나온다. 

> No validator could be found for constraint 'javax.validation.constraints.NotEmpty' validating type 'java.lang.Integer'. Check configuration for 'value'

``@NotEmpty``는 내부구현 처럼 해당 size/length 를 검사할 수 있는 타입이 들어가게 된다.  
CharSequence, Collection, Map, Array를 넣을 수 있다. 





***

### @NotBlank

```java
@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set("");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    boardDto.set(" ");
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}

@Test
public void validationTest() throws Exception{
    BoardDto boardDto = new BoardDto();
    Set<ConstraintViolation<BoardDto>> violations = validator.validate(boardDto);
    assertThat(violations.size()).isEqualTo(1);
}
```

``@NotBlank``는 흥미롭게도 ``" "`` 이렇게 공백도 통과를 시키지 않는다.  
어떻게 이러느냐면, ``NotBlankValidator`` 클래스를 사용해서 character's sequence의  trim 처리를 한 결과 값이 not empty 인지를 보기 때문이다.  

```java
/**
 * Check that a character sequence is not {@code null} nor empty after removing any leading or trailing whitespace.
 *
 * @author Guillaume Smet
 */
public class NotBlankValidator implements ConstraintValidator<NotBlank, CharSequence> {

	/**
	 * Checks that the character sequence is not {@code null} nor empty after removing any leading or trailing
	 * whitespace.
	 *
	 * @param charSequence the character sequence to validate
	 * @param constraintValidatorContext context in which the constraint is evaluated
	 * @return returns {@code true} if the string is not {@code null} and the length of the trimmed
	 * {@code charSequence} is strictly superior to 0, {@code false} otherwise
	 */
	@Override
	public boolean isValid(CharSequence charSequence, ConstraintValidatorContext constraintValidatorContext) {
		if ( charSequence == null ) {
			return false;
		}

		return charSequence.toString().trim().length() > 0;
	}
}
```

``@NotBlank``는 String 만 받을 수가 있다.  





***

``@NotNull`` 에서는 여러 타입이 들어갈 수 있지만 ``@NotEmpty``나 ``@NotBlank`` 에서는 그렇지 않다.  
int, long 등 수에 대한 valid 체크 고민이 있다면 ``@DecimalMax``, ``@DecimalMin``, ``@Max``, ``@Min``, ``@Positive``, ``@PositiveOrZero``, ``@Negative``, ``@NegativeOrZero``,``@Digits``,``@Size`` 등 여러 어노테이션들이 있다.  
시간의 경우에는 ``@Future``, ``@FutureOrPresent``,``@Past``,``@PastOrPresent`` 들이 있다.  
그 외에 ``@Email``,``@Pattern`` 등도 있다는 것을 알아두도록 하자.  



#### Reference

[https://www.baeldung.com/java-bean-validation-not-null-empty-blank](https://www.baeldung.com/java-bean-validation-not-null-empty-blank)

***