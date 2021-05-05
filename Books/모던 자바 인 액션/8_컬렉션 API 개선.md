# 컬렉션 API 개선

## 컬렉션 팩토리

```java
List<String> friends = new ArrayList<>();
friends.add("Raphael");
friends.add("Olivia");
friends.add("Thibaut");
//이 코드는 Arrays.asList() 팩토리 메서드로 간단히 줄일 수가 있다.
List<String> friends = Arrays.asList("Raphael", "Olivia", "Thibaut");
```

고정 크기의 리스트를 만들었기 때문에 요소를 갱신할 순 있지만 새 요소를 추가하거나 요소를 삭제할 순 없다.  
만약 요소를 추가하려하면 ``UnsupportedOperationException``이 발생한다. 내부적으로 고정된 크기의 변환할 수 있는 배열로 구현되었기 때문에 일어나는 일이다. set과 map의 경우 내부적으로 불필요한 객체 할당을 필요로 한다.  

```java
List<String> list = new ArrayList<>(Arrays.asList("a","b","c"));
list.add("d");
//이렇게 만들면 괜찮긴한데... 책을 일단 따르자!
```

자바 9에서는 작은 리스트, 집합, 맵을 쉽게 만들 수 있는 팩토리 메서드를 제공한다.  

### 리스트 팩토리

List.of 팩토리 메서드를 이용해서 간단하게 리스트를 만들 수 있다.  

```java
List<String> friends = List.of("a", "b", "c");
System.out.println(friends);
```

그러나 추가하려고 하면 역시나 ``UnsupportedOperationException``이 발생한다. 이런 제약이 꼭 나쁜 것만은 아니다.  
컬렉션이 의도치 않게 변하는 것을 막을 수 있기 때문이다. 하지만 요소 자체가 변하는 것을 막을 수 있는 방법은 없다.  

### 집합 팩토리

List.of와 비슷한 방법으로 바꿀 수 없는 집합을 만들 수 있다.  

```java
Set<String> friends = Set.of("a", "a", "c");
System.out.println(friends);
```

마찬가지로 추가할 수 없으며 만약 중복된 요소가 위와 같이 들어있다면 ``IllegalArgumentException``이 발생한다.  

### 맵 팩토리

```java
Map<String, Integer> map = Map.ofEntries(entry("a", 1),
                                         entry("b", 2),
                                         entry("c", 3));
```

다음과 같이 ``Map.ofEntries`` 팩토리 메서드를 이용해서 맵을 만들 수 있다.  

<br/>

## 리스트와 집합 처리

자바 8에서는 List, Set 인터페이스에 다음과 같은 메서드를 추가했다.  

* ``removeIf`` : 프레디케이트를 만족하는 요소를 제거한다. List나 Set을 구현하거나 그 구현을 상속받은 모든 클래스에서 이용할 수 있다.
* ``replaceAll`` : 리스트에서 이용할 수 있는 기능으로 UnaryOperator 함수를 이용해 요소를 바꾼다.
* ``sort`` : List 인터페이스에서 제공하는 기능으로 리스트를 정렬한다.

컬렉션을 바꾸는 동작은 에러를 유발하며 복잡함을 더하는데 왜 ``removeIf`` 나 ``replaceAll``를 추가했을까??  

### removeIf 메서드

다음은 숫자로 시작되는 참조 코드를 가진 트랜잭션을 삭제하는 코드다.  

```java
for (Transaction transaction : transactions) {
    if (Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        transactions.remove(transaction);
    }
}
```

위의 코드는 ``ConcurrentModificationException``을 일으킨다. 내부적으로 for-each 루프는 Iterator 객체를 사용하므로 밑의 코드와 같이 해석된다.  

```java
for(Iterator<Transaction> iterator = transactions.iterator(); iterator.hasNext(); ) {
    Transaction transaction = iterator.next();
    if(Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        transactions.remove(transaction);
    }
}
```

두 개의 개별 객체가 컬렉션을 관리하고 있다.  

* Iterator 객체, ``next()``, ``hasNext()``를 이용해 소스를 질의한다.
* Collection 객체 자체, ``remove()``를 호출해 요소를 삭제한다.

결과적으로 반복자의 상태는 컬렉션의 상태와 서로 동기화되지 않는다.  
Iterator 객체를 명시적으로 사용하고 그 객체의 ``remove()`` 메서드를 호출함으로 이 문제를 해결할 수 있다.  

```java
for(Iterator<Transaction> iterator = transactions.iterator; iterator.hasNext(); ) {
    Transaction transaction = iterator.next();
    if(Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        iterator.remove();
    }
}
```

코드가 조금 복잡해졌다. 이 때 ``removeIf`` 메서드로 해결할 수 있다.  

```java
transactions.removeIf(transaction ->
                     Character.isDigit(transaction.getReferenceCode().charAt(0)));
```

### replaceAll 메서드

List 인터페이스의 ``replaceAll`` 메서드를 이용해 리스트의 각 요소를 새로운 요소로 바꿀 수 있다.  
스트림 API를 사용하면 다음처럼 문제를 해결할 수 있었다.  

```java
referenceCodes.stream()
    .map(code -> Character.toUpperCase(code.charAt(0)) +
        code.substring(1))
    .collect(Collectors.toList())
    .forEach(System.out::println);
```

하지만 이 코드는 새 문자열 컬렉션을 만든다. 우리가 원하는 것은 기존 컬렉션을 바꾸는 것이다. 다음처럼 ListIterator 객체(요소를 바꾸는 ``set()`` 메서드를 지원)를 이용할 수 있다.  

```java
for(ListIterator<String> iterator = referenceCodes.listIterator()); iterator.hasNext(); ) {
    String code = iterator.next();
    iterator.set(Character.toUpperCase(code.charAt(0)) + code.substring(1));
}
```

코드가 조금 복잡해졌다. 이전 설명과 같이 컬렉션 객체를 Iterator 객체와 혼용하면 반복과 컬렉션 변경이 동시에 이루어지면서 쉽게 문제를 일으킨다. 자바 8의 기능을 이용해서 간단하게 구현할 수 있다.  

```java
referenceCodes.replaceAll(code -> Character.toUpperCase(code.charAt(0)) + code.substring(1));
```

<br/>

## 맵 처리

### forEach 메서드

맵에서 키와 값을 반복하면서 확인하는 작업은 귀찮은 작업이다. ``Map.Entry<K, V>``의 반복자를 이용해 반복 가능하다.  

```java
for(Map.Entry<String, Integer> entity: ageOfFriends.entrySet()) {
    String friend = entry.getKey();
    Integer age = entry.getValue();
    System.out.println(friend + " is " + age + " years old");
}
```

자바 8에서부터는 Map 인터페이스는 BiConsumer (키와 값을 인수로 받음)를 인수로 받는 forEach 메서드를 지원한다.  

```java
ageOfFriends.forEach((friend, age) -> System.out.println(friend + " is " + age + " years old"));
```

### 정렬 메서드

다음 두 개의 새로운 유틸리티를 이용하면 맵의 항목을 값 또는 키를 기준으로 정렬할 수 있다.  

* ``Entry.comparingByValue``
* ``Entry.comparingByKey``

```java
Map<String, Integer> map = new HashMap<>(Map.ofEntries(entry("kang", 26), entry("choi", 27)));
map.put("lee", 14);
map.put("abcd", 26);


map.entrySet().stream()
    .sorted(Map.Entry.comparingByValue())
    .forEachOrdered(System.out::println);
/*
lee=14
kang=26
abcd=26
choi=27

forEachOrdered는 스트림이 순차적인지 병렬적인지에 관계없이 스트림의 순서대로 스트림의 요소를 처리하게끔 한다.
위 예제에서 forEach로 하면 kang과 abcd의 순서가 바뀌더라...
/*
```

### getOrDefault 메서드

기존에는 찾으려는 키가 존재하지 않으면 널이 반환되므로 ``NullPointerException``을 방지하려면 요청 결과가 널인지 확인해야 한다.  
``getOrDefault`` 메서드를 이용하면 쉽게 해결가능한 문제이다. 이 메서드는 1,2 번째 인수로 키와 기본값을 받으며 맵에 키가 존재하지 않으면 두 번째 인수로 받은 기본값을 반환한다.  

```java
Map<String, Integer> friends = new HashMap<>(Map.ofEntries(
    entry("kang", 26), entry("choi", 27),
    entry("lee", 15), entry("min", 22)));

System.out.println(friends.getOrDefault("kang", 50));
System.out.println(friends.getOrDefault("kim", 40));

// 50
// 40
```

## 계산 패턴

맵에 키가 존재하는지 여부에 따라 어떤 동작을 실행하고 결과를 저장해야 하는 상황이 필요한 때가 있다.  
예를 들어 키를 이용해 값비싼 동작을 실행해서 얻은 결과를 캐시하려 한다. 키가 존재하면 결과를 다시 계산할 필요가 없다.  
다음의 세 가지 연산이 이런 상황에서 도움을 준다.

* ``computeIfAbsent`` : 제공된 키에 해당하는 값이 없으면 (값이 없거나 널), 키를 이용해 새 값을 계산하고 맵에 추가한다.
* ``computeIfPresent`` : 제공된 키가 존재하면 새 값을 계산하고 맵에 추가한다.
* ``compute`` : 제공된 키로 새 값을 계산하고 맵에 저장한다.

정보를 캐시할 때 ``computeIfAbsent``를 활용할 수 있다. 파일 집합의 각 행을 파싱해 SHA-256을 계산한다고 가정하자. 기존에 이미 데이터를 처리했다면 이 값을 다시 계산할 필요가 없다. 맵을 이용해 캐시를 구현했다고 가정하면 다음처럼 MessageDigest 인스턴스로 SHA-256 해시를 계산할 수 있다.  

```java
Map<String, byte[]> dataToHash = new HashMap<>();
MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
lines.forEach(line ->
              dataToHash.computeIfAbsent(line, this::calculateDigest));

private byte[] calculateDigest(String key) {
    return messageDigest.digest(key.getBytes(StandardCharsets.UTF_8));
}
```

여러 값을 저장하는 맵을 처리할 때도 이 패턴을 유용하게 활용할 수 있다.  

```java
String friend = "Raphael";
List<String> movies = friendsToMovies.get(friend);
if(movies == null) {
    movies = new ArrayList<>();
    friendsToMovies.put(friend, movies);
}
movies.add("Star Wars");
```

``computeIfAbsent``는 키가 존재하지 않으면 값을 계산해 맵에 추가하고 키가 존재하면 기존 값을 반환한다.  

```java
friendsToMovie.computeIfAbsent("Raphael", name -> new ArrayList<>()).add("Star Wars");
```

``computeIfAbsent``메서드는 현재 키와 관련된 값이 맵에 존재하며 널이 아닐 때만 새 값을 계산한다.  

메서드 스펙 등 자세한 내용은 [Map의 Collection API에 대해](https://github.com/Be-poz/TIL/blob/master/Java/Map%EC%9D%98%20Collection%20API%20%EC%97%90%20%EB%8C%80%ED%95%B4.md) 참조하면 되겠다.  

### 삭제 패턴

```java
String key ="Raphael";
String value = "Jack Reacher 2";
if(favouriteMovies.containsKey(key) && Objects.equals(favouriteMovies.get(key), value)) {
    favouriteMovies.remove(key);
    return true;
}else{
    return false;
}

=>
favouriteMovies.remove(key, value);	//key, value 값 삭제 여부 반환함. key로만 삭제하는 오버로드된 remove도 있음
```

### 교체 패턴

* ``replaceAll`` : BiFunction을 적용한 결과로 각 항목의 값을 교체한다. 이 메서드는 이전에 살펴본 List의 ``replaceAll``과 비슷한 동작을 수행한다.
* ``replace`` : 키가 존재하면 맵의 값을 바꾼다. 키가 특정 값으로 매핑되었을 때만 값을 교체하는 오버로드 버전도 있다.

```java
Map<String, Integer> friends = new HashMap<>(Map.ofEntries(
    entry("kang", 26), entry("choi", 27),
    entry("lee", 15), entry("min", 22)));

friends.replaceAll((key, value) -> value + 10);
friends.replace("kang", 30);
friends.forEach((name, age) -> System.out.println(name + " " + age));
/*
choi 37
min 32
kang 30
lee 25
*/
```

### 합침

두 그룹의 연락처를 포함하는 두 개의 맵을 합친다고 할 때 ``putAll``을 사용할 수 있다.  

```java
Map<String, String> family = Map.ofEntries(
    entry("Teo", "Star Wars"), entry("Cristina", "James Bond"));

Map<String, String> friend = Map.ofEntries(entry("Raphael", "Star Wars"));
Map<String, String> everyone = new HashMap<>(family);
everyone.putAll(friend);
System.out.println(everyone);
//{Cristina=James Bond, Raphael=Star Wars, Teo=Star Wars}
```

중복된 키가 없다면 위 코드는 잘 동작한다. 값을 좀 더 유연하게 합쳐야 한다면 새로운 ``merge`` 메서드를 이용할 수 있다.  
이 메서드는 중복된 키를 어떻게 합칠지 결정하는 BiFunction을 인수로 받는다.  

```java
Map<String, String> family = Map.ofEntries(
    entry("Teo", "Star Wars"), entry("Cristina", "James Bond"));

Map<String, String> friend = Map.ofEntries(
    entry("Raphael", "Star Wars"), entry("Cristina", "Matrix"));

Map<String, String> everyone = new HashMap<>(family);
friend.forEach((key, value) ->
               everyone.merge(key, value, (movie1, movie2) -> movie1 + " & " + movie2));

everyone.merge("Teo", "whats up", (movie1, movie2) -> movie1 + " & " + movie2);
System.out.println(everyone);
//{Raphael=Star Wars, Cristina=James Bond & Matrix, Teo=Star Wars & whats up}
```

``merge`` 메서드는 널값과 관련된 복잡한 상황도 처리한다.  

> "지정된 키와 연관된 값이 없거나 값이 널이면 [merge]는 키를 널이 아닌 값과 연결한다. 아니면 [merge]는 연결된 값을 주어진 매핑 함수의 [결과] 값으로 대치하거나 결과가 널이면 [항목]을 제거한다."

``merge``를 이용해 초기화 검사를 구현할 수도 있다.  

```java
Map<String, Long> moviesToCount = new HashMap<>();
String movieName = "JamesBond";
long count = moviesToCount.get(movieName);
if (count == null) {
    moviesToCount.put(movieName, 1);
} else {
    moviesToCount.put(movieName, count + 1);
}

=>
    
moviesToCount.merge(movieName, 1L, (key, count) -> count + 1L);
```

"키와 연관된 기존 값에 합쳐질 널이 아닌 값 또는 값이 없거나 키에 널 값이 연관되어 있다면 이 값을 키와 연결"하는데 사용된다.  
키의 반환값이 널이므로 처음에는 1이 사용된다. 그 다음부터는 값이 1로 초기화되어 있으므로 BiFunction을 적용한다.  

<br/>

## 개선된 ConcurrentHashMap

``ConcurrentHashMap``클래스는 동시성 친화적이며 최신 기술을 반영한 HashMap 버전이다.  
``ConcurrentHashMap``은 내부 자료구조의 특정 부분만 잠궈 동시 추가, 갱신 작업을 허용한다. 따라서 동기화된 Hashtable 버전에 비해 읽기 쓰기 연산 성능이 월등하다.  

### 리듀스와 검색

``ConcurrentHashMap``은 스트림에서 봤던 것과 비슷한 종류의 세 가지 새로운 연산을 지원한다.  

* forEach : 각 (키, 값) 쌍에 주어진 액션을 실행
* reduce : 모든 (키, 값) 쌍을 제공된 리듀스 함수를 이용해 결과로 합침
* search : 널이 아닌 값을 반환할 때까지 각 (키,값) 쌍에 함수를 적용

다음처럼 키에 함수 받기, 값, Map.Entry, (키, 값) 인수를 이용한 네 가지 연산 형태를 지원한다.  

* 키, 값으로 연산(forEach, reduce, search)
* 키로 연산(forEachKey, reduceKeys, searchKeys)
* 값으로 연산(forEachValue, reduceValues, searchValues)
* Map.Entry 객체로 연산(forEachEntry, reduceEntries, searchEntries)

이들 연산은 ``ConcurrentHashMap``의 상태를 잠그지 않고 연산을 수행한다는 점을 주목해야 한다. 따라서 이들 연산에 제공한 함수는 계산이 진행되는 동안 바뀔 수 있는 객체, 값, 순서 등에 의존하지 않아야 한다.  

또한 이들 연산에 병렬성 기준값을 지정해야 한다. 맵의 크기가 주어진 기준값보다 작으면 순차적으로 연산을 실행한다. 기준값을 1로 지정하면 공통 스레드 풀을 이용해 병렬성을 극대화한다. Long.MAX_VALUE를 기준값으로 설정하면 한 개의 스레드로 연산을 실행한다. 여러분의 소프트웨어 아키텍처가 고급 수준의 자원 활용 최적화를 사용하고 있지 않다면 기준값 규칙을 따르는 것이 좋다.  

```java
ConcurrentHashMap<String, Long> map = new ConcurrentHashMap<>();
long parallelismThreshold = 1;
Optional<Integer> maxValue = 
    Optional.ofNullable(map.reduceValues(parallelismThreshold, Long::max));
```

### 계수

``ConcurrentHashMap`` 클래스는 맵의 매핑 개수를 반환하는 ``mappingCount`` 메서드를 제공한다. 기존의 size 메서드 대신 새 코드에서는 int를 반환하는 ``mappingCount`` 메서드를 사용하는 것이 좋다. 그래야 개수가 int의 범위를 넘어서는 이후의 상황을 대처할 수 있기 때문이다.  

### 집합뷰

``ConcurrentHashMap`` 클래스는 ``ConcurrentHashMap``을 집합 뷰로 반환하는 ``keySet``이라는 새 메서드를 제공한다. 맵을 바꾸면 집합도 바뀌고 반대로 집합을 바꾸면 맵도 영향을 받는다. ``newKeySet``이라는 새 메서드를 이용해 ``ConcurrentHashMap``으로 유지되는 집합을 만들 수도 있다.  

***