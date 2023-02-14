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
