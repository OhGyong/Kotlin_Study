[Coroutines basics](https://kotlinlang.org/docs/coroutines-basics.html)

코루틴은 비동기적으로 실행되는 코드를 간소화하기 위해 등장한 동시 실행 설계 패턴이다.<br>

스레드는 생성과 전환하는 비용이 많이 들고 다른 스레드의 데이터가 필요하면<br>
해당 스레드의 작업이 끝나기까지 기다려야 하기 때문에 자원이 낭비된다.

반면, 코루틴은 스레드의 실행을 차단하지 않고 정지하거나 다시 실행하는 등의 동작을 할 수 있다.<br>
그리고 스레드 내에 여러 개의 코루틴을 실행하여 여러 작업을 처리할 수 있다.

---
코루틴을 사용하려면 코루틴 빌더(Coroutine Builder)로 코루틴을 생성해야 한다.<br>
그리고 코루틴이 생성되려면 코루틴 범위(Coroutine Scope) 내에서만 가능하다. 

아래 코드를 보자.

```kotlin
fun main() {
    GlobalScope.launch {
        println("1")
    }
    println("2")
}

// 실행 결과
2
```

- launch : 코루틴 빌더

코드 실행 결과 2가 출력됐다. 왜 그럴까?<br>
GlobalScope.launch를 이용해 코루틴을 생성했고, 비동기로 동작하면서 2가 먼저 출력이 됐다.<br>
하지만 2가 출력된 이후에 main 함수가 종료되면서 메인 스레드가 종료됐다.<br>
코루틴은 스레드 내에서 동작한다고 했는데, 스레드가 종료되었기 때문에 1이 출력되지 않은 것이다.

<br>

```kotlin
fun main() {
    GlobalScope.launch {
        println("1")
    }
    println("2")
    Thread.sleep(1000L)
}

// 실행 결과
2
1
```

함수의 마지막에 스레드를 1초간 정지시켰더니 코루틴 블록에서 1이 출력됐다.

<br>

다른 방법으로는 runBlocking을 사용하는 것이다.
```kotlin
fun main() {
    runBlocking {
        println("1")
    }
    println("2")
}

// 실행 결과
1
2
```
- runBlocking : 코루틴 빌더

runBlocking도 launch와 같은 코루틴 빌더이지만<br>
runBlocking은 코루틴 범위 내의 작업이 완료될 때까지 스레드의 동작을 차단한다.<br>
그래서 실행 결과 1, 2가 출력된 것이다.

스레드를 건드리는(생성, 차단 등) 동작은 비효율적이라서<br>
runBlocking은 실제 코드에서는 잘 안 쓰인다고 한다.

<br>

메인 함수를 runBlocking으로 생성하여 메인 스레드가 종료되지 않도록 했다.
```kotlin
fun main() = runBlocking {
    launch {
        println("1")
    }
    delay(1000L)
    println("2")
}

// 실행 결과
1
2
```
- delay : 정지 함수

코루틴에는 delay라는 정지 함수가 있다.<br>
delay는 일정 시간 동안 코루틴을 중단할 수 있다.<br>
따라서 실행 결과가 2, 1이 아닌 1, 2가 출력되었다.