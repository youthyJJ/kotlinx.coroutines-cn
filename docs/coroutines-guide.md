
Kotlin，作为一门编程语言，仅在标准库中提供了细粒度的底层API来供其它的库来使用协程。
不像很多其它的编程语言提供了类似的功能，比如`async` 和 `await`<!--
-->在 Kotlin 中并不是关键字，甚至不是标准库的一部分. 此外, Kotlin 的
_暂停函数_ 概念提供了相比 futures 和 promises 更安全和更不容易出错的方式<!--
-->来实现异步操作。

`kotlinx.coroutines` 是 JetBrain 提供的一个功能丰富的协程库。这个教程覆盖了 `kotlinx.coroutines` 所包含的大量协程使用原语，
包括了 `launch`, `async` 等等. 

这个教程分为不同的话题介绍了 `kotlinx.coroutines` 核心库并包含一些列的例子。

为了使用协程并运行这些代码示例, 你需要在你的模块中添加`kotlinx-coroutines-core`依赖, 详细的解释请见
[项目的README](../README.md#using-in-your-projects).

## 内容目录

* [协程基础](basics.md)
* [取消与超时](cancellation-and-timeouts.md)
* [组合挂起函数](composing-suspending-functions.md)
* [协程上下文与调度器](coroutine-context-and-dispatchers.md)
* [异常处理](exception-handling.md)
* [通道(试验性的)](channels.md)
* [共享的可变状态与并发](shared-mutable-state-and-concurrency.md)
* [Select表达式(试验性的)](select-expression.md)

## 补充引用

* [使用协程来编写UI程序的指南](../ui/coroutines-guide-ui.md)
* [使用协程来编写响应式流的指南](../reactive/coroutines-guide-reactive.md)
* [协程设计文档](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
* [完整的协程API教程](http://kotlin.github.io/kotlinx.coroutines)