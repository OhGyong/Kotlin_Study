[Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

---

### Dispatchers and threads

코루틴은 항상 코틀린 표준 라이브러리로 정의된 CoroutineContext에 의해 표시되는 일부 컨텍스트에서 실행된다.

코루틴 컨텍스트는 다양한 요소의 집합으로, 주 요소는 Job과 Dispatcher라고 할 수 있다.

코루틴 디스패처는 코루틴의 실행을 위해 사용되는 스레드를 결정한다.</br>
(코루틴을 특정 스레드로 제한, 스레드 풀로 디스패치, 아무 제한 없이 실행)

launch와 async 같은 코루틴 빌더는 선택적으로 CoroutineContext를 파라미터로 받는다.<br>
파라미터인 CoroutineContext는 새 코루틴 또는 컨텍스트 요소를 위해 명시적으로 지정하는데 사용된다.

```kotlin
fun main() = runBlocking<Unit> {
    // 1
    launch {
        // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }

    // 2
    launch(Dispatchers.Unconfined) {
        // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }

    // 3
    launch(Dispatchers.Default) {
        // will get dispatched to DefaultDispatcher
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }

    // 4
    launch(newSingleThreadContext("MyOwnThread")) {
        // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}

// 실행 결과
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
main runBlocking      : I'm working in thread main
newSingleThreadContext: I'm working in thread MyOwnThread
```

1. launch 함수가 파라미터 없이 호출될 경우에는 현재 코루틴 범위에 해당하는 컨텍스트를 사용한다.
2. Unconfined는 자신이 호출된 스레드에서 동작한다. → 위 코드에서 메인 스레드로 나온 이유는 메인 스레드에서 호출됐기 때문
3. Default는 코루틴 범위 내에서 명시된 디스패처가 없는 경우에 쓰이고, 공용 백그라운드 스레드 풀을 사용한다.  
4. newSingleThreadContext은 코루틴이 실행되도록 스레드를 새로 생성한다. → 해당 코루틴을 위해 생성된 스레드이므로 필요가 없을 때는 닫아줘야한다.

---

### Unconfined vs confined dispatcher
Dispatchers.Unconfined는 호출한 스레드에서 코루틴을 시작하지만 첫 번째 중단 함수를 만날 때까지만 실행된다.

정지된 이후 다시 재개될 때는 정지 함수가 호출된 스레드에서 다시 시작된다.

Unconfined 디스패처는 CPU 시간을 소비하거나 공유 데이터를 업데이트하지 않아야 한다.

```kotlin
fun main() = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
}

// 실행 결과
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

디스패처는 기본적으로 부모 CoroutineScope를 상속받는다.

위 코드에서 파라미터 없이 호출된 launch의 경우 부모 Scope를 상속 받았기 때문에 runBlocking의 스레드에서 계속 실행된다.

반면 unconfined 코루틴은 delay 함수가 사용한 DefaultExecutor 스레드에서 재개된다.

<br>

참고로 unconfined 디스패처는 사이드 이펙트를 발생시킬 수 있는 고급 메커니즘으로 일반 코드에서 사용하지 않는 것을 추천하고 있다.

---

### 컨텍스트에서의 Job
코루틴의 Job은 컨텍스트의 일부분이며 **`coroutineContext[Job]`**을 사용하여 Job을 얻을 수 있다.

```kotlin
fun main() = runBlocking {
    println("My job is ${coroutineContext[Job]}")
    println("Active is ${coroutineContext[Job]?.isActive}")
}

// 실행 결과
My job is BlockingCoroutine{Active}@335eadca
Active is true
```

위의 코드를 보면 `coroutineContext[Job]?.isActive`가 Cancellation and timeouts에서 다뤘던<br>
isActive의 간략한 표현임을 알 수 있다.

isActive 뿐만 아니라 Job의 여러 확장 함수들이 축약 표현으로 사용되었다.

---

### 코루틴의 자식

어떤 코루틴을 다른 코루틴의 CoroutineScope에서 생성하면 다른 코루틴의 coroutineContext를 통해<br>
컨텍스트를 상속하고, 부모-자식 관계를 형성하게 된다.

새로 생성된 코루틴의 Job도 상위 코루틴 Job의 자식이 되고, 부모 코루틴이 종료되면 자식 코루틴도 재귀적으로 종료된다.

하지만 다음 두 가지 방법으로 이러한 룰을 무시할 수 있다.
1. CoroutineScope의 범위를 명시적으로 표시한 경우(ex-GlobalScope 사용) 해당 범위 내에서는 코루틴이 독립적으로 동작한다.
2. 다른 Job 객체가 새 코루틴의 컨텍스트로 전달되면 상위 범위의 Job을 재정의한다.

이 두 방법은 아래의 코드에서 확인할 수 있다.

```kotlin
fun main() = runBlocking {
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        // it spawns two other jobs
        launch(Job()) {
            println("job1: 새로운 Context를 전달 받음")
            delay(1000)
            println("job1: 영향을 받지 않았음")
        }
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: 일반적인 자식 코루틴")
            delay(1000)
            println("job2: 호출이 안 됨")
        }
        GlobalScope.launch {
            delay(200)
            println("job3: Scope를 명시적으로 표시함")
            delay(1000)
            println("job3: 영향을 받지 않았음")
        }
    }
    delay(500)
    request.cancel() // cancel processing of the request
    println("main: Who has survived request cancellation?")
    delay(1000) // delay the main thread for a second to see what happens
}

// 실행 결과
job1: 새로운 Context를 전달 받음
job2: 일반적인 자식 코루틴
job3: Scope를 명시적으로 표시함
main: Who has survived request cancellation?
job1: 영향을 받지 않았음
job3: 영향을 받지 않았음
```

---

### 부모 코루틴의 의무
부모 코루틴은 항상 모든 자식 코루틴의 실행이 완료될 때까지 기다린다.

```kotlin
fun main() = runBlocking {
    val request = launch {
        launch {
            delay(1000)
        }

        launch {
            delay(2000)
        }
    }

    delay(1000)
    println("request.isActive ${request.isActive}")

    delay(2000)
    println("request.isActive ${request.isActive}")
}

// 실행 결과
request.isActive true
request.isActive false
```

부모 코루틴인 request에 자식 코루틴을 두 개 생성하였다.

1초가 지났을 때 자식 코루틴 하나가 완료되지 않아서 부모 코루틴의 Active가 true로 나왔고<br>
2초가 지나서 자식 코루틴이 모두 종료되자 부모 코루틴의 Active가 false로 나오는것을 확인할 수 있다.