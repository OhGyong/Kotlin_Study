[Cancellatin and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)

---

### 코루틴의 실행 취소
장시간 실행되는 앱에서 동작하는 코루틴을 세밀하게 제어할 수 있어야 한다.

예를 들어 코루틴이 동작하고 있는 화면에서 벗어났을 때,<br>
더 이상의 결과가 필요하지 않아서 코루틴의 동작을 종료시켜야 한다.

launch 함수는 Job 객체를 반환하고 Job 객체는 취소될 수 있다.

```kotlin
fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

Coroutines basics에서 runBlocking과 coroutines를 설명할 때 cancel 함수를 사용한 적이 있다.

cancel은 실행 중인 코루틴의 Job을 취소한다.

위 코드를 보면 1.3초가 지난 뒤에 반복문이 더 이상 돌아가지 않는 것을 확인 할 수 있다.



---
### CancellationException

코루틴의 모든 정지 함수는 취소가 가능하다.

정지 함수는 코루틴이 취소되는지를 계속 체크를 하고, 취소가 되면 CancellationException을 발생시킨다.

```kotlin
fun main() = runBlocking {
    val job = launch {
        try{
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        }catch (e: Exception) {
            println("exception 발생 : $e")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
exception 발생 : kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelling}@5f282abb
main: Now I can quit.
```

위 코드처럼 코루틴 블록에 try-catch을 추가하여 CancellationException이 발생했는지 확인할 수 있다.

만약, 코루틴이 계산 작업을 하고 있거나, CancellationException을 따로 체크하지 않으면 코루틴을 취소할 수 없다.

아래 코드를 보자.
```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        try {
            var nextPrintTime = startTime
            var i = 0
            while (i < 5) { // computation loop, just wastes CPU
                // print a message twice a second
                if (System.currentTimeMillis() >= nextPrintTime) {
                    println("job: I'm sleeping ${i++} ...")
                    nextPrintTime += 500L
                }
            }
        } catch (e:Exception) {
            println("exception 발생 : $e")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel()
    job.join()
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
job: I'm sleeping 4 ...
main: Now I can quit.
```

while 블록이 계속 실행되고 있기 때문에 cancel 함수를 사용했음에도 반복문의 코드가 계속 수행되었다.

CancellationException도 체크하지 못했는지 catch 블록의 코드도 동작하지 않았다.

```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        try {
            var nextPrintTime = startTime
            var i = 0
            while (i < 5) { // computation loop, just wastes CPU
                // print a message twice a second
                if (System.currentTimeMillis() >= nextPrintTime) {
                    println("job: I'm sleeping ${i++} ...")
                    nextPrintTime += 500L
                    delay(10)
                }
            }
        } catch (e:Exception) {
            println("exception 발생")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
exception 발생
main: Now I can quit.
```

만약에 while 블록에 delay를 사용하여 코루틴을 정지시키면<br>
계산이 종료되기 때문에 CancellationException을 체크하게 된다.

**cancelAndJoin**<br>
~~~ 
cancelAndJoin은 cancel과 join을 결합한 Job의 확장함수다.
cancelAndJoin을 사용하면 위의 코드들에서 cancel과 join을 연달아서 작성할 필요가 없다.
~~~

---

### 계산 중인 코루틴 영역 취소시키기

계산 중인 코루틴 영역을 취소시키는 방법으로 두 가지가 있다.

첫 번째는 취소를 체크하도록 하는 정지 함수를 주기적으로 호출하는 것이고,

두 번째는 취소 상태를 명시적으로 체크하는 것이다.

```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i<5) { // cancellable computation loop
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                yield()
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

첫 번째 방법으로 yield를 사용할 수 있다.<br/>
yield는 해당 위치에서 코루틴을 일시 중단한다.

yield는 정지되지 않아도 취소 항상 여부를 확인하며,<br>
Job이 취소되거나 완료되면  CancellationException으로 재개된다.

<br>

```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // cancellable computation loop
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```
두 번째 방법으로는 isActive를 사용할 수 있다.

isActive는 코루틴 범위에서만 사용할 수 있는 확장 속성으로 코루틴이 비활성 상태인지 확인할 수 있다.

---

### 리소스 종료하기
코루틴 블록에서 예외가 발생하거나 구문이 끝나고 리소스를 종료시켜야 하는 상황이 있다.

이럴 때 finally 또는 use를 사용하면 된다.


아래 코드는 finally를 사용하여 리소스를 종료시켰다.
```kotlin
fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("job: I'm running finally")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
main: Now I can quit.
```

<br>

아래 코드는 use를 사용하여 리소스를 종료시켰다.
```kotlin
fun main() = runBlocking {
    val job = launch {
        ClosingCoroutine().use {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}

class ClosingCoroutine : Closeable {
    override fun close() {
        println("closing coroutine")
    }
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
closing coroutine
main: Now I can quit.
```

--- 

### 취소가 불가능한 코드 블록의 실행 (Run non-cancellable block)

```kotlin
fun main() = runBlocking {
    val job = launch {
        try {
            repeat(3) { i ->
                println("job: I'm sleeping $i ...")
                delay(400L)
            }
        } catch (e:Exception) {
            println("Exception 1")
        }finally {
            try {
                delay(1000L)
                println("정지")
            }catch (e:Exception) {
                println("Exception 2")
            }

        }
    }
    delay(1100L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
// 실행 결과
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
Exception 1
Exception 2
main: Now I can quit.
```

코루틴의 finally 블록에서 정지 함수를 사용할 때 코루틴 블록에서 CancellableException이 발생하면

코루틴은 취소가 된 상태이기 때문에 finally 블록은 정지 함수를 사용하지 못하고 CancellationException이 발생하게 된다.<br>
(finally 블록에서 "정지"가 출력되지 않은 이유)

하지만 보통 이것은 문제 되지는 않는다. 리소스를 종료시키는 블록에서는 일반적으로 정지 함수를 사용하지 않기 때문이다.

```kotlin
fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}

// 실행 결과
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
job: And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
```

그럼에도 정지 함수를 사용해야 하는 경우, withContext에 NonCancellable를 전달하여 이를 해결할 수 있다.<br>
([NonCancellabe 참고](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-non-cancellable/))

---
### Timeout
코루틴의 실행 시간이 너무 길어지는 경우 취소 요청을 보내지 않더라도 코루틴을 종료시켜야 한다.

```kotlin
fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }

    launch {
        delay(1300L)
        job.cancelAndJoin()
        println("job finish")
    }.join()
}

// 실행 결과
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
job finish
```

job 코루틴 블록에 반복문을 수행시키고, 별도의 코루틴 블록을 생성하여<br>
1.3초 후 job 코루틴을 종료하도록 했다.

위 코드처럼 별도의 코루틴을 생성하여 Job을 참조하여 수동으로 취소를 시키는 방법이 있지만 번거롭다.

코루틴은 withTimeout 함수를 이용해 이를 해결할 수 있다.

withTimeout 함수에 제한 시간을 걸어주면 코루틴은 종료된다.

```kotlin
fun main() = runBlocking {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}

// 실행 결과
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException ...
```

실행 결과를 보면 TimeoutCancellationException이 발생하는데 이는 CancellationException의 하위 클래스다.

try-catch로 처리를 하거나 withTimeoutOrNull을 사용하여 해결할 수 있다.

---

### withTimeout 자원 누수 막기
withTimeout도 비동기적으로 실행되기 때문에 블록 내부에서 자원이 반환되기 전에 리소스의 값이 제대로 할당 되지 않을 수 있다.

```kotlin
var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch {
                val resource = withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    Resource() // Acquire a resource and return it from withTimeout block
                }
                resource.close() // Release the resource
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}

// 실행 결과
1589
```
위 코드를 보면 acquired 값이 항상 0이 나와야 될 것 같지만, 막상 결과를 보면 그렇지 않다.

이러한 문제를 해결하려면 블록 내부에서 리소스를 반환하는 것이 아니라<br>
블록 외부의 변수에 리소스를 참조하고 해당 변수에 값을 지정해야한다.

아래 코드를 보자.

```kotlin
var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch {
                var resource: Resource? = null // Not acquired yet
                try {
                    withTimeout(60) { // Timeout of 60 ms
                        delay(50) // Delay for 50 ms
                        resource = Resource() // Store a resource to the variable if acquired
                    }
                    // We can do something else with the resource here
                } finally {
                    resource?.close() // Release the resource if it was acquired
                }
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}

// 실행 결과
0
```

withTimeout 블록에서 리소스를 반환하는 대신 외부 변수에 리소스를 저장한 결과 0만 출력되었다.