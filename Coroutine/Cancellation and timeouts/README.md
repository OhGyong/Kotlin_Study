[Cancellatin and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)

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

첫 번째 방법으로 yield를 사용할 수 있다.

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