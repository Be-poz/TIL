# Custom Validator 적용하기

엔티티에 조건을 걸 때에,  
```java
@NoArgsConstructor
@Getter
public class AccountCreateDto {

    @NotBlank
    @Length(min=2, max=20)
    private String loginId;

    @NotBlank
    @Length(min=8, max=50)
    private String password;
```
흔히 다음과 같은 어노테이션을 사용해서 검증을 한다.  
Controller단에서는 대충 다음과 같이 사용될 것이다.  
```java
    @PostMapping
    public ResponseEntity createAccount(@RequestBody @Valid AccountCreateDto accountCreateDto, Errors errors) throws URISyntaxException {
        if(errors.hasErrors()){
            return badRequest(errors);
        }
        Account account = accountService.createAccount(accountCreateDto);
        return ResponseEntity.created(new URI("/sign-up/complete")).body(account);
    }
```

이번 글에서는 해당 @Valid를 @NotBlank와 같은 기존에 존재하는 조건이 아닌 나만의 custom validator를 만들 것이다.  
만들어진 전체 코드는 다음과 같다.  
```java
@Component
@RequiredArgsConstructor
public class AccountCreateDtoValidator implements Validator {

    private final AccountRepository accountRepository;

    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(AccountCreateDto.class);
    }

    @Override
    public void validate(Object target, Errors errors) {
        AccountCreateDto accountCreateDto = (AccountCreateDto) target;
        if(accountRepository.existsByNickname(accountCreateDto.getNickname())){
            errors.rejectValue("nickname", "invalid.Nickname", new Object[]{accountCreateDto.getNickname()}, "이미 사용중인 닉네임입니다.");
        }
        if(accountRepository.existsByEmail(accountCreateDto.getEmail())){
            errors.rejectValue("email", "invalid,Email", new Object[]{accountCreateDto.getEmail()}, "이미 사용중인 이메일입니다.");
        }
        if(accountRepository.existsByLoginId(accountCreateDto.getLoginId())){
            errors.rejectValue("loginId", "invalid.LoginId", new Object[]{accountCreateDto.getLoginId()}, "이미 사용중인 아이디입니다.");
        }
    }
}
```
``Validator``를 implements받아서 작성하면 된다.  
2개의 메서드를 오버라이딩 하게 되는데, supports 내부에는 해당 클래스를 적어두면된다.  

validate에는 조건을 적고 ``Errors``객체 내에 ``rejectValue()``메서드를 통해 필드에러를 정의해주면 된다.  
위의 코드에서의 ``rejectValue()``의 원형은 다음과 같다.  
```java
	void rejectValue(@Nullable String field, String errorCode,
			@Nullable Object[] errorArgs, @Nullable String defaultMessage);
```
물론 오버로딩이 되어있기 때문에 errorArgs를 뺀다던가 defaultMessage를 뺀다던가 할 수도 있다.  

이제 해당 validator를 등록을 해주어야 한다. 여러 등록방법이 있겠지만 2가지를 소개하겠다.  

Validator가 필요한 클래스에  
```java
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        AccountCreateDtoValidator validator=new AccountCreateDtoValidator(accountRepository);
        binder.addValidators(validator);
    }
```
다음과 같이 initBinder를 선언해주는 경우가 있다.  
위 예제들에서의 내가 선언한 validator는 repository 클래스를 받고 ``@RequiredArgsConstructor``를 사용했기 때문에  
해당 repository를 파라미터로 던져준 경우이고, 생성자는 상황마다 다를 것이다.  

위와 같이 initBinder의 호출이 싫다면 필요한 곳에서 validate 해주는 방법이 있다.  
해당 컨트롤러 클래스에 validator를 ``@Autowired``로 주입받아 주고, 
```java
    @PostMapping
    public ResponseEntity createAccount(@RequestBody @Valid AccountCreateDto accountCreateDto, Errors errors) throws URISyntaxException {
        if(errors.hasErrors()){
            return badRequest(errors);
        }
        accountCreateDtoValidator.validate(accountCreateDto,errors);
        if (errors.hasErrors()) {
            return badRequest(errors);
        }
        ...
    }
``` 
다음과 같이 맨 처음에는 기존의 어노테이션으로 생긴 errors들의 유무를 체크하고,  
이후에 validator의 validate 메서드를 사용하여 검증을 한 후에 에러 유무에 따른 로직을 추가해주면 된다.  

