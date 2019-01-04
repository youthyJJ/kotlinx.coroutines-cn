<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/BasicsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class BasicsGuideTest {
--> 

**目录**

<!--- TOC -->

* [协程基础](#协程基础)
  * [你的第一个协程程序](#你的第一个协程程序)
  * [桥接阻塞与非阻塞的世界](#桥接阻塞与非阻塞的世界)
  * [等待一个作业](#等待一个作业)
  * [结构化的并发](#结构化的并发)
  * [作用域构建器](#作用域构建器)
  * [提取函数重构](#提取函数重构)
  * [协程是轻量级的](#协程是轻量级的)
  * [像守护线程一样的全局协程](#像守护线程一样的全局协程)

<!--- END_TOC -->


## 协程基础

这一部分包括基础的协程概念。

### 你的第一个协程程序

运行以下代码：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L) // 非阻塞的等待 1 秒钟（默认时间单位是毫秒）
        println("World!") // 在延迟后打印输出
    }
    println("Hello,") // 协程已在等待时主线程还在继续
    Thread.sleep(2000L) // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)获得完整代码

代码运行的结果：

```text
Hello,
World!
```

<!--- TEST -->

本质上，协程是轻量级的线程。
它们在某些 [CoroutineScope] 上下文中与 [launch] _协程构建器_ 一起启动。
这里我们在 [GlobalScope] 中启动了一个新的协程，这意味着新协程的<!--
-->生命周期只受整个应用程序的生命周期限制。

可以将
`GlobalScope.launch { …… }` 替换为 `thread { …… }`，将 `delay(……)` 替换为 `Thread.sleep(……)` 达到同样目的。 尝试一下。

如果你首先将 `GlobalScope.launch` 替换为 `thread`，编译器会报以下错误：

```
Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
```

这是因为 [delay] 是一个特殊的 _挂起函数_ ，它不会造成线程阻塞，但是会 _挂起_
协程，并且只能在协程中使用。

### 桥接阻塞与非阻塞的世界

第一个示例在同一段代码中混用了 _非阻塞的_ `delay(……)` 与 _阻塞的_ `Thread.sleep(……)`。
这容易让我们记混哪个是阻塞的、哪个是非阻塞的。
让我们显式使用 [runBlocking] 协程构建器来阻塞：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主线程中的代码会立即执行
    runBlocking {     // 但是这个表达式阻塞了主线程
        delay(2000L)  // ……我们延迟 2 秒来保证 JVM 的存活
    } 
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-02.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->

结果是相似的，但是这些代码只使用了非阻塞的函数 [delay]。
调用了 `runBlocking` 的主线程会一直 _阻塞_ 直到 `runBlocking` 内部的协程执行完毕。

这个示例可以使用更合乎惯用法的方式重写，使用 `runBlocking` 来包装
main 函数的执行：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> { // 开始执行主协程
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主协程在这里会立即执行
    delay(2000L)      // 延迟 2 秒来保证 JVM 存活
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)获得完整代码

<!--- TEST
Hello,
World!
-->

这里的 `runBlocking<Unit> { …… }` 作为用来启动顶层主协程的适配器。
我们显式指定了其返回类型 `Unit`，因为在 Kotlin 中 `main` 函数必须返回 `Unit` 类型。

这也是为挂起函数编写单元测试的一种方式：

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

延迟一段时间来等待另一个协程运行并不是一个好的选择。让我们显式<!--
-->（以非阻塞方式）等待所启动的后台 [Job] 执行结束：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 等待直到子协程执行结束
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-03.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->

现在，结果仍然相同，但是主协程与后台作业的持续时间<!--
-->没有任何关系了。好多了。

### 结构化的并发

协程的实际使用还有一些需要改进的地方。
当我们使用 `GlobalScope.launch` 时，我们会创建一个顶层协程。虽然它很轻量，但它运行时仍会<!--
-->消耗一些内存资源。如果我们忘记保持对新启动的<!--
-->协程的引用，它还会继续运行。如果协程中的代码挂起了会怎么样（例如，我们错误地<!--
-->延迟了太长时间），如果我们启动了太多的协程并导致内存不足会怎么样？
必须手动保持对所有已启动协程的引用并 [join][Job.join] 之很容易出错。

有一个更好的解决办法。我们可以在代码中使用结构化并发。
我们可以在执行操作所在的指定作用域内启动协程，
而不是像通常使用线程（线程总是全局的）那样在 [GlobalScope] 中启动。

在我们的示例中，我们使用 [runBlocking] 协程构建器将  `main` 函数转换为协程。
包括 `runBlocking` 在内的每个协程构建器都将 [CoroutineScope] 的实例添加到其代码块所在的作用域中。
我们可以在这个作用域中启动协程而无需显式 `join` 之，因为<!--
-->外部协程（示例中的 `runBlocking`）直到在其作用域中启动的所有协程<!--
-->都执行完毕后才会结束。因此，可以将我们的示例简化为：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { // 在 runBlocking 作用域中启动新协程
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-03s.kt)来获取完整代码

<!--- TEST
Hello,
World!
-->

### 作用域构建器
除了由不同的构建器提供协程作用域之外，还可以使用
[coroutineScope] 构建器声明自己的作用域。它会创建新的协程作用域并且在所有已启动子协程执行完毕之前不会<!--
-->结束。[runBlocking] 与 [coroutineScope] 的主要区别在于后者<!--
-->在等待所有子协程执行完毕时不会阻塞当前线程。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建一个新的协程作用域
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌 launch 之前输出
    }
    
    println("Coroutine scope is over") // 这一行在内嵌 launch 执行完毕后才输出
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-04.kt)来获得完整代码

<!--- TEST
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
-->

### 提取函数重构

让我们在 `launch { …… }` 中提取代码块并分离到另一个函数中。当你<!--
-->在这段代码上展示“提取函数”函数的时候，你得到了一个新的函数并用 `suspend` 修饰。
这是你的第一个 _挂起函数_ 。挂起函数可以像一个普通的函数一样使用内部协程，但是它们拥有一些额外的特性，反过来说，
使用其它的挂起函数，比如这个示例中的 `delay`，可以使协程暂停执行。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// 你的第一个挂起函数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-05.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->


但是如果提取函数包含了一个调用当前作用域的协程构建器？
在这个示例中仅仅使用 `suspend` 来修饰提取出来的函数是不够的。在 `CoroutineScope` 调用 `doWorld` 方法<!--
-->是一种解决方案，但它并非总是适用，因为它不会使API看起来更清晰。
惯用的解决方法是使 `CoroutineScope` 在一个类中作为一个属性并包含一个目标函数，
或者使它外部的类实现 `CoroutineScope` 接口。
作为最后的手段，[CoroutineScope(coroutineContext)][CoroutineScope()] 也是可以使用的，但是这样的结构是不安全的，
因为你将无法在这个作用域内控制方法的执行。只有私有的API可以使用这样的写法。

### 协程是轻量级的

运行下面的代码:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // 启动大量的协程
        launch {
            delay(1000L)
            print(".")
        }
    }
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-06.kt)来获得完整代码

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

它启动了100,000个协程，并且每秒钟每个协程打印一个点。
现在，尝试使用线程来这么做。将会发生什么？（大多数情况下你的代码将会抛出内存溢出错误）

### 像守护线程一样的全局协程

下面的代码在 [GlobalScope] 中启动了一个长时间运行的协程，它在1秒内打印了“I'm sleeping”两次<!--
-->并且延迟一段时间后在main函数中返回：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 在延迟之后结束程序
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)来获取完整代码

你可以运行这个程序并在命令行中看到它打印出了如下三行：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

在 [GlobalScope] 中启动的活动中的协程就像守护线程一样，不能使它们所在的进程保活。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
<!--- END -->


