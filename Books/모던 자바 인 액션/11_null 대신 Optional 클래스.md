# null 대신 Optional 클래스

## 값이 없는 상황을 어떻게 처리할까?

```java
public class Person {
    private Car car;
    public Car getCar() {
        return car;
    }
}

public class Car {
    private Insurance insurance;
    public Insurance getInsurance() {
        return insurance;
    }
}

public class Insurance {
    private String name;
    public String getName() {
        return name;
    }
}

public String getCarInsuranceName(Person person) {
    return person.getCar().getInsurance().getName();
}
```

위의 코드는 아무 문제가 없는 것 처럼 보이지만 차를 소유하지 않은 사람도 많다. 만약 그렇다면 런타임에 ``NullPointerException``이 발생하게 될 것이다. 그렇다면 만약 Person이 null이라면 어떻게 될까 ?? 아니면 ``getInsurance`` 가 null 이면 ??  

### 보수적인 자세로 NullPointerException 줄이기

```java
public String getCarInsuranceName(Person person) {
    if (person != null) {
        Car car = person.getCar();
        if(car != null) {
            Insurance insurance = car.getInsurance();
            if(insurance != null) {
                return insurance.getName();
            }
        }
    }
    return "Unknown";
}
```

위의 경우에는 null 확인 코드 때문에 들여쓰기 수준이 증가하게 된다.  

```java
public String getCarInsuranceName(Person person) {
    if (person == null) {
        return "Unknown";
    }
    Car car = person.getCar();
    if(car == null) {
        return "Unknown";
    }
    Insurance insurance = car.getInsurance();
    if(insurance == null) {
        return "Unknown";
    }
    return insurance.getName();
}
```

위 방법은 null 확인 코드마다 출구가 생기게 된다. 만약 누군가가 null 일 수 있다는 사실을 깜빡 잊었다면 어떤 일이 일어날까??  

### null 때문에 발생하는 문제

* **에러의 근원이다** : NPE는 자바에서 가장 흔히 발생하는 에러다.
* **코드를 어지럽힌다** : 때로는 중첩된 null 확인 코드를 추가해야 하므로 null 때문에 가독성이 떨어진다.
* **아무 의미가 없다** : null은 아무 의미도 표현하지 않는다. 특히 정적 형식 언어에서 값이 없음을 표현하는 방법으로는 적절하지 않다.
* **자바 철학에 위배된다** : 자바는 개발자로부터 모든 포인터를 숨겼다. 하지만 null 포인터는 예외다.
* **형식 시스템에 구멍을 만든다** : null은 무형식이며 정보를 포함하고 있지 않으므로 모든 참조 형식에 null을 할당할 수 있다. 이런 식으로 null이 할당되기 시작하면서 시스템의 다른 부분으로 null이 퍼졌을 때 애초에 null이 어떤 의미로 사용되었는지 알 수 없다.

### 다른 언어는 null 대신 무얼 사용하나?

그루비 같은 언어에서는 안전 내비게이션 연산자(?.)를 도입해서 null문제를 해결했다.  
하스켈은 선택형값을 저장할 수 있는 Maybe라는 형식을 제공한다.  

자바 8은 이 선택형값 개념의 영향을 받아서 ``java.util.Optional<T>`` 라는 새로운 클래스를 제공한다.  

<br/>

## Optional 클래스 소개

값이 있으면 Optional 클래스는 값을 감싼다. 반면 값이 없으면 Optional.empty 메서드로 Optional을 반환한다.  
Optional.empty는 Optional의 특별한 싱글턴 인스턴스를 반환하는 정적 팩토리 메서드다. 

null을 참조하려 하면 NPE이 발생하지만 Optional.empty() 는 Optional 객체이므로 이를 다양한 방식으로 활용할 수 있다.  

```java
public class Person {
    private Optional<Car> car;
    public Optional<Car> getCar() {
        return car;
    }
}

public class Car {
    private Optional<Insurance> insurance;
    public Optional<Insurance> getInsurance() {
        return insurance;
    }
}

public class Insurance {
    private String name;
    public String getName() {
        return name;
    }
}
```

Optional 클래스를 사용하면서 모델의 의미 semantic가 더 명확해졌음을 확인할 수 있다.  

모든 null 참조를 Optional로 대치하는 것은 바람직하지 않다. Optional의 역할은 더 이해하기 쉬운 API를 설계하도록 돕는 것이다.  
즉, 메서드의 시그니처만 보고도 선택형값인지 여부를 구별할 수 있다. Optional이 등장하면 이를 언랩해서 값이 없을 수 있는 상황에 적절하게 대응하도록 강제하는 효과가 있다.  

<br/>

## Optional 적용 패턴

### Optional 객체 만들기

#### 빈 Optional

``Optional<Car> optCar = Optional.empty();`` 정적 팩토리 메서드 ``Optional.empty()``로 빈 Optional 객체를 얻을 수 있다.  

#### null 이 아닌 값으로 Optional 만들기

``Optional<Car> optCar = Optional.of(car);`` 이제 car가 null이라면 즉시 NPE이 발생한다.  

#### null 값으로 Optional 만들기

``Optional<Car> optCar = Optional.ofNullable(car);``  

정적 팩토리 메서드 ``Optional.ofNullable`` 로 null 값을 저장할 수 있는 Optional을 만들 수 있다.  
car가 null이면 빈 Optional 객체가 반환된다.  

### 맵으로 Optional의 값을 추출하고 변환하기

```java
String name = null;
if(insurance != null) {
    name = insurance.getName();
}

=>

Optional<Insurance> optInsurance = Optional.ofNullable(insurance);
Optional<String> name = optInsurance.map(Insurance::getName);
```

위와 같은 유형의 패턴에 사용할 수 있도록 Optional은 ``map`` 메서드를 지원한다.  
Optional의 ``map``은 스트림의 map 메서드와 개념적으로 비슷하다. Optional 객체를 최대 요소의 개수가 한 개 이하인 데이터 컬렉션으로 생각할 수 있다. Optional이 값을 포함하면 map의 인수로 제공된 함수가 값을 바꾼다. Optional이 비어있으면 아무 일도 일어나지 않는다.  

### flatMap으로 Optional 객체 연결

```java
Optional<Person> optPerson = Optional.of(person);
Optional<String> name = optPerson.map(Person::getCar)
    							.map(Car::getInsurance)
    							.map(Insurance::getName);
```

위 코드는 컴파일되지 않는다. 변수 ``optPerson``의 형식은 ``Optional<Person>`` 이므로 map 메서드를 호출할 수 있다.  
그러나 ``getCar``는 ``Optional<Car>`` 형식의 객체를 반환한다. 즉, map 연산의 결과는 ``Optional<Optional<Car>>`` 형식의 객체다.  
``getInsurance``는 또 다른 Optional 객체를 반환하므로 ``getInsurance`` 메서드를 지원하지 않는다.  

스트림의 ``flatMap`` 은 함수를 인수로 받아서 다른 스트림을 반환하는 메서드다. 보통 인수로 받은 함수를 스트림의 각 요소에 적용하면 스트림의 스트림이 만들어진다. 하지만 ``flatMap``은 인수로 받은 함수를 적용해서 생성된 각각의 스트림에서 컨텐츠만 남긴다.  
즉, 함수를 적용해서 생성된 모든 스트림이 하나의 스트림으로 병합되어 평준화된다.  

한 요소가 스트림의 결과를 가질 때에 3번 반복될 시에 세 개의 스트림을 포함하는 하나의 스트림이 생성되지만, ``flatMap``은 이를 하나의 스트림으로 바꿔준다. 마찬가지로 Optional의 ``flatMap`` 메서드로 전달된 함수는 이차원 Optional이 하나의 삼각형을 포함하는 하나의 Optional로 바뀐다.  

#### Optional로 자동차의 보험회사 이름 찾기

```java
public String getCarInsuranceName(Optional<Person> person) {
    return person.flatMap(Person::getCar)
        		.flatMap(Car::getInsurance)
        		.map(Insurance::getName)
        		.orElse("Unknown");	//Optional이 비어있으면 기본값 사용
}
```

이제 위와 같이 사용하면 조건 분기문을 추가해서 코드를 복잡하게 만들지 않으면서도 쉽게 이해할 수 있는 코드가 완성된다.  

### Optional 스트림 조작

```java
public Set<String> getCarInsuranceName(List<Person> persons) {
    return persons.stream()
        .map(Person::getCar)//사람 목록을 각 사람이 보유한 자동차의 Optional<Car> 스트림으로 변환
        .map(optCar -> optCar.flatMap(Car::getInsurance))//flatMap으로 Optional<Insurance>로 변환
        .map(optIns -> optIns.map(Insurance::getName))//Optional<String>으로 매핑
        .flatMap(Optional::stream)//Stream<Optional<String>>을 현재 이름을 포함하는 Stream<String>으로 변환
        .collect(toSet());
}
```

마지막 결과를 얻으려면 빈 Optional을 제거하고 값을 언랩해야 한다.  

```java
Stream<Optional<String>> stream =...
Set<String> result = stream.filter(Optional::isPresent)
    						.map(Optional::get)
    						.collect(toSet());
```

다음과 같이 ``filter``, ``map``을 순차적으로 이용해 결과를 얻을 수 있다.  

하지만 위의 첫 번째 코드와 같이 Optional 클래스의 ``stream()`` 메서드를 이용하면 한 번의 연산으로 같은 결과를 얻을 수 있다.  
이 메서드는 각 Optional이 비어있는지 아닌지에 따라 Optional을 0개 이상의 항목을 포함하는 스트림으로 변환한다. ㄸ라서 이 메서드의 참조를 스트림의 한 요소에서 다른 스트림으로 적용하는 함수로 볼 수 있으며 이를 원래 스트림에 호출하는 ``flatMap`` 메서드로 전달할 수 있다. 이 기법을 이용하면 한 단계의 연산으로 값을 포함하는 Optional을 언랩하고 비어있는 Optional은 건너뛸 수 있다.  

### 디폴트 액션과 Optional 언랩

* ``get()`` 은 값을 읽는 가장 간단한 메서드면서 동시에 가장 안전하지 않은 메서드다. get은 래핑된 값이 있으면 해당 값을 반환하고 값이 없으면 ``NoSuchElementException``을 발생시킨다. 따라서 Optional에 값이 반드시 있다고 가정할 수 있는 상황이 아니면 get 메서드를 사용하지 않는 것이 바람직하다. 결국 이 상황은 중첩된 null 확인 코드를 넣는 상황과 크게 다르지 않다.
* ``orElse`` 메서드를 이용하면 Optional이 값을 포함하지 않을 때 기본값을 제공할 수 있다. 
* ``orElseGet(Supplier<? extends T> other)`` 는 orElse 메서드에 대응하는 게으른 버전의 메서드다. Optional에 값이 없을 때만 Supplier가 실행되기 때문이다. 디폴트 메서드를 만드는 데 시간이 걸리거나 Optional이 비어있을 때만 기본값을 생성하고 싶다면 이를 사용하면 된다.
* ``orElseThrow(Supplier<? extends X> exceptionSupplier)``는 Optional이 비어있을 때 예외를 발생시킨다는 점에서 get 메서드와 비슷하다. 하지만 이 메서드는 발생시킬 예외의 종류를 선택할 수 있다.
* ``ifPresent(Consumer<? super T> consumer)`` 를 이용하면 값이 존재할 때 인수로 넘겨준 동작을 실행할 수 있다. 값이 없으면 아무 일도 일어나지 않는다.
* ``ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction)``. 이 메서드는 Optional이 비어있을 때 실행할 수 있는 Runnable을 인수로 받는다는 점만 ifPresent와 다르다.

### 두 Optional 합치기

Person과 Car 정보를 이용해서 가장 저렴한 보험료를 제공하는 보험회사를 찾는 몇몇 복잡한 비즈니스 로직을 구현한 외부 서비스가 있다고 가정하자.  

```java
public Insurance findCheapestInsurance(Person person, Car car) {
    //다양한 보험회사가 제공하는 서비스 조회
    //모든 결과 데이터 비교
    return cheapestCompany;
}
```

이제 두 Optional을 인수로 받아서 ``Optional<Insurance>``를 반환하는 null 안전 버전의 메서드를 구현해야 한다고 가정하자.  
인수로 전달한 값 중 하나라도 비어있으면 빈 Optional<Insurance>를 반환한다.  

```java
public Optional<Insurance> nullSafeFindCheapestInsurance(Optional<Person> person, Optional<Car> car) {
    if(person.isPresent() && car.isPresent()) {
        return Optional.of(findCheapestInsurance(person.get(), car.get()));
    } else {
        return Optional.empty();
    }
}
```

이 메서드의 장점은 person과 car의 시그니처만으로 둘 다 아무 값도 반환하지 않을 수 있다는 정보를 명시적으로 보여준다는 것이다.  
그러나 구현 코드는 null 확인 코드와 크게 다른 점이 없다.  

### 필터로 특정값 거르기

```java
Insurance insurance = ...;
if(insruance != null && "CambridgeInsurance".equals(insurance.getName())) {
    System.out.println("ok");
}

=>
    
Optional<Insurance> optInsurance = ...;
optInsurance.filter(insurance -> "CambridgeInsurance".equals(insurance.getName()))
    		.ifPresent(x -> System.outprintln("ok"));
```

다음과 같이 filter를 이용하여 코드를 재구현할 수 있다.  
filter 메서드는 프레딬이트를 인수로 받는다. Optional이 비어있다면 filter 연산은 아무 동작도 하지 않는다. Optional에 값이 있으면 그 값에 프레디케이트를 적용한다. 프레디케이트 적용 결과가 true면 Optional에는 아무 변화도 일어나지 않는다. 하지만 결과가 false면 값은 사라져버리고 Optional은 빈 상태가 된다.  

Optional 클래스의 메서드  

| 메서드          | 설명                                                         |
| --------------- | ------------------------------------------------------------ |
| empty           | 빈 Optional 인스턴스 반환                                    |
| filter          | 값이 존재하며 프레디케이트와 일치하면 값을 포함하는 Optional을 반환하고, 값이 없거나 프레디케이트와 일치하지 않으면 빈 Optional을 반환함 |
| flatMap         | 값이 존재하면 인수로 제공된 함수를 적용한 결과 Optional을 반환하고, 값이 없으면 빈 Optional을 반환함 |
| get             | 값이 존재하면 Optional이 감싸고 있는 값을 반환하고, 값이 없으면 NoSuchElementException이 발생함 |
| ifPresent       | 값이 존재하면 지정된 Consumer를 실행하고 값이 없으면 아무 일도 일어나지 않음 |
| ifPresentOrElse | 값이 존재하면 지정된 Consumer를 실행하고 값이 없으면 아무 일도 일어나지 않음 |
| isPresent       | 값이 존재하면 true를 반환하고, 값이 없으면 flase를 반환함    |
| map             | 값이 존재하면 제공된 매핑 함수를 적용함                      |
| of              | 값이 존재하면 값을 감싸는 Optional을 반환하고, 값이 null이면 NullPointerException을 발생함 |
| ofNullable      | 값이 존재하면 값을 감싸는 Optional을 반환하고, 값이 null이면 빈 Optional을 반환함 |
| or              | 값이 존재하면 같은 Optional을 반환하고, 값이 없으면 Supplier에서 만든 Optional을 반환 |
| orElse          | 값이 존재하면 값을 반환하고, 값이 없으면 기본값을 반환함     |
| orElseGet       | 값이 존재하면 값을 반환하고, 값이 없으면 Supplier에서 제공하는 값을 반환함 |
| orElseThrow     | 값이 존재하면 값을 반환하고, 값이 없으면 Supplier에서 생성한 예외를 발생함 |
| stream          | 값이 존재하면 존재하는 값만 포함하는 스트림을 반환하고, 값이 없으면 빈 스트림을 반환 |

<br/>

## Optional을 사용한 실용 예제

### 잠재적으로 null이 될 수 있는 대상을 Optional로 감싸기

``Object value = map.get("key");`` 문자열 "key"에 해당하는 값이 없으면 null이 반환될 것이다. 이 경우 if-then-else를 추가하거나, ofNullable를 이용하는 두 가지 방법이 있다.  

``Optional<Object> value = Optional.ofNullable(map.get("key"));``  

### 예외와 Optional 클래스

자바 API는 어떤 이유에서 값을 제공할 수 없을 때 null 대신 예외를 발생시킬 때도 있다.  
예로들면, ``Integer.parseInt(String)`` 의 경우 문자열을 정수로 바꾸지 못할 때 ``NumberFormatException``을 발생시킨다.  
기존에 값이 null일 수 있을때는 if문으로 null 여부를 확인했ㅈ만 예외를 발생시키는 메서드에서는 try/catch 블록을 사용해야 한다는 점이 다르다.  

```java
public static Optional<Integer> stringToInt(String s) {
    try{
        return Optional.of(Integer.parseInt(s));
    } catch (NumberFormatException e) {
        return Optional.empty();
    }
}
```

다음과 같은 메서드를 포함하는 유틸리티 클래스 ``OptionalUtility``를 만들면 문자열을 ``Optional<Integer>``로 변환할 수 있고,  
기존처럼 거추장스러운 try/catch 로직을 사용할 필요가 없다.  

### 기본형 Optional을 사용하지 말아야 하는 이유

Optional의 최대 요소 수는 한 개이므로 Optional에서는 기본형 특화 클래스로 성능을 개선할 수 없다.  
Optional 클래스의 유용한 메서드 ``map``, ``flatMap``, ``filter`` 등을 기본형 특화 Optional에서는 지원하지 않으므로 사용을 지양해야 한다.  
그리고 기본형 특화 Optional로 생성한 결과는 다른 일반 Optional과 혼용할 수 없다.  

### 응용

프로그램의 설정 인수로 Properties를 전달한다고 가정하자.  

```java
Properties props = new Properties();
props.setProperty("a", "5");
props.setProperty("b", "true");
props.setProperty("c", "-3");
```

프로그램에서는 Properties를 읽어서 값을 초 단위의 지속 시간으로 해석한다. 다음과 같은 메서드 시그니처로 지속 시간을 읽을 것이다.  

``public int readDuration(Properties props, String name)``   
이를 구현해보면,  

```java
public int readDuration(Properties props, String name) {
    String value = props.getProperty(name);
    if(value != null) {
        try{
            int i = Integer.parseInt(value);
            if(i > 0) {
                return i;
            }
        } catch(NumberFormatException e) {}
    }
    return 0;
}
```

if 문과 try/catch 블록이 중첩되면서 구현 코드가 복잡해졌고 가독성이 나빠졌다.  

```java
public int readDuration(Properties props, String name) {
    return Optional.ofNullable(props.getProperty(name))
        			.flatMap(OptionalUtility::stringToInt)
        			.filter(i -> i > 0)
        			.orElse(0);
}
```

***