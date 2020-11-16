# @Builder에 대해

```java
public class Member{
    private final String email;
    private final String name;
    
    public static class Builder{
        private final String email;
        private final String name;
        
        public Builder email(String val){
            email = val;
            return this;
        }
        public Builder name(String val){
            name = val;
            return this;
        }
        public Member build(){
            return new Member(this);
        }
    }
    private Member(Builder builder){
        email = builder.email;
        name = builder.name;
    }
}
```

이게 기본적인 빌더패턴이다. setter를 사용하지 않으므로 불변한 객체를 만들 수가 있다.  
한 번에 객체를 생성해서 객체 일관성이 깨지지 않고, build() 함수가 잘못된 값이 입력되었는지 검증할 수가 있다. Lombok에서는 ``@Builder``를 지원하는데 이를 이용해서 간편하게 패턴을 이용할 수가 있다.  

```java
@Builder
public class Member{
    private String email;
    private String name;
}
```

다음과 같이 어노테이션만 사용해주면 된다.  

```java
Member member = Member.builder()
    			.email("builderPattern@gmail.com")
    			.name("Bepoz")
    			.build();
```

다음과 같이 사용해줄 수가 있다.  

여기서 지향해야 할 점은 클래스 위에 ``@Builder``를 사용하게되면 	``@AllArgsConstructor``와 동일한 역할을 한다는 것이다.  

```java
@Entity
public class Member{
    @Id @GeneratedValue
    private Long id;
    
    @CreatedDate
    private LocalDateTime time;
    
    @OneToMany(mappedBy = "member")
    List<Hobby> hobbies = new ArrayList<>();
    
    private String name;
    private String email;
}
```

대충 이런식으로 엔티티가 있다고 쳤을 때에 클래스 위에다가 ``@Builder``를 사용하게 되면 id, time,hobbies 등이 다 파라미터로 넘어가게 된다. id 같은 경우에 위에서는 DB에 한 번 접근하고 나야 그 값이 나오는건데 문제가 생길 것이다. 그래서 필요한 필드 파라미터를 가진 생성자 위에 어노테이션을 사용해주는 것이 바람직하다.  

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member{
    @Id @GeneratedValue
    private Long id;
    
    @CreatedDate
    private LocalDateTime time;
    
    @OneToMany(mappedBy = "member")
    List<Hobby> hobbies = new ArrayList<>();
    
    private String name;
    private String email;
    
    @Builder
	private Member(String name, String email){
        this.name = name;
        this.email = email;
    }
}
```

위와 같이 사용하게되면 name과 email에 관해서만 받아서 사용하게 된다. private으로 지정해두었는데 어떻게 접근하나요 ?? 위의 빌더패턴 코드를 보면 ``static class Builder`` 임을 알 수가 있다.  static 이므로 그냥 ``Member.Builder()`` 이렇게 호출이 가능하기 때문에 private 으로 지정해 둔것이다. public으로 두면 ``new Member()`` 로 뜨긴하더라. 그리고 내부의 ``this.name = name;`` 작업을 생략하면 값이 들어가지 않는다 당연하지만...   

```java
@Builder
public class Test{
    @Builder.Default
    public int a = 3;
    public int b;
    public int c=3;
    public String name;
}

Test build = Test.builder().a(3).b(3).build();	//a=3, b=3, c=0, name=null
Test build = Test.builder().b(3).name("kang").build();//a=3, b=3, c=0, name=kang
Test build = Test.builder().c(1).build();	//a=3, b=0, c=1, name=kang
```

위의 코드를 보면 a는 빌더로 만들어주면 그 값이, 만들어 주지 않으면 3이,
b와 c는 만들어주면 그 값이, 만들어 주지 않으면 0이,
name은 만들어주면 그 값이, 만들어 주지 않으면 null 이 출력되는 것을 알 수가 있다.  

이건 위의 Builder 코드를 보면 이해가 가능할 것이다. 이제 여기서 기본 값을 주는 방법이 2가지가 있다. 바로 ``final`` 이용과 ``@Builder.Default`` 이용이 있다. final을 걸면 ``builder().`` 으로 해당 값이 나오지가 않는다. ``final``이기 때문에 불변하기에 그렇다. ``@Builder.Default``의 경우에는 이제 값을 줄 수도 있지만 일단 기본값을 설정하고 싶을 때에 사용하는 방법이 되겠다.  

```java
public class Test {
    @Builder.Default
    public int a = 3;
    public int b;
    public final int c=3;
    public int d = 3;
    public String name;
    @Builder
    public Test(int a,int b,String name) {
        this.a=a;
        this.b=b;
        this.name=name;
    }
}

Test build = Test.builder().b(3).build();	//a=0, b=3, name=null
Test build = Test.builder().a(3).b(3).name("kang").build();//a=3,b=3,name=kang
```

다음과 같이 생성자에 쓰는 경우를 살펴보자.  
생성자 파라미터에 쓰인 값들을 이용해서 빌더패턴을 꾸린다. 필드에다가 ``@Builder.Default``를 사용했지만 의미가 없는 것을 볼 수가 있다.  ``final``을 붙은 변수 ``c``는 이미 초기화가 되어있기 때문에 파라미터로 넣고 값을 변경하려들면 빨간줄이 뜨게된다. 만약 필드에서 초기화를 하지 않은 상태라면 상관이 없다. 그러나 애초에 setter를 제외하고 불변하게 만들기 위한 방법 중 하나인 빌더패턴을 이용하는데 굳이..? 라는 생각이 든다.  

d의 경우에도 이미 초기화가 되어있다. 해당 값을 조회해보면 0이 아닌 3이 출력이 된다. ``@Builder`` 로 필드 값 ``a``,``b``,``name`` 만 가져다가 사용했기 때문에 이러한 결과가 나온다는 것을 유추할 수가 있을 것이다. 만약 ``@Builder.Default``와 같은 결과를 받고 싶다면 해당 생성자 내부에 로직을 만들어 주면 해결이 될 것이다. 

***