# 순차적 스트림, 병렬 스트림 그리고 findAny와 findFirst 에 대해

지난번 스터디 때에 손너잘이 밑의 제 코드에 대해서 스트림은 비내리듯이 병렬로 진행되기 때문에 ``findFirst``를 사용하여도 첫 값이 보장이 안되기에 ``findAny``를 사용해야할 것 같다고 지적을 했습니다. 그렇다면 왜 두 메서드를 따로 나뒀을까 싶어서 찾아 봤습니다.  

```java
public enum Grade {
    A((score) -> score >= 95),
    B((score) -> score >= 90),
    C((score) -> score >= 80),
    D((score) -> score >= 70),
    F((score) -> true);
    
    Predicate<Integer> gradePredicate;
    
    Grade(Predicate<Integer> predicate) {
        gradePredicate = predicate;
    }
    
    public static Grade getGradeByScore(int score) {
        return Arrays.stream(Grade.values())
            .filter(grade -> grade.gradePredicate.test())
            .findFirst()
            .orElseThrow(RuntimeException::new);
    }
}
```

[Difference between findAny() and findFirst()](https://stackoverflow.com/questions/35359112/difference-between-findany-and-findfirst-in-java-8) 해당 링크에 다음과 같이 나와있었습니다.  

* ``findAny`` : **it is free to select any element in the stream.** 말 그대로 딱 발견한 값  

* ``findFirst`` : **strictly** the first element of the stream. 무조건 스트림의 첫 요소  

그렇다면 위의 코드를 ``findAny``로 수정한 뒤에 돌렸을 때, 너잘 말대로 비 내리듯이 스트림이 돌아갈텐데 어떤 스트림 요소가 먼저 도착할지 모르는 상황인데 제대로 결과값이 나올 수가 있을까 라는 생각이 들었습니다. 예를 들어, score 변수에 83점이 들어갔는데 첫 스트림의 요소가 Grade.D 라면 그대로 D가 반환될 것이니깐요.  

이런 이유 때문에 조건식을 ``score >= 70`` 이 아니라 ``score >= 70 && score < 80``으로 해야되는건가? 라고 생각했습니다. 조금 이해가 되지않아서 Stream 인터페이스에 들어가봤더니 다음과 같이 설명되어있었습니다.  

> Stream pipelines may execute either sequentially or in parallel. This execution mode is a property of the stream. Streams are created with an initial choice of sequential or parallel execution. (For example, Collection.stream() creates a sequential stream, and Collection.parallelStream() creates a parallel one.) This choice of execution mode may be modified by the sequential() or parallel() methods, and may be queried with the isParallel() method.

요약하자면 평소에 저희가 사용하는 Stream은 sequential한 순차적 스트림이고, 병렬로 사용하고 싶으면 ``parallelStream()`` 을 사용하거나 ``Parallel()`` 메서드를 사용해서 하라는것입니다.  

그래서 한 번 실험해봤습니다.  

```java
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            list.add(i);
        }
        
        list.stream().forEach(System.out::println);
        list.parallelStream().forEach(System.out::println);
```

위의 경우엔 1부터 100,000 까지 순차적으로 찍혔고, 밑의 경우엔 병렬로 진행되어 순차적으로 찍히지 않았습니다.  

```java
        System.out.println(list.parallelStream()
                .filter(i -> i > 5)
                .findAny()
                //.findFirst()
                .orElseThrow(RuntimeException::new));
```

위의 코드를 실행해보니 ``findFisrt()``를 실행을하니 6이 나오고, ``findAny()``를 실행하니 예상불가능한 값이 나왔습니다.  
``findFirst()``는 병렬 스트림이나 순차 스트림 어디든 스트림의 무조건적인 첫 요소를 반환해주는 거였습니다.  

```java
    public static Grade getGradeByScore(int score) {
        return Arrays.stream(Grade.values())
            .filter(grade -> grade.gradePredicate.test())
            .findFirst()
            .orElseThrow(RuntimeException::new);
    }
```

따라서 위의 코드에서는 ``findFirst()``를 사용하나  ``findAny()``를 사용하나 같은 결과값이 나온다는 것을 유추할 수 있었습니다. 병렬 스트림이 아니라 순차 스트림이기 때문이기에 말입니다.  

병렬에서도 무조건적인 첫 스트림 요소를 보장해주는 ``findFirst()``는 그러면 더 비용이 들지 않을까? 라고 생각이 들어 책을 찾아봤습니다. 나와있더라구요! P.168에 ``findAny()``는 찾는 즉시 실행을 종료한다고 합니다. 지금 보니깐 P. 169에도 두 메서드에 대해서 나와있네요. 병렬 스트림에서는 제약이 적은 ``findAny()``를 사용한다고요.  

위의 ``getGradeByScore``도 결국엔 ``findAny()``를 사용하는 것이 더 좋은게 맞는 것 같습니다! (어느 쪽을 사용하나 순차 스트림이니 동일 결과값이 보장되지만 시간 효율 상 이득)  그리고 점수 체크에 대한 조건식도 더 강화시키는 것도 맞는 것 같습니다. 혼자 이해안돼서 찾았던 거여서... 다른 분들도 도움되셨으면 해서 올려봅니다!  

혹시 틀린 내용이나(손너잘... 살살해줘요...) 덧붙일 의견 있으시면 언제든지 고고!  

***