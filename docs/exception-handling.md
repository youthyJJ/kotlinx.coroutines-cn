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


这部分内容包括异常处理以及取消异常。
我们已经知道当协程被取消的时候会在挂起点抛出 [CancellationException]，并且它<!--
-->在协程机制中被忽略了。但是如果一个异常在取消期间被抛出或多个子协程在同一个<!--
-->父协程中抛出异常将会发生什么？

### 异常的传播

协程构建器有两种风格：自动的传播异常（[launch] 以及 [actor]）
或者将它们暴露给用户（[async] 以及 [produce]）。
前者对待异常是不处理的，类似于 Java 的 `Thread.uncaughtExceptionHandler`，
而后者依赖用户来最终消耗<!--
-->异常，比如说，通过 [await][Deferred.await] 或 [receive][ReceiveChannel.receive]
（[produce] 以及 [receive][ReceiveChannel.receive] 在[通道](https://www.kotlincn.net/docs/reference/coroutines/channels.html)中介绍过）。

可以通过一个在 [GlobalScope] 中创建协程的简单示例来进行演示：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // 我们将在控制台打印 Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
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

但是如果不想将所有的异常打印在控制台中呢？
[CoroutineExceptionHandler] 上下文元素被用来将通用的 `catch` 代码块用于在协程中自定义日志记录或异常处理。
它和使用 [`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 很相似。

在 JVM 中可以重定义一个全局的异常处理者来将所有的协程通过
[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) 注册到 [CoroutineExceptionHandler]。
全局异常处理者就如同
[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 
一样，在没有更多的指定的异常处理者被注册的时候被使用。
在 Android 中， `uncaughtExceptionPreHandler` 被设置在全局协程异常处理者中。

[CoroutineExceptionHandler] 仅在预计不会由用户处理的异常上调用，
所以在 [async] 构建器中注册它没有任何效果。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) {
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
Caught java.lang.AssertionError
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

如果协程遇到除 `CancellationException` 以外的异常，它将取消具有该异常的父协程。
这种行为不能被覆盖，且它被用来提供一个稳定的协程层次结构来进行<!--
-->[结构化并发](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/composing-suspending-functions.md#structured-concurrency-with-async)而无需依赖
[CoroutineExceptionHandler] 的实现。
且当所有的子协程被终止的时候，原本的异常被父协程所处理。

> 这也是为什么，在这个例子中，[CoroutineExceptionHandler] 总是被设置在由 [GlobalScope]
启动的协程中。将异常处理者设置在
[runBlocking] 主作用域内启动的协程中是没有意义的，尽管子协程已经设置了异常处理者，
但是主协程也总是会被取消的。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
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
Caught java.lang.ArithmeticException
```
<!--- TEST-->

### 异常聚合

如果一个协程的多个子协程抛出异常将会发生什么？
通常的规则是“第一个异常赢得了胜利”，所以第一个被抛出的异常将会暴露给处理者。
但也许这会是异常丢失的原因，比如说一个协程在 `finally` 块中抛出了一个异常。
这时，多余的异常将会被压制。

> 其中一个解决方法是分别抛出异常，
但是接下来 [Deferred.await] 应该有相同的机制来避免行为不一致并且会导致<!--
-->协程的实现细节（是否已将其部分工作委托给子协程）
泄漏到异常处理者中。


<!--- INCLUDE
import kotlinx.coroutines.exceptions.*
-->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                throw ArithmeticException()
            }
        }
        launch {
            delay(100)
            throw IOException()
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
Caught java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> 注意，这个机制当前只能在 Java 1.7 以上的版本中使用。
在 JS 和原生环境下暂时会受到限制，但将来会被修复。

取消异常是透明的并且会在默认情况下解包：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught original $exception")
    }
    val job = GlobalScope.launch(handler) {
        val inner = launch {
            launch {
                launch {
                    throw IOException()
                }
            }
        }
        try {
            inner.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e
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
Caught original java.io.IOException
```
<!--- TEST-->

### 监督

正如我们之前研究的那样，取消是一种双向机制，在协程的整个层次结构<!--
-->之间传播。但是如果需要单向取消怎么办？

此类需求的一个良好示例是在其作用域内定义作业的 UI 组件。如果任何一个 UI 的子作业<!--
-->执行失败了，它并不总是有必要取消（有效地杀死）整个 UI 组件，
但是如果 UI 组件被销毁了（并且它的作业也被取消了），由于它的结果不再被需要了，它有必要使所有的子作业执行失败。

另一个例子是服务进程孵化了一些子作业并且需要 _监督_
它们的执行，追踪它们的故障并在这些子作业执行失败的时候重启。

#### 监督作业

[SupervisorJob][SupervisorJob()] 可以被用于这些目的。它类似于常规的 [Job][Job()]，唯一的不同是：<!--
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

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
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
Caught java.lang.AssertionError
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
