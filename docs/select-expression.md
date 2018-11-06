<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/SelectGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class SelectGuideTest {
--> 


## 目录

<!--- TOC -->

* [select 表达式（试验性）](#select-expression-experimental)
  * [从通道查询](#selecting-from-channels)
  * [从关闭的通道查询](#selecting-on-close)
  * [查询并发送](#selecting-to-send)
  * [查询延迟值](#selecting-deferred-values)
  * [在延迟值通道上切换](#switch-over-a-channel-of-deferred-values)

<!--- END_TOC -->



## select 表达式（实验性的）

select 表达式可以同时等待多个挂起函数，并 _选择_ <!--
-->第一个可用的。

> Select 表达式在 `kotlinx.coroutines` 中是一个实验性的特性。这些API在<!--
--> `kotlinx.coroutines` 库即将到来的更新中可能会<!--
-->发生改变。

### 从通道中查询

我们现在有两个字符串生产者：`fizz` 和 `buzz` 。其中 `fizz` 生产者每300毫秒产出 “Fizz” 字符串：



```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // 每 300ms 发送 "Fizz"
        delay(300)
        send("Fizz")
    }
}
```



接着 `buzz` 每500毫秒产出 “Buzz!” 字符串：



```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // 每 500ms 发送 "Buzz!"
        delay(500)
        send("Buzz!")
    }
}
```



使用 [receive][ReceiveChannel.receive] 挂起函数，我们可以从一个或另一个通道接收数据。<!--
-->但是 [select] 表达式允许我们使用其<!--
--> [onReceive][ReceiveChannel.onReceive] 子句 _同时_ 从两者接收：



```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> 意味着该 select 表达式不返回任何结果
        fizz.onReceive { value ->  // 这是第一个 select 子句
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // 这是第二个 select 子句
            println("buzz -> '$value'")
        }
    }
}
```



让我们运行7次：

<!--- CLEAR -->



```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.fizz() = produce<String> {
    while (true) { // 每 300ms 发送 "Fizz"
        delay(300)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) { // 每 500ms 发送 "Buzz!"
        delay(500)
        send("Buzz!")
    }
}

suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> 意味着该 select 表达式不返回任何结果
        fizz.onReceive { value ->  // 这是第一个 select 子句
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // 这是第二个 select 子句
            println("buzz -> '$value'")
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren() // 取消 fizz 和 buzz 协程
//sampleEnd
}
```



> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-01.kt)获得完整代码

这段代码的结果如下：

```text
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
buzz -> 'Buzz!'
```

<!--- TEST -->

### 从关闭的通道查询

select 中的 [onReceive][ReceiveChannel.onReceive] 子句执行在已经关闭的通道会失败，并导致相应的 <!--
-->`select` 抛出异常。我们可以使用 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 子句在关闭通道时执行<!--
-->特定操作。以下示例还显示了 `select` 是一个返回<!--
-->其查询方法结果的表达式：



```kotlin
suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'a' is closed" 
            else 
                "a -> '$value'"
        }
        b.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'b' is closed"
            else    
                "b -> '$value'"
        }
    }
```



现在有一个生成四次 “Hello” 字符串的 `a` 通道，<!--
-->和一个产出四次 “World” 字符串的 `b` 通道，我们在这两个通道上使用它：

<!--- CLEAR -->



```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'a' is closed"
            else
                "a -> '$value'"
        }
        b.onReceiveOrNull { value ->
            if (value == null)
                "Channel 'b' is closed"
            else
                "b -> '$value'"
        }
    }

fun main() = runBlocking<Unit> {
//sampleStart
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    repeat(8) { // 打印最早的8个结果
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()
//sampleEnd
}
```



> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-02.kt)获得完整代码

这段代码的结果非常有趣，所以我们将在模式细节中分析它：

```text
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

<!--- TEST -->

有几个结果可以通过观察得出。

首先，`select` _偏向于_ 第一个子句，当可以同时选到多个子句时，<!--
-->第一个子句将被选中。在这里，两个通道都在不断地生成字符串，因此 `a` 通道<!--
-->作为 select 中的第一个子句获胜。然而因为我们使用的是无缓冲通道，所以 `a` 在其调用 <!--
-->[send][SendChannel.send] 时会不时地被挂起，进而 `b` 也有机会发送。

第二个观察结果是，当通道已经关闭时，<!--
-->会立即选择 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 。

### 查询并发送

Select 表达式具有 [onSend][SendChannel.onSend] 子句，可以很好的与<!--
-->选择的偏向特性结合使用。

我们来编写一个整数生成器的示例，当主通道上的<!--
-->消费者无法跟上它时，它会将值发送到 `side` 通道上：



```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // 生产从 1 到 10 的10个数值
        delay(100) // 每 100 ms
        select<Unit> {
            onSend(num) {} // 发送到主通道
            side.onSend(num) {} // 或者发送到 side 通道
        }
    }
}
```



消费者将会非常缓慢，每个数值处理需要250毫秒：

<!--- CLEAR -->



```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // 生产从 1 到 10 的10个数值
        delay(100) // 每 100 ms
        select<Unit> {
            onSend(num) {} // 发送到主通道
            side.onSend(num) {} // 或者发送到 side 通道
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val side = Channel<Int>() // 分配 side 通道
    launch { // 对于 side 通道来说，这是一个很快的消费者
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // 让我们不急切地、适当地来消费数值
    }
    println("Done consuming")
    coroutineContext.cancelChildren()
//sampleEnd
}
```



> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-03.kt)获得完整代码

让我们看看会发生什么：

```text
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

<!--- TEST -->

### 查询延迟值

延迟值可以使用 [onAwait][Deferred.onAwait] 子句查询。<!--
-->让我们启动一个异步函数，它在<!--
-->随机的延迟后会返回延迟字符串：



```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```



让我们启动十几个，每个都延迟随机的时间。



```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```



现在主函数在等待第一个函数完成，并统计仍处于<!--
-->激活状态的延迟值的数量。注意，我们在这里的使用，事实上是把 `select` 表达式作为一种Kotlin DSL，<!--
-->所以我们可以用任意代码为它提供子句。在这种情况下，我们遍历一个<!--
-->延迟值的队列，并为每个延迟值提供 `onAwait` 子句。

<!--- CLEAR -->



```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*
import java.util.*

fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}

fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val list = asyncStringsList()
    val result = select<String> {
        list.withIndex().forEach { (index, deferred) ->
            deferred.onAwait { answer ->
                "Deferred $index produced answer '$answer'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
//sampleEnd
}
```



> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-04.kt)获得完整代码

输出如下：

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 在延迟值通道上切换

我们现在来编写一个通道生产者函数，它消费一个产生延迟字符串的通道，并等待每个接收的<!--
-->延迟值，但只在下一个延迟值到达或者通道关闭之前。此示例将 <!--
-->[onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 和 [onAwait][Deferred.onAwait] 子句放在同一个 `select` 中：



```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // 从第一个接收到的延迟值开始
    while (isActive) { // 循环直到被取消或关闭
        val next = select<Deferred<String>?> { // 从这个 select 中返回下一个延迟值或 null
            input.onReceiveOrNull { update ->
                update // 替换下一个要等待的值
            }
            current.onAwait { value ->  
                send(value) // 发送当前延迟生成的值
                input.receiveOrNull() // 然后使用从输入通道得到的下一个延迟值
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // 跳出循环
        } else {
            current = next
        }
    }
}
```



为了测试它，我们将用一个简单的异步函数，它在特定的延迟后返回特定的字符串：




```kotlin
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```



主函数只是启动一个协程来打印 `switchMapDeferreds` 的结果并向它发送一些<!--
-->测试数据：

<!--- CLEAR -->



```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // 从第一个接收到的延迟值开始
    while (isActive) { // 循环直到被取消或关闭
        val next = select<Deferred<String>?> { // 从这个 select 中返回下一个延迟值或 null
            input.onReceiveOrNull { update ->
                update // 替换下一个要等待的值
            }
            current.onAwait { value ->
                send(value) // 发送当前延迟生成的值
                input.receiveOrNull() // 然后使用从输入通道得到的下一个延迟值
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // 跳出循环
        } else {
            current = next
        }
    }
}

fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}

fun main() = runBlocking<Unit> {
//sampleStart
    val chan = Channel<Deferred<String>>() // 测试使用的通道
    launch { // 启动打印协程
        for (s in switchMapDeferreds(chan)) 
            println(s) // 打印每个获得的字符串
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // 充足的时间来生产 "BEGIN"
    chan.send(asyncString("Slow", 500))
    delay(100) // 不充足的时间来生产 "Slow"
    chan.send(asyncString("Replace", 100))
    delay(500) // 在最后一个前给它一点时间
    chan.send(asyncString("END", 500))
    delay(1000) // 给执行一段时间
    chan.close() // 关闭通道……
    delay(500) // 然后等待一段时间来让它结束
//sampleEnd
}
```



> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-05.kt)获得完整代码

这段代码的结果：

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Deferred.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/on-await.html
<!--- INDEX kotlinx.coroutines.channels -->
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[ReceiveChannel.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive.html
[ReceiveChannel.onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/on-receive-or-null.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[SendChannel.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/on-send.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
