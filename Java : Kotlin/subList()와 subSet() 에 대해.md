# subList()와 subSet() 에 대해

```java
        List<Integer> beforeList = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        List<Integer> afterList = beforeList.subList(0, 2);
        afterList.forEach(System.out::println);
/*출력
1
2
```

``subList()``는 List를 잘라주는 메서드이다. 인수로 ``int fromIndex``, ``int toIndex``를 갖는다.  
내부구현은 다음과 같이 이루어져 있다. (ArrayList 기준)  

```java
    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList<>(this, fromIndex, toIndex);
    }

    private static class SubList<E> extends AbstractList<E> implements RandomAccess {
        private final ArrayList<E> root;
        private final SubList<E> parent;
        private final int offset;
        private int size;

        /**
         * Constructs a sublist of an arbitrary ArrayList.
         */
        public SubList(ArrayList<E> root, int fromIndex, int toIndex) {
            this.root = root;
            this.parent = null;
            this.offset = fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = root.modCount;
        }
        ... 각종 메서드
    }
```

코드를 보면 ``SubList``라는 클래스를 생성해서 리턴을 해주고 있다. ``SubList``생성자는 위의 코드 외에 첫 번째 인수로 ``SubList<E> root``를 가진 생성자 또한 있다. 그리고 각 메서드 들은 ``checkForComodification()``를 호출하고 있는데 기존의 List를 SubList가 참조를 하고 있다. 따라서 원본 list의 값에 이상이 생기면(삭제와 같이) ``java.util.ConcurrentModificationException``이 발생하게 된다.  

```java
        List<Integer> beforeList = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        List<Integer> afterList = beforeList.subList(0, 2);

        beforeList.remove((Integer) 1);
        afterList.forEach(System.out::println);

// Exception in thread "main" java.util.ConcurrentModificationException
```

물론 이후에 추가하는 것은 문제 없이 동작한다. 하지만 List의 사이즈가 정해져 있을 때에는 ``java.lang.UnsupportedOperationException``이 발생하게 된다. ( ex. List<Integer> beforeList = Arrays.asList(1, 2, 3, 4, 5); )

```java
        List<Integer> beforeList = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        List<Integer> afterList = beforeList.subList(0, 2);

        afterList.add(4);
        afterList.forEach(System.out::println);

/*출력
1
2
4
```

``subSet``은 ``TreeSet``과 ``NavigableSet``에 정의되어 있다. 따라서 ``new TreeSet`` 후에 ``Set``으로 받지 못한다.  
``subList`` 처럼 index로 나누는 것이 아닌 값으로 나누게 된다. 밑의 코드를 보면 이해할 것이다.  

```java
        TreeSet<Integer> set = new TreeSet<>(Set.of(1, 3, 2, 5, 7, 4, 6));

        SortedSet<Integer> subSet1 = set.subSet(2, 5);
        NavigableSet<Integer> subSet2 = set.subSet(2, false, 5, true);

        System.out.println("subSet1" + subSet1);
        System.out.println("subSet2" + subSet2);

/*출력
subSet1[2, 3, 4]
subSet2[3, 4, 5]
```

subSet2에서 사용한 ``subSet``의 정의는 다음과 같다. from, to 값 포함 여부에 대한 boolean을 받아준다.  

```java
    public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive)
```

``subSet``은 ``subList``와 달리 ``ConcurrentModificationException``이 발생하지 않는다.  
그 이유는 ``Tree.subSet``은 ``NavigationMap.subMap()``으로 만든 ``TreeSet``을 생성하기 때문에 데이터가 변경되어도 상관없기 때문이다.  
위에서 Map의 구현을 빌려다 썼다. Set은 사실 Map이다. 그 이유는 이곳에서 확인해보자.  
[HashSet, LinkedHashSet, TreeSet 의 원리에 대해](https://github.com/Be-poz/TIL/blob/master/%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0/HashSet%2C%20LinkedHashSet%2C%20TreeSet%20%EC%9D%98%20%EC%9B%90%EB%A6%AC%EC%97%90%20%EB%8C%80%ED%95%B4.md)  

***

### Reference

https://knight76.tistory.com/entry/Java%EC%9D%98-List%EC%9D%98-subList%EC%99%80-Set%EC%9D%98-subSet-%EB%B9%84%EA%B5%90