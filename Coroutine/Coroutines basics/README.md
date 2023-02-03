[Coroutines basics](https://kotlinlang.org/docs/coroutines-basics.html)

### 코루틴?
코루틴은 비동기적으로 실행되는 코드를 간소화하기 위해 등장한 동시 실행 설계 패턴이다.<br>

스레드는 생성과 전환에 쓰이는 비용이 많이 들고 다른 스레드의 데이터가 필요하면<br>
해당 스레드의 작업이 끝나기까지 기다려야 하기 때문에 자원이 낭비된다.

반면, 코루틴은 스레드의 실행을 차단하지 않고 정지하거나 다시 실행하는 등의 동작을 할 수 있다.<br>
그리고 스레드 내에 여러 개의 코루틴을 실행하여 여러 작업을 처리할 수 있다.

---
### 코루틴의 생성
코루틴을 사용하려면 코루틴 빌더(Coroutine Builder)로 코루틴을 생성해야 한다.<br>
이때 코루틴 빌더는 코루틴 범위(Coroutine Scope) 내에서만 선언 가능하다. 

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

1이 출력되는 것을 확인하려면 main 함수의 종료를 막아야한다.
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

함수의 마지막에 스레드를 1초간 정지시켜 코루틴 블록의 코드가 수행되도록 했다.

다른 방법으로는 코루틴의 runBlocking을 사용하는 것이다.
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

runBlocking도 launch와 같은 코루틴 빌더이지만 다른 점이 있다.<br>
runBlocking은 코루틴 범위 내의 작업이 완료될 때까지 스레드의 동작을 차단한다.<br>
그래서 실행 결과 1, 2가 출력된 것이다.

스레드를 건드리는(생성, 차단 등) 동작은 비효율적이라서 runBlocking은 실제 코드에서는 잘 안 쓰인다고 한다.

<br>

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

---
part 2

---
### Job
코루틴 빌더 중 launch는 Job 객체를 반환한다.<br>

Job은 실행된 코루틴에 대한 라이프 사이클과 같은 정보를 지니고 있다.<br>
코루틴을 취소하거나 완료될 때까지 대기를 시키는 것이 가능하기 때문에<br>
Job 객체는 코루틴 핸들러라고 생각하면 될 것 같다.

지금까지 코루틴의 작업이 완료되는 것을 delay를 통해서 임의로 제어를 했다.<br>
하지만 이 방식은 다른 코루틴의 작업이 언제 끝날지 알 수 없기 때문에 좋은 방법이 아니다.<br>

```kotlin
fun main() = runBlocking {
    launch {
        println("1")
    }.join()
    println("2")
}

// 실행 결과
1
2
```

코루틴에서는 launch의 반환 값인 Job 객체를 이용하여 명시적으로 제어를 할 수 있다.<br>
Job의 join은 코루틴의 작업이 완료될 때까지 코루틴을 일시 정지한다.

다른 예시를 보자.
```kotlin
fun main() = runBlocking {
    val job = launch {
        delay(3000L)
        println("1")
    }
    
    println("2")

    job.join()
    println("3")
}

// 실행 결과
2
1
3
```
launch라는 코루틴 빌더로 인해 비동기로 동작이 수행된다.<br>
2가 출력되는 동안 launch의 코루틴 블록은 3초를 정지하게 되고<br>
밖에서는 join을 만나 launch의 작업이 끝나기를 기다린다.<br>
그 결과 2, 1, 3으로 출력되었다.

---

### Scope builder
스코프 빌더를 사용하면 코루틴 범위를 생성할 수 있다.<br>
스코프 빌더에는 coroutineScope가 있다.<br>

```
runBlocking은 코루틴 빌더라고 나왔지만 스코프 빌더로도 볼 수 있지 않을까?
CoroutineScope와 GlobalScope도 스코프 빌더이지 않을까?
```
coroutineScope는 자신이 생성한 코루틴 블록의 작업이 완료될 때까지 코루틴을 정지한다.<br>
```kotlin
fun main() = runBlocking {
    coroutineScope {
        println("1")
    }
    println("2")
}

// 실행 결과
1
2
```

runBlocking처럼 실행 결과가 1, 2로 출력되어 runBlock과 coroutineScope가 비슷해 보일 수 있다.<br>
runBlocking은 자신의 블록 작업이 완료될 때까지 스레드를 차단하는 것이고 coroutineScope는 코루틴을 정지한다.<br>
그래서 runBlocking은 일반 함수, coroutineScope는 정지 함수이다.

```kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        launch {
            println("${Thread.currentThread()}: block1 start")
            delay(500L)
            println("${Thread.currentThread()}: 1")
            delay(500L)
            println("${Thread.currentThread()}: 2")
        }

        coroutineScope {
            launch {
                println("${Thread.currentThread()}: block2 start")
                delay(500L)
                println("${Thread.currentThread()}: 3")
                delay(500L)
                println("${Thread.currentThread()}: 4")
            }
        }
    }
    delay(300)
    println("cancel")
    job.cancel()

    job.join()

    println("${Thread.currentThread()}: finish")
}

// 실행 결과
Thread[DefaultDispatcher-worker-2,5,main]: block1 start
Thread[DefaultDispatcher-worker-1,5,main]: block2 start
cancel
Thread[main,5,main]: finish
```
- cancel : Job을 취소하는 함수

위 코드에서 GlobalScope.launch를 통해 비동기로 동작시켰고,<br>
delay(300)이 되는 동안 block1과 block2가 출력되었고 cancel을 통해 코루틴을 종료시켰다.<br>
그래서 나머지 코루틴 블럭의 나머지 코드가 실행되지 않았다.

coroutineScope를 runBlock으로 바꿔보면
```kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        launch {
            println("${Thread.currentThread()}: block1 start")
            delay(500L)
            println("${Thread.currentThread()}: 1")
            delay(500L)
            println("${Thread.currentThread()}: 2")
        }

        runBlocking {
            launch {
                println("${Thread.currentThread()}: block2 start")
                delay(500L)
                println("${Thread.currentThread()}: 3")
                delay(500L)
                println("${Thread.currentThread()}: 4")
            }
        }
    }
    delay(300)
    println("cancel")
    job.cancel()

    job.join()

    println("${Thread.currentThread()}: finish")
}

// 실행 결과
Thread[DefaultDispatcher-worker-2,5,main]: block1 start
Thread[DefaultDispatcher-worker-1,5,main]: block2 start
cancel
Thread[DefaultDispatcher-worker-1,5,main]: 3
Thread[DefaultDispatcher-worker-1,5,main]: 4
Thread[main,5,main]: finish
```
코루틴을 종료시켰음에도 runBlocking 블록의 코드들이 실행되었다.<br>

coroutineScope가 부모 코루틴이 종료되는 것에 영향을 받은 반면<br>
runBlocking이 현재 스레드의 동작을 차단하면서 cancel의 영향을 안받은게 아닐까 생각이 든다.

---

part3

---

### suspend

```kotlin
suspend fun doWorld() {
    delay(500)
    println("1")
}

fun main() = runBlocking {
    launch { doWorld() }
    launch { println("2") }
    launch {
        delay(500)
        println("3")
    }
    println("4")
}

// 실행 결과
4
2
1
3
```

함수명 앞에 suspend라는 키워드가 붙으면 정지 함수를 의미한다.<br>
suspend가 붙음으로써 delay 같은 실행, 정지 기능을 수행 할 수 있다. 

위 코드 처럼 suspend를 사용하면 개별 함수로 분리가 가능하다.<br>
주의 할 점은 suspend는 코루틴 빌더 또는 다른 suspend 함수 에서 호출되어야 한다.<br> 
(앞에서 본 delay, join을 살펴보면 suspend로 선언되어있다.)

