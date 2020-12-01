# Validator 생성 시 주의해야 할 점, Invalid target 오류

Custom 한 Validator 를 많이들 생성할 것이다. 이번에 나는 프로젝트를 수행 도중에 크게 막히는 부분이 있었다.  

> Invalid target for Validator [com.ticket.captain.festival.validator.FestivalCreateValidator@5d035ab6]: com.ticket.captain.festival.dto.FestivalUpdateDto@3407ded1

바로 다음과 같은 오류였다. 이 오류가 발생할 당시에 코드는 다음과 같았다.  

```java
// validate 메서드는 생략
@Component
@RequiredArgsConstructor
public class FestivalCreateValidator implements Validator {

    private final FestivalService festivalService;

    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(FestivalCreateDto.class);
    }
```

```java
    @PutMapping("update/{festivalId}")
    public ApiResponseDto update(@PathVariable Long festivalId,
                                 @RequestBody FestivalUpdateDto festivalUpdateDto,
                                 Errors errors) {
        if (errors.hasErrors()) {
            String field = errors.getFieldError().getDefaultMessage();
            ExceptionDto exceptionDto = ExceptionDto.builder().message(field).build();
            return ApiResponseDto.VALIDATION_ERROR(exceptionDto);
        }
        return ApiResponseDto.createOK(festivalService.update(festivalId, festivalUpdateDto));
    }
```

```java
    @Test
    @WithMockUser(value = "mock-manager", roles = "MANAGER")
    void updateFestival() throws Exception {

        FestivalUpdateDto updateDto = FestivalUpdateDto.builder()
                .title("Charity Concert")
                .content("Enjoy And Donate")
                .salesEndDate(LocalDateTime.now())
                .festivalCategory(FestivalCategory.CHARITY.toString())
                .build();

        mockMvc.perform(put(API_MANAGER_URL + "/update/" + festival.getId())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(updateDto))
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("data.title").value("Charity Concert"))
                .andExpect(jsonPath("data.content").value("Enjoy And Donate"))
                .andExpect(jsonPath("data.festivalCategory").value("CHARITY"))
                .andDo(print());

    }
```

메서드를 만들고 test를 하는 과정에 있었다. 그런데 저렇게 오류가 나는 것이다. 그래서 로그를 확인 해보았다.  

```java
REQUEST : com.ticket.captain.festival.FestivalManagerController(initBinder) =  [_csrf -> (e1c3b860-7701-4bbd-abac-7d64bdd0f25f)]
RESPONSE : com.ticket.captain.festival.FestivalManagerController(initBinder) = null (9ms)
REQUEST : com.ticket.captain.festival.FestivalManagerController(initBinder) =  [_csrf -> (e1c3b860-7701-4bbd-abac-7d64bdd0f25f)]
```

나는 이때 어 나는 ``@Valid``를 붙여주지도 않았는데 도대체 왜 initBinder가 실행이 됐고 그게 왜 또 2번이나 실행이 되었고, 2 번째 호출에서 ``Invalid Target`` 에러가 대체 왜 난거지??? 정말 멘붕 그 자체였다.  

알고보니 initBinder는 Controller에 들어오는 모든 요청에 따라 다 처리를 해주는 것이었다. 그래서 이 update 메서드도 처리가 되었던 것이다. ``@Valid``를 쓰지 않았음에도 불구하고 말이다. 그런데 여기서 문제가 생긴다.  

```java
    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(FestivalCreateDto.class);
    }
```

validator 의 supports 메서드이다. 내부의 클래스가 FestivalCreateDto.class 이다. 이 코드는 팀프로젝트여서 직접 짠 validator가 아니다. 하지만, 나도 처음에 custom validator를 배울 때에 특정 클래스를 저렇게 넣는 식으로 배웠었다. 왜냐하면 해당 예제에서는 넣은 그 특정 클래스에 대해 validate 을 시작했기 때문이다.  

**update 메서드에서 ``@RequestBody`` 로 FestivalUpdateDto 를 들여온다. 이제 이 상황에서 FestivalCreateDto와 맞지 않기 때문에 ``Invalid Target`` 에러가 발생한 것이다.** 그래서 저 부분을 ``clazz``로 넣어서 해결해주었다.  



어... 그렇다면 대체 왜 initBinder가 2번이 호출이 된거고 위의 말처럼 클래스가 맞지않아 에러가 발생한거면 처음 호출된 initBinder는 어떻게 통과한거죠 ??  

그것은 바로 모든 파라미터에 대해서 검사를 해서 그런 것 같다(추측). 추측이지만 99% 맞는 것 같다.  
근거로 해당 메서드에 ``@PathVariable``를 추가하거나 없애 주었을 때와 비례해서 initBinder가 호출된 것을 알 수가 있었다.  
update 메서드의 파라미터 개수를 보면 2개이기 때문에 2번이 호출된 것이다. ``Long festivalId`` 값이 먼저 있으니 이 값은 통과되고 그 다음 파라미터에서 오류가 난 것이다. 이를 증명하기 위해 파라미터의 위치를 바꿔보았다.  

``@RequestBody FestivalUpdateDto festivalUpdateDto,@PathVariable Long festivalId`` 다음과 같이 바꾸고 진행하니깐  

> REQUEST : com.ticket.captain.festival.FestivalManagerController(initBinder) =  [_csrf -> (a311ae8f-175c-4b9f-990c-fcb9990c28c0)]

이게 딱 1번 나오고 바로 에러가 난 것을 알 수가 있었다.  그리고 클래스가 아닌 기본타입이 들어오면 통과해주는 것 같다.  

그러면 모든 값에 대해서 initBinder를 건다면 ``@Valid``는 왜 하는거죠??  
이 어노테이션을 붙이게 되면 다음과 같이 흘러간다.  

> REQUEST : com.ticket.captain.festival.FestivalManagerController(initBinder) =  [_csrf 
>
> RESPONSE : com.ticket.captain.festival.FestivalManagerController(initBinder) = null (10ms)
>
> REQUEST : com.ticket.captain.festival.FestivalManagerController(initBinder) =  [_csrf 
>
> RESPONSE : com.ticket.captain.festival.FestivalManagerController(initBinder) = null (1ms)REQUEST : com.ticket.captain.festival.FestivalService(findByTitle) =  [_csrf

initBinder 후에 ``findByTitle`` 를 실행하는 것을 볼 수가 있다. 이 메서드는 내가 validate 메서드에 적어둔 메서드이다.  

**즉, ``@Valid``를 붙이게 되면 initBinder 이후에 validate 메서드를 적용한다는 것을 알 수가 있다.**  

***

정말 시간을 많이 잡아먹은 오류였다... 하지만, 큰거 하나 배우고 간다라는 생각이 들어서 해결하고 나니깐 기분은 좋다!

