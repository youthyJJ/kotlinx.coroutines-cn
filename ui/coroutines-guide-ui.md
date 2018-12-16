<!--- INCLUDE .*/example-ui-([a-z]+)-([0-9]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide-ui.md by Knit tool. Do not edit.
package kotlinx.coroutines.javafx.guide.$$1$$2

import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.javafx.JavaFx as Main
import javafx.application.Application
import javafx.event.EventHandler
import javafx.geometry.*
import javafx.scene.*
import javafx.scene.input.MouseEvent
import javafx.scene.layout.StackPane
import javafx.scene.paint.Color
import javafx.scene.shape.Circle
import javafx.scene.text.Text
import javafx.stage.Stage

fun main(args: Array<String>) {
    Application.launch(ExampleApp::class.java, *args)
}

class ExampleApp : Application() {
    val hello = Text("Hello World!").apply {
        fill = Color.valueOf("#C0C0C0")
    }

    val fab = Circle(20.0, Color.valueOf("#FF4081"))

    val root = StackPane().apply {
        children += hello
        children += fab
        StackPane.setAlignment(hello, Pos.CENTER)
        StackPane.setAlignment(fab, Pos.BOTTOM_RIGHT)
        StackPane.setMargin(fab, Insets(15.0))
    }

    val scene = Scene(root, 240.0, 380.0).apply {
        fill = Color.valueOf("#303030")
    }

    override fun start(stage: Stage) {
        stage.title = "Example"
        stage.scene = scene
        stage.show()
        setup(hello, fab)
    }
}
-->
<!--- KNIT     kotlinx-coroutines-javafx/test/guide/.*\.kt -->

# 使用协程进行 UI 编程指南

本篇教程假定你已经熟悉了<!-- 
-->包含[kotlinx.coroutines 指南](../docs/coroutines-guide.md)在内的基础协程概念，并给予<!--
-->如何在 UI 应用程序中使用协程的明确示例。

所有的 UI 程序库都有一个共同的特征。所有的 UI 状态都被限制在单个的<!--
-->主线程中，并且所有更新 UI 的操作都应该发生在该线程中。在使用协程时，
这意味着你需要一个适当的 _协程调度器上下文_ 来限制协程<!--
-->运行与 UI 主线程中。

特别是，`kotlinx.coroutines` 为不同的 UI 应用程序库提供了三个<!--
-->协程上下文模块：
 
* [kotlinx-coroutines-android](kotlinx-coroutines-android) -- `Dispatchers.Main` 为 Android 应用程序提供的上下文。
* [kotlinx-coroutines-javafx](kotlinx-coroutines-javafx) -- `Dispatchers.JavaFx` 为 JavaFX UI 应用程序提供的上下文。
* [kotlinx-coroutines-swing](kotlinx-coroutines-swing) -- `Dispatchers.Swing` 为 Swing UI 应用程序提供的上下文。

当然，UI 调度器被允许通过来自于 `kotlinx-coroutines-core` 的 `Dispatchers.Main` 以及被
[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) API 暴露的相应实现（Android，JavaFx 或 Swing）。
举例来说，假如你编写了一个 JavaFx 应用程序，你使用 `Dispatchers.Main` 或者 `Dispachers.JavaFx` 扩展都是可以的，它们指向同一个对象。

本教程同时包含了所有的 UI 库，因为这些模块中的每一个都只包含一个<!--
-->对象的定义，长度为几页。你可以使用它们中的任何一个来作为例子来编写<!--
-->为你最喜爱的 UI 库编写上下文对象，甚至是没有被包含在本文中的。

## 目录

<!--- TOC -->

* [体系](#setup)
  * [JavaFx](#javafx)
  * [Android](#android)
* [UI 协程基础](#basic-ui-coroutines)
  * [启动 UI 协程](#launch-ui-coroutine)
  * [取消 UI 协程](#cancel-ui-coroutine)
* [在 UI 上下文中使用 actors](#using-actors-within-ui-context)
  * [协程扩展](#extensions-for-coroutines)
  * [最多一个并发任务](#at-most-one-concurrent-job)
  * [事件归并](#event-conflation)
* [阻塞操作](#blocking-operations)
  * [UI 冻结的问题](#the-problem-of-ui-freezes)
  * [结构性并发，生命周期以及协程父子层级结构](#structured-concurrency-lifecycle-and-coroutine-parent-child-hierarchy)
  * [阻塞操作](#blocking-operations)
* [高级主题](#advanced-topics)
  * [没有调度器时在 UI 事件处理器中启动协程](#starting-coroutine-in-ui-event-handlers-without-dispatch)

<!--- END_TOC -->

## 体系

本篇教程中的可运行的示例是通过 JavaFx 来呈现的。其优点是所有的示例可以<!--
-->直接在任何运行在操作系统中而不需要模拟器或任何类似的东西，并且它们可以完全独立存在
（每个示例都在同一个文件中）。
关于在 Android 上重现它们需要进行哪些更改（如果有的话）会有单独的注释。 

### JavaFx

这个 JavaFx 的基础的示例应用程度由一个包含名为 `hello` 的文字标签的窗口构成，它最初<!--
-->包含 "Hello World!" 字符串以及一个右下角的粉色圆形 `fab`（floating action button —— 悬浮动作按钮）。

![JavaFx 的 UI 示例](ui-example-javafx.png)

JavaFX 应用程序中的 `start` 函数调用了 `setup` 函数，将它引用到 `hello` 与 `fab`
节点上。这是本指南其余部分中放置各种代码的地方：

```kotlin
fun setup(hello: Text, fab: Circle) {
    // placeholder
}
```

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-basic-01.kt)获得完整代码

你可以在 Github 上 clone [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 这个项目到你的<!--
-->工作站中在 IDE 中打开这个项目。所有本教程中的示例都在
[`ui/kotlinx-coroutines-javafx`](kotlinx-coroutines-javafx) 模块的 test 文件夹中。 
这样的话你就能够运行并观察每一个示例是如何工作的并<!--
-->在你对它们修改时进行实验。

### Android

请跟随这篇教程——[在 Android 中开始使用 Kotlin](https://kotlinlang.org/docs/tutorials/kotlin-android.html)，
来在 Android Studio 中创建一个 Kotlin 项目。我们也鼓励你添加<!--
-->[Android 的 Kotlin 扩展](https://kotlinlang.org/docs/tutorials/android-plugin.html)
到你的应用程序中。

在 Android Studio 2.3 中，您将获得一个类似于下图所示的应用程序：

![UI example for Android](ui-example-android.png)

到你的应用程序的 `context_main.xml` 文件中，并将 ID “hello” 指定给你写有 “Hello World!” 字符串的文本视图，
因此它在您的应用程序中可用作 “hello” 和 Kotlin Android 扩展。粉红色的悬浮<!--
-->动作按钮在已创建的项目模板中已命名为 “fab”。

在你的应用程序的 `MainActivity.kt` 中移除 `fab.setOnClickListener { ... }` 代码块并替换<!--
-->添加 `setup(hello, fab)` 的调用作为 `onCreate` 函数的最后一行。
在文件的末尾创建一个占位的 `setup` 函数。
这是本指南其余部分中放置各种代码的地方：

```kotlin
fun setup(hello: TextView, fab: FloatingActionButton) {
    // placeholder
}
```

<!--- CLEAR -->

在 `app/build.gradle` 文件的 `dependencies { ... }`
部分中添加 `kotlinx-coroutines-android` 模块的依赖：

```groovy
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.0.1"
```

你可以在 Github 上 clone [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 这个项目到你的<!--
-->工作站中。Android 生成的模板项目位于
[`ui/kotlinx-coroutines-android/example-app`](kotlinx-coroutines-android/example-app) 目录。
你可以在 Android Studio 中打开它并跟着本教程在 Android 上学习。

## UI 协程基础

本节将展示协程在 UI 应用程序中的基础用法。

### 启动 UI 协程

`kotlinx-coroutines-javafx` 模块包含
[Dispatchers.JavaFx][kotlinx.coroutines.Dispatchers.JavaFx] 
调度器，其包含执行
JavaFx 应用程序线程。我们通过 `Main` 引入它，使所有呈现的示例都可以<!--
-->容易的移植到 Android：
 
```kotlin
import kotlinx.coroutines.javafx.JavaFx as Main
```
 
<!--- CLEAR -->

协程被限制在 UI 主线程就可以自如的做任何更新 UI 的操作，并且可以在主线程中进行无阻塞的挂起。
举例来说，我们可以通过命令式编码来执行动画。下面的代码使用
[launch] 协程构建器，将文本倒序的 从 10 更新到 1：

```kotlin
fun setup(hello: Text, fab: Circle) {
    GlobalScope.launch(Dispatchers.Main) { // 在主线程中启动协程
        for (i in 10 downTo 1) { // 从 10 到 1 的倒数
            hello.text = "Countdown $i ..." // 更新文本
            delay(500) // 等待半秒钟
        }
        hello.text = "Done!"
    }
}
```

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-basic-02.kt)获得完整代码

所以，这里将发生什么？由于我们在主 UI 上下文中启动协程，我们可以在该协程内部<!--
-->自如的更新 UI，并同时调用就像 [delay] 这样的 _挂起函数_ 。当 `delay` 函数的等待期间<!--
-->UI 并不会冻结，因为它不会阻塞 UI 线程——它只会挂起协程。

> 相应的代码在 Android 应用程序中表现也是类似的。 
  你只需要在相应的代码中拷贝 `setup` 的函数体到 Android 项目中。 

### 取消 UI 协程

我们可以对 `launch` 函数返回的 [Job] 对象保持一个引用，来使用它<!--
-->当我们想停止一个任务的时候来取消协程。让我们在粉色按钮被点击的时候来取消协程：

```kotlin
fun setup(hello: Text, fab: Circle) {
    val job = GlobalScope.launch(Dispatchers.Main) { // 在主线程中启动协程
        for (i in 10 downTo 1) { // 从 10 到 1 的倒数
            hello.text = "Countdown $i ..." // 更新文本
            delay(500) // 等待半秒钟
        }
        hello.text = "Done!"
    }
    fab.onMouseClicked = EventHandler { job.cancel() } // 在点击时取消协程
}
```

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-basic-03.kt)获得完整代码

现在，如果当倒数仍然在运行时点击圆形按钮，倒数会停止。 
注意，[Job.cancel] 的调用是是完全线程安全和非阻塞的。它仅仅是示意协程取消<!--
-->它的任务，而不会去等待任务事实上的终止。它可以在任何地方被调用。
Invoking it on a coroutine that was already cancelled or has completed does nothing. 

> 相关的代码行在 Android 中如下所示：

```kotlin
fab.setOnClickListener { job.cancel() }  // 在点击时取消协程
```

<!--- CLEAR -->

## 在 UI 上下文中使用 actors

在本节中，我们将展示 UI 应用程序如何在其UI上下文中使用 actor 以确保<!--
-->被启动的协程的数量没有无限制的增长。

### 协程扩展

我们的目标是写一个扩展 _协程构建器_ 函数并命名为 `onClick`，
所以我们可以在在该示例代码中展示当每次圆形按钮被点击的时候都会进行倒数：

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onClick { // 当圆形按钮被点击的时候启动协程
        for (i in 10 downTo 1) { // 从 10 到 1 的倒数
            hello.text = "Countdown $i ..." // 更新文本
            delay(500) // 等待半秒钟
        }
        hello.text = "Done!"
    }
}
```

<!--- INCLUDE .*/example-ui-actor-([0-9]+).kt -->

我们的第一个 `onClick` 实现只是在每次鼠标事件到来时启动了一个新的协程并<!--
-->将相应的鼠标事件传递给提供的操作（仅仅在每次我们需要它时）：

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    onMouseClicked = EventHandler { event ->
        GlobalScope.launch(Dispatchers.Main) { 
            action(event)
        }
    }
}
```  

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-actor-01.kt)获得完整代码

注意，当每次圆形按钮被点击时，它启动了一个新的协程并都将竞争<!--
-->更新文本。尝试一下。它看起来并不是非常棒。我们将在稍后修正它。

> 在 Android 中，相关的扩展可以写给 `View` 类，所以在上面这段代码中展示的
  `setup` 函数可以无需修改就能使用。而在 Android 的
  OnClickListener 中 `MouseEvent` 没有被使用，所以省略它。

```kotlin
fun View.onClick(action: suspend () -> Unit) {
    setOnClickListener { 
        GlobalScope.launch(Dispatchers.Main) {
            action()
        }
    }
}
```

<!--- CLEAR -->

### 最多一个并发任务

在启动一个新协程之前，我们可以取消一个存活中的任务以确保最多一个协程正在进行<!--
-->倒数。然而，这通常不是一个好的主意。[cancel][Job.cancel] 函数仅仅被用来指示<!--
-->退出一个协程。协程会协同的进行取消，在这时，做一些不可取消的事情<!--
-->或以其它方式忽略取消信号。一个好的解决方式是使用一个 [actor] 来执行任务<!--
-->而不应该进行并发。让我们改变 `onClick` 扩展的实现：
  
```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // 在这个节点中启动一个 actor 来处理所有事件
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main) {
        for (event in channel) action(event) // 将事件传递给 action
    }
    // 设置一个监听器来为这个 actor 添加事件
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-actor-02.kt)获得完整代码
  
构成协程和常规事件处理程序的集成基础的关键思想是
[SendChannel] 上的 [offer][SendChannel.offer] 函数不会等待。它会立即将一个元素发送到 actor，
如果可能的话，或者丢弃一个元素。一个 `offer` 事实上返回了一个我们在这里忽略的 `Boolean` 结果。

在这个版本的代码中尝试反复点击圆形按钮。当倒数动作进行中时，
点击动作会被忽略。这会发生的原因是 actor 正忙于执行而不会从通道中接收元素。
默认的，一个 actor 的邮箱由 `RendezvousChannel` 支持，只有当 `receive` 在运行中的时候
`offer` 操作才会成功。 

> 在 Android 中，这里有一个 `View` 在 OnClickListener 中发送事件，所以我们发送一个 `View` 到 actor 来作为信号。 
  相关的 `View` 类的扩展如下所示：

```kotlin
fun View.onClick(action: suspend (View) -> Unit) {
    // 启动一个 actor
    val eventActor = GlobalScope.actor<View>(Dispatchers.Main) {
        for (event in channel) action(event)
    }
    // 设置一个监听器来启用 actor
    setOnClickListener { 
        eventActor.offer(it)
    }
}
```

<!--- CLEAR -->


### 事件归并
 
有时处理最近的事件更合适，而不是在我们忙于处理前一个事件的时候<!--
-->忽略事件。[actor] 协程构建器接收一个可选的 `capacity` 参数来<!--
-->控制此 actor 用于其邮箱的通道的实现。所有关于可用选项的描述于
[`Channel()`][Channel] 工厂函数的文档中给出。

让我们修改代码来使用 `ConflatedChannel` 通过 [Channel.CONFLATED] 修改容量值。这<!--
-->只需要在创建 actor 的这一行作出修改：

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // launch one actor to handle all events on this node
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) { // <--- 修改这里
        for (event in channel) action(event) // 将事件传递给 action
    }
    // 设置一个监听器来为这个 actor 添加事件
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-actor-03.kt)获得完整代码。
  在 Android 中你需要在前面的示例中更新 `val eventActor = ...` 这一行。

现在，当倒数运行中时如果这个圆形按钮被点击，倒数将在结束后重新运行。仅仅一次。 
在倒数进行中时，重复点击将被 _合并_ ，只有最近的事件才会被<!--
-->处理。

对于必须对高频传入的事件做出反应的 UI 应用程序，这也是一种期望的行为，
事件流通过基于最近收到的更新更新其UI。协程通过使用
`ConflatedChannel` 来避免通过引入事件缓冲而造成的延迟。

您可以在上面的代码行中试验 `capacity` 参数，看看它如何影响代码的行为。
设置 `capacity = Channel.UNLIMITED` 参数来创建协程以及 `LinkedListChannel` 邮箱来缓冲所有的<!--
-->事件。在这个案例中，动画会在单击圆形按钮时运行多次。

## Blocking operations

This section explains how to use UI coroutines with thread-blocking operations.

### The problem of UI freezes 

It would have been great if all APIs out there were written as suspending functions that never blocks an 
execution thread. However, it is quite often not the case. Sometimes you need to do a CPU-consuming computation
or just need to invoke some 3rd party APIs for network access, for example, that blocks the invoker thread. 
You cannot do that from the main UI thread nor from the UI-confined coroutine directly, because that would
block the main UI thread and cause the freeze up of the UI.

<!--- INCLUDE .*/example-ui-blocking-([0-9]+).kt

fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) {
        for (event in channel) action(event) // pass event to action
    }
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
-->

The following example illustrates the problem. We are going to use `onClick` extension with UI-confined
event-conflating actor from the last section to process the last click in the main UI thread. 
For this example, we are going to 
perform naive computation of [Fibonacci numbers](https://en.wikipedia.org/wiki/Fibonacci_number):
 
```kotlin
fun fib(x: Int): Int =
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
``` 
 
We'll be computing larger and larger Fibonacci number each time the circle is clicked. 
To make the UI freeze more obvious, there is also a fast counting animation that is always running 
and is constantly updating the text in the main UI dispatcher:

```kotlin
fun setup(hello: Text, fab: Circle) {
    var result = "none" // the last result
    // counting animation 
    GlobalScope.launch(Dispatchers.Main) {
        var counter = 0
        while (true) {
            hello.text = "${++counter}: $result"
            delay(100) // update the text every 100ms
        }
    }
    // compute the next fibonacci number of each click
    var x = 1
    fab.onClick {
        result = "fib($x) = ${fib(x)}"
        x++
    }
}
```
 
> You can get full JavaFx code [here](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-01.kt).
  You can just copy the `fib` function and the body of the `setup` function to your Android project.

Try clicking on the circle in this example. After around 30-40th click our naive computation is going to become
quite slow and you would immediately see how the main UI thread freezes, because the animation stops running 
during UI freeze.

### Structured concurrency, lifecycle and coroutine parent-child hierarchy

A typical UI application has a number of elements with a lifecycle. Windows, UI controls, activities, views, fragments
and other visual elements are created and destroyed. A long-running coroutine, performing some IO or a background 
computation, can retain references to the corresponding UI elements for longer than it is needed, preventing garbage 
collection of the whole trees of UI objects that were already destroyed and will not be displayed anymore.

The natural solution to this problem is to associate a [Job] object with each UI object that has a lifecycle and create
all the coroutines in the context of this job. But passing associated job object to every coroutine builder is error-prone, 
it is easy to forget it. For this purpose, [CoroutineScope] interface should be implemented by UI owner, and then every
coroutine builder defined as an extension on [CoroutineScope] inherits UI job without explicitly mentioning it.

For example, in Android application an `Activity` is initially _created_ and is _destroyed_ when it is no longer 
needed and when its memory must be released. A natural solution is to attach an 
instance of a `Job` to an instance of an `Activity`:
<!--- CLEAR -->

```kotlin
abstract class ScopedAppActivity: AppCompatActivity(), CoroutineScope {
    protected lateinit var job: Job
    override val coroutineContext: CoroutineContext 
        get() = job + Dispatchers.Main
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        job = Job()
    }
        
    override fun onDestroy() {
        super.onDestroy()
        job.cancel()
    } 
}
```

Now, an activity that is associated with a job has to extend ScopedAppActivity

```kotlin
class MainActivity : ScopedAppActivity() {

    fun asyncShowData() = launch { // Is invoked in UI context with Activity's job as a parent
        // actual implementation
    }
    
    suspend fun showIOData() {
        val deferred = async(Dispatchers.IO) {
            // impl      
        }
        withContext(Dispatchers.Main) {
          val data = deferred.await()
          // Show data in UI
        }
    }
}
```

Every coroutine launched from within a `MainActivity` has its job as a parent and is immediately cancelled when
activity is destroyed.

To propagate activity scope to its views and presenters, multiple techniques can be used:
- [coroutineScope] builder to provide a nested scope
- Receive [CoroutineScope] in presenter method parameters
- Make method extension on [CoroutineScope] (applicable only for top-level methods)

```kotlin
class ActivityWithPresenters: ScopedAppActivity() {
    fun init() {
        val presenter = Presenter()
        val presenter2 = ScopedPresenter(this)
    }
}

class Presenter {
    suspend fun loadData() = coroutineScope {
        // Nested scope of outer activity
    }
    
    suspend fun loadData(uiScope: CoroutineScope) = uiScope.launch {
      // Invoked in the uiScope
    }
}

class ScopedPresenter(scope: CoroutineScope): CoroutineScope by scope {
    fun loadData() = launch { // Extension on ActivityWithPresenters's scope
    }
}

suspend fun CoroutineScope.launchInIO() = launch(Dispatchers.IO) {
   // Launched in the scope of the caller, but with IO dispatcher
}
``` 

Parent-child relation between jobs forms a hierarchy. A coroutine that performs some background job on behalf of
the view and in its context can create further children coroutines. The whole tree of coroutines gets cancelled
when the parent job is cancelled. An example of that is shown in the
["Children of a coroutine"](../docs/coroutine-context-and-dispatchers.md#children-of-a-coroutine) section of the guide to coroutines.
<!--- CLEAR -->

### 阻塞操作

The fix for the blocking operations on the main UI thread is quite straightforward with coroutines. We'll 
convert our "blocking" `fib` function to a non-blocking suspending function that runs the computation in 
the background thread by using [withContext] function to change its execution context to [Dispatchers.Default] that is 
backed by the background pool of threads. 
Notice, that `fib` function is now marked with `suspend` modifier. It does not block the coroutine that
it is invoked from anymore, but suspends its execution when the computation in the background thread is working:

<!--- INCLUDE .*/example-ui-blocking-0[23].kt

fun setup(hello: Text, fab: Circle) {
    var result = "none" // the last result
    // counting animation 
    GlobalScope.launch(Dispatchers.Main) {
        var counter = 0
        while (true) {
            hello.text = "${++counter}: $result"
            delay(100) // update the text every 100ms
        }
    }
    // compute next fibonacci number of each click
    var x = 1
    fab.onClick {
        result = "fib($x) = ${fib(x)}"
        x++
    }
}
-->

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
}
```

> You can get full code [here](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-02.kt).

You can run this code and verify that UI is not frozen while large Fibonacci numbers are being computed. 
However, this code computes `fib` somewhat slower, because every recursive call to `fib` goes via `withContext`. This is 
not a big problem in practice, because `withContext` is smart enough to check that the coroutine is already running
in the required context and avoids overhead of dispatching coroutine to a different thread again. It is an 
overhead nonetheless, which is visible on this primitive code that does nothing else, but only adds integers 
in between invocations to `withContext`. For some more substantial code, the overhead of an extra `withContext` invocation is 
not going to be significant.

Still, this particular `fib` implementation can be made to run as fast as before, but in the background thread, by renaming
the original `fib` function to `fibBlocking` and defining `fib` with `withContext` wrapper on top of `fibBlocking`:

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    fibBlocking(x)
}

fun fibBlocking(x: Int): Int = 
    if (x <= 1) x else fibBlocking(x - 1) + fibBlocking(x - 2)
```

> You can get full code [here](kotlinx-coroutines-javafx/test/guide/example-ui-blocking-03.kt).

You can now enjoy full-speed naive Fibonacci computation without blocking the main UI thread. 
All we need is `withContext(Dispatchers.Default)`.

Note, that because the `fib` function is invoked from the single actor in our code, there is at most one concurrent 
computation of it at any given time, so this code has a natural limit on the resource utilization. 
It can saturate at most one CPU core.
  
## Advanced topics

This section covers various advanced topics. 

### Starting coroutine in UI event handlers without dispatch

Let us write the following code in `setup` to visualize the order of execution when coroutine is launched
from the UI thread:

<!--- CLEAR -->

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main) {
            println("Inside coroutine")
            delay(100)
            println("After delay")
        } 
        println("After launch")
    }
}
```
 
> You can get full JavaFx code [here](kotlinx-coroutines-javafx/test/guide/example-ui-advanced-01.kt).

When we start this code and click on a pinkish circle, the following messages are printed to the console:
 
```text
Before launch
After launch
Inside coroutine
After delay
```

As you can see, execution immediately continues after [launch], while the coroutine gets posted onto the main UI thread
for execution later. All UI dispatchers in `kotlinx.coroutines` are implemented this way. Why so? 

Basically, the choice here is between "JS-style" asynchronous approach (async actions
are always postponed to be executed later in the even dispatch thread) and "C#-style" approach
(async actions are executed in the invoker thread until the first suspension point).
While, C# approach seems to be more efficient, it ends up with recommendations like
"use `yield` if you need to ....". This is error-prone. JS-style approach is more consistent
and does not require programmers to think about whether they need to yield or not.

However, in this particular case when coroutine is started from an event handler and there is no other code around it,
this extra dispatch does indeed add an extra overhead without bringing any additional value. 
In this case an optional [CoroutineStart] parameter to [launch], [async] and [actor] coroutine builders 
can be used for performance optimization. 
Setting it to the value of [CoroutineStart.UNDISPATCHED] has the effect of starting to execute
coroutine immediately until its first suspension point as the following example shows:

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main, CoroutineStart.UNDISPATCHED) { // <--- Notice this change
            println("Inside coroutine")
            delay(100)                            // <--- And this is where coroutine suspends      
            println("After delay")
        }
        println("After launch")
    }
}
```
 
> You can get full JavaFx code [here](kotlinx-coroutines-javafx/test/guide/example-ui-advanced-02.kt).

It prints the following messages on click, confirming that code in the coroutine starts to execute immediately:

```text
Before launch
Inside coroutine
After launch
After delay
```
  
<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[CoroutineStart]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/index.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineStart.UNDISPATCHED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-u-n-d-i-s-p-a-t-c-h-e-d.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[SendChannel.offer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/offer.html
[SendChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/index.html
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[Channel.CONFLATED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/-c-o-n-f-l-a-t-e-d.html
<!--- MODULE kotlinx-coroutines-javafx -->
<!--- INDEX kotlinx.coroutines.javafx -->
[kotlinx.coroutines.Dispatchers.JavaFx]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-javafx/kotlinx.coroutines.javafx/kotlinx.coroutines.-dispatchers/-java-fx.html
<!--- MODULE kotlinx-coroutines-android -->
<!--- INDEX kotlinx.coroutines.android -->
<!--- END -->
