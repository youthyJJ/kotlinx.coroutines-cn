<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/DispatcherGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class DispatchersGuideTest {
--> 

## 目录

<!--- TOC -->

* [协程上下文与调度器](#coroutine-context-and-dispatchers)
  * [调度器与线程](#dispatchers-and-threads)
  * [不受限的调度器 vs 受限的调度器](#unconfined-vs-confined-dispatcher)
  * [在协程于线程上调试](#debugging-coroutines-and-threads)
  * [在线程之间跳转](#jumping-between-threads)
  * [上下文中的任务](#job-in-the-context)
  * [子协程](#children-of-a-coroutine)
  * [父协程的职责](#parental-responsibilities)
  * [命名协程以进行调试](#naming-coroutines-for-debugging)
  * [组合上下文中的元素](#combining-context-elements)
  * [可取消的显式任务](#cancellation-via-explicit-job)
  * [线程私有的数据](#thread-local-data)

<!--- END_TOC -->

## 协程上下文与调度器

协程总是运行在一些以
[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
类型为代表的上下文中，它们被定义在了 Kotlin 的标准库中。

协程上下文是各种不同元素的集合。其中主元素是协程中的 [Job]，
我们在前面的文档中见过它，以及它的调度器，而本文将对它进行介绍。

### 调度器与线程

协程上下文包括了一个 _协程调度器_ （请看 [CoroutineDispatcher]），它确定了相应的协程在执行时<!--
-->使用一个或多个线程。协程调度器可以将协程的执行局现在<!--
-->指定的线程中，指定它运行在线程中中或让它不受限的运行。

所有的协程建造器诸如 [launch] 和 [async] 接收一个可选的
[CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 
参数，它可以被用来显式的为一个新协程或其它上下文元素指定一个调度器。

尝试下面的示例：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch { // 运行在父协程的上下文中，即 runBlocking 主协程
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // 不受限的--将工作在主线程中
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // 将会获取默认调度器
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // 将使它获得一个新的线程
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-01.kt)获得完整代码

它执行后得到了以下输出（也许顺序会有所不同）：

```text
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

当调用 `launch { ... }` 时不传参数，它从启动了它的 [CoroutineScope]
承袭了上下文(以及调度器)。在这个案例中，它从 `main` 线程中的 `runBlocking`
主协程承袭了上下文。

[Dispatchers.Unconfined] 是一个特殊的调度器且似乎也运行在 `main` 线程中，但实际上，
它是一种不同的机制，这会在后文中降到。

该默认调度器，当协程在 [GlobalScope] 中启动的时候被使用，
它代表 [Dispatchers.Default] 使用了共享的后台线程池，
所以 `GlobalScope.launch { ... }` 也可以使用类似的调度器—— `launch(Dispatchers.Default) { ... }`。
  
[newSingleThreadContext] 为协程的运行启动了一个新的线程。
一个专用的线程是一种非常昂贵的资源。
在真实的应用程序中两者都必须被释放，当不再需要的时候，使用 [close][ExecutorCoroutineDispatcher.close] 
函数，或存储在一个顶级变量中使它在整个应用程序中被重用。

### 不受限的调度器 vs 受限的调度器
 
[Dispatchers.Unconfined] 协程调度器在被调用的线程中启动协程，但是这只有直到程序运行到<!--
-->第一个挂起点的时候才行。挂起后，它将在完全由该所运行的线程中恢复<!--
-->挂起被调用的函数。Unconfined dispatcher is appropriate when coroutine does not
consume CPU time nor updates any shared data (like UI) that is confined to a specific thread. 

On the other side, by default, a dispatcher for the outer [CoroutineScope] is inherited. 
The default dispatcher for [runBlocking] coroutine, in particular,
is confined to the invoker thread, so inheriting it has the effect of confining execution to
this thread with a predictable FIFO scheduling.

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-02.kt)获得完整代码

执行后的输出：
 
```text
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

<!--- TEST LINES_START -->
 
So, the coroutine that had inherited context of `runBlocking {...}` continues to execute
in the `main` thread, while the unconfined one had resumed in the default executor thread that [delay]
function is using.

> Unconfined dispatcher is an advanced mechanism that can be helpful in certain corner cases where
dispatching of coroutine for its execution later is not needed or produces undesirable side-effects,
because some operation in a coroutine must be performed right away. 
Unconfined dispatcher should not be used in general code.  

### Debugging coroutines and threads

Coroutines can suspend on one thread and resume on another thread. 
Even with a single-threaded dispatcher it might be hard to
figure out what coroutine was doing, where, and when. The common approach to debugging applications with 
threads is to print the thread name in the log file on each log statement. This feature is universally supported
by logging frameworks. When using coroutines, the thread name alone does not give much of a context, so 
`kotlinx.coroutines` includes debugging facilities to make it easier. 

Run the following code with `-Dkotlinx.coroutines.debug` JVM option:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
//sampleStart
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
//sampleEnd    
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-context-03.kt)

There are three coroutines. The main coroutine (#1) -- `runBlocking` one, 
and two coroutines computing deferred values `a` (#2) and `b` (#3).
They are all executing in the context of `runBlocking` and are confined to the main thread.
The output of this code is:

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST FLEXIBLE_THREAD -->

The `log` function prints the name of the thread in square brackets and you can see, that it is the `main`
thread, but the identifier of the currently executing coroutine is appended to it. This identifier 
is consecutively assigned to all created coroutines when debugging mode is turned on.

You can read more about debugging facilities in the documentation for [newCoroutineContext] function.

### 在线程之间跳转

Run the following code with `-Dkotlinx.coroutines.debug` JVM option (see [debug](#debugging-coroutines-and-threads)):

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() {
//sampleStart
    newSingleThreadContext("Ctx1").use { ctx1 ->
        newSingleThreadContext("Ctx2").use { ctx2 ->
            runBlocking(ctx1) {
                log("Started in ctx1")
                withContext(ctx2) {
                    log("Working in ctx2")
                }
                log("Back to ctx1")
            }
        }
    }
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-04.kt)获得完整代码

它演示了一些新技术。其中一个使用 [runBlocking] 来显式指定了一个上下文，并且<!--
-->另一个使用 [withContext] 函数来改变协程的上下文，而仍然驻留在相同的<!--
-->协程中，你可以在下面的输出中看到：

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

注意，在这个例子中，当我们不再需要某个在 [newSingleThreadContext] 中创建的线程的时候，
它使用了标准库中的 Kotlin 标准库中的 `use` 函数来释放该线程。

### 上下文中的任务

The coroutine's [Job] is part of its context. The coroutine can retrieve it from its own context 
using `coroutineContext[Job]` expression:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    println("My job is ${coroutineContext[Job]}")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-05.kt)获得完整代码

It produces something like that when running in [debug mode](#debugging-coroutines-and-threads):

```
My job is "coroutine#1":BlockingCoroutine{Active}@6d311334
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is \"coroutine#1\":BlockingCoroutine{Active}@") -->

Note, that [isActive] in [CoroutineScope] is just a convenient shortcut for
`coroutineContext[Job]?.isActive == true`.

### Children of a coroutine

When a coroutine is launched in the [CoroutineScope] of another coroutine,
it inherits its context via [CoroutineScope.coroutineContext] and 
the [Job] of the new coroutine becomes
a _child_ of the parent coroutine's job. When the parent coroutine is cancelled, all its children
are recursively cancelled, too. 

However, when [GlobalScope] is used to launch a coroutine, it is not tied to the scope it
was launched from and operates independently.
  

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        // it spawns two other jobs, one with GlobalScope
        GlobalScope.launch {
            println("job1: I run in GlobalScope and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-06.kt)获得完整代码

The output of this code is:

```text
job1: I run in GlobalScope and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### Parental responsibilities 

A parent coroutine always waits for completion of all its children. Parent does not have to explicitly track
all the children it launches and it does not have to use [Job.join] to wait for them at the end:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        repeat(3) { i -> // launch a few children jobs
            launch  {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() // wait for completion of the request, including all its children
    println("Now processing of the request is complete")
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-07.kt)获得完整代码

The result is going to be:

```text
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```

<!--- TEST -->

### Naming coroutines for debugging

Automatically assigned ids are good when coroutines log often and you just need to correlate log records
coming from the same coroutine. However, when coroutine is tied to the processing of a specific request
or doing some specific background task, it is better to name it explicitly for debugging purposes.
[CoroutineName] context element serves the same function as a thread name. It'll get displayed in the thread name that
is executing this coroutine when [debugging mode](#debugging-coroutines-and-threads) is turned on.

The following example demonstrates this concept:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking(CoroutineName("main")) {
//sampleStart
    log("Started main coroutine")
    // run two background value computations
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-08.kt)获得完整代码

The output it produces with `-Dkotlinx.coroutines.debug` JVM option is similar to:
 
```text
[main @main#1] Started main coroutine
[main @v1coroutine#2] Computing v1
[main @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

<!--- TEST FLEXIBLE_THREAD -->

### Combining context elements

Sometimes we need to define multiple elements for coroutine context. We can use `+` operator for that.
For example, we can launch a coroutine with an explicitly specified dispatcher and an explicitly specified 
name at the same time: 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-09.kt)获得完整代码

The output of this code  with `-Dkotlinx.coroutines.debug` JVM option is: 

```text
I'm working in thread DefaultDispatcher-worker-1 @test#2
```

<!--- TEST FLEXIBLE_THREAD -->

### Cancellation via explicit job

Let us put our knowledge about contexts, children and jobs together. Assume that our application has
an object with a lifecycle, but that object is not a coroutine. For example, we are writing an Android application
and launch various coroutines in the context of an Android activity to perform asynchronous operations to fetch 
and update data, do animations, etc. All of these coroutines must be cancelled when activity is destroyed
to avoid memory leaks. 
  
We manage a lifecycle of our coroutines by creating an instance of [Job] that is tied to 
the lifecycle of our activity. A job instance is created using [Job()] factory function when
activity is created and it is cancelled when an activity is destroyed like this:


<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class Activity : CoroutineScope {
    lateinit var job: Job

    fun create() {
        job = Job()
    }

    fun destroy() {
        job.cancel()
    }
    // to be continued ...
```

</div>

We also implement [CoroutineScope] interface in this `Actvity` class. We only need to provide an override
for its [CoroutineScope.coroutineContext] property to specify the context for coroutines launched in its
scope. We combine the desired dispatcher (we used [Dispatchers.Default] in this example) and a job:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
    // class Activity continues
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job
    // to be continued ...
```

</div>

Now, we can launch coroutines in the scope of this `Activity` without having to explicitly
specify their context. For the demo, we launch ten coroutines that delay for a different time:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
    // class Activity continues
    fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends
``` 

</div>

In our main function we create activity, call our test `doSomething` function, and destroy activity after 500ms.
This cancels all the coroutines that were launched which we can confirm by noting that it does not print 
onto the screen anymore if we wait: 

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class Activity : CoroutineScope {
    lateinit var job: Job

    fun create() {
        job = Job()
    }

    fun destroy() {
        job.cancel()
    }
    // to be continued ...

    // class Activity continues
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job
    // to be continued ...

    // class Activity continues
    fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends

fun main() = runBlocking<Unit> {
//sampleStart
    val activity = Activity()
    activity.create() // create an activity
    activity.doSomething() // run test function
    println("Launched coroutines")
    delay(500L) // delay for half a second
    println("Destroying activity!")
    activity.destroy() // cancels all coroutines
    delay(1000) // visually confirm that they don't work
//sampleEnd    
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-context-10.kt)获得完整代码

The output of this example is:

```text
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

<!--- TEST -->

As you can see, only the first two coroutines had printed a message and the others were cancelled 
by a single invocation of `job.cancel()` in `Activity.destroy()`.

### Thread-local data

Sometimes it is convenient to have an ability to pass some thread-local data, but, for coroutines, which 
are not bound to any particular thread, it is hard to achieve it manually without writing a lot of boilerplate.

For [`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html), 
[asContextElement] extension function is here for the rescue. It creates an additional context element, 
which keeps the value of the given `ThreadLocal` and restores it every time the coroutine switches its context.

It is easy to demonstrate it in action:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

val threadLocal = ThreadLocal<String?>() // declare thread-local variable

fun main() = runBlocking<Unit> {
//sampleStart
    threadLocal.set("main")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
       println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
//sampleEnd    
}
```  

</div>                                                                                       

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-context-11.kt)

In this example we launch new coroutine in a background thread pool using [Dispatchers.Default], so
it works on a different threads from a thread pool, but it still has the value of thread local variable,
that we've specified using `threadLocal.asContextElement(value = "launch")`,
no matter on what thread the coroutine is executed.
Thus, output (with [debug](#debugging-coroutines-and-threads)) is:

```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

<!--- TEST FLEXIBLE_THREAD -->

`ThreadLocal` has first-class support and can be used with any primitive `kotlinx.coroutines` provides.
It has one key limitation: when thread-local is mutated, a new value is not propagated to the coroutine caller 
(as context element cannot track all `ThreadLocal` object accesses) and updated value is lost on the next suspension.
Use [withContext] to update the value of the thread-local in a coroutine, see [asContextElement] for more details. 

Alternatively, a value can be stored in a mutable box like `class Counter(var i: Int)`, which is, in turn, 
stored in a thread-local variable. However, in this case you are fully responsible to synchronize 
potentially concurrent modifications to the variable in this mutable box.

For advanced usage, for example for integration with logging MDC, transactional contexts or any other libraries
which internally use thread-locals for passing data, see documentation for [ThreadContextElement] interface 
that should be implemented. 

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[newSingleThreadContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html
[ExecutorCoroutineDispatcher.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-executor-coroutine-dispatcher/close.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[newCoroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-coroutine-context.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope.coroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-name/index.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[asContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.lang.-thread-local/as-context-element.html
[ThreadContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-thread-context-element/index.html
<!--- END -->
