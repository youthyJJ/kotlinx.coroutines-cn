<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2019 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../kotlinx-coroutines-core/jvm/test/guide/.*-##\.kt -->
<!--- TEST_OUT ../kotlinx-coroutines-core/jvm/test/guide/test/FlowGuideTest.kt
// This file was automatically generated from flow.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class FlowGuideTest {
--> 

**目录**

<!--- TOC -->

* [异步流](#异步流)
  * [表示多个值](#表示多个值)
    * [序列](#序列)
    * [挂起函数](#挂起函数)
    * [流](#流)
  * [流是冷的](#流是冷的)
  * [流取消](#流取消)
  * [流构建器](#流构建器)
  * [过渡流操作符](#过渡流操作符)
    * [转换操作符](#转换操作符)
    * [限长操作符](#限长操作符)
  * [末端流操作符](#末端流操作符)
  * [流是连续的](#流是连续的)
  * [流上下文](#流上下文)
    * [withContext 发出错误](#withcontext-发出错误)
    * [flowOn 操作符](#flowon-操作符)
  * [缓冲](#缓冲)
    * [合并](#合并)
    * [处理最新值](#处理最新值)
  * [组合多个流](#组合多个流)
    * [Zip](#zip)
    * [Combine](#combine)
  * [展平流](#展平流)
    * [flatMapConcat](#flatmapconcat)
    * [flatMapMerge](#flatmapmerge)
    * [flatMapLatest](#flatmaplatest)
  * [流异常](#流异常)
    * [收集器 try 与 catch](#收集器-try-与-catch)
    * [一切都已捕获](#一切都已捕获)
  * [异常透明性](#异常透明性)
    * [透明捕获](#透明捕获)
    * [声明式捕获](#声明式捕获)
  * [流完成](#流完成)
    * [命令式 finally 块](#命令式-finally-块)
    * [声明式处理](#声明式处理)
    * [仅限上游异常](#仅限上游异常)
  * [命令式还是声明式](#命令式还是声明式)
  * [启动流](#启动流)
  * [流（Flow）与响应式流（Reactive Streams）](#flow-and-reactive-streams)

<!--- END_TOC -->

## 异步流

挂起函数可以异步的返回单个值，但是该如何异步返回<!--
-->多个计算好的值呢？这正是 Kotlin 流（Flow）的用武之地。

### 表示多个值

在 Kotlin 中可以使用[集合][collections]来表示多个值。 
比如说，我们可以拥有一个函数 `foo()`，它返回一个包含三个数字的 [List]，
然后使用 [forEach] 打印它们：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun foo(): List<Int> = listOf(1, 2, 3)
 
fun main() {
    foo().forEach { value -> println(value) } 
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-01.kt)获取完整代码。

这段代码输出如下： 

```text
1
2
3
```

<!--- TEST -->

#### 序列

如果使用一些消耗 CPU 资源的阻塞代码计算数字<!--
-->（每次计算需要 100 毫秒）那么我们可以使用 [Sequence] 来表示数字：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
fun foo(): Sequence<Int> = sequence { // 序列构建器
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们正在计算
        yield(i) // 产生下一个值
    }
}

fun main() {
    foo().forEach { value -> println(value) } 
}
```  

</div>

> 你可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-02.kt)获取完整代码。

这段代码输出相同的数字，但在打印每个数字之前等待 100 毫秒。

<!--- TEST 
1
2
3
-->

#### 挂起函数

然而，计算过程阻塞运行该代码的主线程。
当这些值由异步代码计算时，我们可以使用 `suspend` 修饰符标记函数 `foo`，
这样它就可以在不阻塞的情况下执行其工作并将结果作为列表返回：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*                 
                           
//sampleStart
suspend fun foo(): List<Int> {
    delay(1000) // 假装我们在这里做了一些异步的事情
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    foo().forEach { value -> println(value) } 
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-03.kt)获取完整代码。

这段代码将会在等待一秒之后打印数字。

<!--- TEST 
1
2
3
-->

#### 流

使用 List<Int> 结果类型，意味着我们只能一次返回所有值。
为了表示异步计算的值流（stream），我们可以使用 Flow<Int> 类型（正如同步计算值会使用 Sequence<Int> 类型）：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart               
fun foo(): Flow<Int> = flow { // 流构建器
    for (i in 1..3) {
        delay(100) // 假装我们在这里做了一些有用的事情
        emit(i) // 发送下一个值
    }
}

fun main() = runBlocking<Unit> {
    // 启动并发的协程以验证主线程并未阻塞
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // 收集这个流
    foo().collect { value -> println(value) } 
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-04.kt)获取完整代码。

这段代码在不阻塞主线程的情况下每等待 100 毫秒打印一个数字。在主线程中运行一个<!--
-->单独的协程每 100 毫秒打印一次 “I'm not blocked” 已经经过了验证。

```text
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

<!--- TEST -->

注意使用 [Flow] 的代码与先前示例的下述区别：

* 名为 [flow] 的 [Flow] 类型构建器函数。
* `flow { ... }` 构建块中的代码可以挂起。
* 函数 `foo()` 不再标有 `suspend` 修饰符。  
* 流使用 [emit][FlowCollector.emit] 函数 _发射_ 值。 
* 流使用 [collect][collect] 函数 _收集_ 值。

> 我们可以在 `foo` 的 `flow { ... }` 函数体内使用 [delay] 代替 `Thread.sleep`
以观察主线程在本案例中被阻塞了。

### 流是冷的

Flow 是一种类似于序列的冷流 &mdash; 这段 [flow] 构建器中的代码<!--
-->直到流被收集的时候才运行。这在以下的示例中非常明显：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart      
fun foo(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling foo...")
    val flow = foo()
    println("Calling collect...")
    flow.collect { value -> println(value) } 
    println("Calling collect again...")
    flow.collect { value -> println(value) } 
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-05.kt)获取完整代码。

打印如下：

```text
Calling foo...
Calling collect...
Flow started
1
2
3
Calling collect again...
Flow started
1
2
3
```

<!--- TEST -->
 
这是返回一个流的 `foo()` 函数没有标记 `suspend` 修饰符的主要原因。
通过它自己，`foo()` 会尽快返回且不会进行任何等待。该流在每次收集的时候启动，
这就是为什么当我们再次调用 `collect` 时我们会看到“Flow started”。

### 流取消

流采用与协程同样的协作取消。然而，流的基础设施未引入<!--
-->其他取消点。取消完全透明。像往常一样，流的收集可以在<!--
-->当流在一个可取消的挂起函数（例如 [delay]）中挂起的时候取消，否则不能取消。

下面的示例展示了当 [withTimeoutOrNull] 块中代码在运行的时候流是如何在超时的情况下取消<!--
-->并停止执行其代码的：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart           
fun foo(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // 在 250 毫秒后超时
        foo().collect { value -> println(value) } 
    }
    println("Done")
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-06.kt)获取完整代码。

注意，在 `foo()` 函数中流仅发射两个数字，产生以下输出：

```text
Emitting 1
1
Emitting 2
2
Done
```

<!--- TEST -->

### 流构建器

先前示例中的 `flow { ... }` 构建器是最基础的一个。还有其它构建器<!--
-->使流的声明更简单：

* [flowOf] 构建器定义了一个发射固定值集的流。
* 使用 `.asFlow()`  扩展函数，可以将各种集合与序列转换为流。

因此，从流中打印从 1 到 3 的数字的示例可以写成：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    // 将一个整数区间转化为流
    (1..3).asFlow().collect { value -> println(value) }
//sampleEnd 
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-07.kt)获取完整代码。
 
<!--- TEST
1
2
3
-->

### 过渡流操作符

可以使用操作符转换流，就像使用集合与序列一样。
过渡操作符应用于上游流，并返回下游流。
这些操作符也是冷操作符，就像流一样。这类操作符<!--
-->本身不是挂起函数。它运行的速度很快，返回新的转换流的定义。

基础的操作符拥有相似的名字，比如 [map] 与 [filter]。
流与序列的主要区别在于这些操作符中的代码<!--
-->可以调用挂起函数。

举例来说，一个请求中的流可以<!--
-->使用 [map] 操作符映射出结果，即使执行一个长时间的请求<!--
-->操作也可以使用挂起函数来实现： 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart           
suspend fun performRequest(request: Int): String {
    delay(1000) // 模仿长时间运行的异步工作
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // 一个请求流
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-08.kt)获取完整代码。

它产生以下三行，每一行每秒出现一次：

```text                                                                    
response 1
response 2
response 3
```

<!--- TEST -->

#### 转换操作符

在流转换操作符中，最通用的一种称为 [transform]。它可以用来模仿<!--
-->简单的转换，例如 [map] 与 [filter]，以及实施更复杂的转换。
使用 `transform` 操作符，我们可以 [发射][FlowCollector.emit] 任意值任意次。

比如说，使用 `transform` 我们可以在执行长时间运行的异步请求之前发射一个字符串<!--
-->并跟踪这个响应：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

suspend fun performRequest(request: Int): String {
    delay(1000) // 模仿长时间运行的异步任务
    return "response $request"
}

fun main() = runBlocking<Unit> {
//sampleStart
    (1..3).asFlow() // 一个请求流
        .transform { request ->
            emit("Making request $request") 
            emit(performRequest(request)) 
        }
        .collect { response -> println(response) }
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-09.kt)获取完整代码。

这段代码的输出如下：

```text
Making request 1
response 1
Making request 2
response 2
Making request 3
response 3
```

<!--- TEST -->

#### 限长操作符

限长过渡操作符（例如 [take]）在流触及相应限制的时候会将它的<!--
-->执行取消。协程中的取消操作总是通过抛出异常来执行，这样所有的资源管理<!--
-->函数（如 `try {...} finally {...}` 块）会在取消的情况下正常运行：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun numbers(): Flow<Int> = flow {
    try {                          
        emit(1)
        emit(2) 
        println("This line will not execute")
        emit(3)    
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers() 
        .take(2) // 只获取前两个
        .collect { value -> println(value) }
}            
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-10.kt)获取完整代码。

这段代码的输出清楚地表明，`numbers()` 函数中对 `flow {...}` 函数体的执行在<!--
-->发射出第二个数字后停止：

```text       
1
2
Finally in numbers
```

<!--- TEST -->

### 末端流操作符

Terminal operators on flows are _suspending functions_ that start a collection of the flow.（这句话原文有歧义，先保留英文）
[collect] 是最基础的末端操作符，但是还有另外一些更方便使用的末端操作符：

* 转化为各种集合，例如 [toList] 与 [toSet]。
* 获取第一个（[first]）值与确保流发射单个（[single]）值的操作符。
* 使用 [reduce] 与 [fold] 将流规约到单个值。

举例来说：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart         
    val sum = (1..5).asFlow()
        .map { it * it } // 数字 1 至 5 的平方                        
        .reduce { a, b -> a + b } // 求和（末端操作符）
    println(sum)
//sampleEnd     
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-11.kt)获取完整代码。

打印单个数字：

```text
55
```

<!--- TEST -->

### 流是连续的

流的每次单独收集都是按顺序执行的，除非进行特殊操作的操作符<!--
-->使用多个流。该收集过程直接在协程中运行，该协程调用末端操作符。
默认情况下不启动新协程。
从上游到下游每个过渡操作符都会处理每个发射出的值<!--
-->然后再交给末端操作符。

请参见以下示例，该示例过滤偶数并将其映射到字符串：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart         
    (1..5).asFlow()
        .filter {
            println("Filter $it")
            it % 2 == 0              
        }              
        .map { 
            println("Map $it")
            "string $it"
        }.collect { 
            println("Collect $it")
        }    
//sampleEnd                  
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-12.kt)获取完整代码。

执行：

```text
Filter 1
Filter 2
Map 2
Collect string 2
Filter 3
Filter 4
Map 4
Collect string 4
Filter 5
```

<!--- TEST -->

### 流上下文

流的收集总是在调用协程的上下文中发生。例如，如果有一个流
`foo`，然后以下代码在它的编写者指定的上下文中<!--
-->运行，而无论流 `foo` 的实现细节如何：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
withContext(context) {
    foo.collect { value ->
        println(value) // 运行在指定上下文中
    }
}
``` 

</div>

<!--- CLEAR -->

流的该属性称为 _上下文保存_ 。

所以默认的，`flow { ... }` 构建器中的代码运行在相应流的收集器<!--
-->提供的上下文中。举例来说，考虑打印线程的 `foo` 的实现，
它被调用并发射三个数字：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
//sampleStart
fun foo(): Flow<Int> = flow {
    log("Started foo flow")
    for (i in 1..3) {
        emit(i)
    }
}  

fun main() = runBlocking<Unit> {
    foo().collect { value -> log("Collected $value") } 
}            
//sampleEnd
```                

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-13.kt)获取完整代码。

运行这段代码：

```text  
[main @coroutine#1] Started foo flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

<!--- TEST FLEXIBLE_THREAD -->

由于 `foo().collect` 是在主线程调用的，则 `foo` 的流主体也是在主线程调用的。
这是快速运行或异步代码的理想默认形式，它不关心执行的上下文并且不会<!--
-->阻塞调用者。

#### withContext 发出错误

然而，长时间运行的消耗 CPU 的代码也许需要在 [Dispatchers.Default] 上下文中执行，并且更新 UI
的代码也许需要在 [Dispatchers.Main] 中执行。通常，[withContext] 用于在
Kotlin 协程中改变代码的上下文，但是 `flow {...}` 构建器中的代码必须遵循上下文<!--
-->保存属性，并且不允许从其他上下文中发射（[emit][FlowCollector.emit]）。

尝试运行下面的代码：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
                      
//sampleStart
fun foo(): Flow<Int> = flow {
    // 在流构建器中更改消耗 CPU 代码的上下文的错误方式
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
            emit(i) // 发射下一个值
        }
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value -> println(value) } 
}            
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-14.kt)获取完整代码。

这段代码产生如下的异常：

```text
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
		Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
		but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, DefaultDispatcher].
		Please refer to 'flow' documentation or use 'flowOn' instead
	at ...
``` 

<!--- TEST EXCEPTION -->

#### flowOn 操作符
   
例外的是 [flowOn] 函数，该函数用于更改流发射的上下文。
以下示例展示了更改流上下文的正确方法，该示例还通过<!--
-->打印相应线程的名字以展示它们的工作方式：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // 假装我们以消耗 CPU 的方式进行计算
        log("Emitting $i")
        emit(i) // 发射下一个值
    }
}.flowOn(Dispatchers.Default) // 在流构建器中改变消耗 CPU 代码上下文的正确方式

fun main() = runBlocking<Unit> {
    foo().collect { value ->
        log("Collected $value") 
    } 
}            
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-15.kt)获取完整代码。
  
注意，当收集发生在主线程中，`flow { ... }` 是如何在后台线程中工作的：
  
<!--- TEST FLEXIBLE_THREAD
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
-->

这里要观察的另一件事是 [flowOn] 操作符已改变流的默认顺序性。
现在收集发生在一个协程中（“coroutine#1”）而发射发生在运行于另一个线程中<!--
-->与收集协程并发运行的另一个协程（“coroutine#2”）中。当上游流<!--
-->必须改变其上下文中的 [CoroutineDispatcher] 的时候，[flowOn] 操作符创建了另一个协程。

### 缓冲

从收集流所花费的时间来看，将流的不同部分运行在不同的协程中<!--
-->将会很有帮助，特别是当涉及到长时间运行的异步操作时。例如，考虑一种情况，
`foo()` 流的发射很慢，它每花费 100 毫秒才产生一个元素；而收集器也非常慢， 
需要花费 300 毫秒来处理元素。让我们看看从该流收集三个数字要花费多长时间：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        foo().collect { value -> 
            delay(300) // 假装我们花费 300 毫秒来处理它
            println(value) 
        } 
    }   
    println("Collected in $time ms")
}
//sampleEnd
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-16.kt)获取完整代码。

它会产生这样的结果，整个收集过程大约需要 1200 毫秒（3 个数字，每个花费 400 毫秒）：

```text
1
2
3
Collected in 1220 ms
```

<!--- TEST ARBITRARY_TIME -->

我们可以在流上使用 [buffer] 操作符来并发运行 `foo()` 中发射元素的代码以及收集的代码，
而不是顺序运行它们：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .buffer() // 缓冲发射项，无需等待
            .collect { value -> 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println(value) 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-17.kt)获取完整代码。

它产生了相同的数字，只是更快了，由于我们高效地创建了处理流水线，
仅仅需要等待第一个数字产生的 100 毫秒以及处理每个数字各需<!--
-->花费的 300 毫秒。这种方式大约花费了 1000 毫秒来运行：

```text
1
2
3
Collected in 1071 ms
```                    

<!--- TEST ARBITRARY_TIME -->

> 注意，当必须更改 [CoroutineDispatcher] 时，[flowOn] 操作符使用了相同的缓冲机制，
但是我们在这里显式地请求缓冲而不改变执行上下文。

#### 合并

当流代表部分操作结果或操作状态更新时，可能没有必要<!--
-->处理每个值，而是只处理最新的那个。在本示例中，当收集器处理它们太慢的时候，
[conflate] 操作符可以用于跳过中间值。构建前面的示例：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .conflate() // 合并发射项，不对每个值进行处理
            .collect { value -> 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println(value) 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-18.kt)获取完整代码。

我们看到，虽然第一个数字仍在处理中，但第二个和第三个数字已经产生，因此<!--
-->第二个是 _conflated_ ，只有最新的（第三个）被交付给收集器：

```text
1
3
Collected in 758 ms
```             

<!--- TEST ARBITRARY_TIME -->   

#### 处理最新值

当发射器和收集器都很慢的时候，合并是加快处理速度的一种方式。它通过删除发射值来实现。
另一种方式是取消缓慢的收集器，并在每次发射新值的时候重新启动它。有<!--
-->一组与 `xxx` 操作符执行相同基本逻辑的 `xxxLatest` 操作符，但是在新值产生的时候<!-- but cancel the
-->取消执行其块中的代码。让我们在先前的示例中尝试更换 [conflate] 为 [collectLatest]：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // 假装我们异步等待了 100 毫秒
        emit(i) // 发射下一个值
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .collectLatest { value -> // 取消并重新发射最后一个值
                println("Collecting $value") 
                delay(300) // 假装我们花费 300 毫秒来处理它
                println("Done $value") 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-19.kt)获取完整代码。
 
由于 [collectLatest] 的函数体需要花费 300 毫秒，但是新值每 100 秒发射一次，我们看到该代码块<!-- 
-->对每个值运行，但是只收集最后一个值：

```text 
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
``` 

<!--- TEST ARBITRARY_TIME -->

### 组合多个流

组合多个流有很多种方式。

#### Zip

就像 Kotlin 标准库中的 [Sequence.zip] 扩展函数一样，
流拥有一个 [zip] 操作符用于组合两个流中的相关值：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow() // 数字 1..3
    val strs = flowOf("one", "two", "three") // 字符串
    nums.zip(strs) { a, b -> "$a -> $b" } // 组合单个字符串
        .collect { println(it) } // 收集并打印
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-20.kt)获取完整代码。

示例打印如下：

```text
1 -> one
2 -> two
3 -> three
```
 
<!--- TEST -->

#### Combine

当流表示一个变量或操作的最新值时（请参阅<!--
-->相关小节 [conflation](#合并)），可能需要执行计算，这依赖于<!--
-->相应流的最新值，并且每当上游流产生值的时候<!--
-->都需要重新计算。这种相应的操作符家族称为 [combine]。

例如，先前示例中的数字如果每 300 毫秒更新一次，但字符串每 400 毫秒更新一次，
然后使用 [zip] 操作符合并它们，但仍会产生相同的结果，
尽管每 400 毫秒打印一次结果：

> 我们在本示例中使用 [onEach] 过渡操作符来延时每次元素发射并使<!--
-->该流更具说明性以及更简洁。
 
<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow().onEach { delay(300) } // 发射数字 1..3，间隔 300 毫秒
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // 每 400 毫秒发射一次字符串
    val startTime = System.currentTimeMillis() // 记录开始的时间
    nums.zip(strs) { a, b -> "$a -> $b" } // 使用“zip”组合单个字符串
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-21.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
1 -> one at 437 ms from start
2 -> two at 837 ms from start
3 -> three at 1243 ms from start
-->

然而，当在这里使用 [combine] 操作符来替换 [zip]：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow().onEach { delay(300) } // 发射数字 1..3，间隔 300 毫秒
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // 每 400 毫秒发射一次字符串
    val startTime = System.currentTimeMillis() // 记录开始的时间
    nums.combine(strs) { a, b -> "$a -> $b" } // 使用“combine”组合单个字符串
        .collect { value -> // 收集并打印
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-22.kt)获取完整代码。

我们得到了完全不同的输出，其中，`nums` 或 `strs` 流中的每次发射都会打印一行：

```text 
1 -> one at 452 ms from start
2 -> one at 651 ms from start
2 -> two at 854 ms from start
3 -> two at 952 ms from start
3 -> three at 1256 ms from start
```

<!--- TEST ARBITRARY_TIME -->

### 展平流

Flows represent asynchronously received sequences of values, so it is quite easy to get in a situation where 
each value triggers a request for another sequence of values. For example, we can have the following
function that returns a flow of two strings 500 ms apart:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // wait 500 ms
    emit("$i: Second")    
}
```

</div>

<!--- CLEAR -->

Now if we have a flow of three integers and call `requestFlow` for each of them like this:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
(1..3).asFlow().map { requestFlow(it) }
``` 

</div>

<!--- CLEAR -->

Then we end up with a flow of flows (`Flow<Flow<String>>`) that needs to be _flattened_ into a single flow for 
further processing. Collections and sequences have [flatten][Sequence.flatten] and [flatMap][Sequence.flatMap]
operators for this. However, due the asynchronous nature of flows they call for different _modes_ of flattening, 
as such, there is a family of flattening operators on flows.

#### flatMapConcat

Concatenating mode is implemented by [flatMapConcat] and [flattenConcat] operators. They are the most direct
analogues of the corresponding sequence operators. They wait for the inner flow to complete before
starting to collect the next one as the following example shows:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // wait 500 ms
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapConcat { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-23.kt).            

The sequential nature of [flatMapConcat] is clearly seen in the output:

```text                      
1: First at 121 ms from start
1: Second at 622 ms from start
2: First at 727 ms from start
2: Second at 1227 ms from start
3: First at 1328 ms from start
3: Second at 1829 ms from start
```

<!--- TEST ARBITRARY_TIME -->

#### flatMapMerge

Another flattening mode is to concurrently collect all the incoming flows and merge their values into
a single flow so that values are emitted as soon as possible.
It is implemented by [flatMapMerge] and [flattenMerge] operators. They both accept an optional 
`concurrency` parameter that limits the number of concurrent flows that are collected at the same time
(it is equal to [DEFAULT_CONCURRENCY] by default). 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // wait 500 ms
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapMerge { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-24.kt).            

The concurrent nature of [flatMapMerge] is obvious:

```text                      
1: First at 136 ms from start
2: First at 231 ms from start
3: First at 333 ms from start
1: Second at 639 ms from start
2: Second at 732 ms from start
3: Second at 833 ms from start
```                                               

<!--- TEST ARBITRARY_TIME -->

> Note that the [flatMapMerge] calls its block of code (`{ requestFlow(it) }` in this example) sequentially, but 
collects the resulting flows concurrently, it is the equivalent of performing a sequential 
`map { requestFlow(it) }` first and then calling [flattenMerge] on the result.   

#### flatMapLatest   

In a similar way to the [collectLatest] operator, that was shown in 
["Processing the latest value"](#处理最新值) section, there is the corresponding "Latest" 
flattening mode where a collection of the previous flow is cancelled as soon as new flow is emitted. 
It is implemented by the [flatMapLatest] operator.

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // wait 500 ms
    emit("$i: Second")    
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val startTime = System.currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapLatest { requestFlow(it) }                                                                           
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-25.kt).            

The output here in this example is a good demonstration of how [flatMapLatest] works:

```text                      
1: First at 142 ms from start
2: First at 322 ms from start
3: First at 425 ms from start
3: Second at 931 ms from start
```                                               

<!--- TEST ARBITRARY_TIME -->
  
> Note that [flatMapLatest] cancels all the code in its block (`{ requestFlow(it) }` in this example) on a new value. 
It makes no difference in this particular example, because the call to `requestFlow` itself is fast, not-suspending,
and cannot be cancelled. However, it would show up if we were to use suspending functions like `delay` in there.

### 流异常

Flow collection can complete with an exception when an emitter or code inside the operators throw an exception. 
There are several ways to handle these exceptions.

#### 收集器 try 与 catch

A collector can use Kotlin's [`try/catch`][exceptions] block to handle exceptions: 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value ->         
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-26.kt).

This code successfully catches an exception in [collect] terminal operator and, 
as we see, no more values are emitted after that:

```text 
Emitting 1
1
Emitting 2
2
Caught java.lang.IllegalStateException: Collected 2
```

<!--- TEST -->

#### 一切都已捕获

The previous example actually catches any exception happening in the emitter or in any intermediate or terminal operators.
For example, let's change the code so that emitted values are [mapped][map] to strings,
but the corresponding code produces an exception:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }                 
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-27.kt).

This exception is still caught and collection is stopped:

```text 
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
```

<!--- TEST -->

### 异常透明性

But how can code of the emitter encapsulate its exception handling behavior?  

Flows must be _transparent to exceptions_ and it is a violation of the exception transparency to [emit][FlowCollector.emit] values in the 
`flow { ... }` builder from inside of a `try/catch` block. This guarantees that a collector throwing an exception
can always catch it using `try/catch` as in the previous example.

The emitter can use a [catch] operator that preserves this exception transparency and allows encapsulation
of its exception handling. The body of the `catch` operator can analyze an exception
and react to it in different ways depending on which exception was caught:

* Exceptions can be rethrown using `throw`.
* Exceptions can be turned into emission of values using [emit][FlowCollector.emit] from the body of [catch].
* Exceptions can be ignored, logged, or processed by some other code.

For example, let us emit the text on catching an exception:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun foo(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }                 
        "string $value"
    }

fun main() = runBlocking<Unit> {
//sampleStart
    foo()
        .catch { e -> emit("Caught $e") } // emit on exception
        .collect { value -> println(value) }
//sampleEnd
}            
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-28.kt). 
 
The output of the example is the same, even though we do not have `try/catch` around the code anymore. 

<!--- TEST  
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
-->

#### 透明捕获

The [catch] intermediate operator, honoring exception transparency, catches only upstream exceptions
(that is an exception from all the operators above `catch`, but not below it).
If the block in `collect { ... }` (placed below `catch`) throws an exception then it escapes:  

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-29.kt). 
 
A "Caught ..." message is not printed despite there being a `catch` operator: 

<!--- TEST EXCEPTION  
Emitting 1
1
Emitting 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
	at ...
-->

#### 声明式捕获

We can combine the declarative nature of the [catch] operator with a desire to handle all the exceptions, by moving the body
of the [collect] operator into [onEach] and putting it before the `catch` operator. Collection of this flow must
be triggered by a call to `collect()` without parameters:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    foo()
        .onEach { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
        .catch { e -> println("Caught $e") }
        .collect()
//sampleEnd
}            
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-30.kt). 
 
Now we can see that a "Caught ..." message is printed and so we can catch all the exceptions without explicitly
using a `try/catch` block: 

<!--- TEST EXCEPTION  
Emitting 1
1
Emitting 2
Caught java.lang.IllegalStateException: Collected 2
-->

### 流完成

When flow collection completes (normally or exceptionally) it may need to execute an action. 
As you may have already noticed, it can be done in two ways: imperative or declarative.

#### 命令式 finally 块

In addition to `try`/`catch`, a collector can also use a `finally` block to execute an action 
upon `collect` completion.

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-31.kt). 

This code prints three numbers produced by the `foo()` flow followed by a "Done" string:

```text
1
2
3
Done
```

<!--- TEST  -->

#### 声明式处理

For the declarative approach, flow has [onCompletion] intermediate operator that is invoked
when the flow has completely collected.

The previous example can be rewritten using an [onCompletion] operator and produces the same output:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
//sampleStart
    foo()
        .onCompletion { println("Done") }
        .collect { value -> println(value) }
//sampleEnd
}            
```
</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-32.kt). 

<!--- TEST 
1
2
3
Done
-->

The key advantage of [onCompletion] is a nullable `Throwable` parameter of the lambda that can be used
to determine whether the flow collection was completed normally or exceptionally. In the following
example the `foo()` flow throws an exception after emitting the number 1:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}            
//sampleEnd
```
</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-33.kt). 

As you may expect, it prints:

```text
1
Flow completed exceptionally
Caught exception
```

<!--- TEST -->

The [onCompletion] operator, unlike [catch], does not handle the exception. As we can see from the above
example code, the exception still flows downstream. It will be delivered to further `onCompletion` operators
and can be handled with a `catch` operator. 

#### 仅限上游异常

Just like the [catch] operator, [onCompletion] only sees exceptions coming from upstream and does not
see downstream exceptions. For example, run the following code:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
//sampleEnd
```

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-34.kt). 

We can see the completion cause is null, yet collection failed with exception:

```text 
1
Flow completed with null
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

<!--- TEST EXCEPTION -->

### 命令式还是声明式

Now we know how to collect flow, and handle its completion and exceptions in both imperative and declarative ways.
The natural question here is, which approach is preferred and why?
As a library, we do not advocate for any particular approach and believe that both options
are valid and should be selected according to your own preferences and code style. 

### 启动流

It is easy to use flows to represent asynchronous events that are coming from some source.
In this case, we need an analogue of the `addEventListener` function that registers a piece of code with a reaction
for incoming events and continues further work. The [onEach] operator can serve this role. 
However, `onEach` is an intermediate operator. We also need a terminal operator to collect the flow. 
Otherwise, just calling `onEach` has no effect.
 
If we use the [collect] terminal operator after `onEach`, then the code after it will wait until the flow is collected:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .collect() // <--- Collecting the flow waits
    println("Done")
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-35.kt). 
  
As you can see, it prints:

```text 
Event: 1
Event: 2
Event: 3
Done
```    

<!--- TEST -->
 
The [launchIn] terminal operator comes in handy here. By replacing `collect` with `launchIn` we can
launch a collection of the flow in a separate coroutine, so that execution of further code 
immediately continues:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

//sampleStart
fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- Launching the flow in a separate coroutine
    println("Done")
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-36.kt). 
  
It prints:

```text          
Done
Event: 1
Event: 2
Event: 3
```    

<!--- TEST -->

The required parameter to `launchIn` must specify a [CoroutineScope] in which the coroutine to collect the flow is 
launched. In the above example this scope comes from the [runBlocking]
coroutine builder, so while the flow is running, this [runBlocking] scope waits for completion of its child coroutine
and keeps the main function from returning and terminating this example. 

In actual applications a scope will come from an entity with a limited 
lifetime. As soon as the lifetime of this entity is terminated the corresponding scope is cancelled, cancelling
the collection of the corresponding flow. This way the pair of `onEach { ... }.launchIn(scope)` works
like the `addEventListener`. However, there is no need for the corresponding `removeEventListener` function, 
as cancellation and structured concurrency serve this purpose.

Note that [launchIn] also returns a [Job], which can be used to [cancel][Job.cancel] the corresponding flow collection
coroutine only without cancelling the whole scope or to [join][Job.join] it.

{:#flow-and-reactive-streams}

### 流（Flow）与响应式流（Reactive Streams）

For those who are familiar with [Reactive Streams](https://www.reactive-streams.org/) or reactive frameworks such as RxJava and project Reactor, 
design of the Flow may look very familiar.

Indeed, its design was inspired by Reactive Streams and its various implementations. But Flow main goal is to have as simple design as possible, 
be Kotlin and suspension friendly and respect structured concurrency. Achieving this goal would be impossible without reactive pioneers and their tremendous work. You can read the complete story in [Reactive Streams and Kotlin Flows](https://medium.com/@elizarov/reactive-streams-and-kotlin-flows-bfd12772cda4) article.

While being different, conceptually, Flow *is* a reactive stream and it is possible to convert it to the reactive (spec and TCK compliant) Publisher and vice versa.
Such converters are provided by `kotlinx.coroutines` out-of-the-box and can be found in corresponding reactive modules (`kotlinx-coroutines-reactive` for Reactive Streams, `kotlinx-coroutines-reactor` for Project Reactor and `kotlinx-coroutines-rx2` for RxJava2).
Integration modules include conversions from and to `Flow`, integration with Reactor's `Context` and suspension-friendly ways to work with various reactive entities.
 
<!-- stdlib references -->

[collections]: https://kotlinlang.org/docs/reference/collections-overview.html 
[List]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-list/index.html 
[forEach]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/for-each.html
[Sequence]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/index.html
[Sequence.zip]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/zip.html
[Sequence.flatten]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flatten.html
[Sequence.flatMap]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/flat-map.html
[exceptions]: https://kotlinlang.org/docs/reference/exceptions.html

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[Dispatchers.Main]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
<!--- INDEX kotlinx.coroutines.flow -->
[Flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html
[flow]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html
[FlowCollector.emit]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html
[collect]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html
[flowOf]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-of.html
[map]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/map.html
[filter]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/filter.html
[transform]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/transform.html
[take]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/take.html
[toList]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html
[toSet]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html
[first]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/first.html
[single]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html
[reduce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html
[fold]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/fold.html
[flowOn]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html
[buffer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html
[conflate]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html
[collectLatest]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html
[zip]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/zip.html
[combine]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html
[onEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-each.html
[flatMapConcat]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-concat.html
[flattenConcat]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-concat.html
[flatMapMerge]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-merge.html
[flattenMerge]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flatten-merge.html
[DEFAULT_CONCURRENCY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-d-e-f-a-u-l-t_-c-o-n-c-u-r-r-e-n-c-y.html
[flatMapLatest]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html
[catch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/catch.html
[onCompletion]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/on-completion.html
[launchIn]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/launch-in.html
<!--- END -->
