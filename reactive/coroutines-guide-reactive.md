<!--- INCLUDE .*/example-reactive-([a-z]+)-([0-9]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide-reactive.md by Knit tool. Do not edit.
package kotlinx.coroutines.rx2.guide.$$1$$2

-->
<!--- KNIT     kotlinx-coroutines-rx2/test/guide/.*\.kt -->
<!--- TEST_OUT kotlinx-coroutines-rx2/test/guide/test/GuideReactiveTest.kt
// This file was automatically generated from coroutines-guide-reactive.md by Knit tool. Do not edit.
package kotlinx.coroutines.rx2.guide.test

import kotlinx.coroutines.guide.test.*
import org.junit.Test

class GuideReactiveTest : ReactiveTestBase() {
-->

# 响应式流与协程指南

这篇教程介绍了 Kotlin 协程与响应式流的不同点并展示了<!--
-->如何将它们更好的一起使用。在此之前熟悉包含在
[Guide to kotlinx.coroutines](../docs/coroutines-guide.md) 中的基础协程概念不是必须的， 
但如果熟悉它将会是个很大的加分。如果你熟悉响应式流，你可能发现本指南会<!--
-->更好地介绍协程的世界。

在 `kotlinx.coroutines` 项目中有一系列和响应式流相关的模块：

* [kotlinx-coroutines-reactive](kotlinx-coroutines-reactive) ——为 [Reactive Streams](http://www.reactive-streams.org) 提供的适配
* [kotlinx-coroutines-reactor](kotlinx-coroutines-reactor) ——为 [Reactor](https://projectreactor.io) 提供的适配
* [kotlinx-coroutines-rx2](kotlinx-coroutines-rx2) ——为 [RxJava 2.x](https://github.com/ReactiveX/RxJava) 提供的适配

本指南主要基于 [Reactive Streams](http://www.reactive-streams.org) 的规范并使用
`Publisher` 接口和一些基于 [RxJava 2.x](https://github.com/ReactiveX/RxJava) 的示例，
该示例实现了响应式流的规范。

欢迎你在 Github 上 clone 
[`kotlinx.coroutines` 项目](https://github.com/Kotlin/kotlinx.coroutines)
到你的工作站中，这是为了<!--
-->可以运行所有在本指南中展示的示例。它们被包含在项目的
[reactive/kotlinx-coroutines-rx2/test/guide](kotlinx-coroutines-rx2/test/guide)
路径中。
 
## 目录

<!--- TOC -->

* [响应式流与通道的区别](#differences-between-reactive-streams-and-channels)
  * [迭代的基础](#basics-of-iteration)
  * [订阅与取消](#subscription-and-cancellation)
  * [背压](#backpressure)
  * [Rx 主题 vs 广播通道](#rx-subject-vs-broadcastchannel)
* [操作符](#operators)
  * [Range](#range)
  * [Fused filter-map hybrid](#fused-filter-map-hybrid)
  * [Take until](#take-until)
  * [Merge](#merge)
* [协程上下文](#coroutine-context)
  * [线程与 Rx](#threads-with-rx)
  * [线程与协程](#threads-with-coroutines)
  * [Rx observeOn](#rx-observeon)
  * [使用协程上下文来管理它们](#coroutine-context-to-rule-them-all)
  * [不受限的上下文](#unconfined-context)

<!--- END_TOC -->

## 响应式流与通道的区别

本节主要包含响应式流与以协程为基础的通道的不同点。

### 迭代的基础

[通道]与如下所示的响应式流类有类似的概念：

* Reactive stream [Publisher](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/api/src/main/java/org/reactivestreams/Publisher.java)；
* Rx Java 1.x [Observable](http://reactivex.io/RxJava/javadoc/rx/Observable.html)；
* Rx Java 2.x [Flowable](http://reactivex.io/RxJava/2.x/javadoc/)，`Publisher` 的实现者。

它们都描述了一个异步的有限或无限的元素流（在 Rx 中又名 items），
并且都支持背压。
  
然而，使用 Rx 的术语的话，`Channel` 总是表示了一个条目的 _热_ 流。元素被生产者<!--
-->协程发送到通道并被消费者协程所接收。
在通道中每调用一次 [receive][ReceiveChannel.receive] 就消费一个元素。
让我们用以下的例子来说明：

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    // 创建一个通道，该通道每200毫秒生产一个数字，从 1 到 3
    val source = produce<Int> {
        println("Begin") // 在输出中标记协程开始运行
        for (x in 1..3) {
            delay(200) // 等待 200 毫秒
            send(x) // 将数字 x 发送到通道中
        }
    }
    // 从 source 中打印元素
    println("Elements:")
    source.consumeEach { // 在 source 中消费元素
        println(it)
    }
    // 再次从 source 中打印元素
    println("Again:")
    source.consumeEach { // 从 source 中消费元素
        println(it)
    }
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-01.kt)获得完整代码

这段代码产生了如下输出：

```text
Elements:
Begin
1
2
3
Again:
```

<!--- TEST -->

注意，“Begin” 只被打印了一次，因为 [produce] _协程构建器_ 被执行的时候，
只创建了一个协程来生产元素流。所有被生产的元素都被 
[ReceiveChannel.consumeEach][consumeEach] 
扩展函数消费。没有办法从这个通道重复<!--
-->接收元素。当生产者协程结束时该通道被关闭，
再次尝试接收元素将不会接收到任何东西。

让我们使用 `kotlinx-coroutines-reactive` 模块中的 [publish] 协程构建器 代替 `kotlinx-coroutines-core` 模块中的 [produce]
来重写这段代码。代码保持相似， 
但是在 `source` 接收 [ReceiveChannel] 类型的地方，现在它接收响应式流的
[Publisher](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html) 
类型。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    // 创建一个 publisher，每200毫秒生产一个数字，从 1 到 3
    val source = publish<Int> {
    //           ^^^^^^^  <--- 这里与先前的示例不同
        println("Begin") // 在输出中标记协程开始运行
        for (x in 1..3) {
            delay(200) // 等待 200 毫秒
            send(x) // 将数字 x 发送到通道中
        }
    }
    // 从 source 中打印元素
    println("Elements:")
    source.consumeEach { // 在 source 中消费元素
        println(it)
    }
    // 再次从 source 中打印元素
    println("Again:")
    source.consumeEach { // 在 source 中消费元素
        println(it)
    }
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-02.kt)获得完整代码

现在这段代码的输出变为：

```text
Elements:
Begin
1
2
3
Again:
Begin
1
2
3
```

<!--- TEST -->

这个例子使用高亮标记了响应式流与通道的不同点。响应式流是一种<!--
-->高阶的概念。当通道 _是_ 一个元素的流时，该响应式流定义了一个“食谱”来规定元素<!--
-->在流中如何被生产。它成为了 _订阅_ 上真实的元素流。每一个订阅者也许会接收相同或不同的<!--
-->元素流，这取决于 `Publisher` 的相应实现如何工作。

[publish] 协程构建器已经被用于先前的示例，每次订阅都会启动一个新的协程。
每一次 [Publisher.consumeEach][org.reactivestreams.Publisher.consumeEach] 被调用都创建了一个新的订阅者。
在这段代码中我们有两处调用，所以我们能看到“Begin”被打印了两次。

在 Rx 的术语中这是调用了一个 _冷_ 发布者。大量的标准 Rx 操作符也会生产冷流。我们可以在<!--
-->协程中迭代它们，并且每个订阅者都会生产相同的元素流。

**警告**：它计划在未来的一秒钟内在通道上调用 `consumeEach` 方法<!--
-->来准备好消费元素可以快速的失败，这会<!--
-->立即抛出一个 `IllegalStateException`。
查看 [this issue](https://github.com/Kotlin/kotlinx.coroutines/issues/167)
的细节。

> 注意，我们可以使用 Rx 中的
[publish](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#publish()) 
操作符与 [connect](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/flowables/ConnectableFlowable.html#connect())
方法来替换我们在通道中所看到的类似的行为。

### 订阅与取消

在先前小节的例子中使用 `source.consumeEach { ... }` 代码片段来打开一个订阅<!--
-->并从它接受所有元素。如果我们需要在如何处理<!--
-->从通道中接收到的元素施加更多控制，我们可以使用 [Publisher.openSubscription][org.reactivestreams.Publisher.openSubscription]，
这将在下面的示例中展示：

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.reactive.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // 五个数字的区间
        .doOnSubscribe { println("OnSubscribe") } // 提供了一些可被观察的点
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... 在正在执行的代码中
    var cnt = 0 
    source.openSubscription().consume { // 在源中打开通道
        for (x in this) { // 迭代通道以从中接收元素
            println(x)
            if (++cnt >= 3) break // 当三个元素被打印出来的时候，执行 break
        }
        // 注意：当这段代码执行完成并阻塞的时候 `consume` 取消了该通道
    }
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-03.kt)获得完整代码

它将产生如下输出：
 
```text
OnSubscribe
1
2
3
Finally
```

<!--- TEST -->
 
使用显示的 `openSubscription` 时我们应该使用 [cancel][ReceiveChannel.cancel]
来取消订阅响应的订阅源。这里不需要显示的调用 `cancel`——因为
`consume` 为我们做了这些事。
添加
[doFinally](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#doFinally(io.reactivex.functions.Action))
监听器并打印“Finally”来确认订阅确实被取消了。注意“OnComplete”
永远不会被打印因为我们没有消费所有的元素。

如果对发布者发送出的所有元素执行迭代，
则我们不需要显示的 `cancel`，因为它被 `consumeEach` 自动取消了：

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val source = Flowable.range(1, 5) // 五个数字的区间
        .doOnSubscribe { println("OnSubscribe") } // 提供了一些可被观察的点
        .doOnComplete { println("OnComplete") }   // ...
        .doFinally { println("Finally") }         // ... 在正在执行的代码中
    // iterate over the source fully
    source.consumeEach { println(it) }
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-04.kt)获得完整代码

我们得到如下输出：

```text
OnSubscribe
1
2
3
4
OnComplete
Finally
5
```

<!--- TEST -->

注意，如何使“OnComplete”与“Finally”在最后一个元素“5”之前被打印。在这个示例中它将发生在我们的 `main`
函数在协程中执行时，使用 [runBlocking] 协程构建器来启动它。
我们的主协程在通道中使用 `source.consumeEach { ... }` 扩展函数来接收通道。
当它等待源发射元素的时候该主协程是 _挂起的_ ，
当最后一个元素被 `Flowable.range(1, 5)` 发射时它
_恢复_ 了主协程，它被分派到主线程上打印出来
最后一个元素在稍后的时间点打印，而 source 执行完成并打印“Finally”。

### 背压

Backpressure is one of the most interesting and complex aspects of reactive streams. Coroutines can 
_suspend_ and they provide a natural answer to handling backpressure. 

In Rx Java 2.x a backpressure-capable class is called 
[Flowable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html).
In the following example we use [rxFlowable] coroutine builder from `kotlinx-coroutines-rx2` module to define a 
flowable that sends three integers from 1 to 3. 
It prints a message to the output before invocation of
suspending [send][SendChannel.send] function, so that we can study how it operates.

The integers are generated in the context of the main thread, but subscription is shifted 
to another thread using Rx
[observeOn](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler,%20boolean,%20int))
operator with a buffer of size 1. 
The subscriber is slow. It takes 500 ms to process each item, which is simulated using `Thread.sleep`.

<!--- INCLUDE
import io.reactivex.schedulers.*
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> { 
    // coroutine -- fast producer of elements in the context of the main thread
    val source = rxFlowable {
        for (x in 1..3) {
            send(x) // this is a suspending function
            println("Sent $x") // print after successfully sent item
        }
    }
    // subscribe on another thread with a slow subscriber using Rx
    source
        .observeOn(Schedulers.io(), false, 1) // specify buffer size of 1 item
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(500) // 500ms to process each item
            println("Processed $x")
        }
    delay(2000) // suspend the main thread for a few seconds
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-05.kt)获得完整代码

The output of this code nicely illustrates how backpressure works with coroutines:

```text
Sent 1
Processed 1
Sent 2
Processed 2
Sent 3
Processed 3
Complete
```

<!--- TEST -->

We see here how producer coroutine puts the first element in the buffer and is suspended while trying to send another 
one. Only after consumer processes the first item, producer sends the second one and resumes, etc.


### Rx 主题 vs 广播通道
 
RxJava has a concept of [Subject](https://github.com/ReactiveX/RxJava/wiki/Subject) which is an object that
effectively broadcasts elements to all its subscribers. The matching concept in coroutines world is called a 
[BroadcastChannel]. There is a variety of subjects in Rx with 
[BehaviorSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/BehaviorSubject.html) being
the one used to manage state:

<!--- INCLUDE
import io.reactivex.subjects.BehaviorSubject
-->

```kotlin
fun main() {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two") // updates the state of BehaviorSubject, "one" value is lost
    // now subscribe to this subject and print everything
    subject.subscribe(System.out::println)
    subject.onNext("three")
    subject.onNext("four")
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-06.kt)

This code prints the current state of the subject on subscription and all its further updates:


```text
two
three
four
```

<!--- TEST -->

You can subscribe to subjects from a coroutine just as with any other reactive stream:
   
<!--- INCLUDE 
import io.reactivex.subjects.BehaviorSubject
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.consumeEach
-->   
   
```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // now launch a coroutine to print everything
    GlobalScope.launch(Dispatchers.Unconfined) { // launch coroutine in unconfined context
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
}
```   

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-07.kt)

The result is the same:

```text
two
three
four
```

<!--- TEST -->

Here we use [Dispatchers.Unconfined] coroutine context to launch consuming coroutine with the same behaviour as subscription in Rx. 
It basically means that the launched coroutine is going to be immediately executed in the same thread that 
is emitting elements. Contexts are covered in more details in a [separate section](#coroutine-context).

The advantage of coroutines is that it is easy to get conflation behavior for single-threaded UI updates. 
A typical UI application does not need to react to every state change. Only the most recent state is relevant.
A sequence of back-to-back updates to the application state needs to get reflected in UI only once, 
as soon as the UI thread is free. For the following example we are going to simulate this by launching 
consuming coroutine in the context of the main thread and use [yield] function to simulate a break in the 
sequence of updates and to release the main thread:

<!--- INCLUDE
import io.reactivex.subjects.*
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two")
    // now launch a coroutine to print the most recent update
    launch { // use the context of the main thread for a coroutine
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
    yield() // yield the main thread to the launched coroutine <--- HERE
    subject.onComplete() // now complete subject's sequence to cancel consumer, too    
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)

Now coroutine process (prints) only the most recent update:

```text
four
```

<!--- TEST -->

The corresponding behavior in a pure coroutines world is implemented by [ConflatedBroadcastChannel] 
that provides the same logic on top of coroutine channels directly, 
without going through the bridge to the reactive streams:

<!--- INCLUDE
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    val broadcast = ConflatedBroadcastChannel<String>()
    broadcast.offer("one")
    broadcast.offer("two")
    // now launch a coroutine to print the most recent update
    launch { // use the context of the main thread for a coroutine
        broadcast.consumeEach { println(it) }
    }
    broadcast.offer("three")
    broadcast.offer("four")
    yield() // yield the main thread to the launched coroutine
    broadcast.close() // now close broadcast channel to cancel consumer, too    
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-09.kt)

It produces the same output as the previous example based on `BehaviorSubject`:

```text
four
```

<!--- TEST -->

Another implementation of [BroadcastChannel] is `ArrayBroadcastChannel` with an array-based buffer of
a specified `capacity`. It can be created with `BroadcastChannel(capacity)`. 
It delivers every event to every
subscriber since the moment the corresponding subscription is open. It corresponds to 
[PublishSubject](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/PublishSubject.html) in Rx.
The capacity of the buffer in the constructor of `ArrayBroadcastChannel` controls the numbers of elements
that can be sent before the sender is suspended waiting for receiver to receive those elements.

## Operators

Full-featured reactive stream libraries, like Rx, come with 
[a very large set of operators](http://reactivex.io/documentation/operators.html) to create, transform, combine
and otherwise process the corresponding streams. Creating your own operators with support for
back-pressure is [notoriously](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)
[difficult](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0).

Coroutines and channels are designed to provide an opposite experience. There are no built-in operators, 
but processing streams of elements is extremely simple and back-pressure is supported automatically 
without you having to explicitly think about it.

This section shows coroutine-based implementation of several reactive stream operators.  

### Range

Let's roll out own implementation of 
[range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int))
operator for reactive streams `Publisher` interface. The asynchronous clean-slate implementation of this operator for
reactive streams is explained in 
[this blog post](http://akarnokd.blogspot.ru/2017/03/java-9-flow-api-asynchronous-integer.html).
It takes a lot of code.
Here is the corresponding code with coroutines:

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun CoroutineScope.range(context: CoroutineContext, start: Int, count: Int) = publish<Int>(context) {
    for (x in start until start + count) send(x)
}
```

In this code `CoroutineScope` and `context` are used instead of an `Executor` and all the backpressure aspects are taken care
of by the coroutines machinery. Note, that this implementation depends only on the small reactive streams library
that defines `Publisher` interface and its friends.

It is straightforward to use from a coroutine:

```kotlin
fun main() = runBlocking<Unit> {
    // Range inherits parent job from runBlocking, but overrides dispatcher with Dispatchers.Default
    range(Dispatchers.Default, 1, 5).consumeEach { println(it) }
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-01.kt)

The result of this code is quite expected:
   
```text
1
2
3
4
5
```

<!--- TEST -->

### Fused filter-map hybrid

Reactive operators like 
[filter](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#filter(io.reactivex.functions.Predicate)) and 
[map](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#map(io.reactivex.functions.Function))
are trivial to implement with coroutines. For a bit of challenge and showcase, let us combine them
into the single `fusedFilterMap` operator: 

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T, R> Publisher<T>.fusedFilterMap(
    context: CoroutineContext,   // the context to execute this coroutine in
    predicate: (T) -> Boolean,   // the filter predicate
    mapper: (T) -> R             // the mapper function
) = GlobalScope.publish<R>(context) {
    consumeEach {                // consume the source stream 
        if (predicate(it))       // filter part
            send(mapper(it))     // map part
    }        
}
```

Using `range` from the previous example we can test our `fusedFilterMap` 
by filtering for even numbers and mapping them to strings:

<!--- INCLUDE

fun CoroutineScope.range(start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) send(x)
}
-->

```kotlin
fun main() = runBlocking<Unit> {
   range(1, 5)
       .fusedFilterMap(coroutineContext, { it % 2 == 0}, { "$it is even" })
       .consumeEach { println(it) } // print all the resulting strings
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-02.kt)

It is not hard to see, that the result is going to be:

```text
2 is even
4 is even
```

<!--- TEST -->

### Take until

Let's implement our own version of
[takeUntil](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#takeUntil(org.reactivestreams.Publisher))
operator. It is quite a [tricky one](http://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html) 
to implement, because of the need to track and manage subscription to two streams. 
We need to relay all the elements from the source stream until the other stream either completes or 
emits anything. However, we have [select] expression to rescue us in coroutines implementation:

<!--- INCLUDE
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlinx.coroutines.selects.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T, U> Publisher<T>.takeUntil(context: CoroutineContext, other: Publisher<U>) = GlobalScope.publish<T>(context) {
    this@takeUntil.openSubscription().consume { // explicitly open channel to Publisher<T>
        val current = this
        other.openSubscription().consume { // explicitly open channel to Publisher<U>
            val other = this
            whileSelect {
                other.onReceive { false }          // bail out on any received element from `other`
                current.onReceive { send(it); true }  // resend element from this channel and continue
            }
        }
    }
}
```

This code is using [whileSelect] as a nicer shortcut to `while(select{...}) {}` loop and Kotlin's
[use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 
expression to close the channels on exit, which unsubscribes from the corresponding publishers. 

The following hand-written combination of 
[range](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) with 
[interval](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#interval(long,%20java.util.concurrent.TimeUnit,%20io.reactivex.Scheduler))
is used for testing. It is coded using a `publish` coroutine builder 
(its pure-Rx implementation is shown in later sections):

```kotlin
fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}
```

The following code shows how `takeUntil` works: 

```kotlin
fun main() = runBlocking<Unit> {
    val slowNums = rangeWithInterval(200, 1, 10)         // numbers with 200ms interval
    val stop = rangeWithInterval(500, 1, 10)             // the first one after 500ms
    slowNums.takeUntil(coroutineContext, stop).consumeEach { println(it) } // let's test it
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-03.kt)

Producing 

```text
1
2
```

<!--- TEST -->

### Merge

There are always at least two ways for processing multiple streams of data with coroutines. One way involving
[select] was shown in the previous example. The other way is just to launch multiple coroutines. Let
us implement 
[merge](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#merge(org.reactivestreams.Publisher))
operator using the later approach:

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T> Publisher<Publisher<T>>.merge(context: CoroutineContext) = GlobalScope.publish<T>(context) {
  consumeEach { pub ->                 // for each publisher received on the source channel
      launch {  // launch a child coroutine
          pub.consumeEach { send(it) } // resend all element from this publisher
      }
  }
}
```

Notice, the use of 
[coroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html)
in the invocation of [launch] coroutine builder. It is used to refer
to the context of the enclosing `publish` coroutine. This way, all the coroutines that are
being launched here are [children](../docs/coroutines-guide.md#children-of-a-coroutine) of the `publish`
coroutine and will get cancelled when the `publish` coroutine is cancelled or is otherwise completed. 
Moreover, since parent coroutine waits until all children are complete, this implementation fully
merges all the received streams.

For a test, let us start with `rangeWithInterval` function from the previous example and write a 
producer that sends its results twice with some delay:

<!--- INCLUDE

fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}
-->

```kotlin
fun CoroutineScope.testPub() = publish<Publisher<Int>> {
    send(rangeWithInterval(250, 1, 4)) // number 1 at 250ms, 2 at 500ms, 3 at 750ms, 4 at 1000ms 
    delay(100) // wait for 100 ms
    send(rangeWithInterval(500, 11, 3)) // number 11 at 600ms, 12 at 1100ms, 13 at 1600ms
    delay(1100) // wait for 1.1s - done in 1.2 sec after start
}
```

The test code is to use `merge` on `testPub` and to display the results:

```kotlin
fun main() = runBlocking<Unit> {
    testPub().merge(coroutineContext).consumeEach { println(it) } // print the whole stream
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-04.kt)

And the results should be: 

```text
1
2
11
3
4
12
13
```

<!--- TEST -->

## Coroutine context

All the example operators that are shown in the previous section have an explicit
[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
parameter. In Rx world it roughly corresponds to 
a [Scheduler](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Scheduler.html).

### Threads with Rx

The following example shows the basics of threading context management with Rx.
Here `rangeWithIntervalRx` is an implementation of `rangeWithInterval` function using Rx 
`zip`, `range`, and `interval` operators.

<!--- INCLUDE
import io.reactivex.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> = 
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-01.kt)

We are explicitly passing the 
[Schedulers.computation()](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#computation()) 
scheduler to our `rangeWithIntervalRx` operator and
it is going to be executed in Rx computation thread pool. The output is going to be similar to the following one:

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### Threads with coroutines

In the world of coroutines `Schedulers.computation()` roughly corresponds to [Dispatchers.Default], 
so the previous example is similar to the following one:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = GlobalScope.publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-02.kt)

The produced output is going to be similar to:

```text
1 on thread ForkJoinPool.commonPool-worker-1
2 on thread ForkJoinPool.commonPool-worker-1
3 on thread ForkJoinPool.commonPool-worker-1
```

<!--- TEST LINES_START -->

Here we've used Rx 
[subscribe](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#subscribe(io.reactivex.functions.Consumer))
operator that does not have its own scheduler and operates on the same thread that the publisher -- on a default
shared pool of threads in this example.

### Rx observeOn 

In Rx you use special operators to modify the threading context for operations in the chain. You
can find some [good guides](http://tomstechnicalblog.blogspot.ru/2016/02/rxjava-understanding-observeon-and.html)
about them, if you are not familiar. 

For example, there is
[observeOn](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler)) 
operator. Let us modify the previous example to observe using `Schedulers.computation()`:   

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.schedulers.Schedulers
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = GlobalScope.publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // wait before sending each number
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .observeOn(Schedulers.computation())                           // <-- THIS LINE IS ADDED
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-03.kt)

Here is the difference in output, notice "RxComputationThreadPool":

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### Coroutine context to rule them all

A coroutine is always working in some context. For example, let us start a coroutine
in the main thread with [runBlocking] and iterate over the result of the Rx version of `rangeWithIntervalRx` operator, 
instead of using Rx `subscribe` operator:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
        .consumeEach { println("$it on thread ${Thread.currentThread().name}") }
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-04.kt)

The resulting messages are going to be printed in the main thread:

```text
1 on thread main
2 on thread main
3 on thread main
```

<!--- TEST LINES_START -->

### Unconfined context

Most Rx operators do not have any specific thread (scheduler) associated with them and are working 
in whatever thread that they happen to be invoked in. We've seen it on the example of `subscribe` operator 
in the [threads with Rx](#threads-with-rx) section.
 
In the world of coroutines, [Dispatchers.Unconfined] context serves a similar role. Let us modify our previous example,
but instead of iterating over the source `Flowable` from the `runBlocking` coroutine that is confined 
to the main thread, we launch a new coroutine in `Dispatchers.Unconfined` context, while the main coroutine
simply waits its completion using [Job.join]:

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import io.reactivex.functions.BiFunction
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit
-->

```kotlin
fun rangeWithIntervalRx(scheduler: Scheduler, time: Long, start: Int, count: Int): Flowable<Int> =
    Flowable.zip(
        Flowable.range(start, count),
        Flowable.interval(time, TimeUnit.MILLISECONDS, scheduler),
        BiFunction { x, _ -> x })

fun main() = runBlocking<Unit> {
    val job = launch(Dispatchers.Unconfined) { // launch new coroutine in Unconfined context (without its own thread pool)
        rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
            .consumeEach { println("$it on thread ${Thread.currentThread().name}") }
    }
    job.join() // wait for our coroutine to complete
}
```

> You can get full code [here](kotlinx-coroutines-rx2/test/guide/example-reactive-context-05.kt)

Now, the output shows that the code of the coroutine is executing in the Rx computation thread pool, just
like our initial example using Rx `subscribe` operator.

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST LINES_START -->

Note, that [Dispatchers.Unconfined] context shall be used with care. It may improve the overall performance on certain tests,
due to the increased stack-locality of operations and less scheduling overhead, but it also produces deeper stacks 
and makes it harder to reason about asynchronicity of the code that is using it. 

If a coroutine sends an element to a channel, then the thread that invoked the 
[send][SendChannel.send] may start executing the code of a coroutine with [Dispatchers.Unconfined] dispatcher.
The original producer coroutine that invoked `send`  is paused until the unconfined consumer coroutine hits its next
suspension point. This is very similar to a lock-step single-threaded `onNext` execution in Rx world in the absense
of thread-shifting operators. It is a normal default for Rx, because operators are usually doing very small chunks
of work and you have to combine many operators for a complex processing. However, this is unusual with coroutines, 
where you can have an arbitrary complex processing in a coroutine. Usually, you only need to chain stream-processing
coroutines for complex pipelines with fan-in and fan-out between multiple worker coroutines.

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[ReceiveChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/index.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[BroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-broadcast-channel/index.html
[ConflatedBroadcastChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-conflated-broadcast-channel/index.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
[whileSelect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/while-select.html
<!--- MODULE kotlinx-coroutines-reactive -->
<!--- INDEX kotlinx.coroutines.reactive -->
[publish]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/kotlinx.coroutines.-coroutine-scope/publish.html
[org.reactivestreams.Publisher.consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/consume-each.html
[org.reactivestreams.Publisher.openSubscription]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-reactive/kotlinx.coroutines.reactive/org.reactivestreams.-publisher/open-subscription.html
<!--- MODULE kotlinx-coroutines-rx2 -->
<!--- INDEX kotlinx.coroutines.rx2 -->
[rxFlowable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-rx2/kotlinx.coroutines.rx2/kotlinx.coroutines.-coroutine-scope/rx-flowable.html
<!--- END -->


