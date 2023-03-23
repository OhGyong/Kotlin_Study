[Asynchronous Flow](https://kotlinlang.org/docs/flow.html)

---

### 여러 값을 표시하기(Representing multiple values)
코틀린의 컬렉션을 사용하면 여러 값을 표시할 수 있다.

예를 들어 forEach를 사용하면 리스트의 모든 수를 출력할 수 있다.

```kotlin
fun simple() : List<Int> = listOf(1, 2, 3)

fun main() {
    simple().forEach {println(it)}
}

// 실행 결과
1
2
3
```

비동기의 정지 함수는 단일 값을 반환하는데, 비동기적으로 계산된 값들은 어떻게 반환할 수 있을까?

이 방법을 위해 Kotlin Flow가 존재한다.

---

### 시퀀스(Sequences)
CPU를 많이 소비하는 차단 코드를 사용하여 일련의 값들을 계산한다고 하면, 우리는 우리는 Sequence를 사용할 수 있다.

``` kotlin
fun simple(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(500) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() {
    simple().forEach { value -> println(value) }
}

// 실행 결과
1
2
3
```

---

### 정지 함수(Suspending functions)
시퀀스의 예를 들었던 위의 코드는 스레드를 차단하는 문제가 있다.

정지 함수를 이용해서 여러 값을 표시해보자.

```kotlin
suspend fun simple(): List<Int> {
    delay(1000) // pretend we are doing something asynchronous here
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    simple().forEach { value -> println(value) }
}

// 실행 결과
1
2
3
```

아직까지는 이 코드는 한 번에 모든 값을 받아서 컬렉션으로 표시하고 있다.

---

### Flows
비동기적으로 계산되는 값의 스트림을 나타내기 위해서 ```Sequence<Int>``` 대신에 ```Flow<Int>```을 사용할 수 있다.

```kotlin
fun simple(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to check if the main thread is blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    simple().collect { value -> println(value) }
}

// 실행 결과
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

- Flow의 생성을 위해 flow{} 빌더를 사용
- emit으로 데이터를 보냄
- collect로 데이터를 수집
- flow 블록은 언제든지 중지 가능

---

### Flows are cold
```kotlin
fun simple(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling simple function...")
    val flow = simple()
    println("Calling collect...")
    flow.collect { value -> println(value) }
    println()
    println("Calling collect again...")
    flow.collect { value -> println(value) }
}

// 실행 결과
Calling simple function...
Calling collect...
Flow started
1
2
3

Calling collect again...
Flow started
1
2
3
```

Flow는 시퀀스와 비슷하게 콜드 스트림이다.

flow{} 빌더 내부의 코드 블록은 flow가 수집될 때까지 실행되지 않는다.

코드를 보면 simple이 호출되었음에도 "Flow started"가 출력되지 않다가 collect 이후에 출력되는 것을 확인할 수 있다.

그리고 collect가 호출될 때마다 flow는 다시 실행되는데 "Flow started"가 두 번 출력된 것을 보면 알 수 있다.

---

### Flow Cancellation basics
Flow는 일반적으로 코루틴의 협력적인? 취소를 따른다.

Flow는 코루틴 기능 중 하나인 취소 스택에 의존하여 협력적인 취소를 따르게 된다.

withTimeoutOrNull은 입력된 시간이 초과되면 실행 중인 코드를 취소하고 null을 반환한다.

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // Timeout after 250ms 
        simple().collect { value -> println(value) }
    }
    println("Done")
}

// 실행 결과
Emitting 1
1
Emitting 2
2
Done
```

하지만 Flow 자체적으로 취소 지점을 제공하는 기능은 없다고 한다.

---

### Flow bulilders
Flow의 선언을 허용하는 다른 빌더들이 있다.

- flow{}는 지금까지 다뤘던 가장 기본적인 빌더다.
- flowOf{}는 고정된 값들을 방출하는 Flow를 정의 한다.
- 다양한 컬렉션들과 시퀀스는 .asFlow()로 Flow로 변환이 가능하다.


---

### collect vs collectLatest(Processing the latest value)
Flow의 데이터를 수집하는 방법으로 collectLatest가 있다.

collectLatest는 데이터를 수집하는 도중에 새로운 값이 방출되면 기존의 코드 블록을 취소하고 다시 시작한다.
```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collect { value -> // cancel & restart on the latest value
                println("Collecting $value")
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value")
            }
    }
    println("Collected in $time ms")
}

// 실행 결과
Collecting 1
Done 1
Collecting 2
Done 2
Collecting 3
Done 3
Collected in 1230 ms
```

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // cancel & restart on the latest value
                println("Collecting $value")
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value")
            }
    }
    println("Collected in $time ms")
}

// 실행 결과 
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 645 ms
```

방출하는 부분(delay 100)이 수집하는 부분(delay 300)보다 빠르기 때문에 수집하는 부분에서 연산이 쌓이게 된다.

collectLatest로 바꾸게 되면 쌓이는 순간 해당 블록을 취소하고 새 값을 받은 상태에서 다시 실행하게 된다.

<br/>

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // cancel & restart on the latest value
                println("Collecting $value")
                delay(50) // pretend we are processing it for 300 ms
                println("Done $value")
            }
    }
    println("Collected in $time ms")
}

// 실행 결과
Collecting 1
Done 1
Collecting 2
Done 2
Collecting 3
Done 3
Collected in 398 ms
```

collectLatset에서 delay의 시간을 낮춰보면 collect와 똑같이 동작하는 것을 확인할 수 있다.