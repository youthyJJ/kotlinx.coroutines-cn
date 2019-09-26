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
  * [流构造器](#流构造器)
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

在 Kotlin 中可以使用 [集合][collections] 来表示多个值。 
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

> 你可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-03.kt)获取完整代码。

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

> 你可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-flow-04.kt)获取完整代码。

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

Flows are _cold_ streams similar to sequences &mdash; the code inside a [flow] builder does not
run until the flow is collected. This becomes clear in the following example:

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

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-05.kt).

Which prints:

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
 
This is a key reason the `foo()` function (which returns a flow) is not marked with `suspend` modifier.
By itself, `foo()` returns quickly and does not wait for anything. The flow starts every time it is collected,
that is why we see "Flow started" when we call `collect` again.

### 流取消

Flow adheres to the general cooperative cancellation of coroutines. However, flow infrastructure does not introduce
additional cancellation points. It is fully transparent for cancellation. As usual, flow collection can be 
cancelled when the flow is suspended in a cancellable suspending function (like [delay]), and cannot be cancelled otherwise.

The following example shows how the flow gets cancelled on a timeout when running in a [withTimeoutOrNull] block 
and stops executing its code:

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
    withTimeoutOrNull(250) { // Timeout after 250ms 
        foo().collect { value -> println(value) } 
    }
    println("Done")
}
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-06.kt).

Notice how only two numbers get emitted by the flow in `foo()` function, producing the following output: 

```text
Emitting 1
1
Emitting 2
2
Done
```

<!--- TEST -->

### 流构造器

The `flow { ... }` builder from the previous examples is the most basic one. There are other builders for
easier declaration of flows:

* [flowOf] builder that defines a flow emitting a fixed set of values.
* Various collections and sequences can be converted to flows using `.asFlow()` extension functions.

So, the example that prints the numbers from 1 to 3 from a flow can be written as:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart
    // Convert an integer range to a flow
    (1..3).asFlow().collect { value -> println(value) }
//sampleEnd 
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-07.kt).
 
<!--- TEST
1
2
3
-->

### 过渡流操作符

Flows can be transformed with operators, just as you would with collections and sequences. 
Intermediate operators are applied to an upstream flow and return a downstream flow. 
These operators are cold, just like flows are. A call to such an operator is not
a suspending function itself. It works quickly, returning the definition of a new transformed flow. 

The basic operators have familiar names like [map] and [filter]. 
The important difference to sequences is that blocks of 
code inside these operators can call suspending functions. 

For example, a flow of incoming requests can be
mapped to the results with the [map] operator, even when performing a request is a long-running
operation that is implemented by a suspending function:   

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart           
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-08.kt).

It produces the following three lines, each line appearing after each second:

```text                                                                    
response 1
response 2
response 3
```

<!--- TEST -->

#### 转换操作符

Among the flow transformation operators, the most general one is called [transform]. It can be used to imitate
simple transformations like [map] and [filter], as well as implement more complex transformations. 
Using the `transform` operator, we can [emit][FlowCollector.emit] arbitrary values an arbitrary number of times.

For example, using `transform` we can emit a string before performing a long-running asynchronous request 
and follow it with a response:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
//sampleStart
    (1..3).asFlow() // a flow of requests
        .transform { request ->
            emit("Making request $request") 
            emit(performRequest(request)) 
        }
        .collect { response -> println(response) }
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-09.kt).

The output of this code is:

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

Size-limiting intermediate operators like [take] cancel the execution of the flow when the corresponding limit
is reached. Cancellation in coroutines is always performed by throwing an exception, so that all the resource-management
functions (like `try { ... } finally { ... }` blocks) operate normally in case of cancellation:

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
        .take(2) // take only the first two
        .collect { value -> println(value) }
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-10.kt).

The output of this code clearly shows that the execution of the `flow { ... }` body in the `numbers()` function
stopped after emitting the second number:

```text       
1
2
Finally in numbers
```

<!--- TEST -->

### 末端流操作符

Terminal operators on flows are _suspending functions_ that start a collection of the flow.
The [collect] operator is the most basic one, but there are other terminal operators, which can make it easier:

* Conversion to various collections like [toList] and [toSet].
* Operators to get the [first] value and to ensure that a flow emits a [single] value.
* Reducing a flow to a value with [reduce] and [fold].

For example:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> {
//sampleStart         
    val sum = (1..5).asFlow()
        .map { it * it } // squares of numbers from 1 to 5                           
        .reduce { a, b -> a + b } // sum them (terminal operator)
    println(sum)
//sampleEnd     
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-11.kt).

Prints a single number:

```text
55
```

<!--- TEST -->

### 流是连续的

Each individual collection of a flow is performed sequentially unless special operators that operate
on multiple flows are used. The collection works directly in the coroutine that calls a terminal operator. 
No new coroutines are launched by default. 
Each emitted value is processed by all the intermediate operators from 
upstream to downstream and is then delivered to the terminal operator after. 

See the following example that filters the even integers and maps them to strings:

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

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-12.kt).

Producing:

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

Collection of a flow always happens in the context of the calling coroutine. For example, if there is 
a `foo` flow, then the following code runs in the context specified
by the author of this code, regardless of the implementation details of the `foo` flow:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
withContext(context) {
    foo.collect { value ->
        println(value) // run in the specified context 
    }
}
``` 

</div>

<!--- CLEAR -->

This property of a flow is called _context preservation_.

So, by default, code in the `flow { ... }` builder runs in the context that is provided by a collector
of the corresponding flow. For example, consider the implementation of `foo` that prints the thread
it is called on and emits three numbers:

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

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-13.kt).

Running this code produces:

```text  
[main @coroutine#1] Started foo flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```

<!--- TEST FLEXIBLE_THREAD -->

Since `foo().collect` is called from the main thread, the body of `foo`'s flow is also called in the main thread.
This is the perfect default for fast-running or asynchronous code that does not care about the execution context and
does not block the caller. 

#### withContext 发出错误

However, the long-running CPU-consuming code might need to be executed in the context of [Dispatchers.Default] and UI-updating
code might need to be executed in the context of [Dispatchers.Main]. Usually, [withContext] is used
to change the context in the code using Kotlin coroutines, but code in the `flow { ... }` builder has to honor the context
preservation property and is not allowed to [emit][FlowCollector.emit] from a different context. 

Try running the following code:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
                      
//sampleStart
fun foo(): Flow<Int> = flow {
    // The WRONG way to change context for CPU-consuming code in flow builder
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // pretend we are computing it in CPU-consuming way
            emit(i) // emit next value
        }
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value -> println(value) } 
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-14.kt).

This code produces the following exception:

<!--- TEST EXCEPTION
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
		Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
		but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, DefaultDispatcher].
		Please refer to 'flow' documentation or use 'flowOn' instead
	at ...
-->
   
> Note that we had to use a fully qualified name of the [kotlinx.coroutines.withContext][withContext] function in this example to 
demonstrate this exception. A short name of `withContext` would have resolved to a special stub function that
produces a compilation error to prevent us from running into this problem.   

#### flowOn 操作符
   
The exception refers to the [flowOn] function that shall be used to change the context of the flow emission.
The correct way to change the context of a flow is shown in the example below, which also prints the 
names of the corresponding threads to show how it all works:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
           
//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it in CPU-consuming way
        log("Emitting $i")
        emit(i) // emit next value
    }
}.flowOn(Dispatchers.Default) // RIGHT way to change context for CPU-consuming code in flow builder

fun main() = runBlocking<Unit> {
    foo().collect { value ->
        log("Collected $value") 
    } 
}            
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-15.kt).
  
Notice how `flow { ... }` works in the background thread, while collection happens in the main thread:   
  
<!--- TEST FLEXIBLE_THREAD
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
-->

Another thing to observe here is that the [flowOn] operator has changed the default sequential nature of the flow.
Now collection happens in one coroutine ("coroutine#1") and emission happens in another coroutine
("coroutine#2") that is running in another thread concurrently with the collecting coroutine. The [flowOn] operator
creates another coroutine for an upstream flow when it has to change the [CoroutineDispatcher] in its context. 

### 缓冲

Running different parts of a flow in different coroutines can be helpful from the standpoint of the overall time it takes 
to collect the flow, especially when long-running asynchronous operations are involved. For example, consider a case when
the emission by `foo()` flow is slow, taking 100 ms to produce an element; and collector is also slow, 
taking 300 ms to process an element. Let's see how long it takes to collect such a flow with three numbers:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

//sampleStart
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        foo().collect { value -> 
            delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
    }   
    println("Collected in $time ms")
}
//sampleEnd
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-16.kt).

It produces something like this, with the whole collection taking around 1200 ms (three numbers, 400 ms for each):

```text
1
2
3
Collected in 1220 ms
```

<!--- TEST ARBITRARY_TIME -->

We can use a [buffer] operator on a flow to run emitting code of `foo()` concurrently with collecting code,
as opposed to running them sequentially:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .buffer() // buffer emissions, don't wait
            .collect { value -> 
                delay(300) // pretend we are processing it for 300 ms
                println(value) 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-17.kt).

It produces the same numbers just faster, as we have effectively created a processing pipeline,
having to only wait 100 ms for the first number and then spending only 300 ms to process
each number. This way it takes around 1000 ms to run:

```text
1
2
3
Collected in 1071 ms
```                    

<!--- TEST ARBITRARY_TIME -->

> Note that the [flowOn] operator uses the same buffering mechanism when it has to change a [CoroutineDispatcher],
but here we explicitly request buffering without changing the execution context. 

#### 合并

When a flow represents partial results of the operation or operation status updates, it may not be necessary
to process each value, but instead, only most recent ones. In this case, the [conflate] operator can be used to skip
intermediate values when a collector is too slow to process them. Building on the previous example:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .conflate() // conflate emissions, don't process each one
            .collect { value -> 
                delay(300) // pretend we are processing it for 300 ms
                println(value) 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-18.kt).

We see that while the first number was still being processed the second, and third were already produced, so
the second one was _conflated_ and only the most recent (the third one) was delivered to the collector:

```text
1
3
Collected in 758 ms
```             

<!--- TEST ARBITRARY_TIME -->   

#### 处理最新值

Conflation is one way to speed up processing when both the emitter and collector are slow. It does it by dropping emitted values.
The other way is to cancel a slow collector and restart it every time a new value is emitted. There is
a family of `xxxLatest` operators that perform the same essential logic of a `xxx` operator, but cancel the
code in their block on a new value. Let's try changing [conflate] to [collectLatest] in the previous example:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.*

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
//sampleStart
    val time = measureTimeMillis {
        foo()
            .collectLatest { value -> // cancel & restart on the latest value
                println("Collecting $value") 
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value") 
            } 
    }   
    println("Collected in $time ms")
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-19.kt).
 
Since the body of [collectLatest] takes 300 ms, but new values are emitted every 100 ms, we see that the block
is run on every value, but completes only for the last value:

```text 
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
``` 

<!--- TEST ARBITRARY_TIME -->

### 组合多个流

There are lots of ways to compose multiple flows.

#### Zip

Just like the [Sequence.zip] extension function in the Kotlin standard library, 
flows have a [zip] operator that combines the corresponding values of two flows:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow() // numbers 1..3
    val strs = flowOf("one", "two", "three") // strings 
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string
        .collect { println(it) } // collect and print
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-20.kt).

This example prints:

```text
1 -> one
2 -> two
3 -> three
```
 
<!--- TEST -->

#### Combine

When flow represents the most recent value of a variable or operation (see also the related 
section on [conflation](#合并)), it might be needed to perform a computation that depends on
the most recent values of the corresponding flows and to recompute it whenever any of the upstream
flows emit a value. The corresponding family of operators is called [combine].

For example, if the numbers in the previous example update every 300ms, but strings update every 400 ms, 
then zipping them using the [zip] operator will still produce the same result, 
albeit results that are printed every 400 ms:

> We use a [onEach] intermediate operator in this example to delay each element and make the code 
that emits sample flows more declarative and shorter.
 
<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
    val startTime = System.currentTimeMillis() // remember the start time 
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string with "zip"
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-21.kt).

<!--- TEST ARBITRARY_TIME
1 -> one at 437 ms from start
2 -> two at 837 ms from start
3 -> three at 1243 ms from start
-->

However, when using a [combine] operator here instead of a [zip]:

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking<Unit> { 
//sampleStart                                                                           
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms          
    val startTime = System.currentTimeMillis() // remember the start time 
    nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
        } 
//sampleEnd
}
```  

</div>

> You can get the full code from [here](../kotlinx-coroutines-core/jvm/test/guide/example-flow-22.kt).

We get quite a different output, where a line is printed at each emission from either `nums` or `strs` flows:

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
