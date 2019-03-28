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
-->如何将它们更好的一起使用。在此之前熟悉包含在<!--
-->[协程指南](../docs/coroutines-guide.md)中的基础协程概念不是必须的，
但如果熟悉它将会是个很大的加分。如果你熟悉响应式流，你可能发现本指南会<!--
-->更好地介绍协程的世界。

在 `kotlinx.coroutines` 项目中有一系列和响应式流相关的模块：

* [kotlinx-coroutines-reactive](kotlinx-coroutines-reactive) ——为 [Reactive Streams](https://www.reactive-streams.org) 提供的适配
* [kotlinx-coroutines-reactor](kotlinx-coroutines-reactor) ——为 [Reactor](https://projectreactor.io) 提供的适配
* [kotlinx-coroutines-rx2](kotlinx-coroutines-rx2) ——为 [RxJava 2.x](https://github.com/ReactiveX/RxJava) 提供的适配

本指南主要基于 [Reactive Streams](https://www.reactive-streams.org) 的规范并使用
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

* [响应式流与通道的区别](#响应式流与通道的区别)
  * [迭代的基础](#迭代的基础)
  * [订阅与取消](#订阅与取消)
  * [背压](#背压)
  * [Rx 主题 vs 广播通道](#rx-主题-vs-广播通道)
* [操作符](#操作符)
  * [Range](#range)
  * [熔合 filter 与 map 操作符](#熔合-filter-与-map-操作符)
  * [Take until](#take-until)
  * [Merge](#merge)
* [协程上下文](#协程上下文)
  * [线程与 Rx](#线程与-rx)
  * [线程与协程](#线程与协程)
  * [Rx observeOn](#rx-observeon)
  * [使用协程上下文来管理它们](#使用协程上下文来管理它们)
  * [不受限的上下文](#不受限的上下文)

<!--- END_TOC -->

## 响应式流与通道的区别

本节主要包含响应式流与以协程为基础的通道的不同点。

### 迭代的基础

[Channel] 与如下所示的响应式流类有类似的概念：

* Reactive stream [Publisher](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/api/src/main/java/org/reactivestreams/Publisher.java)；
* Rx Java 1.x [Observable](https://reactivex.io/RxJava/javadoc/rx/Observable.html)；
* Rx Java 2.x [Flowable](https://reactivex.io/RxJava/2.x/javadoc/)，`Publisher` 的实现者。

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
[Publisher](https://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html)
类型。

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> {
    // 创建一个 publisher，每 200 毫秒生产一个数字，从 1 到 3
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
查看[这个提案](https://github.com/Kotlin/kotlinx.coroutines/issues/167)<!--
-->的细节。

> 注意，我们可以使用 Rx 中的
[publish](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#publish())
操作符与 [connect](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/flowables/ConnectableFlowable.html#connect())
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
[doFinally](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#doFinally(io.reactivex.functions.Action))
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

背压是响应式流中最有趣最复杂的概念之一。协程可以<!--
-->_挂起_ 并且它提供了一个自然的解决方式来处理背压。

在 Rx Java 2.x 中一个支持背压的类被称为
[Flowable](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html)。
在下面的示例中我们可以使用 `kotlinx-coroutines-rx2` 模块中的协程构建器 [rxFlowable] 来定义一个
发送从 1 到 3 三个整数的 flowable。
在调用挂起的 [send][SendChannel.send] 函数之前，
它在输出中打印了一条消息，所以我们可以来研究它是如何操作的。

这些整数在主线程的上下文中被产生，
但是在使用 Rx 的
[observeOn](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler,%20boolean,%20int))
操作符后缓冲区大小为 1 的订阅被转移到了另一个线程。
为了模拟订阅者很慢，它使用了 `Thread.sleep` 来模拟消耗 500 毫秒来处理每个元素。

<!--- INCLUDE
import io.reactivex.schedulers.*
import kotlinx.coroutines.*
import kotlinx.coroutines.rx2.*
import kotlin.coroutines.*
-->

```kotlin
fun main() = runBlocking<Unit> { 
    // 协程 —— 在主线程的上下文中快速生成元素
    val source = rxFlowable {
        for (x in 1..3) {
            send(x) // 这是一个挂起函数
            println("Sent $x") // 在成功发送元素后打印
        }
    }
    // 使用 Rx 让一个处理速度很慢的订阅者在另一个线程订阅
    source
        .observeOn(Schedulers.io(), false, 1) // 指定缓冲区大小为 1 个元素
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(500) // 处理每个元素消耗 500 毫秒
            println("Processed $x")
        }
    delay(2000) // 挂起主线程几秒钟
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-05.kt)获得完整代码

这段代码的输出更好地说明了背压是如何在协程中工作的：

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

当尝试发送另一个元素的时候，我们看到这里的处理者协程是如何将第一个元素放入缓冲区并挂起的。
只有当消费者处理了第一个元素，处理者才会发送第二个元素并恢复，等等。


### Rx 主题 vs 广播通道
 
RxJava 有一个 [主题（Subject）](https://github.com/ReactiveX/RxJava/wiki/Subject) 的概念：一个对象可以有效地向所有<!--
-->订阅者广播元素。与此相匹配的概念在协程的世界中被称为
[BroadcastChannel]。在 Rx 中有一种主题——
[BehaviorSubject](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/BehaviorSubject.html)
被用来管理状态：

<!--- INCLUDE
import io.reactivex.subjects.BehaviorSubject
-->

```kotlin
fun main() {
    val subject = BehaviorSubject.create<String>()
    subject.onNext("one")
    subject.onNext("two") // 更新 BehaviorSubject 的状态，“one”变量被丢弃
    // 现在订阅这个主题并打印所有信息
    subject.subscribe(System.out::println)
    subject.onNext("three")
    subject.onNext("four")
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-06.kt)获得完整代码

这段代码打印订阅时主题的当前状态及其所有后续更新：


```text
two
three
four
```

<!--- TEST -->

您可以像使用任何其他响应式流一样从协程订阅主题：
   
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
    // 现在启动一个协程来打印所有东西
    GlobalScope.launch(Dispatchers.Unconfined) { // 在不受限的上下文中启动协程
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
}
```   

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-07.kt)获得完整代码

结果是相同的：

```text
two
three
four
```

<!--- TEST -->

这里我们使用 [Dispatchers.Unconfined] 协程上下文以与 Rx 中的订阅相同的行为启动消费协程。
它基本上意味着启动的协程将立即在同一个线程中执行<!--
-->并发射元素。上下文的更多细节被包含在[单独的小节](#coroutine-context)。

协程的优点是很容易获得单线程 UI 更新的混合行为。
一个典型的 UI 应用程序不需要响应每一个状态改变。只有最近的状态需要被响应。
应用程序状态的一系列背靠背更新只需在UI中反映一次，
尽可能保证 UI 线程是空闲的。在以下的示例中我们将通过模拟在主线程上下文中<!--
-->启动消费者协程并使用 [yield] 函数来模拟中断更新序列<!--
-->并释放主线程：

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
    // 现在启动一个协程来打印最近的更新
    launch { // 为协程使用主线程的上下文
        subject.consumeEach { println(it) }
    }
    subject.onNext("three")
    subject.onNext("four")
    yield() // 使主线程让步来启动协程 <--- 这里
    subject.onComplete() // 现在也结束主题的序列来取消消费者
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-08.kt)获得完整代码

现在协程只处理（打印）最近的更新：

```text
four
```

<!--- TEST -->

纯协程世界中的相应行为由 [ConflatedBroadcastChannel] 实现<!--
-->在协程通道上直接提供相同的逻辑，
没有桥接到响应式流：

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
    // 现在启动一个协程来打印最近的更新
    launch { // 为协程使用主线程的上下文
        broadcast.consumeEach { println(it) }
    }
    broadcast.offer("three")
    broadcast.offer("four")
    yield() // 使主线程让步来启动协程
    broadcast.close() // 现在也结束主题的序列来取消消费者
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-basic-09.kt)获得完整代码

它与基于 `BehaviorSubject` 的先前的示例产生了相同的输出：

```text
four
```

<!--- TEST -->

另一个 [BroadcastChannel] 的实现是 `ArrayBroadcastChannel` ——一个使用数组的缓冲区<!--
-->来规定 `capacity`。它可以使用 `BroadcastChannel(capacity)` 来启动。
它为每一个订阅者<!--
-->自相应订阅开放之时起提供每一个事件。它对应于 Rx 中的
[PublishSubject](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/subjects/PublishSubject.html)。
`ArrayBroadcastChannel` 构造函数中的缓冲区的 capacity 参数控制<!--
-->发送者在挂起等待接收者接收这些元素之前的元素的数量。

## 操作符

全功能的响应式流库，比如 Rx，都伴随着<!--
-->[非常大量的操作符](https://reactivex.io/documentation/operators.html)用于创建、变换、合并<!--
-->以及反转来处理相关的流。创建你自己的并且支持背压的<!--
-->操作符是非常[臭名昭著](https://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)以及<!--
-->[困难](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0)的。

协程与通道则被设计为提供完全相反的体验。这里没有内建的操作符，
但是处理元素流是非常简单并且自动支持背压的，
即使是在你没有明确思考这一点的情况下。

本节将展示以协程为基础而实现的一系列响应式流操作符。

### Range

让我们推出自己的为响应式流 `Publisher` 接口实现的
[range](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int))
操作符。为响应式流提供的本操作符从零开始的异步实现<!--
-->被包含在<!--
-->[这篇博客](https://akarnokd.blogspot.ru/2017/03/java-9-flow-api-asynchronous-integer.html)中。
它需要很多代码。
以下是与协同程序相对应的代码：

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

在这段代码中 `CoroutineScope` 与 `context` 被用来替代一个 `Executor` 并且所有的背压方面都被小心的<!--
-->用于协程机制。注意，此实现仅依赖于那些定义了 `Publisher` 接口和它的朋友们<!--
-->的小型响应式流库。

它可以直接在协程中被使用：

```kotlin
fun main() = runBlocking<Unit> {
    // Range 从 runBlocking 中承袭了父 job，但是使用 Dispatchers.Default 来覆盖调度器
    range(Dispatchers.Default, 1, 5).consumeEach { println(it) }
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-01.kt)获得完整代码

这段代码的结果非常值得我们期待：
   
```text
1
2
3
4
5
```

<!--- TEST -->

### 熔合 filter 与 map 操作符

响应式操作符比如：
[filter](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#filter(io.reactivex.functions.Predicate)) 以及
[map](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#map(io.reactivex.functions.Function))
使用协程实现是非常琐碎的。对于一些挑战和展示，让我们将它们合并<!--
-->到单个的 `fusedFilterMap` 操作符中：

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T, R> Publisher<T>.fusedFilterMap(
    context: CoroutineContext,   // 协程执行的上下文
    predicate: (T) -> Boolean,   // 过滤器 predicate
    mapper: (T) -> R             // mapper 函数
) = GlobalScope.publish<R>(context) {
    consumeEach {                // 消费源流
        if (predicate(it))       // 过滤的部分
            send(mapper(it))     // 变换的部分
    }        
}
```

使用先前 `range` 中的示例我们可以测试我们的 `fusedFilterMap`
来过滤偶数以及将它们映射到字符串：

<!--- INCLUDE

fun CoroutineScope.range(start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) send(x)
}
-->

```kotlin
fun main() = runBlocking<Unit> {
   range(1, 5)
       .fusedFilterMap(coroutineContext, { it % 2 == 0}, { "$it is even" })
       .consumeEach { println(it) } // 打印所有的字符串结果
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-02.kt)获得完整代码

不难看出，结果将是：

```text
2 is even
4 is even
```

<!--- TEST -->

### Take until

让我们为
[takeUntil](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#takeUntil(org.reactivestreams.Publisher))
操作符实现自己的版本。它是非常[难于](https://akarnokd.blogspot.ru/2015/05/pitfalls-of-operator-implementations.html)<!--
-->去实现的，因为需要跟踪和管理两个流的订阅。
我们需要以来源流中的所有元素直到另一个流也执行完成或<!--
-->发射了任何东西。然而，我们有 [select] 表达式可以在协程的实现中拯救我们：

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
    this@takeUntil.openSubscription().consume { // 显式地打开 Publisher<T> 的通道
        val current = this
        other.openSubscription().consume { // 显式地打开 Publisher<U> 的通道
            val other = this
            whileSelect {
                other.onReceive { false }          // 释放任何从 `other` 接收到的元素
                current.onReceive { send(it); true }  // 在这个通道上重新发送元素并继续
            }
        }
    }
}
```

这段代码使用 [whileSelect] 作为比 `while(select{...}) {}` 循环更好的快捷方式，并且 Kotlin 的
[use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 
表达式会在退出时关闭通道，并取消订阅相应的发布者。

在下面手写的
[range](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#range(int,%20int)) 与
[interval](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#interval(long,%20java.util.concurrent.TimeUnit,%20io.reactivex.Scheduler))
的组合被用来测试。它在编码中使用 `publish` 协程构建器
（在下一小节中它将是纯 Rx 实现的）：

```kotlin
fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // 在每次发送数字之前等待
        send(x)
    }
}
```

下面的代码展示了 `takeUntil` 是如何工作的：

```kotlin
fun main() = runBlocking<Unit> {
    val slowNums = rangeWithInterval(200, 1, 10)         // 数字之间有 200 毫秒的间隔
    val stop = rangeWithInterval(500, 1, 10)             // 第一个在 500 毫秒之后
    slowNums.takeUntil(coroutineContext, stop).consumeEach { println(it) } // 让我们测试它
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-03.kt)获得完整代码

执行

```text
1
2
```

<!--- TEST -->

### Merge

使用协程处理多个数据流总是至少有两种方法。一种方法是调用
[select]，这被展示在先前的示例中。另一种方法是只是启动过个协程。让<!--
-->我们使用
[merge](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#merge(org.reactivestreams.Publisher))
操作符来使用第二种的方法：

<!--- INCLUDE
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import org.reactivestreams.*
import kotlin.coroutines.*
-->

```kotlin
fun <T> Publisher<Publisher<T>>.merge(context: CoroutineContext) = GlobalScope.publish<T>(context) {
  consumeEach { pub ->                 // 为每一个 publisher 在源通道上接收元素
      launch {  // 启动一个子协程
          pub.consumeEach { send(it) } // 从这个 publisher 上重新发送所有元素
      }
  }
}
```

注意，
[coroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html)
在调用 [launch] 协程构建器中的用途。它被用来指定
`publish` 协程的上下文。这种方法，所有被启动的协程<!--
-->在这里都是 `publish` 协程的[子协程](../docs/coroutines-guide.md#children-of-a-coroutine)<!--
-->并且当 `publish` 协程被取消或以其它的方式执行完毕时将会被取消。
此外，父协程会等待至所有子协程执行完毕没，这个实现会完全<!--
-->合并所有的响应式流。

对于测试，让我们在先前的示例中使用 `rangeWithInterval` 函数来启动并编写一个<!--
-->生产者在一段时间的延时后发送结果两次：

<!--- INCLUDE

fun CoroutineScope.rangeWithInterval(time: Long, start: Int, count: Int) = publish<Int> {
    for (x in start until start + count) { 
        delay(time) // 在每次发送数字之前等待
        send(x)
    }
}
-->

```kotlin
fun CoroutineScope.testPub() = publish<Publisher<Int>> {
    send(rangeWithInterval(250, 1, 4)) // 数字 1 在 250 毫秒发射，2 在 500 毫秒，3 在 750 毫秒，4 在 1000 毫秒
    delay(100) // 等待 100 毫秒
    send(rangeWithInterval(500, 11, 3)) // 数字 11 在 600 毫秒，12 在 1100 毫秒，13 在 1600 毫秒
    delay(1100) // 在启动完成后的 1.2 秒之后等待 1.1 秒
}
```

这段测试代码在 `testPub` 上使用了 `merge` 并且展示结果：

```kotlin
fun main() = runBlocking<Unit> {
    testPub().merge(coroutineContext).consumeEach { println(it) } // 打印整个流
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-operators-04.kt)获得完整代码

并且结果应该是：

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

## 协程上下文

所有的示例操作符都在先前的示例中显式地设置了
[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
参数。在 Rx 的世界中它大概对应于<!--
-->一个 [Scheduler](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Scheduler.html)。

### 线程与 Rx

下面的示例中展示了基本的在 Rx 中管理线程上下文。
这里的 `rangeWithIntervalRx` 是`rangeWithInterval` 函数使用 Rx 的
`zip`、`range` 以及 `interval` 操作符的一个实现。

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

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-context-01.kt)获得完整代码

我们显式地通过
[Schedulers.computation()](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/schedulers/Schedulers.html#computation())
调度器，并将它用于 `rangeWithIntervalRx` 操作符，所以<!--
-->它将执行在 Rx 的计算线程池。输出将类似于以下内容：

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### 线程与协程

在协程的世界中 `Schedulers.computation()` 大致对应于 [Dispatchers.Default]，
所以先前的示例将变成下面这样：

<!--- INCLUDE
import io.reactivex.*
import kotlinx.coroutines.*
import kotlinx.coroutines.reactive.*
import kotlin.coroutines.CoroutineContext
-->

```kotlin
fun rangeWithInterval(context: CoroutineContext, time: Long, start: Int, count: Int) = GlobalScope.publish<Int>(context) {
    for (x in start until start + count) { 
        delay(time) // 在每次数字发射前等待
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-context-02.kt)获得完整代码

产生的输出将类似于：

```text
1 on thread ForkJoinPool.commonPool-worker-1
2 on thread ForkJoinPool.commonPool-worker-1
3 on thread ForkJoinPool.commonPool-worker-1
```

<!--- TEST LINES_START -->

这里我们使用了 Rx 的
[subscribe](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#subscribe(io.reactivex.functions.Consumer))
操作符，没有自己的调度器和操作符并且运行在同一个线程上，而发布者在本示例中<!--
-->运行在共享的线程池上。

### Rx observeOn 

在 Rx 中你操作使用了特别的操作符来为调用链修改线程上下文。
如果你不熟悉它的话，
你可以从这篇[很棒的教程](https://tomstechnicalblog.blogspot.ru/2016/02/rxjava-understanding-observeon-and.html)中获得指导。

举例来说，这里使用了
[observeOn](https://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#observeOn(io.reactivex.Scheduler))
操作符。让我们修改先前的示例并观察使用 `Schedulers.computation()` 的效果：

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
        delay(time) // 在每次数字发射前等待
        send(x)
    }
}

fun main() {
    Flowable.fromPublisher(rangeWithInterval(Dispatchers.Default, 100, 1, 3))
        .observeOn(Schedulers.computation())                           // <-- 添加了这一行
        .subscribe { println("$it on thread ${Thread.currentThread().name}") }
    Thread.sleep(1000)
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-context-03.kt)获得完整代码

这里的输出有所不同了，提示了“RxComputationThreadPool”：

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST FLEXIBLE_THREAD -->

### 使用协程上下文来管理它们

一个协程总是运行在一些上下文中。举例来说，让我们<!--
-->使用 [runBlocking] 在主线程中启动一个协程，并且使用 Rx 版本的 `rangeWithIntervalRx` 操作符，
替代使用 Rx 的 `subscribe` 操作符遍历结果：

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

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-context-04.kt)获得完整代码

结果信息将会被打印在主线程中：

```text
1 on thread main
2 on thread main
3 on thread main
```

<!--- TEST LINES_START -->

### 不受限的上下文

大多数 Rx 操作符都没有特别地指定相关线程（调度器）并且正在运行<!--
-->在它们恰好被调用的任何线程中。我们在[线程与 Rx](#threads-with-rx) 这一小节中<!--
-->看到了 `subscribe` 运算符的示例。
 
在协程的世界中，[Dispatchers.Unconfined] 则承担了类似的任务。让我们修改先前的示例，
但是将源 `Flowable` 的遍历替换到 `runBlocking` 协程中后，它被限制<!--
-->在了主线程中，我们在 `Dispatchers.Unconfined` 上下文中启动了一个新协程，当主协程<!--
-->只是等待的时候使用 [Job.join]：

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
    val job = launch(Dispatchers.Unconfined) { // 在不受限的山下文中启动一个新协程（没有它自己的线程池）
        rangeWithIntervalRx(Schedulers.computation(), 100, 1, 3)
            .consumeEach { println("$it on thread ${Thread.currentThread().name}") }
    }
    job.join() // 等待我们的协程结束
}
```

> 你可以从[这里](kotlinx-coroutines-rx2/test/guide/example-reactive-context-05.kt)获得完整代码

现在，这段代码中协程执行在了 Rx 的计算线程池并输出，只是<!--
-->就像我们初始的示例中使用了 Rx 的 `subscribe` 操作符。

```text
1 on thread RxComputationThreadPool-1
2 on thread RxComputationThreadPool-1
3 on thread RxComputationThreadPool-1
```

<!--- TEST LINES_START -->

注意，该 [Dispatchers.Unconfined] 上下文应该被谨慎使用。由于降低了局部的堆栈操作以及开销调度的减少，
它也许会在某些测试中提高总体性能，但它也会产生更深的堆栈<!--
-->并且更难以推断使用它的代码的异步性。

如果一个协程将一个元素发送到一个通道，那么调用的线程
[send][SendChannel.send] 可能会开始使用 [Dispatchers.Unconfined] 调度程序执行协程的代码。
原本的生产者协程调用 `send` 后会暂停直至不受限的消费者协程运行至下一个<!--
-->挂起点。这与缺乏 Rx 世界中的锁步单线程 `onNext` 执行<!--
-->线程切换操作符是非常类似的。这在 Rx 中是默认正常的，因为操作符经常做一些非常小块的<!--
-->工作并且你必须做一些复杂处理来合并大量的操作符。然而，这对于协程来说是不常见的，
你可以在一个协程中进行任意复杂的处理。通常，你只需要在多个工作协程之间使用扇入和扇出<!--
-->来为复杂的流水线链接流处理协程。

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


