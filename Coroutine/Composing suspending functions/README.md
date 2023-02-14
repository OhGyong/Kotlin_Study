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

실행 결과를 보면 알 수 있듯이 비동기로 코드가 수행되었고, 수행 시간도 2초가 걸렸다.

하지만 이렇게 코드를 작성하는 것은 매우 번거롭다.

이런 번거로움을 코루틴 빌더인 async가 해결해줄 수 있다.

---