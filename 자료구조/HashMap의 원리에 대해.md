## HashMap의 원리에 대해

이 글은 [이곳](https://sabarada.tistory.com/57?category=826240)에서 제가 정리한 것보다 더 정리가 너무 잘 되어있어서 가져왔습니다. 

**HashMap은 Key, Value를 저장하는 Map의 구현체 중 하나**입니다. 자료구조에 Key를 넣으면 Value를 반환하도록 합니다. 그리고 HashMap은 Key를 Hashing을 하여 저장하여 빠르게 처리 그리하여 HashMap이란 **입력과 삭제에 대해 시간복잡도가 O(1)인 자료구조**라고 합니다.

## initialize

우리가 Java를 사용할때 HashMap을 사용한다고 한다면 가장먼저 초기화를 해야합니다. (Key, Value는 모두 String이라고 하겠습니다.) 그러면 아래와같이 초기화 할 것입니다.

```
Map<String, String> map = new HashMap<>();
```

이때 내부에서는 어떤일이 일어나는지 보겠습니다.

```
/**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

위와 같은 생성자가 실행이 될 것입니다. 내용을 살펴보면 **loadFactor에 DEFAULT_LOAD_FACTOR(0.75)를 주입**하는 것을 알 수 있습니다. 이것만으로는 내부 구조가 어떻게 되어있는지는 알기 힘들것 같습니다. 그러면 이어서 실제 데이터를 넣어보도록 하겠습니다.

## put

```
map.put("test", "test");
```

위와 같은 행위를 했을 때는 어떤 일이 일어날 지 다시 내부를 보도록 하겠습니다.

```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods.
     *
     * @param hash key를 hash한 값
     * @param key key 값
     * @param value value 값
     * @param onlyIfAbsent true 라면 key 중복에 대해 값 교체 없음.
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
```

putVal 이라는 method를 호출하고 있습니다. 우리는 여기에 hash와 putVal이라는 method가 추가로 사용되는 것을 알 수 있으며, 따라가 보도록 하겠습니다. 먼저 hash를 보도록 하겠습니다.

```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

key 값을 int형으로 변형하는 것을 알 수 있습니다. 내부로직으로 볼때 key가 null이라면 0을 리턴하며, **key가 null이 아니라면 해당 key의 Object class에 있는 hashcode() method를 실행하고 리턴 값인 int를 얻어낸 후 16번 오른쪽으로 unsigned right shift한 값과 XOR을 하여 사용할 hash값**을 얻어냅니다.

이렇게 얻어낸 hash값, Key, Value, onlyIfAbsent, evict 값을 이용하여 putVal method를 실행합니다.

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // local variable
        Node<K,V>[] tab;
        Node<K,V> p;
        int n, i;

        // logic
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

위쪽에는 putVal method를 모두 가져왔습니다. 그러면 차근차근 putVal의 로직을 하나씩 따라가 보겠습니다.

```
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

맴버변수 table에 값이 들어있는지 확인 한 후 크기를 확인합니다. table은 한번이라도 put을 했다면 null이 아닐것입니다만, 저희는 처음 put을 하는 것이기 때문에 해당로직이 실행되어지게 됩니다. `resize()`를 확인해보도록 하겠습니다.

```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

**resize()의 코드의 흐름을 확인해보면, 이미 초기화 되어있는지 아닌지 확인후 아래와 같이 새로운 배열을 생성**합니다. 그리고 만약 기존의 배열(초기화가 이미 진행)이 있다면 index의 위치를 재조정합니다.

```
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
```

이로써 우리는 **HashMap은 사실 LinkedList 배열**이라는 사실을 알 수 있었습니다.

이렇게 resize()를 거친 후 우리는 hash & (n - 1)의 index 즉 (hash % n - 1)의 index에 아무런 값이 들어 있지 않으면 아래와 같이 새로운 노드를 아래와 같이 집어넣게됩니다.

```
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

newNode를 들여다 보면 아래와 같이 새로운 Node를 만들어 해당 배열에 넣는 것을 알 수 있습니다.

```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
}
```

Node는 LinkedList를 구현한 객체라고 보시면 되며 구셩요소는 아래와 같습니다.

```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        ...
}
```

하지만 만약 해당 index에 이미 값이 들어 있다면 해당 LinkedList의 마지막에 연결해야합니다. 해당 로직을 한번 보도록 하겠습니다.

```
Node<K,V> e; K k;
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
else {
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
}
```

3가지의 경우의 수가 나올 수 있습니다. 가장먼저 hash값과 실제 값이 같다면(equals가 true)면기존에 있던 값을 교체합니다.

```
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```

그렇지 않다면 아래 로직이 실행됩니다. 먼저 TreeNode인지 판단을 하고 그러다면 TreeNode에 값을 넣으면 그렇지 않다면 마지막 node의 마지막에 새로운 노드를 넣게 됩니다.

```
p.next = newNode(hash, key, value, null);
```

이렇게 **HashMap의 put이 1의 시간복잡도를 가지는 이유는 hashing을 배열의 index값으로 해 그 곳에 바로 집넣기 때문**이라는 것을 알게되었습니다. 다음은 remove에 대해서 알아보도록 하겠습니다.

## remove

**remove는 key값을 기준으로 탐색하여 해당 key 값에 대한 value가 존재한다면 삭제하며 존재히지 않는다면 null값을 반환하는 method**입니다. method는 아래와 같이 구성되어 있습니다.

```
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab;
    Node<K,V> p; 
    int n, index;

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

`removeNode()`를 보도록 하겠습니다. 가장 먼저 해당 table의 index에 값이 있는지 없는지를 판단하여 있어야 로직을 실행합니다.

```
if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null)
```

그리고 해당 index의 첫번째 node가 equals로 true가 나오는지 확인합니다. 그렇지 않다면 해당 index의 내부를 순회하며 equals가 true인 경우가 있을 때 까지 순회하며 임시변수 node에 넣습니다.

```
if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
```

node에 값이 있다라는 것은 지울 Node가 존재한다는 것입니다. node에 값이 없다면 null을 바로 return 하며 값이 있다면 해당 node를 삭제합니다.

```
if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
    if (node instanceof TreeNode)
        ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
    else if (node == p)
        tab[index] = node.next;
    else
        p.next = node.next;
    ++modCount;
    --size;
    afterNodeRemoval(node);
    return node;
}
```

## 정리

이렇게 우리는 Hashmap의 put과 remove 메서드를 통해 HashMap에 대해서 알아보았습니다. 정리를 해보면

- HashMap은 `**Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]**` 으로 되어있다.
- put으로 data를 넣으면 Node<K,V>[hashing 값 & 배열 크기 - 1]에 값이 들어가고, key가 같고 value가 다르면(equals가 false) linkedlist의 자료구조로 이어서 붙는다.
- remove를 하게되면 Node<K,V>[hashing 값 & 배열 크기 - 1]에 값이 있는지 확인 후 key 값을 비교후 가져오거나 없다면 null.
- put이 많아져 들어있는 양이 **`loadFactor`를 넘게되면 배열의 크기를 재산정**함(현재 크기의 2배로)
- **하나의 index에 들어있는 값이 많아지면 tree의 형태**로 변환.(Java8 이상)
- Hashmap의 **시간복잡도가 O(1)인 이유는 key값을 (hashing & n - 1)으로 직접 index를 가져오기 때문**이다.

그리고 내부구조가 아래와 같음을 알 수 있었습니다.



![img](https://blog.kakaocdn.net/dn/cWl8Le/btqBs06fxq6/5v3qidfoClN8Dd73rd4TP0/img.png)hashmap의 structure



## 참조

https://www.w3schools.com/java/java_hashmap.asp

https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html