# Reflection 사용법 정리

## 클래스 객체 검색

### Object.getClass();

```java
    class Bepoz {
        int value;

        public Bepoz(int value) {
            this.value = value;
        }
    }

Bepoz bepoz = new Bepoz(10);
Class<? extends Bepoz> aClass = bepoz.getClass();
```

### .class 를 이용

```java
Class<Bepoz> bepozClass = Bepoz.class;
```

### Class.forName()

```java
Class<?> aClass1 = Class.forName("reflection.Bepoz");
//Bepoz 클래스는 reflection 패키지 내부에 있음
```

### 기본 유형 래퍼의 TYPE 필드

```java
Class<Double> type = Double.TYPE;
Class<Integer> type1 = Integer.TYPE;
Class<Void> type2 = Void.TYPE;
```

### 클래스를 반환하는 메서드

```java
public class Bepoz extends Parent {
    int value;

    public Bepoz(int value) {
        this.value = value;
    }
}

Class<?> superclass = bepoz.getClass().getSuperclass();
Class<? super Bepoz> superclass1 = Bepoz.class.getSuperclass();
//Parent.class 가 반환됨
```

<br/>

## 필드, 메서드, 생성자 이용하기

![image](https://user-images.githubusercontent.com/45073750/132459346-a09f8bbd-0fad-4873-90e8-a10f75d0f22f.png)

접근 제어자를 무시하고 갖고오고 싶으면 ``Declared`` 붙은 것 사용하자. 하지만, 부모한테 있는 것들은 받아올 수 없다.  
없이 사용하면 ``private`` 은 들고 올 수 없다. ``default-private`` 도 안되는 것을 확인했다.  

### 필드 가져오기

```java
public class Bepoz extends Parent {

    public int publicField;
    private int privateField;

    public Bepoz(int publicField) {
        this.publicField = publicField;
    }

    public int getPublicField() {
        return publicField;
    }

    public void setPublicField(int publicField) {
        this.publicField = publicField;
    }
}

Field pubField = aClass.getField("publicField");
//public int reflection.Bepoz.publicField
Field priField = aClass.getDeclaredField("secret");
//private int reflection.Bepoz.privateField
```

### 필드 값 변경하기

```java
        Bepoz bepoz = new Bepoz(10);
        Class<? extends Bepoz> aClass = bepoz.getClass();
        Field pubField = aClass.getField("publicField");
        pubField.set(bepoz, 100);
        Field priField = aClass.getDeclaredField("privateField");
        priField.setAccessible(true);
        priField.set(bepoz, 1000);
```

``perivateField`` 의 경우 ``private`` 이어서 바로 ``set`` 으로 변경할 수가 없다.  
``setAccessible(true);`` 를 이용해서 ``set`` 이 가능하게끔 하였다.  

### 메서드 가져오고 실행하기

```java
    public void publicMethod() {
        System.out.println("publicMethod");
    }

    public void publicMethod(int param) {
        System.out.println("publicMethodWithParams - " + param);
    }

//Bepoz 클래스에 추가
Method publicMethod = aClass.getMethod("publicMethod");
Method publicMethodWithParam = aClass.getMethod("publicMethod", int.class);
publicMethod.invoke(bepoz);
publicMethodWithParam.invoke(bepoz, 100);
```

마찬가지로 ``getDeclaredMethod``가 마련되어 있다. ``main(String[] args)`` 함수 호출을 원할 경우에는 ``invoke(Object obj, Object args...)`` 의 ``obj`` 에 ``null`` 을 넣어주자.  

### 생성자 가져오고 객체 생성하기

```java
Bepoz bepoz = new Bepoz(10);
Class<? extends Bepoz> aClass = bepoz.getClass();

Constructor<? extends Bepoz> constructor = aClass.getConstructor(int.class);
Bepoz madeBepoz = constructor.newInstance(10000);
assertThat(madeBepoz.publicField).isEqualTo(10000);
```

생성자 또한 메서드와 마찬가지로 ``private`` 생성자를 가져오고 생성하기를 원한다면,  

```java
Constructor<? extends Bepoz> constructor = aClass.getDeclaredConstructor(int.class);
constructor.setAccessible(true);
Bepoz madeBepoz = constructor.newInstance(10000);
assertThat(madeBepoz.publicField).isEqualTo(10000);
```

``Declared~~`` 와 ``setAccessible`` 를 사용하면 된다.  

<br/>

docs를 보고 간단히 따라해보았다. 다른 정보가 필요하다면 docs에서 찾아서 더 사용하면 될 것 같닫.  

***

### REFERENCE

https://docs.oracle.com/javase/tutorial/reflect/index.html

