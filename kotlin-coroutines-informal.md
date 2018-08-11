# 协程



## 用例

协程可以被看作为 *可挂起的计算*  的实例。换句话说，可以在某些点上挂起，稍后在另一个线程上恢复执行。协程调用对方（来回传递数据），可以形成多任务协同机制。



### 异步计算

协程的第一个用例是异步计算（通过 async/await 在 C# 和其他语言中处理）。让我们来看看如何使用回调来完成这些计算。

```kotlin
inChannel.read(buf) {
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
   
    outChannel.write(buf) {
        ...
        ...
        outFile.close()          
    }
}
```

注意，我们在回调内部有一个回调， 而它节省了大量的样板（没有必要将  `buf ` 参数显式传递到回调，它们之被看作闭包的一部分），缩进级别每次都在增长。



同样的计算可以直截了当地表达为协程：

```kotlin
launch(CommonPool) {
    
    val bytesRead = inChannel.aRead(buf) 
    
    ...
    ...
    process(buf, bytesRead)
    
    outChannel.aWrite(buf)
      
    ...
    ...
    outFile.close()
}
```

`aRead()` 和 `aWrite()` 是特别的 *挂起函数* —— 它们可以 *挂起*  代码执行（这并不意味着阻塞运行他们的线程），然后在调用完成时 *恢复*  代码执行。如果我们眯着眼睛去想象所有在 `aRead()` 之后的代码已经被包装成一个 lambda 传给 `aRead()` 作为回调，再对 `aWrite()` 做同样的事情，我们就可以看到和上面相同的代码，只是可读性增加了。



以一种非常通用的方式支持协程是我们的明确目标，所以在这个例子中，`launch{}`, `.aRead()`, 和 `.aWrite()` 只是适应协程工作的库函数：`launch` 是 *协程建造者* —— 它在一些 context（在例子中使用的是 `CommonPool`） 中创建并启动协程。`aRead()` 和 `aWrite()` 作为特别的 *挂起函数* 隐式地接受 *continuations*（continuations 只是一般的回调）。



### Futures



### Generators



### 异步 UI



### 更多用例



## 协程概述

本部分概述了能够编写协程的语言机制和管理其语义的标准库。



### 协程的实验状态



### 术语

* *协程* —— 是一个*可挂起计算* 的 *实例* 。它在概念上类似于一个线程，在这个意义上，它需要一个代码块运行，并具有类似的生命周期 —— 它可以被创建和启动，但它不绑定到任何特定的线程。它可以在一个线程中 *挂起* 其执行， 并在另一个线程中 *恢复* 。而且，像一个 future 或一个 promise，它在完成时可能伴随着结果或异常。

* *挂起函数* —— 一个函数被标记了 `suspend` 修饰符。它可能会 *挂起* 执行代码, 而不通过调用其他挂起函数来阻塞当前执行线程。一个挂起函数不能在常规代码中被调用，除了在其他挂起函数或挂起 lambda 中（见下方）。举个例子，如用例所示，`.await()` 和 `yield()` ，是在库中定义的挂起函数。标准库提供了用于定义其他所有挂起函数所使用的基础挂起函数。

* *挂起 lambda* —— 一个必须在协程中运行的代码块。它看起来完全像一个普通的 lambda 表达式，但它的函数类型被  `suspend` 修饰符标记。就像常规 lambda 表达式是匿名局部函数的短语法形式一样，挂起lambda 是匿名挂起函数的短语法形式。它可能会 *挂起* 执行代码, 而不通过调用其他挂起函数来阻塞当前执行线程。举个例子，如用例所示，跟在 `launch` , `future` , 和 `BuildSequence` 函数后面花括号之后的代码块，就是挂起 lambda。

* *挂起函数类型*  —— 一种用于挂起函数和挂起 lambda 的函数类型。它就像一个常规的函数类型，但具有 `suspend` 修饰符。举个例子，`suspend () -> Int ` 是一个没有参数、返回 `Int` 的挂起函数的函数类型。一个像这样声明 —— `suspend fun foo(): Int` 的挂起函数符合上述函数类型。

* *协程建造者* —— 一个使用一些挂起 lambda 作为参数，创建一个协程，可选地提供某种形式对其结果的访问的函数。举个例子，用例中的 `launch{}`, `future{}`, 和 `buildSequence{}` 是库中定义的协程建造者。 标准库提供了用于定义其他所有协程建造者所使用的基础协程建造者。

* *挂起点* —— 是协程执行过程中可能会被 *挂起* 的一个点。从语法上说，一个挂起点是对一个挂起函数的调用，但 *实际* 的挂起在挂起函数调用了标准库中的原始挂起函数时发生。

* *continuation* —— 是一个挂起的协程在挂起点时的一个状态。它在概念上代表它在挂起点之后的剩余执行代码。一个例子：

  ```kotlin
  buildSequence {
      for (i in 1..10) yield(i * i)
      println("over")
  }  
  ```

  在这里，每次调用挂起函数 `yield()`，协程挂起时，*其执行的剩余* 被看作 continuation，所以我们有 10 个 continuation：第一次运行循环后 `i=2` ，挂起；第二次运行循环后 `i=3`，挂起……最后一个打印 "over" 并完成协程。该协程在此创建，但尚未启动，由它的初始 *continuation* 所表示，后者由它整个执行组成，类型为 `Continuation<Unit> ` 。

如上所述，驱动协程的要求之一是灵活性：我们希望能够支持许多现有的异步 API 和其他用例,，并将硬编码的部分最小化到编译器中。因此， 编译器只负责支持挂起函数、挂起 lambda 时和相应的挂起函数类型。标准库中的原始函数很少， 其余的则留给应用程序库。

### Continuation 接口

这是标准库中接口 `Continuation` 的定义，代表了一个通用的回调：

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resume(value: T)
   fun resumeWithException(exception: Throwable)
}
```

context 在协程上下文中详细介绍，表示于任意用户定义的与协程所关联的上下文。函数 `resume` 和 `resumeWithException` 是用于提供一个成功结果（通过 `resume`）或报告失败（通过 `resumeWithException`）。

### 挂起函数

一个典型的 *挂起函数* 例如 `.await()` 的实现看起来是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null)
                cont.resume(result)
            else
                cont.resumeWithException(exception)
        }
    }
```

`suspend` 修饰符表明这个函数可以挂起协程的执行。这个特别的函数是作为类型 `CompletableFuture<T>` 的扩展函数，以便其使用阅读时与实际执行顺序相应的为自然顺序 —— 从左到右：

```kotlin
asyncOperation(...).await()
```

`suspend` 修饰符可以用于任何函数：顶层函数、扩展函数、成员函数，或操作符函数。

> 注意，在目前版本中，局部函数、属性的 getter/setter，和构造器不能拥有 `suspend` 修饰符。
>
> 这个限制在将来会被取消。

挂起函数可以调用任何常规函数，但要真正挂起执行，必须调用一些其他的挂起函数。特别的是，这个 `await` 实现调用了在标准库中定义的顶层挂起函数 `suspendCoroutine`：

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

当  `suspendCoroutine` 在一个协程中被调用时（它只可能在协程中被调用，因为它是一个挂起函数），它捕获了协程的执行状态到一个 *continuation* 实例，然后将其传给指定的 `block` 作为参数。为了恢复执行，代码块需要调用  `continuation.resume()`  或 `continuation.resumeWithException()` 在该线程或其他线程中。*实际* 的挂起发生当 `suspendCoroutine` 代码块没有返回上述两者。如果协程在代码块中直接被恢复，协程未被看作已经暂停并继续执行。



传给  `continuation.resume()`  的值作为  `suspendCoroutine()` 的返回值，相继成为 `.await()` 的返回值。



多次恢复一个协程是不被允许的，并会产生  `IllegalStateException`。 



### 协程建造者

挂起函数不能够从常规函数中调用，所以标准库提供了用于在常规非挂起范围中开启协程的函数。这是简单 *协程建造者* `launch` 的实现：

```kotlin
fun launch(context: CoroutineContext, block: suspend () -> Unit) =
        block.startCoroutine(StandaloneCoroutine(context))

private class StandaloneCoroutine(override val context: CoroutineContext): Continuation<Unit> {
    override fun resume(value: Unit) {}

    override fun resumeWithException(exception: Throwable) {
        val currentThread = Thread.currentThread()
        currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
    }
}
```

这个实现定义了一个简单类  `StandaloneCoroutine`，它代表了这个协程，实现了 `Continuation` 接口，用于捕获它的完成。这个协程的完成调用了它的 *completion continuation*。它的 `resume` 或 `resumeWithException` 函数在协程完成时被调用，相应地伴随着结果或异常。因为  `launch`  "fire-and-forget"  一个协程，它为了挂起函数 *(喵喵喵?)*，定义为返回 `Unit` 类型，实际上忽略了在它 `resume` 函数中的结果。如果协程执行伴随异常，当前线程的 uncaught exception handler 会被用于去报告这个异常。

> 注意：这个简单实现返回了 `Unit` ，没有提供任何协程状态的访问。实际上在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实现要更加复杂，因为它返回了一个 `Job` 实例，代表这个协程并且可以被取消。

context 在协程上下文中详细介绍。这里可以说，在库中定义协程建造者时包含 `context` 参数，为了更好的与其他库中定义的有用的上下文元素组合是一种很好的风格。

 `startCoroutine`  在标准库中作为挂起函数类型的扩展函数。它的签名是这样的：

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
```

 `startCoroutine`  创建协程并在当前线程中立刻启动执行（但请参阅下面的备注），直到第一个挂起点会返回。挂起点时协程主体中调用某些挂起函数，它由相应挂起函数代码来定义协程恢复执行的时间和方式。

> 注意：continuation 拦截器（来源于 context）在后文中会提到，它能够将协程的执行，*包括* 它的初始 continuation 调度到另一个线程中。

### 协程 Context

协程 context 是一个可以附加到协程中的一组持久的用户定义对象。它可以包括负责协程线程策略的对象，日志，协程执行的安全性和事务方面，协程的标识和名称等等。这里是协程和 context 的简单心智模型 *(喵喵喵?)* 。把协程看作一个轻量线程。在这种情况下，协程 context 仅仅像一个 thread-local 变量的 set。不同之处在于 thread-local 是可变的，协程 context 是不可变的。但对于协程这并不是一个严重的限制，因为他们是如此轻量以至于当 context 改变时可以很容易地开一个新的协程。

标准库没有包含 context 的任何具体实现，但拥有接口和抽象类。这样，所有这些方面可以以可组合的方式在库中定义，因此来自不同库的各个方面可以和平共存为同一个 context。

从概念上讲，协程 context 是一组元素的索引，其中每个元素有唯一的键。它是一个 set 和 map 的混合。它的元素有像在 map 中的键，但它的键直接与元素关联，更像在一个 set 中。标准库定义了  `CoroutineContext` 的最小接口：

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```

 `CoroutineContext`  本身有四种可用的核心操作：

* 操作符 `get` 提供对给定键元素类型安全的访问，可以使用 `[..]` 符号。对于可以这样做的原因解释请见 [Kotlin 操作符重载](https://kotlinlang.org/docs/reference/operator-overloading.html)。
* 函数 `fold` 类似于标准库中 [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) 扩展函数，提供迭代 context 中所有元素的方法。
* 操作符 `plus` 类似于标准库的 [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) 扩展函数，返回两个 context 的组合, 右边的元素加上替换左侧的相同键的元素。 *(喵喵喵?)*
* 函数 `minueKey` 返回不包含指定键的 context。

协程 context 的一个 `Element` 就是它自己。它是仅具有此元素的单个 context。 *(喵喵喵?)*

这样就可以通过协程 context 元素的库定义，使用 `+` 连接它们来创建一个复合 context。举个例子，如果一个库定义 `auth` 元素带着用户授权信息，一些其他的库定义 `CommonPool` 对象带着一些协程执行信息，你就可以使用协程建造者 `launch{}` 并且使用组合 context `launch(auth + CommonPool){...}` 调用。

> 注意： [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 提供了几个 context 元素，包括将协程执行调度在一个共享后台线程池中的 `CommonPool`。

所有库定义的 context 元素应该继承标准库提供的  `AbstractCoroutineContextElement` 类。对于库定义的 context 元素，建议使用一下样式。下面的这个例子显示了一个设想的授权 context 元素，它储存了当前用户名：

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

将 context 的 `Key` 定义为相应元素类的伴生对象能够流畅访问 context 中的相应元素。这是一个设想的挂起函数实现，它需要检查当前用户名：

```kotlin
suspend fun secureAwait(): Unit = suspendCoroutine { cont ->
    val currentUser = cont.context[AuthUser]?.name
    // do something user-specific
}
```

### Continuation 拦截器

让我们回想一下 [异步 UI](x) 用例。异步 UI 应用程序必须保证协程程序体始终在 UI 线程中执行，尽管各种挂起函数在任意线程中回复协程执行。这是使用 *Continuation 拦截器*  完成的。首先，我们要充分了解协程的生命周期。思考一下用了协程建造者 `launch{}` 的代码片段：

```kotlin
launch(CommonPool) {
    initialCode() // initial code 执行
    f1.await() // 挂起点 #1
    block1() // 执行 #1
    f2.await() // 挂起点 #2
    block2() // 执行 #2
}
```
协程从 `initialCode` 开始执行，直到第一个挂起点。在挂起点时，协程 *挂起。*一段时间后按照相应挂起函数的定义，协程恢复执行 `block1`，接着再次挂起，在 *完成* 后恢复执行 `block2`。Continuation 拦截器可以拦截和包装与 `initialCode` ,`block1`, 和`block2` 执行相对应的 continuation，使其恢复到之后的挂起点。

协程的 initial code 被视为 *initial continuation* 的 resumption。标准库提供了以下接口：

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
}
```
 `interceptContinuation` 包装了协程的 continuation。每当协程被挂起时，协程框架用下行代码包装实际后续恢复的 `continuation`：

```kotlin
val facade = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```
协程框架为每个实际的 continuation 实例缓存结果 facade。有关详细信息, 请参阅实现细节部分。

> 注意，像 `await` 这样的挂起函数可能会或可能不会实际挂起协程的执行。举个例子，看看在 [挂起函数](#挂起函数) 部分提到 `await` 的实现，当 future 已经完成时（在这种情况下 `resume` 会立刻被调用，协程的执行没有被挂起）就不会使协程真正挂起。只有当协程执行真正被挂起时，一个 continuation 才会被拦截，即当 `suspendCoroutine` 函数体返回但并没有调用 `resume`。

让我们来看看一个用于 `Swing` 拦截器的具体实例代码，它将执行调度在 Swing UI 事件调度线程上。我们从一个包装类 `SwingContinuation` 开始，它检查当前线程并且确保 continuation 只在 Swing 事件调度线程中恢复。如果执行已经在 UI 线程中发生， `Swing` 仅仅合适地调用 `cont.resume`，否则它会使用 `SwingUtilities.invokeLater` 将 continuation 的执行调度在 Swing UI 线程上。

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> by cont {
    override fun resume(value: T) {
        if (SwingUtilities.isEventDispatchThread()) cont.resume(value)
        else SwingUtilities.invokeLater { cont.resume(value) }
    }

    override fun resumeWithException(exception: Throwable) {
        if (SwingUtilities.isEventDispatchThread()) cont.resumeWithException(exception)
        else SwingUtilities.invokeLater { cont.resumeWithException(exception) }
    }
}
```
接着定义了一个 `Swing` 对象作为相应的 context 元素，并实现了 `ContinuationInterceptor` 接口：

```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

> 你可以从 [这里](https://github.com/Kotlin/kotlin-coroutines/blob/master/examples/context/swing.kt) 获得这部分代码。
>
> 注意：`Swing` 对象在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中实际实现还支持了 debug 功能，这提供并显示了协程的标识和当前运行协程的线程名。

现在，可以使用协程建造者 `launch{}` 带着 `Swing` 参数来执行一个完全运行在 Swing 事件调度线程中的协程：

```kotlin
launch(Swing) {
   // code in here can suspend, but will always resume in Swing EDT
}
```

### Restricted 的挂起



## 更多例子



### 包装回调



### 建造 futures



### 非阻塞延迟



### 合作单线程多任务处理



## 异步 Sequences



### Channels



### Mutexes



## 进阶话题



### 资源管理和 GC



### 并发和线程



## 异步编程样式



## 实现细节



### 协程传递样式



### 状态机



### 编译挂起函数



### 协程内部