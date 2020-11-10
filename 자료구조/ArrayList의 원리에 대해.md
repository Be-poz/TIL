# ArrayList의 원리에 대해

ArrayList 는 LinkedList 와 같이 List 인터페이스를 구현하는 클래스이다.  

배열은 처음에 할당된 만큼의 공간을 사용할 수 있는 반면 ArrayList는 계속해서 add 할 수 있다. 그래서 사용하기 엄청편한데, 그렇다면 내부구현이 어떤식으로 돌아가길레 계속 추가할 수 있는걸까 ??  

```java
ArrayList<Integer> list = new ArrayList<>();
list.add(3);

//ArrayList.java 내의 코드
    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;

    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }

    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size = s + 1;
    }

    private Object[] grow() {
        return grow(size + 1);
    }

    private Object[] grow(int minCapacity) {
        return elementData = Arrays.copyOf(elementData,
                                           newCapacity(minCapacity));
    }

    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity <= 0) {
            if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);
            if (minCapacity < 0) // overflow
                throw new OutOfMemoryError();
            return minCapacity;
        }
        return (newCapacity - MAX_ARRAY_SIZE <= 0)
            ? newCapacity
            : hugeCapacity(minCapacity);
    }

//Arrays.java 내의 코드
    public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }

    @HotSpotIntrinsicCandidate
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

ArrayList의 구현코드를 보면서 add 메서드를 따라가보았다. 참 재밌는 일이 벌어진다.  
결론부터 말하자면 ArrayList는 배열이다. 배열이 아닌척 태연한 척 할 뿐!  
``new ArrayList();`` 가 호출이 되면 빈 Object 배열 객체가 elementData에 할당이 된다. size는 주석의 설명대로 ArrayList의 크기이다.  

add가 호출이되면 또 다른 오버로딩된 add를 호출한다. 넣을 element 와 가지고 있는 element 배열 그리고 size를 넘겨준다. 현재 사이즈가 배열의 길이와 같다면 grow 함수를 호출 그렇지 않다면 해당 인덱스에 값 저장 후에 사이즈를 늘려준다.  

grow 메서드에서는 size+1을 파라미터로 넘겨준다. 이제 이 값을 minCapacity로 받아 Arrays.java 의 copyOf 메서드를 호출한다. 그 전에 minCapacity를 newCapacity를 거치게 하는데, 여러 조건들이 붙지만 이것도 간단히 살펴보면 이전 elementData의 길이에 해당 길이의 반을 더한 길이를 리턴해준다. 근데 이 값이 minCapacity 보다 작다면 minCapacity 값을 리턴한다(2개의 조건이 더 있지만 코드를 직접 보길 바란다).  

이제 copyOf 에서 현재 배열, 새로운 길이, 배열의 클래스 정보를 또 다른 copyOf에 넘긴다.  
넘긴 배열이 Object 배열 클래스 정보와 같으면 해당 길이의 Object 배열을 copy 변수에 초기화 그렇지 않다면 새로운 타입의 배열을 저장한다. 이후 arraycopy를 이용해 기존의 배열의 copy에 앞에서부터 복사하고 이 배열을 리턴해주는 형식이다.  

ArrayList 이 녀석... 겉보기엔 배열같아 보이지는 않지만 속은 배열 그 자체였다! 우아해보이지만 물 속에서 발을 힘차게 구르고 있는 백조 같은 느낌이 들었다.  



```java
    /**
     * Inserts the specified element at the specified position in this
     * list. Shifts the element currently at that position (if any) and
     * any subsequent elements to the right (adds one to their indices).
     *
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        System.arraycopy(elementData, index,
                         elementData, index + 1,
                         s - index);
        elementData[index] = element;
        size = s + 1;
    }
```

내친김에 인덱스에다가 element를 add하는 add 메서드도 한 번 보자.  
해당 인덱스에서 부터를 인덱스+1의 위치에 s-index 만큼 붙이고, 인덱스 위치에 element를 삽입한다.  
정말 재밌게 돌아가는 것을 알 수가 있다. LinkedList랑은 완전 아주아주 다른 녀석이다...  

remove 메서드도 확인결과 arraycopy를 이용하는 것을 확인할 수가 있었다.  
get은 해당 index가 유효한 index 값인지 체크 후에 elementData에서 index로 콕 집어서 리턴해준다.  
contains 는 순차탐색을 하면서 값을 찾는다.  

같은 List를 구현하지만 LinkedList와 이렇게나 다를 수가 있다니 너무 재밌는 것 같다.  

***