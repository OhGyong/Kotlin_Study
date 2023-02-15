[Composing suspending functions](https://kotlinlang.org/docs/composing-suspending-functions.html)

---

### 순차적으로 실행

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    println("가")
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    println("나")
    return 29
}

// 실행 결과
가
나
The answer is 42
Completed in 3024 ms
```

코루틴은 기본적으로 순차적으로 실행한다.

그래서 실행 결과가 '가'가 출력된 이후에 '나'가 출력된 것이다.

동작이 완료되는 시간도 3초가 넘는 것을 확인할 수 있다.

위 동작을 비동기적으로 바꾸려면 각 함수를 코루틴에 담고 전체 결과를 기다릴 코루틴을 생성해야 한다.

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        var one = 0
        var two = 0

        launch {
            launch {
                one = doSomethingUsefulOne()
                println("one")
            }
            launch {
                two = doSomethingUsefulTwo()
                println("two")
            }
        }.join()

        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    println("가")
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    println("나")
    return 29
}

// 실행 결과
나
가
The answer is 42
Completed in 2040 ms
```

실행 결과를 보면 알 수 있듯이 비동기로 코드가 수행되었고, 완료 시간도 2초가 걸렸다.

하지만 이렇게 코드를 작성하는 것은 매우 번거롭다.

이런 번거로움을 코루틴 빌더인 async가 해결해줄 수 있다.

---

### async를 이용한 비동기 처리
async는 launch와 같은 코루틴 빌더이지만 반환 값이 다르다는 차이가 있다.

launch의 경우 Job을 반환했고, async의 경우 Deffered를 반환한다.

Deffered는 await을 통해 async의 코루틴 블록 결과를 받을 수 있다.

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

// 실행 결과
The answer is 42
Completed in 2023 ms
```

실행 결과를 보면 비동기로 동작하여 완료 시간이 2초로 나왔다.

---

### async로 지연 실행하기
async의 start 인자로 CoroutineStart.LAZY를 전달하면 해당 코루틴을 지연시킬 수 있다.

이 경우 await에 의해 결과가 필요하거나 start 함수가 호출될 때 코루틴이 동작한다.

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // some computation
        one.start() // start the first one
        two.start() // start the second one
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

// 실행 결과
The answer is 42
Completed in 3037 ms
```

aync의 start 파라미터에 CoroutineStart.LAZY를 전달하여 코루틴의 실행을 지연시켰다가<br>
start 함수를 만나서 async 블록이 실행된다.

<br>

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }

        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

// 실행 결과
The answer is 42
Completed in 3045 ms
```
start 함수없이 await 함수만 있을 경우에는 결과 값을 기다리기 때문에<br>
순차적으로 실행되어 완료 시간이 3초가 걸렸다.

CoroutineStart.LAZY를 전달한 상태에서 start나 await을 사용하지 않을 경우<br>
해당 코루틴 블록은 실행되지 않는다.

---

### Async를 사용한 함수 스타일

우리는 async 스타일의 함수를 정의할 수 있다.

예시로 GlobalScope에 async 코루틴 빌더를 사용하여 비동기 스타일의 함수를 작성할 수 있다.

```kotlin
// note that we don't have `runBlocking` to the right of `main` in this example
fun main() {
    val time = measureTimeMillis {
        // we can initiate async actions outside of a coroutine
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

// 실행 결과
The answer is 42
Completed in 2132 ms
```

async를 사용한 함수 스타일은 다른 프로그래밍 언어에서 널리 사용되지만<br>
코틀린의 코루틴과 함께 이러한 스타일 사용하는 것은 권장되지 않는다.

~Async 함수와 await 함수 사이에 오류가 발생하여 코루틴이 중단되어도<br>
여전히 ~Async 함수가 백그라운드에서 실행될 가능성이 있어 문제가 된다고 한다.

---

### async를 사용하여 구조화된 동시성 만들기
async 함수는 CoroutineScope의 확장 함수로 정의되어 있기 때문에<br>
coroutineScope 안에서 async를 호출할 수 있다.

```kotlin
fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(2000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}

// 실행 결과
The answer is 42
Completed in 2025 ms
```

위와 같은 코드로 작성하면 동시성을 챙길 수 있고<br>
concurrentSum 함수에서 문제가 발생했을 때 해당 범위의 모든 코루틴이 취소된다.<br>


Exception을 발생시켜서 확인해보자.
```kotlin
fun main() = runBlocking {
    try {
        val time = measureTimeMillis {
            println("The answer is ${concurrentSum()}")
        }
        println("Completed in $time ms")
    } catch (e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async {
        try {
            doSomethingUsefulOne()
        }catch (e: Exception) {
            println("one : ArithmeticException")
        }
        0
    }

    val two = async {
        delay(5000L)
        throw ArithmeticException()
        doSomethingUsefulTwo()
    }

    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    repeat(5) {
        delay(2000L)
        println("doSomethingUsefulOne $it")
    }
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    repeat(5) {
        delay(5000L)
        println("doSomethingUsefulTwo $it")
    }
    return 29
}

// 실행 결과
doSomethingUsefulOne 0
doSomethingUsefulOne 1
one : ArithmeticException
Computation failed with ArithmeticException
```

concurrentSum 함수가 호출되고 doSomethingUsefulOne에서 2초마다 "doSomethingUsefulOne"이 출력된다.

5초 뒤에 two 변수에서 Exception이 발생하면서 동작하고 있던 one 변수의 async가 종료되었고<br>
main 함수에서도 catch 블록이 실행된 것을 확인할 수 있다.

부모 async에서 자식 async 하나가 종료되면<br>
부모 async와 같은 필드의 async가 모두 종료되는 것을 참고하자.

