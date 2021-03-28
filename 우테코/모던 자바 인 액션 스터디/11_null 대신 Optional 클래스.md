# 11. null 대신 Optional 클래스

대부분의 프로그래머는 null 확인 코드를 추가해서 null 예외 문제를 해결하려 든다.  

변수를 접근할 때마다 중첩된 if가 추가되면서 코드 들여쓰기 수준이 증가한다. 이와같은 반복 패턴코드를 '깊은 의심'이라고 부른다. 이를 반복하다보면 코드의 구조가 엉망이 되고 가독성이 떨어진다.  

null로 인해서 다음과 같은 문제가 발생한다.  

* 에러의 근원이다
* 코드를 어지럽힌다
* 아무 의미가 없다
* 자바 철학에 위배된다
* 형식 시스템에 구멍을 만든다

자바 8에서는 Optional이라는 새로운 클래스를 제공한다. Optional은 선택형값을 캡슐화하는 클래스다. ``Optional<Car>`` 처럼 객체를 감싼다. 반면 값이 없으면 ``Optional.empty`` 메서드로 Optional을 반환한다. null을 참조하려 하면 NPE가 발생하지만, ``Optional.empty()``는 Optional 객체이므로 이를 다양한 방식으로 활용할 수 있다.  

1. 빈 Optional을 만드는 법 : ``Optional<Car> optCar = Optional.empty();``
2. null이 아닌 값으로 Optional 만들기 : ``Optional<Car> optCar = Optional.of(car);``
3. null 값으로 Optional 만들기 : ``Optional<Car> optCar = Optional.ofNullable(car);`` 로 null 값을 저장할 수 있는 Optional을 만들 수 있다. car가 null이면 빈 Optional 객체가 반환된다. 

```java
String name = null;
if(insruance != null) {
    name = insurance.getName();
}

Optional<Insurance> optInsurance = Optional.ofNullable(insurance);
Optional<String> name = optInsurance.map(Insurance::getName);
```

다음과 같이 표현 가능하다.  

만약 getName이 내부에서 Optional로 감싼 Name을 리턴하고 Name 안에서도 Optional로 감싸고 그런 경우에는  
``Optional<Optional<Name>>`` 다음과 같은 반환형식을 갖게될 것이다. 이 때에는 ``flatMap``을 사용해서 해결해주자.  

```java
    static class Alphabets {
        private final String alphabets;

        public Alphabets(String name) {
            this.alphabets = name;
        }

        public String getAlphabets() {
            return alphabets;
        }
    }

    static class Name {
        private final Optional<Alphabets> alphabets;

        public Name(String name) {
            this.alphabets = Optional.of(new Alphabets(name));
        }

        public Optional<Alphabets> getAlphabets() {
            return alphabets;
        }
    }

    static class Fruit {
        private final Optional<Name> name;

        public Fruit(String name) {
            this.name = Optional.of(new Name(name));
        }

        public Optional<Name> getName() {
            return name;
        }
    }

    public static void main(String[] args) {
        Optional<Fruit> optFruit = Optional.of(new Fruit("apple"));
        Optional<Name> name = optFruit.flatMap(Fruit::getName);
        Optional<Alphabets> alphabets = name.flatMap(Name::getAlphabets);
        alphabets.map(Alphabets::getAlphabets).orElse("asdf");
    }
```

이런 느낌으로 말이다.  

```java
        Optional<Fruit> optFruit = Optional.of(new Fruit("apple"));
        List<String> collect = optFruit.stream()
                .map(Fruit::getName)
                .map(optName -> optName.flatMap(Name::getAlphabets))
                .map(optAlpha -> optAlpha.map(Alphabets::getAlphabets))
                .flatMap(Optional::stream)
//                .map(Optional::get)
                .collect(Collectors.toList());
```

또는 다음과 같은 방식으로 스트림을 이용해 사용할 수도 있다.  

* ``get()`` 은 값을 읽는 가장 간단한 메서드면서 동시에 가장 안전하지 않은 메서드다. NPE가 발생할 수 있다.  
* ``orElse(T other)`` 를 이용하면 Optional이 값을 포함하지 않을 때 기본값을 제공할 수 있다.
* ``orElseGet(Supplier<? extends T> other)``는 ``orElse`` 메서드에 대응하는 게으른 버전의 메서드다. Optional에 값이 없을 때만 Supplier가 실행되기 때문이다.  
* ``orElseThrow(Upplier<? extends X> exceptionSupplier)``는 Optional이 비어있을 때 예외를 발생시킨다는 점에서 get 메서드와 비슷하다. 하지만 이 메서드는 발생시킬 예외의 종류를 선택할 수 있다.
* ``ifPresent(Consumer<? super T> consumer)``를 이용하면 값이 존재할 때 인수로 넘겨준 동작을 실행할 수 있다. 값이 없으면 아무 일도 일어나지 않는다.

``filter`` 는 값이 존재하면 인수로 제공된 함수를 적용한 결과 Optional을 반환하고, 값이 없으면 빈 Optional을 반환한다.  

```java
        Optional<Alphabets> optAlpha = Optional.of(new Alphabets("asdf"));
        optAlpha.filter(alphabets -> "asdfd".equals(alphabets.getAlphabets()))
                .ifPresent(alphabets -> System.out.println("exist"));
// 필터의 프레디케이트와 일치한다면 그대로 ifPresent을 수행하겠지만 그 값이 false라면 값은 사라져 버릴 것이다.
// Optional이 비어있다면 filter 연산은 아무 동작도 하지 않는다.
```

기본형 Optional은 성능 개선을 기대할 수 없고 map, flatMap, filter 등을 지원하지 않으므로 사용을 지양해야 한다.  

***