# 8. 컬렉션 API 개선

Arrays.asList() 나 List.of 나 추가를 하려고 하면 UnsupportedOperationException이 발생하게된다.  

Set은 Set.of 로 생성한다.  

Map은 다음과 같이 2가지 방법이 있다.  

```java
Map<String, Integer> ageOfFriends = Map.of("Raphael", 30, "Olivia", 25);

Map<String, Integer> ageOfFriends = Map.ofEntries(entry("Raphael", 30),
									             entry("Olivia", 25));
```

자바 8에서는 List와 Set 인터페이스에 다음과 같은 메서드를 추가했다.  

* removeIf : 프레디케이트를 만족하는 요소를 제거한다. List나 Set을 구현하거나 그 구현을 상속받은 모든 클래스에서 이용할 수 있다.
* replaceAll : 리스트에서 이용할 수 있는 기능으로 UnaryOperator 함수를 이용해 요소를 바꾼다.
* sort : List 인터페이스에서 제공하는 기능으로 리스트를 정렬한다.

removeIf는 동기화 문제를 해결하기 위해 만들어 졌다.  

```java
for(Transaction transaction : transactions) {
    if(Character.isDigit(transaction.getReferenceCode().charAt(0))){
        transactions.remove(transaction);
    }
}

=>
    
for(Iterator<Transaction> iterator = transactions.iterator(); iterator.hasNext; ) {
    Transaction transaction = iterator.next();
    if(CHaracter.isDigit(transaction.getReferenceCode().charAt(0))) {
        transactions.remove(transaction);
    }
}
```

위 코드에서 두 개의 개별 객체가 컬렉션을 관리한다. Iterator객체가 next(), hasNext()를 이용해 소스를 질의하고, Collection 객체 자체가 remove()를 호출해 요소를 제거한다. 결과적으로 반복자의 상태는 컬렉션의 상태와 서로 동기화되지 않는다. 이를 Iterator 객체를 명시적으로 사용하는 것으로 문제를 해결할 수 있다.  

```java
for(Iterator<Transaction> iterator = transations.iterator(); iterator.hasNext(); ) {
    Transaction transaction = iterator.next();
    if(Character.isDigit(transaction, getReferenceCode().charAt(0))) {
        iterator.remove();
    }
}

=>
    
transactions.removeIf(transaction -> Character.isDigit(transaction.getReferenceCode().charAt(0)));
```

replaceAll은 각 요소를 새로운 요소로 바꾸어서 새 컬렉션을 만드는 것이 아니라 기존 컬렉션을 바꾸는 것이다. 표현은 다음과 같다.  

```java
referenceCodes.replaceAll(code -> Character.toUpperCase(code.charAt(0) + code.substring(1)));
```

Map은 BiConsumer를 인수로 받는 forEach로 코드를 가지고 놀 수 있다.(하지만, forEach는 지양해야한다)  

Map은 자바 8에 들어서 같은 키 값일 때 LinkedList에서 정렬된 트리를 이용하도록 바뀌었다.  

그리고 ``sorted(Entry.comparingByKey() / Entry.comparingByValue)`` 를 통해 key 나 value를 기준으로 정렬을 할 수 있다.  

``get(key)`` 시에 해당 key 값이 존재하지 않을 수도 있으므로 ``getOrDefault(key, default 값)`` 이렇게 사용할 수도 있다.  

``replaceAll``은 list와 유사하게 수행된다. BiConsumer를 인자로 가지고 있다.  

Map을 합칠 때에 ``putAll`` 대신 ``merge``를 이용해서 중복된 키에 대한 처리를 BiFunction으로 처리할 수 있다.  

``ConcurrentHashMap``은 동시성 친화적이며 최신 기술을 반영한 ``HashMap`` 버전이다. 내부 자료구조의 특정 부분만 잠궈 동시 추가, 갱신 작업을 허용한다. 맵의 매핑 개수를 반환할 때에는 ``size``대신 ``mappingCount`` 메서드를 사용하는 것이 매핑의 개수가 int의 범위를 넘어서는 이후의 상황을 대처할 수 있기에 좋다. ``forEach``, ``reduce``, ``search`` 세 가지 새로운 연산을 지원한다.  

***

### ``파즈``

8장 문제입니다.

1. `Arrays.asList`와`List.of` 의 차이점은 무엇일까요 ??
2. `map.put`과`map.replace`두메서드 다 값 변경이 가능한데 그렇다면 왜 `replace`라는 메서드가 따로 있을까요 ??
3. Map에는 `computeIfAbsent`이라는 메서드가 존재합니다. 만약 key가 존재하지 않는다면 키를 이용해서 새 값을 계산하고 맵을 추가하는 메서드입니다. 추가로, Map에는 `putIfAbsent`라는 메서드도 있습니다. 키가 존재 하지 않는다면 value를 바로 집어넣어주는 메서드입니다.

```java
    public static void main(String[] args) {
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.putIfAbsent(2, calculate(2));
        map.computeIfAbsent(2, key -> calculate(key));
    }    public static int calculate(int n) {
        return n + 5;
}
```

  그렇다면 위와 같은 상황에서 두 메서드는 어떤 차이점이 있을까요??
4.

```java
Map<String, String> family = new HashMap<>(Map.ofEntries(
        Map.entry("Teo", "Star Wars"),
        Map.entry("Cristina", "James Bond")
));family.merge("Teo", "asdf", (s, s2) -> null);
family.merge("Cristina", "JB2", (s, s2) -> s + " % " + s2);
family.merge("Bepoz", "Evans", (s, s2) -> s + "***" + s2);
```

출력 결과를 예상해 보세요.  

### 답안

1번 문제 답안

1. List.of 는 set으로 값 변경이 불가능하다.

```java
List<Integer> asList = Arrays.asList(1, 2, 3);
List<Integer> listOf = List.of(1, 2, 3);asList.set(0, 10);
listOf.set(0, 10);		//UnsupportedOperationException
```

List.of는 set으로 값 변경을 시도하면 컴파일 에러가 발생하게 된다.2. List.of 는 null을 허용하지 않는다.

```java
List<Integer> asList = Arrays.asList(1, 2, null);
List<Integer> listOf = List.of(1, 2, null);		//NPE
```

null을 받아들이는 Arryas.asList와 달리 List.of는 거부한다.3. List.of는 null 여부를 contains 확인도 못하게 한다.

```java
List<Integer> asList = Arrays.asList(1, 2, 3);
List<Integer> listOf = List.of(1, 2, 3);boolean asListResult = asList.contains(null);
boolean listOfResult = listOf.contains(null);		//NPE
```

\4. Arrays.asList는 원본의 배열의 변화에 반응한다.

```java
Integer[] arr = {1, 2, 3};List<Integer> asList = Arrays.asList(arr);
List<Integer> listOf = List.of(arr);arr[0] = 10;System.out.println(asList);
System.out.println(listOf);
/*
[10, 2, 3]
[1, 2, 3]
 */
```

arr의 값이 변하자 asList의 값 또한 변한 것을 확인할 수가 있다.만약 `new ArrayList<>(List.of / asList);` 를 쓰게되는 경우에는, 어떤 것을 사용하는지는 취향차이라고 생각한다.(반박 받음)  

2번 문제 답안
`map.put(key, value)` 는 해당 key 값에 value를 추가한다. 이미 존재하는 key 라면 value를 덮어씌운다.
`map.replace(key, value)` 는 해당 key 값을 value로 갈아치운다.
그러면 그냥 `put`을 사용하면 되는데 왜 굳이 `replace`라는 메서드가 존재하는 것일까 ??

```java
  default V replace(K key, V value) {
    V curValue;
    if (((curValue = get(key)) != null) || containsKey(key)) {
      curValue = put(key, value);
   }
    return curValue;
 }
```

`replace`의 세부내용은 다음과 같다. null이 아닐 때만 `put`을 실행을 하고 있다.

```java
    Map<String, Integer> map = new HashMap<>(Map.of("bepoz", 100));
    map.replace("giraffe", 150);
    System.out.println(map);    map.put("giraffe", 150);
    System.out.println(map);
/*
{bepoz=100}
{bepoz=100, giraffe=150}
```

`put`은 해당 key 값이 없으면 그대로 추가해버리지만, `replace`는 해당 key 값이 존재할 때만 바꿔주므로 조금 더 안정성이 있다는 것을 확인할 수가 있다.  

3번 문제 답안
`putIfAbsent`는 일단 `calculate`메서드를 실행하고 해당 결과를 파라미터로 보낸다.
반면, `computeIfAbsent`는 먼저 key 값의 존재 유무를 확인하고 없을 시에 `calculate`메서드가 실행이 된다.
만약, `map.putIfAbsent(2,2)`, `map.computeIfAbsent(2, key -> 2)` 일 경우에는 차이가 없을 것이다.  

4번 문제 답안
{Bepoz=Evans, Cristina=James Bond % JB2}

```java
  default V merge(K key, V value,
      BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
    Objects.requireNonNull(remappingFunction);
    Objects.requireNonNull(value);
    V oldValue = get(key);
    V newValue = (oldValue == null) ? value :
         remappingFunction.apply(oldValue, value);
    if (newValue == null) {
      remove(key);
   } else {
      put(key, newValue);
   }
    return newValue;
 }
```

newValue가 null 이면 `remove` 해버리기 때문에 Teo는 사라졌다.
Bepoz라는 기존 키가 없었기 때문에 바로 Evans로 추가되었다. Function을 거치지 않기에 *** 가 붙지 않았다.  

### ``중간곰``

많이 바쁘시죠? 간단한 문제를 준비했습니다.
여유 있으신 분들만 풀어보세요~ 
방법이 여러 개 있습니다. **stream은 쓰지 말고** 만들어보세요.8장 문제 ![:축구공:](https://a.slack-edge.com/production-standard-emoji-assets/13.0/google-medium/26bd@2x.png)️
2022년에 카타르에서 월드컵이 열린다.
도박을 좋아하는 사람들이 우승할 것 같은 국가에게 자신의 돈을 걸고 있다.
Toto 클래스는 각각 사람들이 우승을 예상하는 나라코드(nation)와 거는 돈(money)로 이루어져 있다.
totos 리스트에는 사람들의 토토 내역이 들어 가있다.
totos를 순회하며 stats에 나라별로 걸린 금액의 총합을 계산해보자!

```java
public class TotoApp {
    public static void main(String[] args) {
        List<Toto> totos = Arrays.asList(
                new Toto("br", 2000),
                new Toto("fr", 30000),
                new Toto("de", 20000),
                new Toto("fr", 1000),
                new Toto("kr", 15000),
                new Toto("jp", 10),
                new Toto("en", 1500),
                new Toto("kr", 7000),
                new Toto("kr", 4000),
                new Toto("br", 1000),
                new Toto("en", 1030),
                new Toto("br", 1000)
        );        Map<String, Integer> stats = new HashMap<>();
        for (Toto toto : totos) {
            String nation = toto.getNation();
            int money = toto.getMoney();
            // 이곳에 코드를 넣으시오.
            // ...
            // ...
            // ...
        }
        System.out.println(stats);
        // 기대 결과
        // {br=4000, de=20000, jp=10, kr=26000, en=2530, fr=31000}
    }    static class Toto {
        private String nation;
        private int money;        private Toto(final String nation, final int money) {
            this.nation = nation;
            this.money = money;
        }        public String getNation() {
            return nation;
        }        public int getMoney() {
            return money;
        }
    }
}
```

### 답안

```java
stats.merge(nation, money, Integer::sum);
```

### ``웨지``

8장 문제 대신에~~
**ConcurrentHashMap의 동기화 처리 방식**
Map interface를 구현한 3가지 구현체의 차이를 알아봅시다

- HashMap : Thread 에 안전하지 않다.
- Hashtable : Thread safe. 데이터 관련 함수에 synchronized 키워드가 선언 되어 있다.
- ConcurrentHashMap : Thread safe, 어떻게 Thread-safe를 보장하는지 알아보자

synchronized 키워드가 100배 정도 성능에 안 좋은 영향을 준다고 하죠?
Hashtable과는 다르게 ConcurrentHashMap은 주요 메소드마다 synchronized키워드가 선언되어있지는 않습니다.
put할 때 두 가지로 동작하게 되는데요,

1. 빈 해시 버킷에 노드를 삽입하는 경우, lock 을 사용하지 않고 [Compare and Swap](http://tutorials.jenkov.com/java-concurrency/compare-and-swap.html) (비교 후 일치하면 변경)을 이용하여 새로운 노드를 해시 버킷에 삽입(원자성 보장)

![image (1)](https://user-images.githubusercontent.com/45073750/112744503-a3951b80-8fdb-11eb-9fdb-db6a9a8b507a.png)

(1) 무한 루프. table 은 내부적으로 관리하는 가변 배열이다.
(2) 새로운 노드를 삽입하기 위해, 해당 버킷 값을 가져와(tabAt 함수) 비어 있는지 확인한다.(== null)
(3) 다시 Node 를 담고 있는 volatile 변수에 접근하여 Node 와 기대값(null) 을 비교하여(casTabAt 함수) 같으면 새로운 Node 를 생성해 넣고, 아니면 (1)번으로 돌아간다(재시도).  

\2. 이미 노드가 존재 하는 경우는 `synchronized (노드가 존재하는 해시 버킷 객체)` 를 이용해 하나의 스레드만 접근할 수 있도록 제어한다.
서로 다른 스레드가 같은 해시 버킷에 접근할 때만 해당 블록이 잠기게 된다.  

![image (2)](https://user-images.githubusercontent.com/45073750/112744518-c45d7100-8fdb-11eb-9061-119a32b95a14.png)

3줄 요약

1. 자바 8 이전의 ConcurrentHashMap은 영역을 16개로 분리하여 각각의 영역을 잠그는 방식이었다.
2. 자바 8 이후부터의 ConcurrentHashMap은 키가 존재할 때 put할 경우 각 테이블 버킷을 독립적으로 잠근다
3. 값이 null인 경우에는 잠그지는 않고, CAS (체크 후 삽입)을 통해 삽입한다.  

### ``검프``

8장 문제!!

1. java.util.Collections의 정적 메소드를 통해 반환 받은 컬렉션 또는. 컬렉션을 반환하는 유팅성 클래스들의 정적 메소드들은 전부\ 요소를 추가하거나 삭제할 수 없게 되어있어요. 그 이유는 무엇일까요?

```java
List<String> strings = Collections.emptyList();
Map<Integer, Integer> integers = Collections.emptyMap();
List<String> stringArray = Arrays.asList("hi", "bi", "me");
List<Integer> integers1 = List.of(0, 1, 2, 3); 
```

1. 또한, 아래의 코드는 정상 작동할까요?

```java
Set<String> set = new HashSet<>(Arrays.asList("hi", "ss", "me"));
set.add("hi");
```

### 답안

답: 1. 각각의 이유는 손너잘이 잘 말해줌

1. emptyList, emptyMap의 경우는 싱글톤을 이용하여 성능향상을 하기 때문에 변경을 불가하게 해야 한다.
2. asList의 경우는 인자로 들어오는 array를 직접 참조 하기때문에 추가가 불가하다(변경은 가능하다)
3. of의 경우는, 불변을 통해 데이터 안정성을 보장하기 위해 불변하게 설계가 되어있다.

위의 유틸성 클래스들은 전부 내부에서 자체 구현한 컬렉션이 있다. 이들은 공통적으로 불변으로 이뤄져있는데, 데이터 목적성과 정확성을 내부 구현체의 반환값을 변경하는 것을 막는다.

1. 정상작동한다. 다만 값이 제대로 들어가지 않는다.(Arrays.asList의 요소를 복사해서 새로운 HashSet의 요소로 넣어주기 떄문에 )

set은 중복이 불가능한데, add시에 boolean값으로 추가가 됐는지 안됐는지 반환한다. (자바 쓰레기.. 엘레강트 저자가 봤으면 혼냈다! )  

### ``찰리``

8장 문제
pobi, cu, json이 참가한 레이싱 게임입니다.
`movedCarNames` 에는 움직인 순서대로 드라이버의 이름들이 들어있습니다.
경기 직전 레이싱 결과를 계산하는 로직을 보고만 pobi는 마음이 불편하다가
급기야 레이싱 도중 차를 멈추고 getOrDefault 말고 다른걸 써보는건 어떻겠냐고 피드백을 줍니다.Map의 계산 패턴 메서드 또는 merge를 사용해서 결과 계산 로직을 바꿔주세요!

```java
for (String movedDriverName : movedDriverNames) {
    racingResult.put(movedName, (racingResult.getOrDefault(movedCarName, 0) + 1));
}
아래는 전체 코드입니다.
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;public class WonkaQuestion {
    public static void main(String[] args) {
        List<String> movedDriverNames = Arrays.asList(
                "pobi",
                "json",
                "cu",
                "cu",
                "cu",
                "pobi",
                "pobi",
                "cu",
                "json",
                "json",
                "pobi"
        );        Map<String, Integer> racingResult = new HashMap<>();        for (String movedDriverName : movedDriverNames) {
            //racingResult.put(movedDriverName, (racingResult.getOrDefault(movedDriverName, 0) + 1));            // Map의 계산패턴 메서드 또는 merge를 사용해서
            // 위에 주석 처리된 코드와 같은 결과를 보여주는 코드를 작성해주세요!
        }
        System.out.println(racingResult);
    }
}
```

### 답안

```java
racingResult.compute(movedCarName, (k, v) -> v + 1);
racingResult.merge(movedCarName, 1, (oldValue, newValue) -> oldValue + newValue);
racingResult.merge(movedCarName, 1, Integer::sum);
```

***