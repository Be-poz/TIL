# equals, ==, hashCode 의 차이와 재정의 시점에 대해

보통 값 타입들을 비교할 때에 == 연산자를 사용한다.  

```java
int a = 3;
int b = 3;
//a == b   true
```

하지만, String 타입에서는 == 비교를 사용하면 원하는 결과값을 얻어내지 못할 것이다.  
왜냐하면, == 연산은 정확히는 주소값을 비교하는 연산자이기 때문이다. String은 new를 사용해서 새롭게 인스턴스를 만들고 메모리에 올리기 때문에 다른 주소값을 참조하기에 == 연산 시에 false가 리턴된다.  

```java
String a = "abcd";
String b = "abcd";
String c = new String("abcd");
String d = new String("abcd");
System.out.println(a==b);	//true
System.out.println(a==c);	//false
System.out.println(c==d);	//false
```

a와 b는 같은 주소값을 참조하기에 true 나머지는 다르므로 false가 뽑힌 것을 알 수가 있다.  
반면, String의 equals 메서드는 내부의 값을 비교하기 때문에 다음과 같은 결과가 나오게 된다.  

```java
String a = "abcd";
String b = new String("abcd");
System.out.println(a.equals(b))		//true

	public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String aString = (String)anObject;
            if (coder() == aString.coder()) {
                return isLatin1() ? StringLatin1.equals(value, aString.value)
                                  : StringUTF16.equals(value, aString.value);
            }
        }
        return false;
    }
```

equals 내부 구현코드를 살펴보면 내부 값을 비교해서 리턴을 해주는 것을 알 수가 있다.  
== 연산자에 대해서 추가적으로 재밌는 점은  

```java
static int a = 3;
static int b = 3;
System.out.println(a == b);	//false
```

다음과 같은 경우에는 static이 클래스가 메모리에 올라가는 시점에 메모리에 같이 올라가므로 참조하는 주소값이 다르기 때문에 false가 출력이 되는 것을 알 수가 있다.  

hashCode는 해당 값을 hashFunction 한 값을 출력한다.  

```java
String a = "abcd";
String b = new String("df");
String c = new String("abcd");
System.out.println(a.hashCode());	//2987074
System.out.println(b.hashCode());	//3202
System.out.println(c.hashCode());	//2987074

	public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            hash = h = isLatin1() ? StringLatin1.hashCode(value)
                                  : StringUTF16.hashCode(value);
        }
        return h;
    }
```

String의 경우 해당 문자열을 해시화하여 출력을 해준다.  
hashCode는 객체를 다룰 때에 자주 사용하게 된다.  

```java
    class Cat{
        int a;
        int b;

        Cat(int a, int b) {
            this.a = a;
            this.b = b;
        }
    }
```

다음과 같은 ``Cat`` 클래스가 존재할 때,  

```java
Cat cat = new Cat(1, 2);
Cat cat2 = new Cat(1, 2);
Cat cat3 = cat2;
System.out.println(cat == cat2);		//false
System.out.println(cat.equals(cat2);	//false
System.out.println(cat2.equals(cat3);	//true
```

== 연산자와 equals 메서드는 둘 다 false가 나오게 된다.  
주소값이 다르므로 ==는 false가 나오게 될 것이고, 기본적인 equals 또한 내부적으로 ==연산자를 사용하므로 false를 리턴할 것이다. cat2 와 cat3은 동일한 곳을 참조하기에 true를 리턴한 것을 알 수가 있다.  

객체 내에 equals 메서드를 오버라이딩해서 원하는대로 구조를 변경할 수가 있다.  

```java
        @Override
        public boolean equals(Object obj) {
            if(this == obj) return true;
            if(obj instanceof Cat){
                Cat c = (Cat)obj;
                return this.a == c.a && this.b == c.b;
            }
            return false;
        }

System.out.println(cat.equals(cat2);	//true
```

다음과 같이 말이다. 이제 equals 메서드를 실행하였을 때에 내부 값이 같으면 true를 리턴하는 것을 볼 수가 있다. **즉, 객체의 값이 같은지 확인하고 싶을 때에 재정의한다고 볼 수 있겠다.**  

```java
Cat cat = new Cat(1, 2);
Cat cat2 = new Cat(1, 2);
Cat cat3 = cat2;
System.out.println(cat.hashCode());	//2083562754
System.out.println(cat2.hashCode();	//1239731077
System.out.println(cat3.hashCode());//1239731077
```

hashCode는 해당 객체를 앞선 String 과 같이 해시함수를 통과시킨 값을 리턴을 해준다.  
만약 equals를(Cat예제처럼 오버라이딩 한 것 말고 default) 통해서 두 객체가 같다고 판단이 되면 두 객체의 hashCode 값은 같다는 것을 cat2 와 cat3 의 hashCode 결과를 보고 알 수가 있다.  

**그러나, equals 값이 다르다고해서 hashCode값이 항상 다른 것은 아니다. 같을 수 있다. 이런 경우를 해시충돌이라고 부른다.**  

**hashCode나 equals는 재정의하려한다면 둘 다 해주어야 한다.**  

```java
        Map<Cat, Integer> map = new HashMap<>();
        Cat cat = new Cat(1, 2);
        Cat cat2 = new Cat(2, 3);
        Cat cat3 = new Cat(1, 2);
        map.put(cat, 1);
        map.put(cat2, 2);
        System.out.println(map.get(cat3));	// null
```

만약 hashCode를 재정의하지 않는다면 위의 값은 null이 나오게 될 것이다. 기대했던 값은 1 이었을 텐데 말이다. 그 이유는 cat과 cat3의 해시값이 다르기 때문이다. 그래서 해당 해시값으로 저장된 key가 없으므로 null을 출력하게 된다. equals가 아무리 오버라이딩 되어있다고 하더라도 말이다.  

```java
        @Override
        public int hashCode() {
            return Objects.hash(a, b);
        }

		//Object.java의 hash메서드 코드
	    public static int hash(Object... values) {
	        return Arrays.hashCode(values);
	    }
```

위와 같이 hashCode를 오버라이딩 했다. 그런데 여기서 equals가 재정의 되어있지 않다면?! 다시 null이 나오게 된다. 왜냐하면, 그 이유는 해시 값이 같을 때에 일어나는 해시충돌과 관련이 있다.  

해시충돌이 일어나게 되면 java의 HashMap은 chaining 기법을 사용하게 되는데, 이는 해당 해시값이 같은데 key 값이 동일하지 않을 때에 해당 해시값의 버킷이 LinkedList를 가지며 값들을 가지게 된다(동일할 때에는 값을 덮어쓴다, LinkedList 크기가 8이상일 경우에는 Tree의 형태를 띄게 된다).  

이 동일한지에 대한 유무를 equals를 통해 알 수 있는 것이다.  

```java
        Map<Cat, Integer> map = new HashMap<>();
        Cat cat = new Cat(1, 2);
        Cat cat2 = new Cat(1, 2);
        map.put(cat, 1);
        System.out.println(map.get(cat2));	// null
```

hashCode만 재정의 해놓는다면 cat2는 cat과 같은 해시값을 가져 map의 인덱스에는 잘 찾아가게 될 것이다. 하지만 이제 해당 해시값의 버킷에서 list 또는 tree를 탐색할 때에 key 값이 같은 객체를 리턴해야 하는데 equals 메서드가 재정의 되지 않은 상태라면 이 동일성 유무를 판별해 낼 수가 없다는 것이다!!  

따라서 equals 와 hashCode 메서드는 재정의를 같이 하거나 둘 다 하지 않거나 해야한다.  

***