# LinkedList의 원리에 대해

LinkedList(연결 리스트)는 노드로 이루어진 선형의 자료구조이다. 

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    transient int size = 0;

    /**
     * Pointer to first node.
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     */
    transient Node<E> last;
    
    ...
}

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

간략히 내부코드를 보면 다음과 같다.  
노드가 데이터를 담고 있으며, 이전 노드와 다음 노드에 대한 정보도 가지고 있다.  
즉,  노드가 서로 한 줄로 연결되어 있는 구조라는 뜻이다.  

이 Node를 LinkedList에 추가하거나 삭제한다고 해도 추가된 앞/뒤의 노드의 링크, 삭제된 앞/뒤의 노드의 링크만 변경이 일어나고 나머지 노드들에게는 영향이 가지 않는다.  

인덱스가 없기 때문에 중간의 데이터를 삭제해도 모든 인덱스가 밀리거나 당겨지는 일이 없다. 따라서 데이터 추가/삭제가 용이하지만 마찬가지로 인덱스가 없기 때문에 특정 요소 접근을 위해 순차 탐색을 진행하기 때문에 탐색 속도가 떨어진다.  따라서, 탐색 또는 정렬을 많이 사용할 때에는 배열을 데이터 추가 또는 삭제가 많을 경우에는 LinkedList를 사용하는 것이 좋다.  

대표적으로 Queue가 있다. ``Queue<Integer>que = new LinkedList<>();`` 다음과 같이 사용한다.  
Queue는 인터페이스이고 LinkedList는 Queue를 implements 하는 클래스이다.  
Queue에는 ``addFirst``, ``addLast``와 같은 메서드가 없고 LinkedList 내부에 있는데 위와 같이 선언함으로써 Queue에서 해당 메서드를 사용할 수 없게된다.  

```java
    LinkedList<Integer> list = new LinkedList<>();
    list.offer(1);
    list.add(1);

    public boolean offer(E e) {
        return add(e);
    }

    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

여러 메서드가 있는데 먼저 offer와 add를 살펴보자. offer는 add를 호출하므로 결국 동일한 동작을 한다. 그리고 add는 linkLast를 호출하는데 가장 마지막에 노드를 달아주는 것을 알 수 있다. 

```java
list.addLast(1);
list.addFirst(1);
```

메서드명 그대로 처음과 긑에 노드를 달아준다. addLast는 linkLast를 호출한다. 즉, add와 addLast는 동일하게 동작한다.  

```java
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

add 에는 해당 인덱스에 삽입할 수도 있다. 인덱스 값을 주면 된다. 그리고 linkBefore의 구현을 보면 삽입되는 위치의 앞 뒤 노드들의 각 링크를 추가되는 newNode로 변경한 것을 볼 수가 있다. 이 포인터 값만 변할뿐 밀리거나 당겨지지 않는다.  

```java
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

위의 add 메서드를 보면 ``linkBefore(element, node(index))`` 가 있는데 해당 node 메서드의 구현코드를 보면 순차탐색을 해서 인덱스 위치의 node를 찾은 후 return 해주는 것을 알 수가 있다.  

```java
        list.remove(1);			//index remove
        list.remove((Integer)1);//Object remove
```

list의 remove 메서드를 보면 index 위치 삭제와 해당 객체 삭제가 있다. 첫 번째 remove는 index가 1의 위치에 속하는 것을 삭제하는거고, 두 번째는 값이 1인 노드를 삭제하는 것이다.  
삭제는 ``poll()``과 ``pop()``도 있다. 둘의 차이점은 ``poll()``은 값이 존재하지 않을 때에 null를 리턴하고 ``pop()``은 ``NoSuchElementException()``를 리턴한다.  

```java
        LinkedList<Integer> list = new LinkedList<>();
        list.add(1);
        list.add(2);
        list.add(1);
        list.remove((Integer)1);
        for (Integer in : list) {
            System.out.print(in+" ");
        }
```

다음과 같이 돌리면 ``2 1`` 이렇게 출력이 된다. 왜 앞의 1이 삭제되었냐 하면

```java
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

구현 코드를 보면 First 노드에서부터 순차탐색을 하면서 찾기 때문이다.  
주석에도 다음과 같이 나와있다. ``Removes the first occurrence of the specified element from this list,``  
unlink 메서드는 삭제할 노드의 앞 뒤 노드를 이어주는 작업인 것을 알 수가 있다.  

이외에도 ``size()``,``clear()``,``poll()``,``getFirst()``,``getLast()`` 등 다양한 메서드가 존재한다.  