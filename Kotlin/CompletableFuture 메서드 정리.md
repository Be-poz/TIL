# CompletableFuture 메서드 정리

## 비동기 작업 실행

### supplyAsync(Supplier<T>)

```kotlin
val supplyAsync = CompletableFuture.supplyAsync {
    val sentence = "supplyAsync!"
    println(sentence)
    sentence
}
val result = supplyAsync.join()
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

실패한 CompletableFuture을 반환함. 결과값 넘기는데 위의 코드에서는 print 찍기 전에 예외가 발생할 것임.

<br/>

## 콜백 처리

### thenApply(Function<T, U>)

### thenApplyAsync(Function<T, U>)

### thenAccept(Consumer<T>)

### thenAcceptAsync(Consumer<T>)

### thenRun(Runnable)

