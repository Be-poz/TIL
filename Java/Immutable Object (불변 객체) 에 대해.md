# Immutable Object (불변 객체) 에 대해

불변 객체는 생성 후에 그 상태를 바꿀 수 없는 객체를 말한다. 반대 말로는 가변 객체(mutable)가 있다.  
불변 객체는 복제나 비교를 위한 조작을 단순화 시켜주고 성능 개선에 도움을 주지만, 변경 가능한 데이터를 많이 가지고 있는 경우에는 오히려 독이 되기도 한다.  

```java
class Ex{
    public int num;
    
    public Ex(int num){
        this.num = num;
    }
}
```

위의 클래스는 불변이 아니다. 외부에서 값을 변경할 수 있기 때문이다.  

```java
class Ex{
    private final int num;
    
    pbulic Ex(int num){
        this.num = num;
    }
}
```

위의 클래스는 불변이다. ``final ``이기 때문에 Setter로 변경하려고 해도 불가능하다.  

불변객체를 사용하게되면 해당 객체를 믿고 사용할 수가 있다. 변경되지 않기 때문이다. 신뢰도가 높아진다고 할 수가 있다. 동기화도 따로 필요 없다는 것을 유추할 수 있다.  

불변하게 만들기 위해서 가장 먼저 생각할 수 있는 방법은 위의 코드에서 보였듯이 ``final`` 를 붙이는 것이다. ``fianl`` 를 붙이게 되면 당연히 Setter 메서드도 사용할 수 없다. 하지만 이것은 원시타입의 경우일 때에 그렇고 참조 타입일 때에는 조금 더 신경써줘야 한다.  

<br>

***

#### 참조 타입의 불변 객체  

##### 일반 객체일 경우

```java
public class Animal{
    private final Age age;
    
    public Animal(final Age age){
        this.age = age;
    }
    //getter
}

class Age{
    private int value;
    
    public Age(final int value){
        this.value = value;
    }
    
    public void setValue(final int value){
        this.value = value;
    }

    public int getValue(){
        return value;
    }
}
```

Animal 클래스는 ``final``을 사용하고, Setter 가 없지만 불변객체가 될 수 없다.  
왜냐하면, Animal 클래스의 필드인 Age의 값이 변경될 수 있기 때문이다.  

```java
Age age = new Age(1);
Animal animal = new Animal(age);

animal.getAge().setValue(10);		// 바뀌어버린다!!
```

하지만 다음과 같이 Age 또한 불변처리를 해버리면 Animal 클래스 또한 불변 객체가 된다.

```java
class Age{
    private final int value;
    
    public Age(final int value){
        this.value = value;
    }
    //getter
}
```

**참조 변수도 불변 객체여야 한다.**  

<br>

##### 배열일 경우

```java
public class ArrayObj{
    private final int[] array;
    
    public ArrayObj(final int[] array){
        this.array = Arrays.copyOf(array, array.length);
    }
    
    public int[] getArray(){
        return (array == null) ? null : array.clone();
    }
}
```

```java
        int[] arr1 = {1, 2, 3};
        ArrayObj aobj = new ArrayObj(arr1);
        aobj.getArray()[1] = 2;
        for (Integer a : aobj.getArray()) {
            System.out.println(a);
        }
// 1 2 3 그대로 나온다!
```

**원시 타입이 아니고 Animal과 같은 참조 타입이면 당연히 해당 객체는 불변 객체여야 한다.**  

<br>

##### List일 경우

```java
public class ListObj{
    private final List<Animal> animals;
    
    public ListObj(final List<Animal> animals){
        this.animals = new ArrayList<>(animals);
    }
    
    public List<Animal> getAnimals(){
        return Collections.unmodifiableLIst(animals);
    }
}
```

다음과 같이 설정하면 해결이 된다.  

다시 한 번 생각을 해보자, 만약 List나 배열 같은 경우 생성자에서 그냥 인자를 받아서 그대로 해당 객체의 필드 값에 넣어준다면

```java
    public static void main(String[] args) {
        List<Animal> animals = new ArrayList<>();
        animals.add(new Animal(new Age(1)));

        ListObj listObj = new ListObj(animals);

        for (Animal animal : listObj.getList()) {
            System.out.println(animal.getAge().getValue());
        }
        System.out.println();

        animals.add(new Animal(new Age(2)));
        for (Animal animal : listObj.getList()) {
            System.out.println(animal.getAge().getValue());
        }
    }
```

다음과 같이 넣어준 animals 에 새롭게 값을 추가한다면 listObj 에도 그대로 반영이 되는 결과가 일어날 것이다. (생성자에서 this.list = list 해줬을 때 말하는 거다) 조금 더 간단하게 살펴보자  

```java
    static class ListObj{
        private final List<Integer> list;

        public ListObj(final List<Integer> list) {
            this.list = list;
        }


        public List<Integer> getList() {
            return list;
        }
    }

    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        list.add(3);

        ListObj listObj = new ListObj(list);
        list.add(4);

        for (Integer i : listObj.getList()) {
            System.out.println(i);
        }
    }
```

값이 원래는 3만 나와야 하는데 4까지 출력되는 결과가 나온다. 그리고 getter를 신경써줘야 하는 경우도 한 번 봐보자  

```java
static class ListObj{
    private final List<Integer> list;

    public ListObj(final List<Integer> list) {
        this.list = new ArrayList<>(list);
    }


    public List<Integer> getList() {
        return list;
    }
}

public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(3);

    ListObj listObj = new ListObj(list);

    listObj.getList().add(4);

    for (Integer i : listObj.getList()) {
        System.out.println(i);
    }
}
```

이 경우도 3만 나와야하는 결과가 4까지 출력이 된다는 것이다.  

참조타입을 불변 객체로 사용하고 싶을 때에는 항상 오늘 배운 것을 유의하고 사용하자!!  

***