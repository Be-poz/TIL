# HashMap 의 원리에 대해

Map은 알다시피 중복 없는 key를 통해 value를 저장하는 형태를 가진다.  
HashMap은 이름만 들어도 Hashing 을 이용한 Map일 것이다. 이제 정확히 어떤 내부 구현이 되어있는지 알아보자.  

가장 상위에 Map 인터페이스가 존재하고 HashTable과 HashMap이 이들을 implements 하여 구현된다. LinkedHashMap은 HashMap을 상속받는다. 이 클래스들 말고 인터페이스도 Map을 받는데, 바로 SortedMap 인터페이스이다. 이 녀석을 또 다시 NavigableMap 인터페이스가 받고 이를 implements한게 바로 TreeMap이다.  

```java
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

	/**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
	/**
     * Constructs an empty {@code HashMap} with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;

    /**
     * Returns the number of key-value mappings in this map.
     *
     * @return the number of key-value mappings in this map
     */
    public int size() {
        return size;
    }

    /**
     * Returns {@code true} if this map contains no key-value mappings.
     *
     * @return {@code true} if this map contains no key-value mappings
     */
    public boolean isEmpty() {
        return size == 0;
    }

```

여기까지만 대충보면 일반적으로 ``new HashMap<>();`` 를 하게되면 대충 16크기의 default 와 hashtable를 위한 loadfactor를 0.75f로 설정 후 생성해주고 size는 내부의 size 필드를 리턴, isEmpty는 size==0 의 boolean을 리턴해준다.  

```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
```

해당 key의 value를 받기위해서 사용하는 get 메서드는 Node를 만들고 getNode 메서드에 파라미터로 넘어온 key 값을 주고 만든 Node에다가 집어넣고 그 값의 null 여부에 따라 null 또는 해당 Node의 value를 리턴해주는 것을 볼 수가 있다.  

Node 클래스는 hash, key, value, 그리고 next 라는 다음 노드의 정보를 가지고 있고, 생성자와 기타 메서드를 가지고 있다. containsKey 메서드 또한 get과 마찬가지로 getNode 메서드를 통해서 해당 리턴을 해준다. getNode가 null이 아니면 true를 맞다면 false를말이다. 

```java

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

    transient Node<K,V>[] table;
```

getNode 메서드를 읽어보면, Node 배열 tab, 단일 Node first, e, int n, key k 주어지고
tab에 table 초기화하고 null 아니고, n에 tab길이 넣고 0 초과고, first에 (n-1)&hash위치의 tab 값이 null 이 아닐 때에, first 확인해서 first가 해당 get 하려는 Node면 first를 return 해준다. 아니라면 e.next 로 node를 넘겨보면서 쭉 탐색하고 없으면 null을 리턴하게 된다.  

LinkedHashMap 은 넣은 값의 순서를 그대로 기억하고, TreeMap은 삽입을 하면 키가 오름차순으로 정렬이 된다 (문자열은 유니코드 기준). HashTable도 Map 인터페이스를 받아 구현하는데, HashMap 과의 차이점은 HashTable은 동기화를 지원해준다는 점이다. 하지만, 그냥 ConcurrentHashMap을 사용해주는 것이 좋다고 한다.  

HashMap은 내부에 Node를 이용한다는 것을 알 수가 있었다.

***