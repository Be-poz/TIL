# LinkedHashMap의 원리에 대해

## HashMap을 순차적으로 읽으면 어떤 순서로 읽는가 ?

이 글은 [이곳](https://sabarada.tistory.com/120?category=826240) 의 글을 그대로 가져온 글입니다.

HashMap은 Node를 꺼낼 때 넣은 순서를 보장하지 않는다고 합니다. 그렇다면 Node 순회에 대해서 읽는 순서는 어떻게 될까요 Map interface에서 아래와 같은 문구를 발견하였습니다.

> The {@code Map} interface provides three *collection views*, which allow a map's contents to be viewed as a set of keys, collection of values, or set of key-value mappings. The *order* of a map is defined as the order in which the iterators on the map's collection views return their elements. Some map implementations, like the {@code TreeMap} class, make specific guarantees as to their order; others, like the {@code HashMap} class, do not.

**주목하실 부분은 제일 아래의 Some~~ 부분입니다. 직역하자면 TreeMap 등과 같은 구현 Map은 순서를 보장하지만 HashMap과 같은 Map은 보장하지 않는다라는 문구입니다.**

추가로 아래는 HashMap의 keySet의 foreach 구문입니다. **보시면 단순하게 0번 부터 순회하는 것을 알 수 있습니다. 이렇게 순회하게 되면 hash값이 낮은 순서부터 순회하게 됩니다.** 그렇다면 고정이지 않을까? 라고 생각하시는 분들도 있을 것입니다. 하지만 아닙니다. **데이터가 늘어난다면 loadfactor를 넘는 순간 hashcode의 재 분배가 일어나게되므로 순서는 뒤바뀌게 됩니다. 따라서 특정한 순서를 보장할 수 없는 것입니다.**

```
public final void forEach(Consumer<? super K> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (Node<K,V> e : tab) {
            for (; e != null; e = e.next)
                action.accept(e.key);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

## LinkedHashMap

그렇다면 이제 본격적으로 LinkedHashMap에 대해서 알아보도록 하겠습니다. **LinkedHashMap은 기존 HashMap 자료구조에서 LinkedList 자료구조를 섞어놓은 자료구조입니다.** 이렇게 자료구조를 2개 합쳐서 LinkedHashMap은 for, foreach를 통해 탐색했을 때 Map 자료구조에 넣은 순서를 유지할 수 있는 HashMap이 되었습니다. 이렇게만 말하면 뜬 구름 잡는 이야기 같습니다. 이제 코드를 보면서 어떻게 가능한지 함께 확인해보도록 하겠습니다.

## 초기 값

LinkedHashMap은 HashMap을 상속받고 있습니다. 즉, HashMap에서 사용하는 모든 메서드를 사용할 수 있다는 것입니다. 그리고 **여기에 head와 tail, before과 after라는 변수가 추가되었습니다.** 이 변수들을 이용하여 Map에 들어온 순서를 기억하는 것입니다.

```
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
/**
    * The head (eldest) of the doubly linked list.
    */
transient LinkedHashMap.Entry<K,V> head;

/**
    * The tail (youngest) of the doubly linked list.
    */
transient LinkedHashMap.Entry<K,V> tail;

static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

이미지로 나타내면 아래와 같은 이미지가 됩니다.



![img](https://blog.kakaocdn.net/dn/bmNyyy/btqLhO7vCDG/wXaMjSxOPJPGEI8vQnKk31/img.png)linkedHashMap 데이터 1개



## 추가

LinkedHashMap은 HashMap의 `put` 메서드를 동일하게 사용합니다. 동일하게 사용하며 put의 내부 메서드인 `putVal` 메서드를 보시면 linkedHashMap 전용으로 사용하는 메서드가 3개 있습니다. 전체 흐름을보고 각각에 대해서 로직을 보며 알아보는 시간을 가져보겠습니다. 아래 코드는 HashMap Class의 데이터를 넣는 메서드입니다. LinkedList와 연관이 있는 로직은 3개입니다. 첫번째는 새로운 노드를 생성하는 `newNode(...)` 부분, 두번째는 key 중복이 존재할 때 Node간의 link를 교체하는 `afterNodeAccess(...)` 메서드 호출 부분, 그리고 마지막으로 Node를 정리하는 `afterNodeInsertion(...)` 부분입니다.

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    ... 중략 ...
    if ((p = tab[i =ㅋ (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null); // 1. key 중복 없을 때 새로운 노드 추가
    else {
        ... 중략 ...
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); // 2. key 중복이 존재할때 Node Link 교체하기
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict); // 3. Node 추가 후 정리하기
    return null;
}
```

새로운 노드를 추가하는 `newNode` 부분을 좀 더 보도록 하겠습니다. newNode가 실행되면 LinkedHashMap의 Node 인스턴스를 하나 생성합니다. 그리고 `linkNodeLast` 메서드를 통해 기존에 들어있는 마지막 Node와 새로 추가한 노드를 연결합니다.

```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

// link at the end of list
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

만약 HashMap의 Key 중복이 존재한다면 `afterNodeAccess(...)` 메서드를 실행합니다. 해당 메서드는 LinkedHashMap에서 동일 key(equals)에 대해서 Node의 변경할 시 사용합니다. 이전 Node를 향하고 있던 Link를 새로운 NOde를 향하도록 변경합니다.

```
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

LinkedHashMap의 put 메서드를 통해 알 수 있는 LinkedHashMap의 자료주조를 이미지로 나타내면 아래와 같습니다.



![img](https://blog.kakaocdn.net/dn/X9eN7/btqK84ju2TV/5uFeEurrFwW1OqcZIdU1oK/img.png)linkedHashMap 데이터 3개



## 삭제

이번에는 노드를 삭제하는 `remove` 메서드에 대해서 알아보도록 하겠습니다. node 삭제또한 HashMap의 remove를 함께 이용합니다. linkedList 용으로 남겨져 있는 부분이 `afterNodeRemoval(...)`입니다. HashMap으로 실행하면 해당 노드는 아무런 코드없이 해당 메서드가 실행되지만 LinkedHashMap으로 실행하면 오버라이딩 된 메서드가 실행됩니다.

```
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        ... 중략 ...
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            ... 중략 ...
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

LinkedHashMap의 `afterNodeRemoval(...)` 메서드는 삭제 노드에 대해서 before와 after의 link를 끊는 작업을 합니다.

```
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

## 순환

마지막으로 순환(foreEach)에서 어떻게 LinkedHashMap이 순서를 유지하는지 보겠습니다. LinkedHashMap의 순환에 대해서 보겠습니다. 아래 코드가 순환코드입니다. 코드의 for 구문을 보시면 head에서 시작해서 e.after가 null일때 까지 반복합니다.

```
public final void forEach(Consumer<? super K> action) {
    if (action == null)
        throw new NullPointerException();
    int mc = modCount;
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
        action.accept(e.key);
    if (modCount != mc)
        throw new ConcurrentModificationException();
}
```

***

