<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/SharedStateGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class SharedStateGuideTest {
-->
## 目录

<!--- TOC -->

* [共享的可变状态和并发](#shared-mutable-state-and-concurrency)
  * [问题](#the-problem)
  * [没有发挥作用的volatile](#volatiles-are-of-no-help)
  * [线程安全的数据结构](#thread-safe-data-structures)
  * [以细粒度限制线程](#thread-confinement-fine-grained)
  * [以粗粒度限制线程](#thread-confinement-coarse-grained)
  * [互斥](#mutual-exclusion)
  * [Actors](#actors)

<!--- END_TOC -->

## 共享的可变状态和并发

协程可用多线程调度器（比如默认的 [Dispatchers.Default] ）并发执行。这样就可以提出所有常见的并发问题。主要的问题是同步访问**共享的可变状态**。协程领域对这个问题的一些解决方案类似于多线程领域中的解决方案，但其他解决方案则是独一无二的。

### 问题

我们启动一百个协程，它们做一千次相同的动作。我们同时会测量它们的完成时间，以便进一步的比较：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

</div>

我们从一个非常简单的动作开始：在 [GlobalScope] 中使用多线程的 [Dispatchers.Default] 递增一个共享的可变变量。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-01.kt)获得完整代码

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码最后打印出什么结果？它不太可能打印出“Counter = 100000”，因为一千个协程在一百个线程中同时递增计数器而且没有做并发处理。

> 注意：如果你的运行机器使用两个或者更少的cpu，那么你总是会看到 100000，因为线程池在这种情况下只会在一个线程中运行。要重现这个问题，可以做如下的变动：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val mtContext = newFixedThreadPoolContext(2, "mtPool") // explicitly define context with two threads
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    CoroutineScope(mtContext).massiveRun { // use it instead of Dispatchers.Default in this sample and below
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-01b.kt)获得完整代码

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

### 没有发挥作用的volatile

有一种常见的误解：volatile 可以解决并发问题。让我们尝试一下：


<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

@Volatile // in Kotlin `volatile` is an annotation
var counter = 0

fun main() = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-02.kt)获得完整代码

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码运行速度更慢了，但我们仍然没有得到 “Counter = 100000”，因为 volatile 变量保证可线性化（这是“原子”的技术术语）读取和写入变量，但在大量动作（在我们的示例中即“递增”操作）发生时并不提供原子性。


### 线程安全的数据结构

一种对线程、协程都有效的常规解决方法，就是使用线程安全（也称为同步的、可线性化、原子）的数据结构，它为需要在共享状态上执行的相应操作提供所有必需的同步处理。在简单的计数器场景中，我们可以使用具有 `incrementAndGet` 原子操作的 `AtomicInteger` 类：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = AtomicInteger()

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-03.kt)获得完整代码

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这是针对此类特定问题的最快解决方案。它适用于普通计数器、集合、队列和其他标准数据结构以及它们的基本操作。然而，它并不容易扩展为应对复杂状态、或复杂操作没有现成的线程安全实现的情况。


### 以细粒度限制线程

_限制线程_ 是解决共享可变状态问题的一种方案，其中对特定共享状态的所有访问权都限制在单个线程中。它通常应用于UI程序中：所有UI状态都局限于单个事件分发线程或应用主线程中。这在协程中很容易实现，通过使用一个单线程上下文：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun { // run each coroutine with DefaultDispathcer
        withContext(counterContext) { // but confine each increment to the single-threaded context
            counter++
        }
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-04.kt)获得完整代码

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行非常缓慢，因为它进行了 _细粒度_ 的线程限制。每个增量操作都得使用 [withContext] 块从多线程 [Dispatchers.Default] 上下文切换到单线程上下文。

### 以粗粒度限制线程

在实践中，线程限制是在大段代码中执行的，例如：状态更新类业务逻辑中大部分都是限于单线程中。下面的示例演示了这种情况，在单线程上下文中运行每个协程。这里我们使用 [CoroutineScope()] 函数来切换协程上下文为 [CoroutineScope]：


<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    CoroutineScope(counterContext).massiveRun { // run each coroutine in the single-threaded context
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-05.kt)获得完整代码

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行更快而且打印出了正确的结果。

### 互斥

该问题的互斥解决方案是使用永远不会同时执行的 _关键代码块_ 来保护共享状态的所有修改。在阻塞的世界中，你通常会为此目的使用 `synchronized` 或者 `ReentrantLock`。在协程中的替代品叫做 [Mutex] 。它具有 [lock][Mutex.lock]  和 [unlock][Mutex.unlock] 方法，可以隔离关键的部分。关键的区别在于 `Mutex.lock()` 是一个挂起函数，它不会阻塞线程。

还有 [withLock] 扩展函数，可以方便的替代常用的 `mutex.lock(); try { ... } finally { mutex.unlock() }` 模式：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val mutex = Mutex()
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        mutex.withLock {
            counter++        
        }
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-06.kt)获得完整代码

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

此示例中锁是细粒度的，因此会付出一些代价。但是对于某些必须定期修改共享状态的场景，它是一个不错的选择，但是没有自然线程可以限制此状态。


### Actors

一个 [actor](https://en.wikipedia.org/wiki/Actor_model) 是由以下元素组成的一个实体：一个协程、它的状态受限封装在此协程中、以及一个与其他协程通信的 _通道_ 。一个简单的 actor 可以简单的写成一个函数，但是一个拥有复杂状态的 actor 更适合由类来表示。

有一个 [actor] 协程构建器，它可以方便地将 actor 的 _邮箱通道_ 组合到其作用域中（用来接收消息）、组合发送 channel 与结果集对象，这样对 actor 的单个引用就可以作为其句柄持有。

使用 actor 的第一步是定一个 actor 要处理的消息类。Kotlin 的 [密封类](https://kotlinlang.org/docs/reference/sealed-classes.html) 很适合这种场景。我们使用 `IncCounter` 消息（用来递增计数器）和 `GetCounter` 消息（用来获取值）来定义 `CounterMsg` 密封类。后者需要发送回复。[CompletableDeferred] 通信原语表示未来可知（可传达）的单个值，此处用于此目的。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply
```

</div>

接下来我们定义一个函数，使用 [actor] 协程构建器来启动一个 actor：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// This function launches a new counter actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

</div>


主函数代码很简单：


<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply

// This function launches a new counter actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val counter = counterActor() // create the actor
    GlobalScope.massiveRun {
        counter.send(IncCounter)
    }
    // send a message to get a counter value from an actor
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // shutdown the actor
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-sync-07.kt)获得完整代码

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

actor 本身执行时所处上下文（就正确性而言）无关紧要。一个 actor 是一个协程，而一个协程是按顺序执行的，因此将状态限制到特定协程可以解决共享可变状态的问题。实际上，actor 可以修改自己的私有状态，但只能通过消息互相影响（避免任何锁定）。

actor 在高负载下比锁更有效，因为在这种情况下它总是有工作要做，而且根本不需要切换到不同的上下文。


> 注意， [actor] 协程构建器是 [produce] 协程构建器的双重构件。一个 actor 与它接收消息的通道相关联，而一个 producer 与它发送元素的通道相关联。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
<!--- END -->
