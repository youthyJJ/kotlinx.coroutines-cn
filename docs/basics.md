<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.$$1$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/BasicsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.experimental.guide.test

import org.junit.Test

class BasicsGuideTest {
--> 

## 目录

<!--- TOC -->

* [协程基础](#coroutine-basics)
  * [你的第一个协程程序](#your-first-coroutine)
  * [桥接阻塞和非阻塞的世界](#bridging-blocking-and-non-blocking-worlds)
  * [等待一个任务](#waiting-for-a-job)
  * [结构性的并发](#structured-concurrency)
  * [范围建造器](#scope-builder)
  * [提取函数重构](#extract-function-refactoring)
  * [协程是轻量级的](#coroutines-are-light-weight)
  * [像守护线程一样的全局协程](#global-coroutines-are-like-daemon-threads)

<!--- END_TOC -->


## 协程基础

这一部分包括基础的协程概念。

### 你的第一个协程程序

运行以下代码:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) {
    GlobalScope.launch { // launch new coroutine in background and continue
        delay(1000L) // 无阻塞的等待1秒钟(默认时间单位是好喵)
        println("World!") // 在延迟后打印输出
    }
    println("Hello,") // 主线程的协程将会继续等待
    Thread.sleep(2000L) // 阻塞主线程2秒钟来保证JVM存活
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-01.kt)获得完整代码

代码运行的结果:

```text
Hello,
World!
```

<!--- TEST -->

本质上, 协程是轻量级的线程.
它们在[CoroutineScope]上下文中和[launch] _协同构造器_一起被启动。 
这里我们在[GlobalScope]中启动了一些新的协程, 存活时间是指新的协程的存活时间被限制在了整个应用的存活时间之内。  

你可以使用一些协程操作来替换一些线程操作，比如：
用`GlobalScope.launch { ... }` 替换 `thread { ... }` 用 `delay(...)` 替换 `Thread.sleep(...)`。 尝试一下。

如果你开始使用 `GlobalScope.launch` 来替换 `thread`, 编译器将会抛出错误:

```
Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
```

这是因为 [delay] 是一个特别的 _暂停函数_ ,它不会造成线程阻塞, 但是 _暂停_
函数只能在协程中使用.

### 桥接阻塞和非阻塞的世界

第一个例子中在相似的代码中包含了 _非阻塞的_ `delay(...)` 和 _阻塞的_ `Thread.sleep(...)`。 
它非让容易的让我们看出来哪一个是阻塞的，哪一个是非阻塞的. 
来一起使用明确的阻塞[runBlocking]协程构造器:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) { 
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主线程中的代码会立即执行
    runBlocking {     // 但是这个函数阻塞了主线程
        delay(2000L)  // ...我们延迟2秒来保证JVM的存活
    } 
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-02.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->

结果是相似的，但是这些代码只使用了非阻塞的函数[delay]. 
在主线程中调用了`runBlocking`, _阻塞_ 会持续到`runBlocking`中的协程执行完毕。 

这个例子可以使用更多的惯用方法来重写, 使用`runBlocking`来包装main方法:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking { // 开始执行主协程
    GlobalScope.launch { // 在后台开启一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主协程在这里会立即执行
    delay(2000L)      // 延迟2秒来保证JVM存活
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-02b.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->

这里的`runBlocking<Unit> { ... }`作为一个适配器被用来启动最高优先级的主协程。 
我们明确的声明`Unit`为返回值类型, 因为Kotlin中的`main`函数返回`Unit`类型。

这也是一种使用暂停函数来实现单元测试的方法:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
 
```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // 这里我们可以使用暂停函数来实现任何我们喜欢的断言风格
    }
}
```

</div>

<!--- CLEAR -->
 
### 等待一个任务

延迟一段时间来等待另一个协程开始工作并不是一个好的选择。让我们明确地等待
(使用非阻塞的方法)一个后台[Job]执行结束:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = GlobalScope.launch { // 启动一个新的协程并保持对这个任务的引用
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 等待直到子协程执行结束
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-03.kt)来获得完整代码

<!--- TEST
Hello,
World!
-->

现在, 结果仍然相同, 但是主协程与后台任务的持续时间没有任何关系. 这样写会更好.

### 结构性的并发

这里还有一些东西我们期望的写法被使用在协程的练习中。 
当我们使用`GlobalScope.launch`时我们创建了一个最高优先级的协程. 甚至, 虽然它是轻量级的, 
但是它在运行起来的时候仍然消耗了一些内存资源. 甚至如果我们失去了一个对新创建的协程的引用, 它仍然会继续运行. 
如果一段代码在协程中挂起(举例来说, 我们错误的延迟了太长时间), 如果我们启动了太多的协程, 是否会导致内存溢出？ 
如果我们手动引用所有的协程和[join][Job.join]是非常容易出错的. 

这有一个更好的解决办法. 我们可以在你的代码中使用结构性并发. 
用来代替在[GlobalScope]中启动协程, 就像我们使用线程时那样(线程总是全局的), 
我们可以在一个具体的范围中启动协程并操作. 

在我们的例子中, 我们有一个被转换成使用[runBlocking]的协程构造器的`main`函数
每一个协程构造器, 包括`runBlocking`, 添加了一个实例在[CoroutineScope]范围的代码块中. 
我们可以在一个协程还没有明确的调用`join`之前在这个范围内启动它们, 因为一个外部的协程
(我们的例子中的`runBlocking`) 没有在所有的协程在它们的范围内启动完成后执行完毕, 从而, 我们可以使我们的例子更简单:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking { // this: CoroutineScope
    launch { // launch new coroutine in the scope of runBlocking
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

### 范围构造器
除了由不同的构造器提供的协程范围, 也是可以使用
[coroutineScope] 构造器来声明你自己的范围. 它启动了一个新的协程范围并且在所有子协程执行结束后并没有执行完毕. 
[runBlocking]和[coroutineScope]主要的不同之处在于后者在等待所有的子协程执行完毕时并没有使当前线程阻塞.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建一个新的协程范围
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // 该行将在嵌套启动之前执行打印
    }
    
    println("Coroutine scope is over") // 该行将在嵌套结束之后才会被打印
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

让我们在`launch { ... }`中提取代码块并分离到另一个函数中. 当你 
在这段代码上展示"提取函数"函数的时候, 你的到了一个新的函数并用`suspend`修饰.
这是你的第一个 _暂停函数_. 暂停函数可以像一个普通的函数一样使用内部协程, 但是它们拥有一些额外的特性, 反过来说, 
使用其它的暂停函数, 比如这个例子中的`delay`, 可以使协程暂停执行.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// 你的第一个暂停函数
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


但是如果提取函数包含了一个调用当前范围的协程构造器?
在这个例子中仅仅使用`suspend`来修饰提取出来的函数是不够的. 在`CoroutineScope`调用`doWorld`方法 
是一种解决方案, 但它并非总是适用, 因为它不会使API看起来更清晰.
惯用的解决方法是使`CoroutineScope`在一个类中作为一个属性并包含一个目标函数, 
或者使它外部的类实现`CoroutineScope`接口.
作为最后的手段, [CoroutineScope(coroutineContext)][CoroutineScope()] 也是可以使用的, 但是这样的结构是不安全的 
因为你将无法在这个范围内控制方法的执行. 只有私有的API可以使用这样的写法.

### 协程是轻量级的

运行下面的代码:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking {
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

它启动了100,000个协程, 并且每秒钟每个协程打印一个点. 
现在, 尝试使用线程来这么做. 将会发生什么? (大多数情况下你的代码将会抛出内存溢出错误)

### 像守护线程一样的全局协程

下面的代码在[GlobalScope]中启动了一个长时间运行的协程, 它在1秒内打印了 "I'm sleeping"两次并且延迟一段时间后在main函数中返回:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun main(args: Array<String>) = runBlocking {
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 在延迟之后结束程序
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-basic-07.kt)来获取完整代码

你可以运行这个程序并在命令行中看到它打印出了如下三行:

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

在[GlobalScope]中启动的活动中的协程就像守护线程一样, 不能使它们所在的进程保活。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/join.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/coroutine-scope.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope.html
<!--- END -->


