# 코틀린 쿡북

### 널 허용 변수

```kotlin
class Person(val first: String, val middle: String?, val last:String)

val me = Person("kang", null, "bepoz")
```

? 붙이면 null이 들어올 수도 있다는 뜻이다.  

```kotlin
var me = Person("kang", null, "bepoz")

if (me.middle != null) {
  val middleNameLength = me.middle.length      // 타입 변환 불가능!
  val middleNameLength = me.middle!!.length
}
```

var 는 변경이 가능한 변수이고, val은 자바에서의 final 이다.  
위에서 ``var me`` 로 해놨기 때문에 if 문으로 널체크를 해도 그 이후에 middle 값이 null이 될 수 있기 때문에 주석 부분에 쓰여진대로 타입 변환이 불가능하다. ``!!`` 를 사용해서 널이 아님은 단언(assert) 할 수 있다. 하지만 지양해야 한다. 잘못하다간 NPE가 발생할 수 있다.  

```kotlin
val middleNameLength = me.middle?.length
val middleNameLength = me.middle?.length ?: 0
```

``?`` 를 붙이면 널이 들어올 수도 있다는 뜻이다. ``?:`` 는 엘비스 연산자인데, 만약 널이 들어온다면 우측의 값으로 주겠다는 뜻이다.  
엘비스 연산자의 이름은 엘비스가 옆으로 누운 모양이기 때문이다!!!  

<br/>

### 널 허용성 지시자

```kotlin
var s: String = "Hello World!"
var t: String? = null
```

널을 집어넣기 위해서는 ``?`` 가 붙어야 한다.  

<br/>

### 자바를 위한 메서드 중복

```kotlin
@JvmOverloads
fun addProduct(name: String, price: Double = 0.0, desc: String? = null) =
    "Adding product with $name, ${desc ?: "None"}, and " +
            NumberFormat.getCurrencyInstance().format(price)

data class Product @JvmOverloads constructor(
    val name: String,
    val price: Double = 0.0,
    val desc: String? = null
)

//java
@Test
void checkOverloads() {
  assertAll("overloads called from Java",
            () -> System.out.println(OverloadsKt.addProduct("Name", 5.0, "Desc")),
            () -> System.out.println(OverloadsKt.addProduct("Name", 5.0)),
            () -> System.out.println(OverloadsKt.addProduct("Name"))
           );
}
```

코틀린은 기본값을 지정하고 호출 시에 생략을 할 수 있으나 자바는 그럴 수 없다. 자바에서 기본값이 지정되어 있는 코틀린 코드를 들고오기 위해서는 ``@JvmOverloads`` 를 사용해야 한다.  

<Br/>

### 명시적으로 타입 변환하기

```kotlin
var intVar = 3
//var longVar = intVar
var longVar = intVar.toLong()
```

코틀린에서는 자동 형변환이 되지 않는다. 변환 메서드를 이용해 직접 변환시켜주어야 한다.  
``toByte()``, ``toChar()``, ``toShort()``, ``toInt()``, ``toLong()``, ``toFloat()``, ``toDouble()`` 를 이용하면 된다.  

<Br/>

### 다른 기수로 출력, 거듭 제곱

```kotlin
2.toString(2) == "10"
42.toString(2) == "101010"
"11".toInt(2) == 3
"10".toInt(2) == 2

2.0.pow(2) == 4
//double과 float에는 pow가 있지만 int, long에는 없기 때문에 중위(infix) 함수를 만들어서 해결할 수 있다.
infix fun Int.`**`(x: Int) = toDouble().pow(x).toInt()
2 `**` 2 == 4
```

<br/>

### to로 Pair 인스턴스 생성하기

```kotlin
// 코틀린 표준 라이브러리에있는 mapOf 함수의 시그니처, vararg는 가변인자
fun <K,V> mapOf(vararg pairs: Pair<K,V>): Map<K,V>

//Pair는 first, second의 이름으로 두 개의 원소를 갖는 데이터 클래스다. 시그니처는 다음과 같다.
data class Pair<out A, out B> : Serializable

//to 함수를 사용하여 Pair를 만드는 것이 더 일반적이다.
public infix fun <A,B> A.to(that: B): Pair<A,B> = Pair(this, that)

@Test
fun `create map using infix to function`() {
  val map = mapOf("a" to 1, "b" to 2, "c" to 3)
  assertAll(
    { assertThat(map, hasKey("a")) },
    { assertThat(map, hasKey("b")) },
    { assertThat(map, hasKey("c")) },
    { assertThat(map, hasKey("c")) },
    { assertThat(map, hasValue(1)) },
    { assertThat(map, hasValue(2)) },
    { assertThat(map, hasValue(3)) },
  )

  val p1 = Pair("a", 1)
  val p2 = "a" to 1

  assertAll(
    { assertThat(p1.first, `is`("a")) },
    { assertThat(p1.second, `is`(1)) },
    { assertThat(p2.first, `is`("a")) },
    { assertThat(p2.second, `is`(1)) },
    { assertThat(p1, `is`(equalTo(p2))) }
  )
}
```

<br/>

### 사용자 정의 획득자와 설정자 생성하기

코틀린은 기본적으로 public 이다. 그냥 객체.필드 로 접근해서 사용한다.  

```kotlin
class Person {
  var name: String = "bepoz"
  var age: Int = 100
}

Person().name						//이렇게 가져오고
Person().name = "kang"  //이렇게 할당한다.
```

그런데 ``set()`` 과 ``get()`` 을 이용하여 할당 시, 가져올 시에 사용자가 원하는 입맛대로 조절할 수 있다.  

```kotlin
class Person {
  var name: String = "bepoz"
      get() {
        return field + " get!"
      }
      set(value) {
        field = field + value 
      }
}

val person = Person()
println(person.name)			// bepoz get!
person.name = " plus"			// name이 bepoz plus가 된다
println(person.name)			// bepoz plus get!
```

이런 식이다. 주의할 점은 내부에서 field가 아닌 필드명을 사용하면(위의 코드에서는 name) name을 쓰는순간 get이 호출된다는 것이다. 만약 set의 로직을 ``field = name + value`` 로 하게된다면 ``name`` 을 호출했으니 갖고오기 위해 ``get()`` 을 호출하고  
결국 ``field = "bepoz get!" + value`` 가 되는 것이다. 만약 ``get()`` 에서 ``return name + "!"`` 하게 된다면 계속해서 ``get()`` 을 호출해서 무한루프에 빠지게된다.  

<br/>

### const와  val의 차이 이해하기

```kotlin
class Task(val name: String, _priority: Int = DEFAULT_PRIORITY) {

    companion object {
        const val MIN_PRIORITY = 1
        const val MAX_PRIORITY = 5
        const val DEFAULT_PRIORITY = 3
    }

    var priority = validPriority(_priority)
    set(value) {
        field = validPriority(value)
    }

    private fun validPriority(p: Int) = p.coerceIn(MIN_PRIORITY, MAX_PRIORITY)
}
```

val은 값이 변경 불가능한 변수임을 나타낸다. const는 컴파일 타임 상수이며 반드시 객체나 동반 객체(companion object) 선언의 최상위 속성 또는 멤버여야 한다. 컴파일 타임 상수는 문자열 또는 기본 타입의 래퍼 클래스이며, 사용자 정의 획득자(getter)를 가질 수 없다.  
컴파일 타임 상수는 컴파일 시점에 값을 사용할 수 있도록 main 함수를 포함한 모든 함수의 바깥쪽에서 할당돼야 한다.  

<Br/>

### 데이터 클래스 정의하기

클래스 정의에 data를 추가하면 코틀린 컴파일러는 일관된  equals와  hashCode 함수, 클래스와 속성 값을 보여주는 toString 함수, copy 함수와 구조 분해를 위한 component 함수 등 일련의 함수를 생성한다.  

```kotlin
data class Product(
  val name: String,
  var price: Double,
  var onSale: Boolean = false
)

val product1 = Product("baseball", 10.0)
val product2 = Product("baseball", 10.0, false)

assertEquals(product1, product2)
assertEquals(product1.hashCode(), product2.hashCode())

val product1 = Product("baseball", 10.0)
val product2 = product1.copy(name = "basketball")
// copy를 이용하면 원본과 같은 속성 값을 가진 인스턴스로 복사를 해준다.


assertEquals(product2.name, "basketball")
assertThat(product2.price, `is`(closeTo(12.0, 0.01)))
```

``copy()`` 는 깊은 복사가 아니고 얕은 복사이다.  

```kotlin
data class OrderItem(val product: Product, val quantity: Int)

val item1 = OrderItem(Product("baseball", 10.0), 5)
val item2 = item1.copy()

assertTrue { item1 == item2 }
assertFalse { item1 === item2 }
assertTrue { item1.product == item2.product }
assertTrue { item1.product === item2.product }
```

OrderItem 비교 시에 얕은 복사이기에 ``===`` 는 false로 나온다. ``equals`` 함수는 ``==`` 를 통해 호출되고, 레퍼런스 동등 연산자가 ``===`` 이다. OrderItem 내부의 Product는 같은 인스턴스를 공유하는 것을 확인할 수가 있다.  

<br/>

### 지원 속성 기법

```kotlin
class Customer(val name: String) {
  private var _messages: List<String>? = null

  val messages: List<String>
  get() {
    if (_messages == null) {
      _messages = loadMessages()
    }
    return _messages!!
  }

  private fun loadMessages(): MutableList<String> =
  mutableListOf(
    "Initial contact",
    "Convinced them to use Kotlin",
    "Sold training class. Sweet."
  ).also { println("Loaded messages") }
}

val customer = Customer("bepoz").apply { messages }
assertEquals(3, customer.messages.size)
```

객체.프로퍼티 를 하게되면 ``get()`` 을 통한 호출을 하므로 ``val message: List<String>`` 처럼 초기화를 안해줘도 바로 밑에 ``get()`` 을 재정의해줬기 때문에 괜찮다. 처음 messages에 접근할 때에 데이터들이 load 된 것을 나타낸 코드다.  지연로딩이기도 한데 조금 더 쉽게 구현할 수 있다.  

```kotlin
class Customer(val name: String) {
  val messages: List<String> by lazy { loadMessages() }

  private fun loadMessages(): MutableList<String> =
  mutableListOf(
    "Initial contact",
    "Convinced them to use Kotlin",
    "Sold training class. Sweet."
  ).also { println("Loaded messages") }
}
```

<br/>

### 연산자 오버로딩

+, -, *, / 등등 여러 연산자를 오버로딩 하여 사용할 수 있다.

```kotlin
data class Point(val x: Int, val y: Int)

operator fun Point.unaryMinus() = Point(-x, -y)

val point = Point(10, 20)
println(-point) // Point(x=-10, y=-20)
```

연산자와 함수명은 다음과 같다.  

* a+b : plus
* a-b : minus
* a*b : times
* a/b : div
* a%b : rem
* +a : unaryPlus
* -a : unaryMinus
* !a : not
* ++a, a++ : inc
* --a, a-- : dec

<br/>

### lateinit

```kotlin
class LateInitDemo {
lateinit var name: String
}

@Test
fun `lateinit test`() {
assertThrows<UninitializedPropertyAccessException> {
LateInitDemo().name
}

assertDoesNotThrow { LateInitDemo().apply { name = "bepoz" } }
}
```

lateinit 사용은 최대한 지양해야한다. 

<br/>

### equals 재정의를 위해 안전 타입 변환, 레퍼런스 동등, 엘비스 사용하기

두 객체를 동등하다고 판단하면 두 객체의 hashCode도 같아야 하며, equals 함수가 재정의되면  hashCode 함수도 재정의돼야 한다.  
equals 구현의 좋은 예는 다음과 같다.  

```kotlin
override fun equals(other: Any?): Boolean {
  if (this === other) return true
  val otherVersion = (other as? KotlinVersion) ?: return false
  return this.version == otherVersion.version
}

//KotlinVersion은 클래스명이다.
```

레퍼런스 동등성을 확인하고, 안전 타입 변환이 되는지 확인하고 그 후에 속성 동등 여부를 검사한다.  

hashCode는 만약 어떤 클래스의 프로퍼티로 name이 있다면 ``override fun hashCode() = name.hashCode()`` 를 하면된다.  

<br/>

### 싱글톤 생성하기

```kotlin
object MySingleton {
  val myProperty = 3

  fun myFunction() = "Hello"
}
```

코틀린에서 싱글톤을 생성할 때에는 ``object`` 를 붙이면 된다.  
접근시에는 ``Mysingleton.myProperty`` , ``Mysingleton.myFunction()`` 이렇게 접근하면 된다.  

<br/>

### Nothing 

결코 존재할 수 없는 값을 나타내기 위해 사용된다.  

```kotlin
public class Nothing private constructor()
// private 으로 생성자를 만들어놨다
```

그럼 이거를 언제쓰냐? 

```kotlin
fun doNothing(): Nothing = throw Exception("Nothing at all")
// 리턴 타입을 명시해야 하는데 리턴하지 않으므로 Nothing 리턴

val x = null
// 정보가 없기 때문에 추론된 타입은 Nothing 이다.

val x = if (Random.nextBoolean()) "true" else throw Exception("nope")
// Nothing은 모든 타입의 하위 타입이다. 위의 코드는 String vs Nothing 이기 때문에 String인 것을 컴파일러가 알 수 있다. 
```

<br/>

### fold 사용하기

fold 함수는 배열 또는 반복 가능한 컬렉션에 적용할 수 있는 축약 연산이다.  

```kotlin
inline fun <R> Iterable<T>.fold(
		initial: R,
  	operation: (acc: R, T) -> R
): R
```

첫 번째 인자는 accumulator(누적자)의 초기값이며 두 번째는 두 개의 인자를 받아 누적자를 위해 새로운 값을 리턴하는 함수다.  

```kotlin
fun sum(vararg nums: Int) = nums.fold(0) { acc, n -> acc + n }
```

```kotlin
fun recursiveFactorial(n: Long): BigInteger = 
	when (n) {
    0L, 1L -> BigInteger.ONE
    else -> BigInteger.valueOf(n) * recursiveFactorial(n - 1)
  }
// 위의 식을 아래와 같이 변경 가능
fun recursiveFactorial(n: Long): BigInteger = 
	when (n) {
    0L, 1L -> BigInteger.ONE
    else -> (2..n).fold(BigInteger.ONE) { acc, i ->
                                        acc * BigInteger.valueOf(i)}
  }
```

누적 값의 타입과 범위의 원소 타입이 다를 수도 있다.  

```kotlin
fun fibonacciFold(n: Int) =
	(2 until n).fold(1 to 1) { (prev, curr), _ -> curr to (prev + curr) }.second
```

<br/>

### reduce 함수를 사용해 축약하기

reduce는 fold와 비슷한데 누적자의 초기값이 없다는 것이 큰 차이점이다. 누적자의 초기값은 컬렉션의 첫 번째 값으로 초기화된다.  
비어있는 컬렉션은 예외를 발생시킨다.  

```kotlin
fun sumReduce(vararg nums: Int) = nums.reduce { acc, i -> acc + i }
fun sumReduceDoubles(vararg nums: Int) = nums.reduce { acc, i -> acc + 2 * i }
```

컬렉션이 { 3, 1, 4, 1, 5, 9 } 라고 하였을 때 두 번째 함수에서 조심해야 할 것은 3은 초기값을 초기화하는데 사용되기 때문에 i 로 들어가지 않는다는 것이다. i는 1부터가 될 것이다. 따라서 모두 합한 값의 2배인 46이 아니라 43이 결과가 된다.  

<Br/>

### 꼬리 재귀 적용하기

```kotlin
fun recursiveFactorial(n: Long): BigInteger =
when(n) {
  0L, 1L -> BigInteger.ONE
  else -> BigInteger.valueOf(n) * recursiveFactorial(n - 1)
}

assertThrows<StackOverflowError> { recursiveFactorial(10_000) }
```

새로운 재귀 호출은 콜 스택에 프레임을 추가하기 때문에 메모리를 초과하게 된다.  
꼬리 재귀로 알려진 접근법은 콜 스택에 새 스택 프레임을 추가하지 않게 구현하는 특별한 종류의 재귀다. 꼬리 재귀의 구현을 위해 지귀 호출이 연산의 마지막에 수행되도록 팩토리얼 알고리즘을 다시 작성해보겠다.  

```kotlin
tailrec fun factorial(n: Long, acc: BigInteger = BigInteger.ONE): BigInteger =
when (n) {
  0L -> BigInteger.ONE
  1L -> acc
  else -> factorial(n - 1, acc * BigInteger.valueOf(n))
}

assertThat(factorial(75000).toString().length, `is`(333061))
```

디컴파일을 해보면 재귀 호출은 컴파일러에 의해 while 루프를 사용하는 반복 알고리즘으로 리팩토링 되는 것을 확인할 수가 있다.  
재귀함수가 오직 스스로만 호출하는 형태를 띄어야만 리팩토링 된다. ``1 + 자기자신함수`` 이런거는 안된다는 뜻이다. 오직 자기 자신만!  

<br/>

### 배열 다루기

```kotlin
val strings = arrayOf("this", "is", "an", "array", "of", "strings")
val indices = strings.indices
for ((index, value) in strings.withIndex()) {}
for (string in strings) {}

val nullStringArray = arrayOfNulls<String>(5)
val squares = Array(5) { i -> (i * i).toString() }
```

arrayOf 로 배열을 사용할 수 있으며 널로만 채워진 배열을 생성할 수 있다. 이 때에도 특정 타입을 선택해야 한다는 점이 특이한 점이다.  
Array 클래스를 이용해서 생성할 때에는 람다를 넣어 배열 생성에 로직을 가할 수 있다. 위의 결과는 {"0", "1", "4", "9",  "16"} 이 된다.  
오토박싱과 언박싱 비용을 방지하기 위해 기본 타입을 나타내는 클래스도 있다. ex) ``boolean``, ``ArrayOf``, ``byteArrayOf``, ``intArrayOf`` 등등..  

코틀린에는 명시적인 기본 타입은 없지만 값이 널 허용 값인 경우 생성된 바이트코드는  Integer와 와  Double과 같은 자바 래퍼 클래스를 사용하고, 널 비허용 값인 경우 생성된 바이트코드는  int와 double 같은 기본 타입을 사용한다.  

<Br/>

### 컬렉션 생성하기

```kotlin
val list = listOf<Int>(1, 2, 3, 4)
val mutableList = mutableListOf<Int>(1, 2, 3, 4)

setOf<>()
mutableSetOf<>()

mapOf<>()
mutableMapOf<>()

val list = LinkedList<Int>()
list.add(3)
list.add(1)
list.addLast(999)
list.addFirst(0)
list[2] = 4
list.addAll(listOf(11,22,33))
for (i in list) {
  println(i)
}
// 0 3 4 999 11 22 33
```

기본적인 코틀린 컬렉션은 불변이다. 가변을 원할 때에는 ``mutable`` 이 붙은 것을 사용하면 된다.  

<Br/>

### 컬렉션에서 읽기 전용 뷰 생성하기

```kotlin
val mutableNums = mutableListOf(1, 2, 3, 4, 5)
val readOnlyNums = mutableNums.toList()
assertEquals(mutableNums, readOnlyNums)
assertNotSame(mutableNums, readOnlyNums)
// toList() 를 이용해서 불변인 읽기 전용 컬렉션을 생성할 수 있다. 내부 값은 같으나 같은 객체를 나타내는 것은 아니다.
// mutableNums에 값을 추가한다고해서 readOnlyNums에 추가되지 않는다.
// 밑의 코드처럼 List 타입의 레퍼런스에 가변 리스트를 할당하면 mutableNums에 값을 추가해도 반영이 된다.

var readOnlySameNums: List<Int> = mutableNums
assertEquals(mutableNums, readOnlySameNums)
assertSame(mutableNums, readOnlySameNums)
```

<br/>

### 컬렉션에서 맵 만들기

```kotlin
val keys = 'a' .. 'f'
val map = keys.associate { it to it.toString().repeat(5).capitalize() }
println(map)

val map2 = keys.associateWith { it.toString().repeat(5).capitalize() }
println(map2)

//{a=Aaaaa, b=Bbbbb, c=Ccccc, d=Ddddd, e=Eeeee, f=Fffff} 둘 다 결과값은 다음과 같다.
```

``associate`` 은 인자가 ``Pair<Char, String>`` 이며, ``associateWith`` 는 ``String`` 인 상태이다.  

<br/>

### 컬렉션이 빈 경우 기본값 리턴하기

```kotlin
data class Product(val name: String, var price: Double, var onSale: Boolean = false)

fun onSaleProducts_ifEmptyCollection(products: List<Product>) =
products.filter { it.onSale }
.map { it.name }
.ifEmpty { listOf("none") }
.joinToString ( separator = ", " )

fun onSaleProducts_ifEmptyString(products: List<Product>) =
products.filter { it.onSale }
.map { it.name }
.joinToString ( separator = ", " )
.ifEmpty { "none" }
```

전자는 컬렉션이 비었을 경우 기본 리스트를 제공하는 것이고 후자는 빈 문자열인 경우 기본 문자열을 제공하는 것이다.  

<Br/>

### 주어진 범위로 값 제한하기

```kotlin
val range = 3..8
assertThat(5, `is`(5.coerceIn(range)))
assertThat(range.start, `is`(1.coerceIn(range)))
assertThat(range.endInclusive, `is`(9.coerceIn(range)))
```

``coerceIn`` 은 range 안의 값이면 그대로 return, 경계값을 넘어갔다면 range의 최소, 최대값을 리턴한다.  

<br/>

### 컬렉션을 윈도우로 처리하기

```kotlin
val range = 0..10
val chunked = range.chunked(3) // 0,1,2 / 3,4,5 / 6,7,8 / 9,10

assertThat(range.chunked(3) { it.sum() }, `is`(listOf(3, 12, 21, 19)))
range.chunked(3) { it.sum()}
```

```kotlin
public fun <T, R> Iterable<T>.chunked(size: Int, transform: (List<T>) -> R): List<R> {
    return windowed(size, size, partialWindows = true, transform = transform)
}
// chunked의 내부구현은 windowed를 이요한다.
// 인자 순서대로, 각 윈도우에 포함될 원소의 개수, 각 단계마다 전진할 원소의 개수, 마지막 부분이 필요한 원소의 개수를 갖지 못한 경우, 해당 부분을 그대로 유지할지 여부를 알려주는 불리언 값

val windowed = range.windowed(3, 2, true)
for (ints in windowed) {
  println(ints)
}
// 0,1,2 / 2,3,4 / 4,5,6 / 6,7,8 / 8,9,10 / 10
```

<br/>

### 리스트 구조 분해하기

```kotlin
val list = listOf("a", "b", "c", "d", "e", "f", "g")
val (a, b, c, d, e) = list
println("$a $b $c $d $e") // a b c d e
```

위의 방법으로  list에 접근할 수 있는 이유는 코틀린이 지원하는  ``componentN`` 이라는 함수가 list 내부에 정의되어 있기 때문이다.  
component1 .. component5 까지 정의되어 있다. 즉 앞 5개의 값까지 가져올 수 있다는 것이다.  

```kotlin
data class Person(val name: String, val age: Int, val isMarried: Boolean)

val me = Person("kang", 100, false)
val (name, _, isMarried) = me
```

다음과 같이 사용할 수 있는 이유도 모두 데이터 클래스의 주 생성자에 들어있는 프로퍼티에 대해서 컴파일러가 자동으로  ``componoentN`` 함수를 만들어주기 때문이다. 사용하지 않는 값은 밑줄로 대체하여 사용할 수 있다.  

<Br/>

### 다수의 속성으로 정렬하기

```kotlin
data class Person(val name: String, val age: Int, val isMarried: Boolean)

val people = listOf(
  Person("abc", 21, true),
  Person("aba", 21, true),
  Person("abd", 11, false),
  Person("pqa", 61, true),
  Person("zx", 51, false),
)

val sorted = people.sortedWith(
  compareBy({ it.age }, { it.name }, { it.isMarried })
)

sorted.forEach { println(it) }
/*
Person(name=abd, age=11, isMarried=false)
Person(name=aba, age=21, isMarried=true)
Person(name=abc, age=21, isMarried=true)
Person(name=zx, age=51, isMarried=false)
Person(name=pqa, age=61, isMarried=true)
```

```kotlin
val comparator = compareBy(Person::age)
.reversed()
.thenBy(Person::name)
.thenBy(Person::isMarried)
val sorted2 = people.sortedWith(comparator)

sorted2.forEach{ println(it)}
/*
Person(name=pqa, age=61, isMarried=true)
Person(name=zx, age=51, isMarried=false)
Person(name=aba, age=21, isMarried=true)
Person(name=abc, age=21, isMarried=true)
Person(name=abd, age=11, isMarried=false)
```

이런식으로 comparator를 따로 정의한 후에 사용할 수도 있다.  

<br/>

### 사용자 정의 이터레이터 정의하기

```kotlin
data class Player(val name: String)
class Team(val name: String, 
           val players: MutableList<Player> = mutableListOf()){

  fun addPlayers(vararg people: Player) {
    players.addAll(people)
  }
}

val team = Team("Warriors")
team.addPlayers(Player("kang"), Player("kim"), Player("lee"))

for (player in team.players) {
  println(player)
}
```

팀의 플레이어 목록에 접근하려면 위와 같이 했어야 했을 것이다. 이를 iterator 라는 이름의 연산자 함수를 정의해서 간단하게 만들 수 있다.  

```kotlin
class Team(val name: String,
           val players: MutableList<Player> = mutableListOf()): Iterable<Player> {

  override operator fun iterator(): Iterator<Player> = players.iterator()

  fun addPlayers(vararg people: Player) {
    players.addAll(people)
  }
}
```

또는 그냥 ``Team`` 클래스 안에 ``operator fun iterator(): Iterator<Player> = players.iterator()`` 를 선언하는 방법을 사용할 수도 있다. Team 클래스가 Iterable 인터페이스를 구현하도록 변경되었고 그 이유는 Iterable 인터페이스에 추상 연산자 함수  iterator가 있기 때문이다. 그리고 이렇게 되었을 때에 아래와 같이 동작한다.  

```kotlin
team.map { it.name }.joinToString() 
//kang, kim, lee
```

<br/>

### 타입으로 컬렉션을 필터링하기

```kotlin
val list = listOf("a", LocalDate.now(), 3, 1, 4, "b")

val strings = list.filterIsInstance<String>()
val stringsMutable = list.filterIsInstanceTo(mutableListOf<String>())
```

``filterIsInstance`` 를 통해 원하는 타입을 필터링해서 추출할 수 있다.  
``filterIsInstanceTo`` 를 통해 원하는 컬렉션의 타입을 명시해 해당 타입의 인스턴스로 컬렉션을 채울 수도 있다.  

<br/>

### 범위를 수열로 만들기

코틀린에서 1 .. 5 처럼 범위를 지정할 수 윘는데 이것은 Comparable 인터페이스를 구현하는 모든 제네릭 타입 T에  ``rangeTo`` 라는 이름의 확장 함수가 추가되어 있기 때문이다. 

```kotlin
val startDate = LocalDateTime.now()
val midDate = startDate.plusDays(3)
val endDate = startDate.plusDays(5)

val dateRange = startDate .. endDate

assertTrue(startDate in dateRange)
assertTrue(midDate in dateRange)
assertTrue(endDate in dateRange)
assertTrue(startDate.minusDays(1) !in dateRange)
assertTrue(endDate.plusDays(1) !in dateRange)

for (date in dateRange)							// 컴파일 에러
(startDate .. endDate).forEach			// 컴파일 에러
```

이런 식으로 사용할 수 있다. 하지만 범위를 순회할 수는 없는데 그 이유는 범위가 수열이 아니라는 점이다. 수열은 순서 있는 값의 연속이다. 사용자 정의 수열은 표준 라이브러리인  ``IntProgression``, ``LongProgression``, ``CharProgression`` 처럼 ``Iterable`` 인터페이스를 구현해야 한다.  

```kotlin
internal class LocalDateProgressionIterator(
  start: LocalDate,
  val endInclusive: LocalDate,
  val step: Long
) : Iterator<LocalDate> {

  private var current = start
  override fun hasNext() = current <= endInclusive
  override fun next(): LocalDate {
    val next = current
    current = current.plusDays(step)
    return next
  }
}

class LocalDateProgression(
  override val start: LocalDate,
  override val endInclusive: LocalDate,
  val step: Long = 1
) : Iterable<LocalDate>, ClosedRange<LocalDate> {

  override fun iterator(): Iterator<LocalDate> = LocalDateProgressionIterator(start, endInclusive, step)

  infix fun step(days: Long) = LocalDateProgression(start, endInclusive, step)
}
```

<br/>

### 지연 시퀀스 사용하기

```kotlin
(100 until 200)
	.map { println("map value: ${it}"); it * 2 }  // 100개 계산
	.filter { println("filter value: ${it}"); it % 3 == 0 }	// 100개 계산
	.first()
/*
map value: 100
...
map value:199
filter value: 200
...
filter value: 398 */

(100 until 200)
	.map { println("map value: ${it}"); it * 2 }	// 100개 계산
	.first { println("filter value: ${it}"); it % 3 == 0 }	// 3개 계산
/* 
map value: 100
...
map value: 199
filter value: 200
filter value: 202
filter value: 204
```

후자의 예제는 첫 번째 원소를 발견하는 순간 진행을 멈춘다. 특정 조건에 다다를 때까지 오직 필요한 데이터만을 처리하는 방식을 쇼트 서킷이라 부른다.  

```kotlin
(100 until 200).asSequence()
						.map { println("map value: ${it}"); it * 2 }						
						.filter { println("filter value: ${it}"); it % 3 == 0 } 
						.first()
/*
map value: 100
filter value: 200
map value: 101
filter value: 202
map value: 102
filter value: 204
```

시퀀스를 사용하게되면 각 원소는 다음 원소로 진행하기 전에 완전한 전체 파이프라인에서 처리되어 6개의 연산만이 수행되는 것을 확인할 수 있다.  

```kotlin
var givenList = listOf<Int>()
//var givenList = listOf<Int>(1, 2)

val result = givenList.asSequence()
.map { println("map value: ${it}"); it * 2 }
.filter { println("filter value: ${it}"); it % 3 == 0 }
.firstOrNull()

println(result) //null
```

위와 같이 시퀀스가 비는 경우가 발생하여 ``.first()`` 순서에서 예외가 발생할 수 윘다. 이럴 때에는  ``.firstOrNull()`` 을 사용하면 된다.  

시퀀스에 대한 연산은 중간 연산과 최종 연산이라는 범주로 나뉜다. ``map`` 과  ``filter`` 같은 중간 연산은 새로운 시퀀스를 리턴한다.  
``first`` 또는  ``toList`` 같은 최종 연산은 시퀀스가 아닌 다른 것을 리턴한다.  

최종 연산 없이는 시퀀스가 데이터를 처리하지 않는다(lazy한 동작방식).  

이 때문에 위 예제 주석코드에서 1, 2를 넣고 돌렸을 때에도 ``filter`` 가 내뱉는 결과 값이 시퀀스이기 때문에 빈 시퀀스가 되어 예외가 발생하는 것을 확인할 수가 있다.  

+) 자바의 스트림에서도 동일하게 ``filter``나 ``sorted`` 와 같은 중간연산과 ``count``, ``toList`` 와 같은 단말 연산이 존재하고 단말 연산이 파이프라인에 실행하기 전까지 아무 연산도 수행되지 않는 lazy 한 동작을 하고 쇼트서킷을 이용하는 ``findAny``, ``limit`` 도 있기 때문에 이것과 빗대어 생각하면 이해하기 편할 것 같다.

<br/>

 ### 시퀀스 생성하기

```kotlin
val numSequence1 = sequenceOf(3, 1, 4, 1, 6, 7)
val numSequence2 = listOf(3, 1, 4, 1, 6, 8).asSequence()
```

다음과 같은 방법으로 시퀀스 생성이 가능하다.  

```kotlin
fun Int.isPrime() =
    this == 2 || (2..ceil(sqrt(this.toDouble())).toInt()) //ceil: 올림 함수, sqrt: 제곱근 함수
        .none { divisor -> this % divisor == 0 }

fun nextPrime(num: Int) =
    generateSequence(num + 1) { it + 1 }
        .first(Int::isPrime)

public fun <T : Any> generateSequence(seed: T?, nextFunction: (T) -> T?): Sequence<T> =
    if (seed == null)
        EmptySequence
    else
        GeneratorSequence({ seed }, nextFunction)
```

위의 코드와 같이 ``generateSequence`` 를 통해 무한 시퀀스를 생성할 수 윘다. 첫 번째 파라미터는 초기값이고 두 번째는 다음 값을 생산하는 함수가 인자로 들어간다.  

<br/>

### 무한 시퀀스 다루기

```kotlin
fun Int.isPrime =
		num == 2 || (2..ceil(sqrt(num.toDouble())).toInt())
		  .none { divisor -> num % divisor == 0 }

fun nextPrime(num: Int) =
		generateSequence(num + 1) { it + 1 }
				.first { it -> isPrime(it) }

fun firstNPrimes(count: Int) =
    generateSequence(2, ::nextPrime)
        .take(count)								// 요청한 수만큼만 원소를 가져오는 중간 연산
				.toList()

println(firstNPrimes(5)) // 2, 3, 5, 7, 9
```

위의 코드처럼 ``take()``을 이용할 수도 있고 아래와 같이 마지막에 널을 리턴하게 하여 무한대의 원소를 갖는 시퀀스를 잘라내는 방법도 있다.  
시퀀스는 null을 리턴받으면 종료한다.  

```kotlin
fun primesLessThan(max: Int): List<Int> =
    generateSequence(2) { n -> if (n < max) nextPrime(n) else null }
        .toList()
				.dropLast(1)

println(primesLessThan(10))
// [2, 3, 5, 7]
```

또는  ``takeWhile`` 을 이용해서 아래와 같이 사용할 수도 있다.  

```kotlin
fun primesLessThan(max: Int): List<Int> =
    generateSequence(2, ::nextPrime)
        .takeWhile { it < max }
        .toList()
```

<br/>

### 시퀀스에서 yield 하기

지금까지 시퀀스 사용을 기존 데이터에서 ``sequenceOf`` 를 하거나, 컬렉션을  ``asSequence`` 를 이용해 변환하거나 ``generateSequence`` 에 함수를 제공해 값을 생성했다. 이제 ``sequence`` 함수를 이용해 시퀀스를 사용해보겠다.  

```kotlin
public fun <T> sequence(
		block: suspend SequenceScope<T>.() -> Unit
): Sequence<T> = Sequence { iterator(block) }

fun fibonacciSequence() = sequence {
    var terms = Pair(0, 1)

    while (true) {
        yield(terms.first)
        terms = terms.second to terms.first + terms.second
    }
}

println(fibonacciSequence().take<Int>(10).toList())
//[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

``sequence`` 함수는 주어진 블록에서 평가되는 시퀀스를 생성한다. 이 블록은 인자 없는 람다 함수이며 void를 리턴하고(코틀린에서는 Unit) 평가 후에 ``SequenceScope`` 타입을 받는다. ``sequence`` 함수를 사용하기 위해서는 중단함수인 ``yield`` 가 있는 람다를 제공해야 한다.  

``yield`` 함수는 iterator에 값을 제공하고 다음 값을 요청할 때까지 값 생성을 중단한다.  
코루틴과 잘 동작하며 코루틴에 값을 제공한 후에 다음 값을 요청할 때까지 해당 코루틴을 중단시킬 수 있다.  

```kotlin
fun yieldExample() = sequence {
    val start = 0
    yield(start)
    yieldAll(1..5 step 2)
    yieldAll(generateSequence(8) { it * 3})
}

println(yieldExample().take(10).toList())
//[0, 1, 3, 5, 8, 24, 72, 216, 648, 1944]
println(yieldExample().take(5).toList())
//[0, 1, 3, 5, 8]
```

``yieldAll`` 을 이용한 시퀀스 생성도 가능하다.  

 <Br/>

### apply로 객체 생성 후에 초기화하기

영역 함수(스코프 함수)인  ``apply`` 를 사용하면 인스턴스의 속성과 함수를 사용할 수 있다.  
이를 이용하여 객체의 생성자 인자만으로 할 수 없는 초기화 작업을 할 수가 있다.  

```kotlin
inline fun <T> T.apply(block: T.() -> Unit): T

class Person(var name: String, var age: Int){
    fun getOlder() {
        age += 1
    }
}

// apply 사용 전
val person = Person("kang", 99)
person.name += " junior"
person.getOlder()

// apply 사용 후
val person2 = Person("kang", 99).apply {
    name += "junior"
    getOlder()
}
```

``apply`` 함수는 this를 인자로 전달하고  this를 리턴하는 확장 함수다.  
``person.`` 를 안붙이고 해당 인스턴스의 변수를 조작하고 함수 호출도 가능해서 코드가 더 깔끔해졌다.  

<br/>

### 부수 효과를 위해 also 사용하기

코드 흐름을 방해하지 않고 메세지를 출력하거나 다른 부수 효과를 생성하고 싶을 때 사용된다.  
객체의 유효성 검사 등을 할 때에 사용된다고 한다.  

```kotlin
inline fun <T> T.also(block: (T) -> Unit): T

val book = createBook()
	.also { println(it) }
  .also { Logger.getAnonymousLogger().info(it.toString()) }

val person = Person("kang", 100).also {
    it.name = "asdf"
    it.getOlder()
}
```

``it`` 을 사용해서 접근할 수 있다.  

<br/>

### let 함수와 엘비스 연산자 사용하기

``let`` 함수는 컨텍스트 객체가 아닌 블록의 결과를 리턴한다.

```kotlin
inline fun <T, R> T.let(block: (T) -> R): R

fun processingString(str: String) =
    str.let {
        when {
            it.isEmpty() -> "Empty"	//it으로 접근
            it.isBlank() -> "Blank"
            else -> it.capitalize()
        }
    }

fun processingString(str: String?) =
    str?.let {
        when {
            it.isEmpty() -> "Empty"
            it.isBlank() -> "Blank"
            else -> it.capitalize()
        }
    } ?: "Null"

val result = Person("kang", 101).let {
            it.name
        }
println(result) // kang
```

  <br/>

### 임시 변수로  let 사용하기

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
val resultList = numbers.map { it.length }.filter { it > 3 }
println(resultList)
```

위와 같이 임시 변수인 ``resultList`` 에 할당하고 print 하고 싶지 않을 때에  ``let`` 을 이용하여 처리할 수 있다.  

 ```kotlin
val letResult = numbers.map { it.length }.filter { it > 3 }.let {
    println(it)
} //kotlin.Unit

val alsoResult = numbers.map { it.length }.filter { it > 3 }.also {
    println(it)
} //[5, 4, 4]
 ```

``also`` 로도 처리할 수 있다. ``also`` 는 컨텍스트 객체를 리턴하는 반면 ``let`` 은 블록의 결과를 리턴한다. 이 경우에는 ``Unit`` 이다.  

<br/>

### 대리자를 사용해서 합성 구현하기

코틀린은 어떤 속성의 획득자와 설정자가 대리자라고 불리는 다른 객체에서 구현되어 있다는 것을 암시하기 위해 속성에 by 키워드를 사용한다.

```kotlin
interface Dialable {
  fun dial(number: String): String
}

class Phone: Dialable {
  override fun dial(number: String) = "Dialing $number..."
}

interface Snappable {
  fun takePicture(): String
}

class Camera: Snappable {
  override fun takePicture() = "Taking pircutre..."
}

class SmartPhone(
  private val phone: Dialable = Phone(),
  private val camera: Snappable = Camera(),
): Dialable by phone, Snappable by camera

---

private val smartPhone: SmartPhone = SmartPhone();

assertEquals("Dialing 555-1234...", smartPhone.dial("555-1234"))
assertEquals("Taking picture...", smartPhone.takePicture())
```

위의 코드는 자바로 보면 다음과 같다.  

```java
public final class SmartPhone implements Dialable, Snappable {
  private final Dialable phone;
  private final Snappable camera;
  
  public SmartPhone(@NotNull Dialable phone, @NotNull Snappable camera) {
    this.phone = phone;
    this.camera = camera;
  }
  
  @NotNull
  public String dial(@NotNull String number) {
    return this.phone.dial(nubmer);
  }
  
  @NotNull
  public String takePicture() {
    return this.camera.takePicture();
  }
}
```

SmartPhone에 포함된 객체는 SmartPhone을 통해 노출된 것이 아니라 오직 포함된 객체의 public 함수만이 노출된다.  
Dialable과 Snappable 인터페이스에 선언되어 있고, 이와 일치하는 함수만 사용 가능하다.  

<br/>

### lazy 대리자 사용하기

어떤 속성이 필요할 때까지 해당 속성의 초기화를 지연시키고 싶을 때에 lazy 대리자를 사용한다.  

```kotlin
val ultimateAnswer: Int by lazy {
  println("computing the answer")
  20
}

println(ultimateAnswer)
println(ultimateAnswer)
/*
computing the answer
20
20
```

처음 호출 시에만 ``computing the answer`` 를 출력하게된다. 첫 호출을 받은 lazy가 받은 람다를 실행하고 변수에 20을 리턴하게된다.  
이후 다시 출력할 때에는 변수에 저장된 20을 출력하게 되는 것이다.  

<br/>

### 값이 널이 될 수 없게 만들기

보통 코틀린 클래스의 속성은 클래스 생성 시에 초기화된다. 속성 초기화를 지연시키는 한 가지 방법은 속성에 처음 접근하기 전에 속성이 사용되면 예외를 던지는 대리자를 제공하는 notnull 함수를 사용하는 것이다.

```kotlin
var shouldNotBeNull: String by Delegates.notNull<String>()
assertThrows<IllegalStateException> { shouldNotBeNull }

shouldNotBeNull = "Hello, World!"
assertDoesNotThrow { shouldNotBeNull }
assertEquals("Hello, World!", shouldNotBeNull)
```

``notNull`` 대리자를 이용한 ``shouldNotBeNull`` 이 그냥 사용이되면 ``IllegalStateException`` 이 던져지게된다.  

<br/>

### observable과 vetoable 대리자 사용하기

속성의 변경을 가로채서, 필요에 따라 변경을 거부하고 싶을 때에 observable과 vetoable를 사용한다.  
변경 감지에는 observable 함수를 사용하고, 변경의 적용 여부를 결정할 때는 vetoable 함수와 람다를 사용하면 된다.  

```kotlin
var watched: Int by Delegates.observable(1) { prop, old, new ->
                                             println("${prop.name} changed from $old to $new")
                                            }

var checked: Int by Delegates.vetoable(0) { prop, old, new ->
                                           println("Trying to change ${prop.name} from $old to $new")
                                           new >= 0
                                          }

assertEquals(1, watched)
watched *= 2
assertEquals(2, watched)
watched *= 2
assertEquals(4, watched)

assertEquals(0, checked)
checked = 5
assertEquals(5, checked)
checked = -3
assertEquals(5, checked)
/*
watched changed from 1 to 2
watched changed from 2 to 4
Trying to change checked from 0 to 5
Trying to change checked from 5 to -3
```

``watched`` 는 변경이 이루어 질 때 마다 내부의 람다 함수를 호출한다. 따라서 println 을 수행한다.  
checked 또한 마찬가지인데, 0 이상인 값에 대해서만 변경을 받아들이고 있는 것을 확인할 수가 있다.  

<Br/>

### 대리자로서 Map 제공하기

Map을 제공해 객체를 초기화할 수 있다.  

```kotlin
data class Project(val map: MutableMap<String, Any?>) {
  val name: String by map
  var priority: Int by map
  var completed: Boolean by map
}

val project = Project(
  mutableMapOf(
    "name" to "Learn Kotlin",
    "priority" to 5,
    "completed" to true))

assertAll(
  { assertEquals("Learn Kotlin", project.name) },
  { assertEquals(5, project.priority) },
  { assertTrue(project.completed)}
)
```

이 코드가 가능한 이유는 ``MutableMap`` 에 ``ReadWriteProperty`` 대리자가 되는데 필요한 올바른 시그니처의 ``setValue`` 와 ``getValue`` 확장 함수가 있기 때문이다.  이런 기능이 필요한 이유는 무엇일까?  

```kotlin
fun getMapFromJSON() =
    Gson().fromJson<String, Any?>(
        """{ "name": "Learn Kotlin", "priority", 5, "completed": true}""",
        MutableMap::class.java
    )
val project = Project(getMapFromJSON())

assertAll(
    { assertEquals("Learn Kotlin", project.name) },
    { assertEquals(5, project.priority) },
    { assertTrue(project.completed)}
)
```

다음과 같이 JSON을 파싱하거나 다른 동적인 작업을 하는 애플리케이션에서 사용할 수 있기 때문이다.  

<Br/>

### 사용자 정의 대리자 만들기

어떤 클래스의 속성이 다른 클래스의 획득자와 설정자를 사용하고 싶을 때에 ``ReadOnlyProperety`` 와 ``ReadWriteProperty`` 를 구현하는 클래스를 생성함으로써 직접 속성 대리자를 작성하면 된다. 굳이 위 두 인터페이스를 구현할 필요는 없고 시그니처와 동일한 ``getValue``, ``setValue`` 함수만으로도 충분하다.  

```kotlin
class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}

class Example {
    var p: String by Delegate()
}

fun main() {
    val e = Example()
    println(e.p)
    e.p = "NEW"
}

/*
delegates.Example$3c98385c, thank you for delegating 'p' to me!
NEW has been assigned to 'p' in delegates.Example$3c98385c.
```

<br/>

### use로 리소스 관리하기

자바에는 try-with-resources 구조를 사용하여 리소스를 관리할 수 있지만, 코틀린에는 try-with-resources 구조를 지원하지 않는다.  
코틀린은 Closeable에는 확장 함수 use, Reader와 File에는 useLine을 추가했다.  

```kotlin
File("/Users/user/Desktop/word").useLines { line ->
    line.filter { it.length > 20 }
        .sortedByDescending(String::length)
        .take(10)
        .toList()
```

위의 로직은 해당 파일에서 가장 긴 단어 10개를 리턴하는 함수다. use와 useLines의 시그니처는 다음과 같다.  

```kotlin
public inline fun <T> File.useLines(charset: Charset = Charsets.UTF_8, block: (Sequence<String>) -> T): T 

public inline fun <T : Closeable?,R> T.use(block: (T) -> R):R
```

<br/>

### 파일에 기록하기

```kotlin
File("/Users/user/Desktop/example.txt").printWriter().use { writer ->
    writer.println("example data")
}
```

<br/>

### 코틀린 버전 알아내기

```kotlin
println("${KotlinVersion.CURRENT}")
```

``KotlinVersion.CURRENT`` 를 통해 알아낼 수 있다.  

<br/>

### 반복적으로 람다 실행하기

주어진 람다 식을 여러 번 실행하고 싶을 때에 코틀린 내장 ``repeat`` 함수를 사용하면 된다.  

```kotlin
repeat(5) {
    println("Counting: $it")
}
/*
Counting: 0
Counting: 1
Counting: 2
Counting: 3
Counting: 4
```

<br/>

### 완벽한 when 강제하기

```kotlin
fun printMod3(n:Int) {
    when (n % 3) {
        0 -> println("$n % 3 == 0")
        1 -> println("$n % 3 == 1")
        2 -> println("$n % 3 == 2")
    }
}

fun printMod3SingleStatement(n: Int) = when (n % 3) {
    0 -> println("$n % 3 == 0")
    1 -> println("$n % 3 == 1")
    2 -> println("$n % 3 == 2")
    else -> println("problem occurred")
}

fun printMod3(n:Int) {
    when (n % 3) {
        0 -> println("$n % 3 == 0")
        1 -> println("$n % 3 == 1")
        2 -> println("$n % 3 == 2")
    }.exhaustive
}
```

when 절은 자바의 switch 문과 비슷하게 동작하지만 자바와는 다르게 각 섹션마다 break로 탈출하거나 값을 리턴하기 위해서 switch 문 바깥에서 변수를 선언할 필요가 없다.  

첫 번째 함수와 같이 when 표현식이 값을 리턴하지 않는다면 코틀린은 when 식이 완벽하길 요구하지 않는다.  

하지만 두 번째 식과 같이 무언가 할당을 해야하는 경우 완벽한 조건식이 필요해서 else가 필요하다. 개발자는 3 나머지 연산이 0,1,2 밖에 안나온다는 것을 알지만 말이다. ``.exhaustive`` 를 통해 할당을 받지 않는 경우에도 완벽한 when 절을 강제할 수가 있다.  

<br/>

### 정규표현식과 함께 replace 함수 사용하기

```kotlin
sertAll(
    { assertEquals("one*two*", "one.two.".replace(".", "*")) },
    { assertEquals("********", "one.two.".replace(".".toRegex(), "*")) }
)
```

<br/>

### 바이너리 문자열로 변환하고 되돌리기

```kotlin
val str = 42.toString(radix = 2)
assertThat(str, `is`("101010"))

val num = "101010".toInt(radix = 2)
assertThat(num, `is`(42))
```

<Br/>

### 실행 가능한 클래스 만들기

```kotlin
data class AstroResult(
    val message: String,
    val number: Number,
    val people: List<AssignMent>
)

data class AssignMent(
    val craft: String,
    val name: String
)

class AstroRequest {
    companion object {
        private const val ASTRO_URL = "http://api.open-notify.org/astros.json"
    }

    operator fun invoke(): AstroResult {
        val responseString = URL(ASTRO_URL).readText()
        return Gson().fromJson(responseString, AstroResult::class.java)
    }
}

@Test
fun `invoke`() {
    val request = AstroRequest()
    val result = request()
    println(result.message)
    println(result.people)
}
```

해당 api는 지금 이 순간에 우주에 있는 우주 비행사의 수를 나타내는 JSON 데이터를 리턴한다.  

invoke 연산자 함수를 통해서 클래스 인스턴스를 바로 실행할 수 있다.  

<Br/>

### 경과 시간 측정하기

```kotlin
@Test
fun `time`() {
    fun doubleIt(x: Int): Int {
        Thread.sleep(100L)
        println("doubling $x with on thread ${Thread.currentThread().name}")
        return x * 2
    }

    println("${Runtime.getRuntime().availableProcessors()} processors")
    var time = measureTimeMillis {
        IntStream.rangeClosed(1, 6)
            .map { doubleIt(it) }
            .sum()
    }
    println("Sequential stream took ${time}ms")

    time = measureTimeMillis {
        IntStream.rangeClosed(1, 6)
            .parallel()
            .map { doubleIt(it) }
            .sum()
    }

    println("Parallel stream took ${time}ms")
}
```

코드 블록이 실행되는데 걸린 시간을 구할 때 ``measureTimeMillis``, ``measureNanoMillis`` 를 사용한다.  

<Br/>

### 스레드 시작하기

코드 블록을 동시적 스레드에서 실행하고 싶을 때, kotlin.concurrent 패키지의 thread 함수를 사용한다.  

```kotlin
(0..5).forEach { n ->
    val sleepTime = Random.nextLong(range = 0..1000L)
    thread {
        Thread.sleep(sleepTime)
        println("${Thread.currentThread().name} for $n after ${sleepTime}ms")
    }
}
/*
Thread-2 for 2 after 184ms
Thread-5 for 5 after 207ms
Thread-4 for 4 after 847ms
Thread-0 for 0 after 917ms
Thread-3 for 3 after 967ms
Thread-1 for 1 after 980ms
```

<Br/>

### TODO로 완성 강제하기

```kotlin
@Test
fun `todo`() {
    fun completeThis() {
        TODO("finish this")
    }
    completeThis()
}

/*
An operation is not implemented: finish this
kotlin.NotImplementedError: An operation is not implemented: finish this
```

<br/>

### 자바에게 예외 알리기

코틀린 함수가 자바엣 ㅓ체크 예외차로 여겨지는 예외를 던지는 경우 자바에게 해당 예외가 체크 예외임을 알려 주고 싶을 때에 ``@Throws`` 어노테이션을 사용한다.  

코틀린의 모든 예외는 언체크 예외다. 

```kotlin
// 코틀린 함수
fun weHaveProblem() { 
  throws IOException("File or resource not found")
}

// 자바에서 호출 시, IOException으로 충돌
public static void doNothing() {
  weHaveProblem();
}

// try-catch를 하게되면 컴파일이 안됨
public static void useTryCatchBlock() {
    try {
      weHaveProblem();
    } catch (IOException e) {
      e.printStackTrace();
    }
}

// throws를 붙이면 컴파일은 되지만 불필요한 throws 절이라고 경고함
public static void useThrowClause() throws IOException {
  weHaveProblem();
}

// 코틀린 함수에 @Throws를 붙이면 바로 위 2가지 방법 모두 가능하게된다
@Throws(IOException::class)
fun weHaveProblem() {
  throws IOException("File or resource not found")
}
```

<br/>
