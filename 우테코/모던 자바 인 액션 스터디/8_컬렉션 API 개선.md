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