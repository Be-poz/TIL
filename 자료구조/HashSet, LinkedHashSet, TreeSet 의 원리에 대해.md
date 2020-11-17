# HashSet, LinkedHashSet, TreeSet 의 원리에 대해

HashSet과 TreeSet은 Set 인터페이스의 구현 클래스이다. LinkedHashSet은 HashSet을 상속받아 구현이 된다. Set은 알다시피 중복값을 허용을 안한다. LinkedHashSet은 HashSet과는 다르게 저장 순서를 유지해주고 TreeSet은 들어온대로 정렬을 해준다. 그렇다면 이들의 내부 코드는 어떻게 되어있길레 이런 기능들을 하는건지 살펴보자.  

```java
    public HashSet() {
        map = new HashMap<>();
    }

    public int size() {
        return map.size();
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

	private static final Object PRESENT = new Object();
```

위 코드는 HashSet의 내부 코드이다. 보면 HashMap을 이용하고 있다는 것을 알고 있다. size, contains, add 등 그냥 다 HashMap 메서드를 호출해서 사용하고 있다. add 시에 빈 Object 객체를 집어 넣어주고 있다는 것을 알 수가 있다. 그러니깐 HashSet은 contains 메서드만 봐도 알겠지만 HashMap에서 key 만 이용한다는 것이다. value에는 그냥 더미 Object를 집어넣어주면서 말이다.  

```java
    public LinkedHashSet() {
        super(16, .75f, true);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

LinkedHashSet은 HashSet을 상속받는다. 메서드를 그대로 사용하고 인스턴스 생성 시에 LinkedHashMap을 받는다는 차이점이 있다. HashSet이 HashMap을 받았듯이 LinkedHashSet은 LinkedHashMap을 이용한다.  

```java
	private transient NavigableMap<E,Object> m;

    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<>());
    }

    public int size() {
        return m.size();
    }

    public boolean isEmpty() {
        return m.isEmpty();
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }

    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }
```

TreeSet은 TreeMap 을 생성해서 NavigableMap 타입으로 받아두고 size, isEmpty, contains, add 등의 메서드를 실행한다. HashSet이 HashMap을 받아서 그대로 메서드를 사용했던 것과 동일한 방식이다.  



Set은 사실 Map에서 key 값만 사용한 컬렉션이라고 보면 된다. 따라서, Map에 대한 지식이 우선적으로 있어야 하는 컬렉션이다.  

***