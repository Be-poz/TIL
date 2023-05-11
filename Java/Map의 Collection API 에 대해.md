# Map의 Collection API 에 대해

HashMap은 자바를 사용하다보면 필연적으로 많이 사용하게 되는 컬렉션이다.  
이를 더욱 사용하기 편리하게 해주는 Collection API 에 대해 알아보겠다.  

<br/>

## getOrDefault

```java
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
```

```java
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.put(2, 2);

        System.out.println(map.getOrDefault(1, 10));
        System.out.println(map.getOrDefault(2, 10));
        System.out.println(map.getOrDefault(3, 10));

// 출력 결과
// 1
// 2
// 10 
```

만약 key의 값이 있다면 그 값을 들고오고 없다면 지정해둔 defaultValue를 리턴해준다.  

<br/>

## putIfAbsent

```java
    default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }

        return v;
    }
```

```java
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.put(2, 2);

        map.putIfAbsent(1, 10);
        map.putIfAbsent(2, 10);
        map.putIfAbsent(3, 10);
        System.out.println(map.get(3));

// 출력 결과
// 10
```

만약에 해당 key 값이 없다면 value를 집어 넣어준다.  

<br/>

## compute

```java
    default V compute(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        V oldValue = get(key);

        V newValue = remappingFunction.apply(key, oldValue);
        if (newValue == null) {
            // delete mapping
            if (oldValue != null || containsKey(key)) {
                // something to remove
                remove(key);
                return null;
            } else {
                // nothing to do. Leave things as they were.
                return null;
            }
        } else {
            // add or replace old mapping
            put(key, newValue);
            return newValue;
        }
    }
```

```java
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.put(2, 2);
        map.compute(1, (key, value) -> value + 3); // 4 return
        System.out.println(map.get(1));

// 출력 결과
// 4
// 만약 map.compute(4, ~~); 이었다면 NPE 발생
```

내부 구현코드를 보면 key 값으로 value를 구하고 이 두 파라미터를 remappingFunction 에 돌려서 새로운 value를 반환해준다. 문제는 해당 key값이 존재하는지 확인을 안하고 해준다는 점이다.  

<br/>

## computeIfAbsent

```java
    default V computeIfAbsent(K key,
            Function<? super K, ? extends V> mappingFunction) {
        Objects.requireNonNull(mappingFunction);
        V v;
        if ((v = get(key)) == null) {
            V newValue;
            if ((newValue = mappingFunction.apply(key)) != null) {
                put(key, newValue);
                return newValue;
            }
        }

        return v;
    }
```

```java
        Map<Integer, Integer> map = new HashMap<>(Map.ofEntries(entry(1, 1), entry(2, 2), entry(3, 3)));
        System.out.println(map.computeIfAbsent(4, key -> key + 2));	//6
        System.out.println(map.computeIfAbsent(3, key -> key + 2));	//3
        System.out.println(map.get(4));								//6
```

만약 해당 key 값이 존재하지 않는다면 key를 파라미터로 하여 mappingFunction을 수행한 후에 map에 put 해준다.  
이때, 해당 값을 반환해준다. 만약 존재한다면 기존의 value를 반환해준다. 이것은 puteAbsent도 마찬가지다.  

이 방법은 다음과 같은 상황일 때 쓰일 수 있을 것이다.  

```java
    public static void main(String[] args) {
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.put(2, 2);
        map.put(3, 3);

        if (!map.containsKey(4)) {
            map.put(4, calculate(4));
        }
        System.out.println(map.get(4));

        map.computeIfAbsent(5, key -> calculate(key)); // class::calculate으로 축약가능
        System.out.println(map.get(5));
    }

    public static int calculate(int n) {
        return n + 5;
    }
```

<br/>

## computeIfPresent

```java
    default V computeIfPresent(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        V oldValue;
        if ((oldValue = get(key)) != null) {
            V newValue = remappingFunction.apply(key, oldValue);
            if (newValue != null) {
                put(key, newValue);
                return newValue;
            } else {
                remove(key);
                return null;
            }
        } else {
            return null;
        }
    }
```

```java
        Map<Integer, Integer> map = 
            new HashMap<>(Map.ofEntries(entry(1, 1), entry(2, 2), entry(3, 3)));
        System.out.println(map.computeIfPresent(2, (key, value) -> key + value + 1));	//5
        System.out.println(map.computeIfPresent(4, (key, value) -> key + value + 1));	//null
		System.out.println(map.get(2))													//5
```

해당 key 값이 존재하는 경우에 key와 value를 인수로 하여 remappingFunction을 수행해준다.  
key 값이 존재하여 함수가 진행되었을 때에 해당 결과 값을 반환해준다. 만약 해당 key가 존재 하지않는다면 null을 반환한다.

이 방법은 다음과 같을 때에 쓰일 수 있을 것이다.  

```java
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);

        if (map.containsKey(1)) {
            map.put(1, map.get(1) + 1);
        }
        System.out.println(map.get(1));							// 2 출력

        map.computeIfPresent(1, (key, value) -> value + 1);
        System.out.println(map.get(1));							// 3 출력
```

``containsKey``로 확인하여 ``get`` 후에 연산을 하는 것이 ``computeIfPresent ``로 깔끔하게 해결되었다.  

<br/>

## putIfAbsent 와 computeIfAbsent의 차이점

``putIfAbsent`` 는 key와 value를 파라미터로 보내고 key가 없을 시에 해당 value를 put 한다.  

``computeIfAbsent`` 는 key와 mappingFunction을 설정하고 해당 key 값이 없을 시에 function을 실행한다.  

```java
    public static void main(String[] args) {
        Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 1);
        map.putIfAbsent(2, calculate(2));
        map.computeIfAbsent(2, key -> calculate(key));
    }

    public static int calculate(int n) {
        return n + 5;
    }
```

위의 코드에서 ``putIfAbsent``는 ``calculate`` 메서드가 실행이된 결과를 파라미터로 보내게 되고,  
``computeIfAbsent``는 2라는 key 값의 존재 유무를 먼저 확인하고 없을 시에 메서드가 실행이 된다.  

``map.putIfAbsent(2, 2)``, ``map.computeIfAbsent(2, key -> 2)`` 이 경우에는 전혀 상관이 없을 것이다.  

<br/>

***

### Reference

https://dzone.com/articles/how-to-use-java-hashmap-effectively