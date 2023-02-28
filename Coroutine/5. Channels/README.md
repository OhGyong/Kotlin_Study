[Channels](https://kotlinlang.org/docs/channels.html)

---

### Channel basics

코루틴 간에 데이터를 전달은 어떻게 할 수 있을까?

Deferred는 코루틴 간에 단일 값을 전송하는 편리한 방법을 제공하고,<br>
값의 스트림을 전달하는 방법으로는 Channel이 있다.

Channel을 선언하고 값을 넣을 때는 send(), 꺼낼 때는 recieve()를 사용한다.<br>
(BlockingQueue와 개념적으로 매우 유사하다.)

```kotlin
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}

// 실행 결과
1
4
9
16
25
Done!
```

---

### Closing and iteration over channels
Queue와는 달리 Channel은 close를 통해 더 이상의 데이터가 오지 않음을 표시할 수 있다.

개념적으로 close는 Channel에 특별한 토큰을 보낸다고 볼 수 있다.

수신하는 쪽에서는 Channel로부터 더 편하게 데이터를 받기 위해 일반적인 for 루프를 사용하게 되는데,<br/>
close 토큰을 받자마자 수신하던 반복문은 곧장 멈추게 된다.

이럴 경우 cloase 토큰이 Channel에 도달하기 전에 전달된 모든 데이터들은 수신이 보장된다.

```kotlin
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}

// 실행 결과
1
4
9
16
25
Done!
```

위 코드에서는 Channel을 close로 종료하기 전까지 Channel에서 전송한 값을 모두 받는 것을 확인할 수 있다.

아래는 Channel에서 값을 보내는 도중에 close 처리를 해봤다.

```kotlin
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        launch {
            try {
                for (x in 1..5) {
                    delay(100)
                    channel.send(x * x)
                }
            } catch (e: java.lang.Exception) {
                println("error : $e")
            }
        }
        delay(400)
        channel.close() // we're done sending
    }

    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}

// 실행 결과
1
4
9
Done!
error : kotlinx.coroutines.channels.ClosedSendChannelException: Channel was closed
```

close 토큰이 Channel에 전달되기 전에 전송된 값들은 모두 출력되는 것을 확인할 수 있다.

--- 

### Building channel producers

코루틴이 어떤 데이터 스트림을 생성하는 패턴은 꽤나 흔한 일이며,<br/>
동시성 코드에서 종종 볼 수 있는 producer-consumer 패턴의 종류이다.

Channel을 매개변수로 사용하는 함수를 통해 producer를 추상화할 수 있으나,<br/>
함수로부터 결과를 반드시 반환해야한다는 일반적인 상식과는 반대된다.

producer의 추상화 작업을 쉽게 해주는 produce라는 코루틴 빌더가 있고,
