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

<!--- TOC -->

* [select 表达式（试验性）](#select-expression-experimental)
  * [从通道查询](#selecting-from-channels)
  * [从关闭的通道查询](#selecting-on-close)
  * [查询并发送](#selecting-to-send)
  * [查询延迟值](#selecting-deferred-values)
  * [在延迟值通道上切换](#switch-over-a-channel-of-deferred-values)

<!--- END_TOC -->



## select 表达式（试验性）

select 表达式可以同时等待多个挂起函数，并 _选择_ 第一个可用的。

> Select 表达式是 `kotlinx.coroutines` 的试验性功能。它们的 API 在 `kotlinx.coroutines` 库即将到来的更新中可能会有很大的变化。

### 从通道中查询

我们现在有两个字符串生产者：`fizz` 和 `buzz` 。其中 `fizz` 生产者每300毫秒产出 “Fizz” 字符串：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}
```



接着 `buzz` 每500毫秒产出 “Buzz!” 字符串：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.buzz() = produce<String> {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}
```



使用 [receive][ReceiveChannel.receive] 挂起函数，我们可以从一个或另一个通道接收数据。但是 [select] 表达式允许我们使用其 [onReceive][ReceiveChannel.onReceive] 子句同时从两者接收：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}
```



让我们运行7次：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.fizz() = produce<String> {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}

suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
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
    coroutineContext.cancelChildren() // cancel fizz & buzz coroutines
//sampleEnd        
}
```

</div>

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

select 中的 [onReceive][ReceiveChannel.onReceive] 子句在已经关闭的通道会失败，并导致相应的 `select` 抛出异常。我们可以使用 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 子句在关闭通道时执行特定操作。以下示例还显示了 `select` 是一个返回其查询方法结果的表达式：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

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

</div>

现在有一个产出四次 “Hello” 字符串的 `a` 通道、一个产出四次 “World” 字符串的 `b` 通道，我们在这两个通道上使用它：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

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
    repeat(8) { // print first eight results
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()  
//sampleEnd      
}    
```

</div>

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

首先，`select` 偏向于第一个子句，当可以同时选到多个子句时，第一个子句将被选中。在这里，两个通道都在不断地生成字符串，因此作为 select 中的第一个子句的通道获胜。然而因为我们使用的是无缓冲通道，所以 `a` 在其发送调用时会不时被挂起，进而 `b` 也有机会发送。

第二个观察结果是，当通道已经关闭时，会立即选择 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 。



### 查询并发送

Select 表达式具有 [onSend][SendChannel.onSend] 子句，可以很好的与选择的偏向特性结合使用。

我们来编写一个整数生成器的示例，当主通道上的消费者无法跟上它时，它会将值发送到 `side` 通道上：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel     
        }
    }
}
```

</div>

消费者将会非常缓慢，每个数值处理需要250毫秒：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel     
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val side = Channel<Int>() // allocate side channel
    launch { // this is a very fast consumer for the side channel
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // let us digest the consumed number properly, do not hurry
    }
    println("Done consuming")
    coroutineContext.cancelChildren()  
//sampleEnd      
}
```

</div> 

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

延迟值可以使用 [onAwait][Deferred.onAwait] 子句查询。让我们启动一个延迟随机时间后返回延迟字符串的异步方法：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}
```

</div>

让我们启动十几个，每个都延迟随机的时间。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

</div>

现在主函数等待第一个函数完成，并统计仍处于激活状态的延迟值的数量。注意，我们在这里的使用，事实上是把 `select` 表达式作为一种Kotlin DSL，所以我们可以用任意代码为它提供子句。在这种情况下，我们遍历一个延迟值的队列，为每个延迟值提供 `onAwait` 子句。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

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

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-select-04.kt)获得完整代码

输出如下：

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### 在延迟值通道上切换

我们现在来编写一个通道生产者函数，它消费一个产生延迟字符串的通道，并等待每个接收的延迟值，但只在下一个延迟值到达或者通道关闭之前。此示例将 [onReceiveOrNull][ReceiveChannel.onReceiveOrNull] 和 [onAwait][Deferred.onAwait] 子句放在同一个 `select` 中：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->  
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}
```

</div>

为了测试它，我们将用一个简单的异步函数，它在特定的延迟后返回特定的字符串：


<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}
```

</div>

主函数只是启动一个协程来打印 `switchMapDeferreds` 的结果并向它发送一些测试数据：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*
    
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->  
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
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
    val chan = Channel<Deferred<String>>() // the channel for test
    launch { // launch printing coroutine
        for (s in switchMapDeferreds(chan)) 
            println(s) // print each received string
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // enough time for "BEGIN" to be produced
    chan.send(asyncString("Slow", 500))
    delay(100) // not enough time to produce slow
    chan.send(asyncString("Replace", 100))
    delay(500) // give it time before the last one
    chan.send(asyncString("END", 500))
    delay(1000) // give it time to process
    chan.close() // close the channel ... 
    delay(500) // and wait some time to let it finish
//sampleEnd
}
```

</div>

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
