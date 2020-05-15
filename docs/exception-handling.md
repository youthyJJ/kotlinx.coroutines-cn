<!--- TEST_NAME ExceptionsGuideTest -->

**目录**

<!--- TOC -->

* [异常处理](#异常处理)
  * [异常的传播](#异常的传播)
  * [CoroutineExceptionHandler](#coroutineexceptionhandler)
  * [取消与异常](#取消与异常)
  * [异常聚合](#异常聚合)
  * [监督](#监督)
    * [监督作业](#监督作业)
    * [监督作用域](#监督作用域)
    * [监督协程中的异常](#监督协程中的异常)

<!--- END -->

## 异常处理

This section covers exception handling and cancellation on exceptions.
We already know that cancelled coroutine throws [CancellationException] in suspension points and that it
is ignored by the coroutines' machinery. Here we look at what happens if an exception is thrown during cancellation or multiple children of the same
coroutine throw an exception.

### 异常的传播

Coroutine builders come in two flavors: propagating exceptions automatically ([launch] and [actor]) or
exposing them to users ([async] and [produce]).
When these builders are used to create a _root_ coroutine, that is not a _child_ of another coroutine,
the former builder treat exceptions as **uncaught** exceptions, similar to Java's `Thread.uncaughtExceptionHandler`,
while the latter are relying on the user to consume the final
exception, for example via [await][Deferred.await] or [receive][ReceiveChannel.receive] 
([produce] and [receive][ReceiveChannel.receive] are covered later in [Channels](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/channels.md) section).

可以通过一个使用 [GlobalScope] 创建根协程的简单示例来进行演示：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = GlobalScope.launch { // root coroutine with launch
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // 我们将在控制台打印 Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // root coroutine with async
        println("Throwing exception from async")
        throw ArithmeticException() // 没有打印任何东西，依赖用户去调用等待
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-01.kt)获取完整代码。

这段代码的输出如下（[调试](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/docs/coroutine-context-and-dispatchers.md#调试协程与线程)）：

```text
Throwing exception from launch
Exception in thread "DefaultDispatcher-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

<!--- TEST EXCEPTION-->

### CoroutineExceptionHandler

It is possible to customize the default behavior of printing **uncaught** exceptions to the console.
[CoroutineExceptionHandler] context element on a _root_ coroutine can be used as generic `catch` block for
this root coroutine and all its children where custom exception handling may take place.
It is similar to [`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)).
You cannot recover from the exception in the `CoroutineExceptionHandler`. The coroutine had already completed
with the corresponding exception when the handler is called. Normally, the handler is used to
log the exception, show some kind of error message, terminate, and/or restart the application.

在 JVM 中可以重定义一个全局的异常处理者来将所有的协程通过
[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) 注册到 [CoroutineExceptionHandler]。
全局异常处理者就如同
[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 
一样，在没有更多的指定的异常处理者被注册的时候被使用。
在 Android 中， `uncaughtExceptionPreHandler` 被设置在全局协程异常处理者中。

`CoroutineExceptionHandler` is invoked only on **uncaught** exceptions &mdash; exceptions that were not handled in any other way.
In particular, all _children_ coroutines (coroutines created in the context of another [Job]) delegate handling of
their exceptions to their parent coroutine, which also delegates to the parent, and so on until the root,
so the `CoroutineExceptionHandler` installed in their context is never used. 
In addition to that, [async] builder always catches all exceptions and represents them in the resulting [Deferred] object, 
so its `CoroutineExceptionHandler` has no effect either.

> Coroutines running in supervision scope do not propagate exceptions to their parent and are
excluded from this rule. A further [Supervision](#supervision) section of this document gives more details.  

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    val job = GlobalScope.launch(handler) { // root coroutine, running in GlobalScope
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) { // also root, but async instead of launch
        throw ArithmeticException() // 没有打印任何东西，依赖用户去调用 deferred.await()
    }
    joinAll(job, deferred)
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-02.kt)获取完整代码。

这段代码的输出如下：

```text
CoroutineExceptionHandler got java.lang.AssertionError
```

<!--- TEST-->

### 取消与异常

取消与异常紧密相关。协程内部使用 `CancellationException` 来进行取消，这个<!--
-->异常会被所有的处理者忽略，所以那些可以被 `catch` 代码块捕获的异常<!--
-->仅仅应该被用来作为额外调试信息的资源。
当一个协程使用 [Job.cancel] 取消的时候，它会被终止，但是它不会取消它的父协程。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-03.kt)获取完整代码。

这段代码的输出如下：

```text
Cancelling child
Child is cancelled
Parent is not cancelled
```

<!--- TEST-->

If a coroutine encounters an exception other than `CancellationException`, it cancels its parent with that exception. 
This behaviour cannot be overridden and is used to provide stable coroutines hierarchies for
[structured concurrency](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/composing-suspending-functions.md#structured-concurrency-with-async).
[CoroutineExceptionHandler] implementation is not used for child coroutines.

> 在本例中，[CoroutineExceptionHandler] 总是被设置在由 [GlobalScope]
启动的协程中。将异常处理者设置在
[runBlocking] 主作用域内启动的协程中是没有意义的，尽管子协程已经设置了异常处理者，
但是主协程也总是会被取消的。

The original exception is handled by the parent only when all its children terminate,
which is demonstrated by the following example.

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    val job = GlobalScope.launch(handler) {
        launch { // 第一个子协程
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // 第二个子协程
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-04.kt)获取完整代码。

这段代码的输出如下：

```text
Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
CoroutineExceptionHandler got java.lang.ArithmeticException
```
<!--- TEST-->

### 异常聚合

When multiple children of a coroutine fail with an exception the
general rule is "the first exception wins", so the first exception gets handled.
All additional exceptions that happen after the first one are attached to the first exception as suppressed ones. 

<!--- INCLUDE
import kotlinx.coroutines.exceptions.*
-->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // it gets cancelled when another sibling fails with IOException
            } finally {
                throw ArithmeticException() // the second exception
            }
        }
        launch {
            delay(100)
            throw IOException() // the first exception
        }
        delay(Long.MAX_VALUE)
    }
    job.join()  
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-05.kt)获取完整代码。

> 注意：上面的代码将只在 JDK7 以上支持 `suppressed` 异常的环境中才能正确工作。

这段代码的输出如下：

```text
CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> 注意，这个机制当前只能在 Java 1.7 以上的版本中使用。
在 JS 和原生环境下暂时会受到限制，但将来会被修复。

Cancellation exceptions are transparent and are unwrapped by default:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        val inner = launch { // all this stack of coroutines will get cancelled
            launch {
                launch {
                    throw IOException() // the original exception
                }
            }
        }
        try {
            inner.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e // cancellation exception is rethrown, yet the original IOException gets to the handler  
        }
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-06.kt)获取完整代码。

这段代码的输出如下：

```text
Rethrowing CancellationException with original cause
CoroutineExceptionHandler got java.io.IOException
```
<!--- TEST-->

### 监督

As we have studied before, cancellation is a bidirectional relationship propagating through the whole
hierarchy of coroutines. Let us take a look at the case when unidirectional cancellation is required. 

此类需求的一个良好示例是在其作用域内定义作业的 UI 组件。如果任何一个 UI 的子作业<!--
-->执行失败了，它并不总是有必要取消（有效地杀死）整个 UI 组件，
但是如果 UI 组件被销毁了（并且它的作业也被取消了），由于它的结果不再被需要了，它有必要使所有的子作业执行失败。

另一个例子是服务进程孵化了一些子作业并且需要 _监督_
它们的执行，追踪它们的故障并在这些子作业执行失败的时候重启。

#### 监督作业

[SupervisorJob][SupervisorJob()] 可以被用于这些目的。
它类似于常规的 [Job][Job()]，唯一的不同是：<!--
-->SupervisorJob 的取消只会向下传播。这是非常容易从示例中观察到的：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // 启动第一个子作业——这个示例将会忽略它的异常（不要在实践中这么做！）
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("First child is failing")
            throw AssertionError("First child is cancelled")
        }
        // 启动第两个子作业
        val secondChild = launch {
            firstChild.join()
            // 取消了第一个子作业且没有传播给第二个子作业
            println("First child is cancelled: ${firstChild.isCancelled}, but second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // 但是取消了监督的传播
                println("Second child is cancelled because supervisor is cancelled")
            }
        }
        // 等待直到第一个子作业失败且执行完成
        firstChild.join()
        println("Cancelling supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-01.kt)获取完整代码。

这段代码的输出如下：

```text
First child is failing
First child is cancelled: true, but second one is still active
Cancelling supervisor
Second child is cancelled because supervisor is cancelled
```
<!--- TEST-->


#### 监督作用域

对于*作用域*的并发，[supervisorScope] 可以被用来替代 [coroutineScope] 来实现相同的目的。它只会单向的传播<!--
-->并且当作业自身执行失败的时候将所有子作业全部取消。作业自身也会在所有的子作业执行结束前等待，
就像 [coroutineScope] 所做的那样。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("Child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("Child is cancelled")
                }
            }
            // 使用 yield 来给我们的子作业一个机会来执行打印
            yield()
            println("Throwing exception from scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught assertion error")
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-02.kt)获取完整代码。

这段代码的输出如下：

```text
Child is sleeping
Throwing exception from scope
Child is cancelled
Caught assertion error
```
<!--- TEST-->

#### 监督协程中的异常

常规的作业和监督作业之间的另一个重要区别是异常处理。
监督协程中的每一个子作业应该通过异常处理机制处理自身的异常。
这种差异来自于子作业的执行失败不会传播给它的父作业的事实。
It means that coroutines launched directly inside [supervisorScope] _do_ use the [CoroutineExceptionHandler]
that is installed in their scope in the same way as root coroutines do
(see [CoroutineExceptionHandler](#coroutineexceptionhandler) section for details). 

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    supervisorScope {
        val child = launch(handler) {
            println("Child throws an exception")
            throw AssertionError()
        }
        println("Scope is completing")
    }
    println("Scope is completed")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-03.kt)获取完整代码。

这段代码的输出如下：

```text
Scope is completing
Child throws an exception
CoroutineExceptionHandler got java.lang.AssertionError
Scope is completed
```
<!--- TEST-->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineExceptionHandler]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[SupervisorJob()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[supervisorScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
<!--- END -->
