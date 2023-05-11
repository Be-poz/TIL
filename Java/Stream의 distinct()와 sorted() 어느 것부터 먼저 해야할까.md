# Stream의 distinct()와 sorted() 어느 것부터 먼저 해야할까??

```java
numbers.stream()
                .distinct()
                .sorted()
                .collect(Collectors.toList());
numbers.stream()
                .sorted()
                .distinct()
                .collect(Collectors.toList());
```

다음과 같이 위의 두 코드 중 어떤 방식을 선택해야 할까??  

``sorted()``는 흔히 우리가 생각하는 작동방식과 살짝 다르다.  

```java
List<Integer> list = List.of(2, 5, 4, 7, 3, 98, 2);
List<Integer> collect = list.stream()
        .peek(System.out::println)
        .distinct()
        .peek(i -> System.out.println("fjrst" + i))
        .collect(Collectors.toList());
System.out.println("collect = " + collect);

/*
2
fjrst2
5
fjrst5
4
fjrst4
7
fjrst7
3
fjrst3
98
fjrst98
2
collect = [2,5,4,7,3,98]
```

```java
List<Integer> list = List.of(2, 5, 4, 7, 3, 98, 2);
List<Integer> collect = list.stream()
        .peek(System.out::println)
        .sorted()
        .peek(i -> System.out.println("fjrst" + i))
        .collect(Collectors.toList());
System.out.println("collect = " + collect);

/*
2
5
4
7
3
98
2
fjrst2
fjrst2
fjrst3
fjrst4
fjrst5
fjrst7
fjrst98
collect = [2, 2, 3, 4, 5, 7, 98]
```

위의 두 코드를 보시다시피 ``sorted()``는 해당 위치에서 데이터들을 모아서 정렬을 한 뒤에 다시 요소를 보내준다. 데이터가 적으면 ``sorted()`` 후 ``distinct()``가 빠른 경우도 있다. 겹치는 데이터가 어느 정도 있다는 가정하에 ``distinct()`` 후 ``sorted()`` 가 빠르다. 따라서 만약 요소들이 많을 경우에는 Heap 메모리 부족이 발생하는 상황이 발생할 수 있다.  

``distinct()``를 먼저 사용하는 것이 안전하다. 그리고 속도적 측면에서도 대부분 유리하다.  

***