# CompletableFuture 메서드 정리

## 비동기 작업 실행

### supplyAsync(Supplier<T>)

```kotlin
val supplyAsync = CompletableFuture.supplyAsync {
    val sentence = "supplyAsync!"
    println(sentence)
    sentence
}
val result = supplyAsync.join() // "supplyAsync!"
```

비동기로 처리하고 결과값을 반환함

### runAsync(Runnable)

```kotlin
val runAsync = CompletableFuture.runAsync {
    val sentence = "runAsync!"
    println(sentence)
    sentence
}
val result = runAsync.join() // null임
```

비동기로 처리하고 결과값 반환 안함

### completedFuture(T)

```kotlin
val completedFuture = CompletableFuture.completedFuture("completed!")
val result = completedFuture.join()
```

성공한 CompletableFuture을 반환함. 결과값도 같이 넘김

### failedFuture(Throwable)

```kotlin
val failedFuture = CompletableFuture.failedFuture<String>(RuntimeException("exception occurred!"))
val result = failedFuture.join()
println(result)
```

실패한 ``CompletableFuture``을 반환함. 결과값 넘기는데 위의 코드에서는 print 찍기 전에 예외가 발생할 것임.

<br/>

## 콜백 처리

### thenApply(Function<T, U>)

```kotlin
val random = Random.nextInt(3)

CompletableFuture.supplyAsync {
    "supplyAsync!"
}.thenApply { passedValue ->   // "supplyAsync!"
    Thread.sleep(random.toLong() * 1000)
    println("${Thread.currentThread()}-$i have slept for $random second(s).")
    "thenApply!"
}

val result = supplyAsync.join() // "thenApply!"
```

``Function<T, U>``가 들어가는 것 처럼 값을 받아서 값을 반환한다. 동기로 동작한다. 즉, ``supplyAsync``에서 사용한 쓰레드를 그대로 사용한다.

### thenApplyAsync(Function<T, U>)

```kotlin
val random = Random.nextInt(3)

CompletableFuture.supplyAsync {
    "supplyAsync!"
}.thenApplyAsync { passedValue ->   // "supplyAsync!"
    Thread.sleep(random.toLong() * 1000)
    println("${Thread.currentThread()}-$i have slept for $random second(s).")
    "thenApplyAsync!"
}
val result = supplyAsync.join() // "thenApplyAsync!"
```

``Function<T, U>``가 들어가는 것 처럼 값을 받아서 값을 반환한다. 비동기로 동작한다. 즉, ``supplyAsync``에서 사용한 쓰레드가 아닐 수도 있다.  
따로 ``Executors``를 명시하지 않으면 ``supplyAsync``가 사용했던 것과 동일한 ``ForkJoinPool``을 사용할 수가 있다. 만약 원치 않는다면 2번째 인자로 executors를 선언해주면 된다.  

``thenApply``와 굉장히 헷갈리고 언제 이걸 사용하지 싶을 수 있는데 [이 링크](https://blog.krecan.net/2013/12/25/completablefutures-why-to-use-async-methods/)를 참고하여 감을 잡도록 하자.

### thenAccept(Consumer<T>)

```kotlin
CompletableFuture.supplyAsync {
    10
}.thenApplyAsync { value ->
    value * 10
}.thenAccept { value ->
    println(value)   // 100
}
```

값을 받아 사용하고 반환하는게 없다. 동기로 동작한다.

### thenAcceptAsync(Consumer<T>)

```kotlin
CompletableFuture.supplyAsync {
    10
}.thenApplyAsync { value ->
    value * 10
}.thenAcceptAsync { value ->
    println(value)
}
```

값을 받아 사용하고 반환하는게 없다. 비동기로 동작한다.

### thenRun(Runnable)

```kotlin
CompletableFuture.supplyAsync {
    10
}.thenApplyAsync { value ->
    value * 10
}.thenRun {
    println("I don't care your return value!!!")
}
```

값을 받아 사용하지 않고 그냥 실행한다. 

<br/>

## 결과 결합

### thenCompose(Function<T, CompletableFuture<U>>)

```kotlin
fun main(args: Array<String>) {
    val wow = getFirst().thenCompose {
        println(Thread.currentThread())
        getSecond()
    }
    wow.get()
}

private fun getFirst() = CompletableFuture.supplyAsync {
    println(Thread.currentThread())
    "first"
}

private fun getSecond() = CompletableFuture.supplyAsync {
    println(Thread.currentThread())
    // 다른 api콜 요청 로직
    println(Thread.currentThread())
    "second"
}
```

첫 번째 ``CompletableFuture``의 결과를 이용해서 새로운 ``CompletableFuture``를 실행한다. 
``thenApply`` 와 다른점은 ``thenCompose`` 대신 ``thenApply``를 사용하게되면 ``CompletableFuture<CompletableFuture<String>>`` 의 타입을 반환타입으로 가지게 된다. 이것을 ``thenCompose``로 flat 하게 만들어 주는 것이다. ``flatMap`` 이라고 생각하면 된다.  

비동기로 요청을 했기 때문에 api 콜과 같은 네트워크 요청은 블로킹이 아니기 때문에 요청을 하고 쓰레드 반환이 될 것이다.  
위의 로직을 보면 현재 쓰레드 표시를 해놨는데 ``getFirst()``와 ``getSecond()`` 호출은 메인 쓰레드가 아닌 다른 쓰레드에서 맡을 것이다.  
``thenCompose``는 메인 쓰레드가 호출을 하게될 것이다. 

### thenComposeAsync(Function<T, CompletableFuture<U>)

``thenCompose``와 동일한데, 만약 다른 쓰레드가 호출하길 원하거나 다른 쓰레드 풀 이용을 원한다면  ``thenComposeAsync``를 사용하면 된다.  

### thenCombine(CompletableFuture<U>, BiFunction<T, U, R>)

```kotlin
val ten = CompletableFuture.supplyAsync {
    10
}

val twenty = CompletableFuture.supplyAsync {
    20
}

val combined = ten.thenCombine(twenty) { result1, result2 ->
    result1 + result2
}

println(combined.get())   // 30
```

두 future의 결과값을 가져다가 조합할 수가 있다.  

### thenCombineAsync(CompletableFuture<U>, BiFunction<T, U, R>)

``thenCombine``을 비동기로 처리할 수 있다.

### allOf(CompletableFuture<?>...)

```kotlin
val future1 = CompletableFuture.supplyAsync {
    Thread.sleep(1000)
    println("Future 1 completed")
    "Result 1"
}

val future2 = CompletableFuture.supplyAsync {
    Thread.sleep(2000)
    println("Future 2 completed")
    "Result 2"
}

val allFuture = CompletableFuture.allOf(future1, future2)

allFuture.join()
println("All futures completed")
```

varargs로 넣은 future가 모두 완료될 때 까지 대기하게 된다.

### anyOf(CompletableFuture<?>...)

```kotlin
val future1 = CompletableFuture.supplyAsync {
    Thread.sleep(1000)
    println("Future 1 completed")
    "Result 1"
}

val future2 = CompletableFuture.supplyAsync {
    Thread.sleep(2000)
    println("Future 2 completed")
    "Result 2"
}

val anyFuture = CompletableFuture.anyOf(future1, future2)

anyFuture.join()
println("Any future completed")
```

varargs로 넣은 future 중 하나라도 완료되면 즉시 완료된다.

<br/>

## 예외 처리

### exceptionally(Function<Throwable, T>)

```kotlin
val future = CompletableFuture.supplyAsync {
    throw RuntimeException("something went wrong")
    "Result"
}.exceptionally { ex ->
    println(ex.message)
    "default"
}

println(future.join()) // default
```

예외 발생 시 기본값을 제공할 수 있다. 400, 500번대 status code 들은 예외가 아니다.  

### handle(BiFunction<T, Throwable, U>)

```kotlin
val future = CompletableFuture.supplyAsync {
    //throw RuntimeException("something went wrong")
    "Result"
}.handle { result, ex ->
    if (ex != null) {
        "Recovered from error: ${ex.message}"
    } else {
        result + "1"
    }
}

println(future.join())   // Result1
```

정상 케이스와 예외 케이스 모두 처리가능하다. 

### whenComplete(BiConsumer<T, Throwable>)

```kotlin
val future = CompletableFuture.supplyAsync {
//        throw RuntimeException("something went wrong")
    "Result"
}.whenComplete { result, ex ->
    if (ex != null) {
        "Recovered from error: ${ex.message}"
    } else {
        result + "1"
    }
}

println(future.join()) // Result
```

정상 케이스와 예외 케이스 모두 처리 가능하지만 ``BiConsumer`` 이기 때문에 값 변경 없이 그대로 반환한다.

### exceptionallyCompose(Function<Throwable, CompletableFuture<T>>)

```kotlin
val future = CompletableFuture.supplyAsync {
    throw RuntimeException("something went wrong")
    "Result"
}.exceptionallyCompose { ex ->
    println("Handling exception: $ex.message")
    CompletableFuture.supplyAsync { "Recovered from error" }
}

println(future.join())
```

예외가 발생했을 때 다른 CompletableFuture을 호출한다. 예외 발생 시 ``thenCompose``를 사용한다고 생각하면 된다.  

<br/>

## 완료 및 마무리

### join()

완료될 때 까지 대기한다. unchecked exception을 던진다.

### get()

완료될 때 까지 대기한다. checked exception을 던져서 처리해줘야하지만 kotlin에서는 이걸 강제하지 않기 때문에 신경을 덜 써도 된다. 하지만 그러다가 런타임에 발생할 수도 있다.

### getNow(T)

```kotlin
val future = CompletableFuture.supplyAsync {
    Thread.sleep(2000L)
    "Result"
}
val result = future.getNow("right now")
println(result)
```

즉시 값을 가져오나 완료되지 않았으면 지정한 기본 값을 반환한다.

### complete(T)

```kotlin
val future = CompletableFuture.supplyAsync {
//        Thread.sleep(100)
    "Result"
}
Thread.sleep(100)
future.complete("right now")
println(future.join())
```

해당 future을 즉시 완료시키고 값을 정해준다, 하지만 그 전에 이미 future가 완료되었다면 완료된 값을 반환한다.  
위의 경우 ``complete`` 호출 전에 완료가 되어 ``Result``를 반환한다.

### completeExceptionally(Throwable)

```kotlin
val future = CompletableFuture.supplyAsync {
//        Thread.sleep(100)
    "Result"
}
Thread.sleep(100)
future.completeExceptionally(RuntimeException())
println(future.join())
```

future을 강제 예외를 던져 종료시킨다. ``complete`` 때와 마찬가지로 그 전에 future가 완료되면 완료된 값을 받아온다.  

---

### REFERENCE

https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html

https://blog.krecan.net/2013/12/25/completablefutures-why-to-use-async-methods/

https://stackoverflow.com/questions/47489338/what-is-the-difference-between-thenapply-and-thenapplyasync-of-java-completablef

