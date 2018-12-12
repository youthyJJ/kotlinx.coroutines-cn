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
[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) API 暴露的相应实现（Android, JavaFx 或 Swing）。
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
  * [最多一个的并发任务](#at-most-one-concurrent-job)
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

你可以在 Github 上 clone  [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 这个项目到你的<!--
-->工作站中在 IDE 中打开这个项目。所有本教程中的示例都在
[`ui/kotlinx-coroutines-javafx`](kotlinx-coroutines-javafx) 模块的 test 文件夹中。 
这样的话你就能够运行并观察每一个示例是如何工作的并<!--
-->在你对它们进行改变时进行实验。

### Android

请跟随这篇教程——[在 Android 中开始使用 Kotlin](https://kotlinlang.org/docs/tutorials/kotlin-android.html)，
来在 Android Studio 中创建一个 Kotlin 项目。我们也鼓励你添加<!--
-->[Android 的 Kotlin 扩展](https://kotlinlang.org/docs/tutorials/android-plugin.html)
到你的应用程序中。

在 Android Studio 2.3 中，您将获得一个类似于下图所示的应用程序：

![UI example for Android](ui-example-android.png)

到你的应用程序的 `context_main.xml` 文件中，并将 ID "hello" 指定给你写有 "Hello World!" 字符串的文本视图，
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

The `kotlinx-coroutines-javafx` module contains 
[Dispatchers.JavaFx][kotlinx.coroutines.Dispatchers.JavaFx] 
dispatcher that dispatches coroutine execution to
the JavaFx application thread. We import it as `Main` to make all the presented examples 
easily portable to Android:
 
```kotlin
import kotlinx.coroutines.javafx.JavaFx as Main
```
 
<!--- CLEAR -->

Coroutines confined to the main UI thread can freely update anything in UI and suspend without blocking the main thread.
For example, we can perform animations by coding them in imperative style. The following code updates the
text with a 10 to 1 countdown twice a second, using [launch] coroutine builder:

```kotlin
fun setup(hello: Text, fab: Circle) {
    GlobalScope.launch(Dispatchers.Main) { // launch coroutine in the main thread
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
}
```

> 你可以点击[这里](kotlinx-coroutines-javafx/test/guide/example-ui-basic-02.kt)获得完整代码

So, what happens here? Because we are launching coroutine in the main UI context, we can freely update UI from 
inside this coroutine and invoke _suspending functions_ like [delay] at the same time. UI is not frozen
while `delay` waits, because it does not block the UI thread -- it just suspends the coroutine.

> The corresponding code for Android application is the same. 
  You just need to copy the body of `setup` function into the corresponding function of Android project. 

### 取消 UI 协程

We can keep a reference to the [Job] object that `launch` function returns and use it to cancel
coroutine when we want to stop it. Let us cancel the coroutine when pinkish circle is clicked:

```kotlin
fun setup(hello: Text, fab: Circle) {
    val job = GlobalScope.launch(Dispatchers.Main) { // launch coroutine in the main thread
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
    fab.onMouseClicked = EventHandler { job.cancel() } // cancel coroutine on click
}
```

> You can get full code [here](kotlinx-coroutines-javafx/test/guide/example-ui-basic-03.kt)

Now, if the circle is clicked while countdown is still running, the countdown stops. 
Note, that [Job.cancel] is completely thread-safe and non-blocking. It just signals the coroutine to cancel 
its job, without waiting for it to actually terminate. It can be invoked from anywhere.
Invoking it on a coroutine that was already cancelled or has completed does nothing. 

> The corresponding line for Android is shown below: 

```kotlin
fab.setOnClickListener { job.cancel() }  // cancel coroutine on click
```

<!--- CLEAR -->

## Using actors within UI context

In this section we show how UI applications can use actors within their UI context make sure that 
there is no unbounded growth in the number of launched coroutines.

### Extensions for coroutines

Our goal is to write an extension _coroutine builder_ function named `onClick`, 
so that we can perform countdown animation every time when the circle is clicked with this simple code:

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onClick { // start coroutine when the circle is clicked
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
}
```

<!--- INCLUDE .*/example-ui-actor-([0-9]+).kt -->

Our first implementation for `onClick` just launches a new coroutine on each mouse event and
passes the corresponding mouse event into the supplied action (just in case we need it):

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    onMouseClicked = EventHandler { event ->
        GlobalScope.launch(Dispatchers.Main) { 
            action(event)
        }
    }
}
```  

> You can get full code [here](kotlinx-coroutines-javafx/test/guide/example-ui-actor-01.kt)

Note, that each time the circle is clicked, it starts a new coroutine and they all compete to 
update the text. Try it. It does not look very good. We'll fix it later.

> On Android, the corresponding extension can be written for `View` class, so that the code
  in `setup` function that is shown above can be used without changes. There is no `MouseEvent`
  used in OnClickListener on Android, so it is omitted.

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

### At most one concurrent job

We can cancel an active job before starting a new one to ensure that at most one coroutine is animating 
the countdown. However, it is generally not the best idea. The [cancel][Job.cancel] function serves only as a signal
to abort a coroutine. Cancellation is cooperative and a coroutine may, at the moment, be doing something non-cancellable
or otherwise ignore a cancellation signal. A better solution is to use an [actor] for tasks that should
not be performed concurrently. Let us change `onClick` extension implementation:
  
```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // launch one actor to handle all events on this node
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main) {
        for (event in channel) action(event) // pass event to action
    }
    // install a listener to offer events to this actor
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> You can get full code [here](kotlinx-coroutines-javafx/test/guide/example-ui-actor-02.kt)
  
The key idea that underlies an integration of an actor coroutine and a regular event handler is that 
there is an [offer][SendChannel.offer] function on [SendChannel] that does not wait. It sends an element to the actor immediately,
if it is possible, or discards an element otherwise. An `offer` actually returns a `Boolean` result which we ignore here.

Try clicking repeatedly on a circle in this version of the code. The clicks are just ignored while the countdown 
animation is running. This happens because the actor is busy with an animation and does not receive from its channel.
By default, an actor's mailbox is backed by `RendezvousChannel`, whose `offer` operation succeeds only when 
the `receive` is active. 

> On Android, there is `View` sent in OnClickListener, so we send the `View` to the actor as a signal. 
  The corresponding extension for `View` class looks like this:

```kotlin
fun View.onClick(action: suspend (View) -> Unit) {
    // launch one actor
    val eventActor = GlobalScope.actor<View>(Dispatchers.Main) {
        for (event in channel) action(event)
    }
    // install a listener to activate this actor
    setOnClickListener { 
        eventActor.offer(it)
    }
}
```

<!--- CLEAR -->


### Event conflation
 
Sometimes it is more appropriate to process the most recent event, instead of just ignoring events while we were busy
processing the previous one.  The [actor] coroutine builder accepts an optional `capacity` parameter that 
controls the implementation of the channel that this actor is using for its mailbox. The description of all 
the available choices is given in documentation of the [`Channel()`][Channel] factory function.

Let us change the code to use `ConflatedChannel` by passing [Channel.CONFLATED] capacity value. The 
change is only to the line that creates an actor:

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // launch one actor to handle all events on this node
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) { // <--- Changed here
        for (event in channel) action(event) // pass event to action
    }
    // install a listener to offer events to this actor
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```  

> You can get full JavaFx code [here](kotlinx-coroutines-javafx/test/guide/example-ui-actor-03.kt).
  On Android you need to update `val eventActor = ...` line from the previous example. 

Now, if a circle is clicked while the animation is running, it restarts animation after the end of it. Just once. 
Repeated clicks while the animation is running are _conflated_ and only the most recent event gets to be 
processed. 

This is also a desired behaviour for UI applications that have to react to incoming high-frequency
event streams by updating their UI based on the most recently received update. A coroutine that is using
`ConflatedChannel` avoids delays that are usually introduced by buffering of events.

You can experiment with `capacity` parameter in the above line to see how it affects the behaviour of the code.
Setting `capacity = Channel.UNLIMITED` creates a coroutine with `LinkedListChannel` mailbox that buffers all 
events. In this case, the animation runs as many times as the circle is clicked.

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

### Blocking operations

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
