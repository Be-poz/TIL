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

