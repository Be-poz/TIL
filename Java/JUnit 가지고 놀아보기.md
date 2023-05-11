# JUnit 가지고 놀아보기

```java
        String[] strings = "1,2".split(",");
        assertThat(strings).contains("1");			//true
        assertThat(strings).containsExactly("1");	//false
        assertThat("Cowboy Bebop").contains("Cowboy");
        assertThat("The Legend Of Zelda")
                .startsWith("The")
                .contains("he L")
                .endsWith("Zelda");
```

위와 같이 contains를 다양하게 이용할 수 있다. 위의 코드 외에도 다양한 contains 관련 메서드가 있다.  

```java
        assertThat(100).as("100 equals Test").isEqualTo(1000);
```

에러 메세지 출력 시에 ``.as()`` 를 통해 메세지를 정해줄 수도 있다. 그러나 반드시 assertion 이전에 나와야 한다.  

> org.opentest4j.AssertionFailedError: [100 equals Test] 
> Expecting:
>  <100>
> to be equal to:
>  <1000>
> but was not.

```java
        List<Integer> list = new ArrayList(Arrays.asList(1, 2, 3, 4, 5, 6, 7));
        assertThat(list).filteredOn(i -> i > 5)
                .containsExactly(6, 7);
```

``filteredOn()`` 를 통해 필터조건을 걸수도 있다. 위와 같이 Predicate이 아니라 첫 번째 인수에는 Property Name을 두 번째 인수에는 Expected Value를 넣어서 사용할 수도 있다.  

```java
	List<Human> list = new ArrayList<>();
	Human kim = new Human("Kim", 22); 
    Human park = new Human("Park", 25); 
    Human lee = new Human("Lee", 25); 
    Human amy = new Human("Amy", 22); 
    Human jack = new Human("Jack", 22); 
    list.add(kim); 
    list.add(park); 
    list.add(lee); 
    list.add(amy); 
    list.add(jack); 
    assertThat(list).filteredOn("age", 25).containsOnly(park, lee);

출처: https://pjh3749.tistory.com/241 [JayTech의 기술 블로그]
```

위의 다른 분의 코드에 이어나가보겠다.  

```java
assertThat(list).extracting("name").contains("Kim", "Park", "Lee", "Amy", "Jack");
assertThat(list).extracting("name", String.class).contains("Kim", "Park", "Lee", "Amy", "Jack");
assertThat(list).extracting("name", "age")
    .contains(tuple("Kim", 22), 
              tuple("Park", 25), 
              tuple("Lee", 25), 
              tuple("Amy", 22), 
              tuple("Jack",22));
출처: https://pjh3749.tistory.com/241 [JayTech의 기술 블로그]
```

extracting으로 Property를 추출해서 검증할 수 있으며 ``String.class`` 를 명시하는 것과 같이 타입검증을 강하게 걸 수도 있다.  
튜플로도 추출이 가능한 것을 확인할수 있다.  

```java
		String str = "abc";

		assertThatThrownBy(() -> {
            str.charAt(3);
        }).isInstanceOf(IndexOutOfBoundsException.class);

        assertThatExceptionOfType(IndexOutOfBoundsException.class)
                .isThrownBy(() -> {
                    str.charAt(3);
                });
```

다음과 같이 예외처리 또한 할 수 있다.

```java
        assertThatThrownBy(() -> {
            throw new Exception("Exception Occurred");
        }).isInstanceOf(Exception.class)
                .hasMessageMatching("Exception Occurred");
```

``hasMessageMatching`` 을 통해 익셉션 메세지 일치여부를 확인할 수도 있다. 이것 또한 ``hasMessageContaining``, ``hasMessageEndingWith`` 등 다양한 메서드들이 있다.  

* 자주 발생하는 Exception의 경우 Exception 별 메서드를 제공한다.
  * assertThatIllegalArgumentException()
  * assertThatIllegalStateException()
  * assertThatIOException()
  * assertThatNullPointerException()

```java
    Set<Integer> numbers;

    @BeforeEach
    void setUp() {
        numbers = new HashSet<>();
        numbers.add(1);
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
    }

    @Test
    void setContainsTest() {
        assertThat(numbers.contains(1)).isTrue();
        assertThat(numbers.contains(2)).isTrue();
        assertThat(numbers.contains(3)).isTrue();
    }
```

위 코드에서 일일이 contains 값을 확인하기란 정말 귀찮은 작업이다.  

```java
    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3})
    void setContainsTest(int input) {
        assertTrue(numbers.contains(input));
    }
//String 의 경우 @ValueSource(strings = {"a", "b"}) 식이며, 파라미터로 String을 사용하면 된다.
```

다음과 같이 ``@ParameterizedTest`` 와 ``@ValueSource`` 를 이용해서 중복코드를 제거할 수 있었다.  
위의 경우에는 true인 경우만 테스트 가능했다. 이제 입력 값에 따라 결과 값이 다른 테스트에 대한 경우도 가능하도록 해본다.  

```java
    @ParameterizedTest
    @CsvSource(value = {"TEst,TEST", "tEst,TEST", "jaVa,JAVA"})
    void toLowerCase_ShouldGenerateTheExpectedLowerCaseValue(String input, String expected) {
        String actualValue = input.toUpperCase();
        assertEquals(actualValue, expected);
    }
```

``@CsvSource()`` 를 이용했다. 구분자의 default는 ``,`` 인데, 커스텀하게 바꿀 수도 있다.  

``@CsvSource(value = {"TEst:TEST", "tEst:TEST", "jaVa:JAVA"}, delimiter = ':')`` 다음과 같이 말이다.  

```java
    @ParameterizedTest
    @CsvSource(value = {"1,true", "2,true", "3,true", "4,false"})
    void toLowerCase_ShouldGenerateTheExpectedLowerCaseValue(int input, boolean expected) {
        assertEquals(numbers.contains(input), expected);
    }
```

위의 Set 예시를 사용해서 표현해봤다. int 와 boolean 으로 받아와서 비교를 했다.  

***
