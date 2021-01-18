# Comparable과 Comparator의 차이에 대해

## Comparable

Comparable 인터페이스는 해당 인터페이스를 구현한 객체에게 기본 정렬 규칙을 설정하는 목적으로 사용된다. 정렬 수행 시 **기본적으로 적용**되는 정렬 기준이 되는 메서드를 정의하는 인터페이스다.  

```java
int[] nums = {4, 5, 1, 3, 2, 7, 6};
Arrays.sort(nums);
Arrays.stream(nums).forEach(System.out::println);
// 오름차순으로 출력됨
```

Java에서 제공되는 정렬이 가능한 클래스들은 모두 Comparable 인터페이스를 구현하고 있으며, 정렬 시에 이에 맞게 정렬이 수행된다.  
위의 코드도 마찬가지이며 오름차순으로 출력이 된다.  

그렇다면 다음의 경우에는 어떤식으로 출력이 될까 ?   

```java
    public static class Product {
        int price;
        int quantity;

        Product(int price, int quantity) {
            this.price = price;
            this.quantity = quantity;
        }
    }
    public static void main(String[] args) {
        Product product1 = new Product(1000, 4);
        Product product2 = new Product(100, 5);
        Product product3 = new Product(2000, 1);
        Product product4 = new Product(1500, 6);
        Product[] products = {product1, product2, product3, product4};

        Arrays.sort(products);
        Arrays.stream(products).forEach(System.out::println);
    }
```

결과는 ``Exception in thread "main" java.lang.ClassCastException: class tpckg.test$Product cannot be cast to class java.lang.Comparable`` 다음과 같은 예외가 나오게 된다. price로 정렬을 해야할지 quantity로 정렬을 해야할지 그 기준을 모르기에 예외가 터진다.  

```java
    public static class Product implements Comparable<Product> {
        int price;
        int quantity;

        Product(int price, int quantity) {
            this.price = price;
            this.quantity = quantity;
        }

        @Override
        public int compareTo(Product o) {
            return this.price - o.price;
        }
    }
    public static void main(String[] args) {
        Product product1 = new Product(1000, 4);
        Product product2 = new Product(100, 5);
        Product product3 = new Product(2000, 1);
        Product product4 = new Product(1500, 6);
        Product[] products = {product1, product2, product3, product4};

        Arrays.sort(products);
        Arrays.stream(products).forEach(p -> System.out.println(p.price));
    }
```

다음과 같이 Comparable를 받아서 compareTo 메서드를 구현한 후에 출력을 해보면  
100 1000 1500 2000 의 순서로 compareTo 메서드에 정의한 대로 정렬이 되는 것을 알 수가 있다.  

compareTo 메서드는 그것이 구현된 객체와 매개변수로 넘어 온 타겟 객체를 비교하여 원하는 정렬에 따른 int 값을 반환해 주어야 한다. 기준을 정한 후, 이 객체가 매개변수의 객체보다 작으면 음수, 크면 양수 같다면 0을 리턴하게 된다. 양수일 경우 두 객체의 자리가 바뀌게 되고 음수와 0일 때는 바뀌지 않는다.  

int, String 등 기본타입들이 오류 없이 정렬이 되는 것은 이미 해당 클래스 내부에 Comparable들이 구현되어있기 때문이라는 것을 예상할 수가 있다.  

이렇게 알 수 있듯이, Comparable은 **기본 정렬 규칙을 설정**하기 위한 인터페이스이다.  

<br/>

## Comparator

Comparator 인터페이스는 정렬 가능한 클래스들의 **기본 정렬 기준과 다르게 정렬**하고 싶을 때 사용하는 인터페이스다. 보통 내림차순을 사용하고자 할 때 많이 사용하며 익명 클래스를 자주 이용한다.  

```java
    public static class Product implements Comparable<Product> {
        int price;
        int quantity;

        Product(int price, int quantity) {
            this.price = price;
            this.quantity = quantity;
        }

        @Override
        public int compareTo(Product o) {
            return this.price - o.price;
        }
    }

    public static class QuantitySort implements Comparator<Product> {
        @Override
        public int compare(Product o1, Product o2) {
            return o1.quantity - o2.quantity;
        }
    }
    public static void main(String[] args) {
        Product product1 = new Product(1000, 4);
        Product product2 = new Product(100, 5);
        Product product3 = new Product(2000, 1);
        Product product4 = new Product(1500, 6);
        Product[] products = {product1, product2, product3, product4};

        Arrays.sort(products, new QuantitySort());
        Arrays.stream(products).forEach(p -> System.out.println(p.quantity));
    }
```

코드를 보면 Product는 기본 정렬 방식이 price 오름차순이었다. 이것을 특정 상황에서 quantity를 기준으로 삼고 싶어서 Comparator를 구현한 클래스를 생성해서 파라미터에 넣어주었다. 결과는 1, 4, 5, 6으로 quantity로 정렬 되었다.  

```java
Arrays.sort(products, new Comparator<Product>() {
    @Override
    public int compare(Product o1, Product o2) {
        return o2.price - o1.price;
    }
});
Arrays.stream(products).forEach(p -> System.out.println(p.price));
```

다음과 같이 익명 클래스를 이용할 수도 있다. 기본 정렬은 가격 오름차순이지만 내림차순으로 정렬되게끔 해보았다. compare 메서드는 compareTo와는 다르게 비교대상  객체 외부에 생성하기 때문에 비교할 대상 두 개에 대한 정보를 모두 매개변수로 받는다.  

이렇게 알 수 있듯이, Comparator는 **기본 정렬 규칙과 다르게 정렬**하고 싶을 때 사용한다.  

<br/>

***

### Reference

https://dev-daddy.tistory.com/23  

https://gmlwjd9405.github.io/2018/09/06/java-comparable-and-comparator.html