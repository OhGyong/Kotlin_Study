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

