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

수신하는 쪽에서는 Channel로부터 데이터를 받기 위해 for 루프를 사용하는 것이 더 편리하다.

개념적으로 close는 Channel에 특별한 토큰을 보낸다고 볼 수 있다.

이 토큰을 받자마자 수신하던 반복문은 곧장 멈추게 되기 때문에<br>
닫혔다는 신호가 도달하기 전에 전달된 모든 데이터들은 수신된다는 보장이 있다.