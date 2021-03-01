# 강한참조와 약한참조 그리고 WeakHashMap 에 대해

### 강한 참조

new 할당 후 새로운 객체를 만들어 해당객체를 참조하는 방식이다.  
참조가 해제되지 않는 이상 GC의 대상이 되지 않는다.  

### 약한 참조

```java
Integer prime = 1;
WeakReference<Integer> weak = new WeakReference<Integer>(prime);
```

다음과 같이 ``WeakReference`` Class를 이용하여 생성이 가능하다. prime == null이 되면(해당 객체를 가리키는 참조가 WeakReference 뿐일 경우) GC 대상이 된다.  

```java
Fruit apple;
Fruit orange;

Fruit strong;
WeakReference weak;

public void Test() {
    apple = new Fruit("apple");
    orange = new Fruit("orange");
    
    strong = apple;
    weak = new WeakReference(orange);
    
    Console.WriteLine(strong == null ? "null" : strong.name);//apple 출력
    Console.WriteLine(weak == null ? "null" : weak.name);//orange 출력
    
    apple = null;
    ornage = null;
    
    //GC 강제 수행
    System.GC.Collect(0, GCCollectionMode.Forced);
    System.GC.WaitForFullGCComplete();
    
    //강한참조는 strong이 참조하고 있으므로 객체가 해제되지 않았지만,
    //약한참조는 원본에 null 값을 넣으면 weak이 참조하고 있어도 객체가 해제된다.
    Console.WriteLine(strong == null ? "null" : strong.name);//apple 출력
    Console.WriteLine(weak == null ? "null" : weak.name);//null 출력
}
```

<br/>

### WeakHashMap

일반적인 HashMap의 경우 일단 Map안에 Key와 Value가 put되면 사용여부와 관계없이 해당 내용은 삭제되지 않는다. 만약 Key에 해당하는 객체가 더 이상 존재하지 않게 된다면 해당 객체를 Key로 하는 HashMap의 Element도 더 이상 꺼낼 일이 없을 경우가 발생할 것이다.  

WeakHashMap은 WeakReference의 특성을 이용하여 HashMap의 Element를 자동으로 GC 해버린다. Key에 해당하는 객체가 더 이상 사용되지 않는다고 판단되면 제거한다는 의미이다.  

```java
    public static void main(String[] args) {
        WeakHashMap<Fruit, String> map = new WeakHashMap<>();
        Fruit apple = new Fruit("apple");
        Fruit orange = new Fruit("orange");
        map.put(apple, "test a");
        map.put(orange, "test b");
        apple = null;
        System.gc();
        map.entrySet().forEach(System.out::println );
    }
```

위 코드의 결과로 test a 에 관한 내용은 출력되지 않는다.  

주의할 점은 만약 기존에 캐싱되어 있는 경우의 객체이다. 대표적으로 String이나 Integer -128 ~ 127 범위의 경우 캐싱이 되어있으니(String은 string pool)  ``new``를 이용해서 사용하도록 하자.  

***

### Reference

https://aroundck.tistory.com/3057  

http://blog.breakingthat.com/2018/08/26/java-collection-map-weakhashmap/  

https://ktko.tistory.com/entry/%EC%9E%90%EB%B0%94-%EA%B0%95%ED%95%9C%EC%B0%B8%EC%A1%B0Strong-Reference%EC%99%80-%EC%95%BD%ED%95%9C%EC%B0%B8%EC%A1%B0Weak-Reference