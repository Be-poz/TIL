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

### const와  val의 차이 이해하가ㅣ

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



