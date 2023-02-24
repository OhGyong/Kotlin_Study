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