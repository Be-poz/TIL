# Collections.singletonList() VS Arrays.asList() 에 대해

```java
        List<String> list1 = new ArrayList<>(Arrays.asList("abc"));
        List<String> list2 = Collections.singletonList("abc");
```

다음과 같이 작성을 하게되면 ``asList`` 부분에 불빛이 들어오고 ``Collections.singletonList()`` 로 대체하라고 안내문이 나온다. 그 이유는 메모리 절약을 위해서이다. 즉, ``Arrays.asList()``는 ``Collections.singletonList()``보다 메모리를 더 차지, 사이즈가 더 크다는 것을 추측할 수 있다.  

### Collections.singletonList()

불변하며, size가 1로 고정되며 값과 구조를 변경할 수 없다.  

### Arrays.asList()

값 변경이 가능하며, 소유한 배열의 고정된 사이즈의 목록을 반환하고 구조를 변경할 수 없다. ArrayList의 인스턴스를 반환한다.  

***

#### Reference

https://stackoverflow.com/questions/26027396/arrays-aslist-vs-collections-singletonlist/52536126#52536126