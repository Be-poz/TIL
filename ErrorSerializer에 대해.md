# ErrorSerializer에 대해

``return ResponseEntity.ok().body(객체)`` 가 있을 때에 body에 일반적인 객체를 넣으면 알아서 json 형태로 변환이 된다.



ObjectMapper(다양한 Serializer를 가지고 있음)가 자바 빈 스펙을 따르는 객체의 경우에 Bean Serializer를 이용해서 변환해주기 때문이다.  하지만 **Errors 객체는 예외이다.**  
Errors는 자바 빈 스펙을 준수하지 않기 때문에 ObjectMapper에서 Bean Serializer를 통해 Serialization이 불가능하다.

그래서 따로 Serializer를 만든 후에 ObjectMapper에 등록을 해주면 Errors를 Serialization 해줄 수 있게 된다.

```java
@JsonComponent
public class ErrorSerializer extends JsonSerializer<Errors> {
    @Override
    public void serialize(Errors errors, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeStartArray();
        errors.getFieldErrors().forEach(e->{
            try{
                gen.writeStartObject();
                gen.writeStringField("field", e.getField());
                gen.writeStringField("objectName", e.getObjectName());
                gen.writeStringField("code", e.getCode());
                gen.writeStringField("defaultMessage", e.getDefaultMessage());
                Object rejectedValue = e.getRejectedValue();
                if (rejectedValue != null) {
                 gen.writeStringField("rejectedValue",rejectedValue.toString());
                }

                gen.writeEndObject();
            }catch(IOException e1){
                e1.printStackTrace();
            }
        });
        errors.getGlobalErrors().forEach(e->{
            try{
                gen.writeStartObject();
                gen.writeStringField("objectName", e.getObjectName());
                gen.writeStringField("code", e.getCode());
                gen.writeStringField("defaultMessage", e.getDefaultMessage());
                gen.writeEndObject();
            }catch(IOException e1){
                e1.printStackTrace();
            }
        });
        gen.writeEndArray();
    }
}
```

다음과 같이 말이다.  

``gen.writeStartArray()`` 와 ``gen.writeEndArray()``로 시작과 끝을 열고 닫고, 에러의 글로벌 에러와 필드 에러에 따라 코드를 작성해준다. 

FieldError는 rejectValue() 메서드를 사용하여 에러 정보를 담은 경우이고, 
GlobalError는 reject() 메서드를 사용하여 에러 정보를 담은 경우이다.  

이제 해당 Serializer를 ObjectMapper에 등록을 해주어야 하는데, boot에서 제공하는 @JsonComponent를 사용하면 손쉽게 등록이 가능해진다.  

이제 ``return ResponseEntity.ok().body(errors)``가 가능해진다.  

![error1](https://user-images.githubusercontent.com/45073750/96000437-dbffcd80-0e71-11eb-8024-4ac9cf359c25.PNG)

결과는 다음과 같이 나오게 된다.
