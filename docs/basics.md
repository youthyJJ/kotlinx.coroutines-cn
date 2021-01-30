<!--- TEST_NAME BasicsGuideTest -->

**目录**

<!--- TOC -->

- [协程基础](#协程基础)

    - [第一个协程程序](#第一个协程程序)

    - [桥接阻塞与非阻塞的世界](#桥接阻塞与非阻塞的世界)

    - [等待一个作业](#等待一个作业)

    - [结构化的并发](#结构化的并发)

    - [作用域构建器](#作用域构建器)

    - [提取函数重构](#提取函数重构)

    - [协程很轻量](#协程很轻量)

    - [全局协程像守护线程](#全局协程像守护线程)

<!--- END -->

## 协程基础

这一部分包括基础的协程概念。

### 第一个协程程序

运行以下代码：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch {   // 在后台启动一个新的协程并继续
        delay(1000L)       // 非阻塞的等待 1 秒钟（默认时间单位是毫秒）
        println("World!")  // 在延迟后打印输出
    }
    println("Hello,")      // 协程已在等待时主线程还在继续
    Thread.sleep(2000L)    // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt)获取完整代码。

代码运行的结果：

```text
Hello,
World!
```

<!--- TEST -->

本质上，协程是轻量级的线程。

它们在某些 [CoroutineScope] 上下文中与 [launch] ___协程构建器___ 一起启动。

上方代码中我们在 [GlobalScope] 中启动了一个新的协程，这意味着这个协程的生命周期只会受整个应用程序的生命周期限制。

你可以尝试将 `GlobalScope.launch { … }` 替换为 `thread { … }`，并将 `delay(…)` 替换为 `Thread.sleep(…)` ，这样也能够达到同样目的。

试试看吧（不要忘记导入 `kotlin.concurrent.thread` 哦）。

> 提示:
>
> 如果你只将 `GlobalScope.launch` 替换为 `thread`，编译器会报以下错误：
> 
> ```Log
> Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
> ```
> 
> 这是因为 [delay] 是一个特殊的 ___挂起函数___ ，它不会造成线程阻塞，但是会 _挂起_ 协程，并且只能在协程中使用。

### 桥接阻塞与非阻塞的世界

第一个示例在同一段代码中混用了 _非阻塞的_ `delay(…)` 与 _阻塞的_ `Thread.sleep(…)` 。

这容易让我们混淆哪个是阻塞的、哪个是非阻塞的。

让我们显式使用 [runBlocking] 协程构建器来阻塞：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch {  // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,")     // 主线程中的代码会立即执行
    runBlocking {         // 但是这个表达式阻塞了主线程
        delay(2000L)      // 我们延迟 2 秒来保证 JVM 的存活
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-02.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

结果是相似的，但是这些代码只使用了 _非阻塞式_ 的 [delay] 函数。

调用了 `runBlocking` 的主线程会一直 _阻塞_ 直到 `runBlocking` 内部的协程执行完毕。

上方的示例如果用以下的方式重写看起来更符合编码的习惯，使用 `runBlocking` 来包装 `main` 函数的执行：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> { // 开始执行主协程
    GlobalScope.launch {         // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,")            // 主协程在这里会立即执行
    delay(2000L)                 // 延迟 2 秒来保证 JVM 存活
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-03.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

> 注意
> 
> 上方的代码中我们使用 `runBlocking<Unit> { … }` 作为用来启动 _顶层主协程_ 的 ___适配器___。
> 
> 我们显式指定了其返回类型 `Unit`，因为在 Kotlin 中 `main` 函数必须返回 `Unit` 类型。
> 
> 这也是为挂起函数编写单元测试的一种方式：

<!--- INCLUDE
import kotlinx.coroutines.*
-->

<div class="sample" markdown="1" theme="idea" data-highlight-only>
 
```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // 这里我们可以使用任何喜欢的断言风格来使用挂起函数
    }
}
```

</div>

<!--- CLEAR -->
 
### 等待一个作业

通过 `delay(…)` 函数延迟一段时间来等待另一个协程运行并不是一个好的选择。让我们显式（以非阻塞方式）等待所启动的后台 [Job] 执行结束：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {         // 启动一个主协程
    val job = GlobalScope.launch { // 启动一个子协程持有它的引用
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join()                     // 通过join等待子协程执行结束
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-04.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

现在，打印的结果仍然是相同的，但是主协程不用再掐着表计算正在后台作业的子协程需要多少时间了。好多了。

### 结构化的并发

协程的实际使用还有一些需要改进的地方。

当我们使用 `GlobalScope.launch` 时，我们会创建一个 _顶层主协程_ 。虽然它很轻量，但它运行时仍会消耗一些内存资源。

- 如果我们忘记持有新启动的协程的引用，它还会在后台继续运行。

- 如果协程中的代码挂起了会怎么样（例如，我们错误地延迟了太长时间）？

- 如果我们启动了太多的协程并导致内存不足会怎么样？

这要求我们必须手动持有所有已启动协程的引用，还要调用它们的 [join][Job.join] 函数，这很容易出错。

有一个更好的解决办法。我们可以在代码中使用 ___结构化并发___ 。

我们可以在一个已有的协程作用域中启动新的子协程，而不是像通常使用线程（线程总是全局的）那样在 [GlobalScope] 中启动。

在下方的示例中，我们使用 [runBlocking] 协程构建器将  `main` 函数转换为协程。

值得说明的是包括 `runBlocking` 在内，所有的协程构建器都会将 [CoroutineScope] 的实例附加到他们的代码块所在的作用域中。

我们可以在这个作用域中启动协程而无需显式地调用 `join` 函数，因为外部主协程（示例中的 `runBlocking`）直到在其作用域中启动的所有子协程都执行完毕后才会结束。因此，可以将我们的示例简化为：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  // this: CoroutineScope
    launch {                // 在 runBlocking 的作用域中启动一个新协程
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-05.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

### 作用域构建器

除了由不同的构建器提供协程作用域之外，还可以使用 [coroutineScope][_coroutineScope] 构建器声明自己的作用域。它会创建一个协程作用域，并且会在这个作用域内已启动子协程全部执行完毕才结束。

> 注意
>
> [runBlocking] 与 [coroutineScope][_coroutineScope] 可能看起来很类似，因为它们都会等待自身的 _协程代码块_ 以及 _作用域中启动的所有子协程_ 结束。
> 
> 他们的主要区别在于，[runBlocking] 方法会 _阻塞_ 当前线程来等待，而 [coroutineScope][_coroutineScope] 不会，它只是在协程中挂起，会释放底层线程用于其他用途，不会 _阻塞_ 线程。
> 
> 由于存在这点差异，[runBlocking] 是 _常规函数(fun)_，而 [coroutineScope][_coroutineScope] 是 _挂起函数(suspend fun)_。

可以通过以下示例来演示：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    coroutineScope {                         // 创建一个协程作用域
        launch {                             // 启动一个内嵌协程
            delay(500L)
            println("Task from nested launch")
        }

        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌协程执行"前"输出
    }

    println("Coroutine scope is over")       // 这一行会在内嵌协程执行"后"输出
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-06.kt)获取完整代码。

<!--- TEST
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
-->

> 注意
>
> 上方的代码中，当等待内嵌协程延迟执行(500ms)时，控制台在输出了 “Task from coroutine scope” 之后，很快就执行并输出 “Task from runBlocking” ，尽管此时 [coroutineScope][_coroutineScope] 尚未结束。

### 提取函数重构

我们试着将  `launch { … }` 内部的代码块抽取到独立的函数中。当你对这段代码执行“抽取函数”重构时，你会得到一个带有 `suspend` 修饰符的新函数。

这是你的第一个 ___挂起函数___ 。在协程内部，你可以像调用普通函数一样使用挂起函数，在挂起函数内部，同样可以使用其他挂起函数（如本例中的 `delay` ）来 _挂起_ 协程的执行。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// 这或许是你的第一个挂起函数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-07.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

> 注意
> 
> 如果在想要抽取为函数的代码块中，包含了一个在当前协程作用域中调用的协程构建器的话，该怎么办？
> 
> 在这种情况下，抽取出来的函数上只有 `suspend` 修饰符是不够的，因为调用协程构建器方法需要一个 `CoroutineScope` 。
>
> 有一种方式是为 `CoroutineScope` 写一个 `doWorld` 扩展方法，但这可能并非总是适用，因为它并没有使 API 更加清晰。
> 
> 惯用的解决方案是要么显式将 `CoroutineScope` 提升为 __包含了该函数的类的一个   字段__ ，要么当外部类实现了 `CoroutineScope` 时隐式取得。
> 
> 作为最后的手段，可以使用 [CoroutineScope(coroutineContext)][CoroutineScope()]，不过这种方法结构上不安全，因为你不能再控制该方法执行的作用域。只有私有 API 才能使用这个构建器。

### 协程很轻量

运行以下代码：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // 启动大量的协程
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-08.kt)获取完整代码。

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

它启动了 10 万个协程，并且在 5 秒钟后，每个协程都输出一个点。

现在，尝试使用线程来实现。会发生什么？（很可能你的代码会产生某种内存不足的错误）

### 全局协程像守护线程

以下代码在 [GlobalScope] 中启动了一个长期运行的协程，该协程每秒输出“I'm sleeping”两次，之后在主函数中延迟一段时间后返回。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 在延迟后退出
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-09.kt)获取完整代码。

你可以运行这个程序并看到它输出了以下三行后终止：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

在 [GlobalScope] 中启动的活动协程并不会使进程保活。

它们就像守护线程，这也就是 [第一个协程程序](#第一个协程程序) 中需要调用 Thread 的 `sleep` 函数阻塞线程的原因。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
<!--- END -->


