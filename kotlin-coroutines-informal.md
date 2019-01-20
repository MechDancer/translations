# 协程

## 用例

协程可以被看作为*可挂起的计算*  的实例。换句话说，可以在某些点上挂起，稍后在另一个线程上恢复执行。协程相互调用（来回传递数据），即可形成多任务协同机制。

### 异步计算

最能描述协程功能用例是异步计算（在 C# 及其他语言中通过 `async`/`await` 实现）。让我们来看看如何使用回调来完成这些计算。不妨以异步 I/O 为例（下面的 API 经过简化）：

```kotlin
// 异步读数据到 `buf`，完成后执行 λ
inChannel.read(buf) {
    // 这个 λ 会在读完后执行
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
    
    // 异步从 `buf` 写数据, 完成后执行 λ
    outChannel.write(buf) {
        // 这个 λ 会在写完后执行
        ...
        ...
        outFile.close()          
    }
}
```

注意，我们在回调内部有一个回调，虽然这能节省大量没有意义的代码（例如，没有必要将  `buf ` 参数显式传递到回调，它们被看作是闭包的一部分），但缩进级别每次都在增长，而且只要嵌套超过一层，大家都知道能产生多少麻烦（百度“回调地狱”，看看 JavaScript 迫害了多少人）。

同样的计算可以直截了当地表达为协程（前提是有一个库，使 I/O 接口适配协程）：

```kotlin
launch {
    // 异步读时挂起
    val bytesRead = inChannel.aRead(buf) 
    // 读完成后才执行这一行
    ...
    ...
    process(buf, bytesRead)
    // 异步写时挂起
    outChannel.aWrite(buf)
    // 写完成后才执行这一行
    ...
    ...
    outFile.close()
}
```

`aRead()` 和 `aWrite()` 是特别的*挂起函数* —— 它们可以*挂起* 代码执行（这并不意味着阻塞运行他们的线程），然后在调用完成时*恢复*  代码执行。如果我们眯着眼睛去想象所有在 `aRead()` 之后的代码已经被包装成一个 lambda 传给 `aRead()` 作为回调，再对 `aWrite()` 做同样的事情，我们就可以看到和上面相同的代码，只是可读性增加了。

以一种非常通用的方式支持协程是我们的明确目标，所以在这个例子中，`launch{}`、`.aRead()` 和 `.aWrite()` 只是适应协程工作的库函数；`launch` 是*协程建造者* —— 它创建并启动协程；`aRead()` 和 `aWrite()` 作为特别的*挂起函数* 隐式地接受*续体*（[*续体*](https://aisia.moe/2018/02/08/kotlin-coroutine-kepa/) 就是一般的回调）。

> 关于 `launch{}` 的示例代码在[协程建造者](#协程建造者)一节，关于 `aRead()` 的示例代码在[包装性回调](#包装性回调)一节。

注意，显式传入的回调要在循环中异步调用通常非常棘手，但在协程中这不过是稀松平常的小事：

```kotlin
launch {
    while (true) {
        // 异步读时挂起
        val bytesRead = inFile.aRead(buf)
        // 读完继续执行
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // 异步写时挂起
        outFile.aWrite(buf) 
        // 写完继续执行
        ...
    }
}
```

可想而知，在协程中处理异常也会稍微方便一些。

### 特性

还有另一种表达异步计算的方式：通过期货（或者叫做诺言、延时）。我们在示例中使用一个虚构的 API，对图像应用一个覆盖：

```kotlin
val future = runAfterBoth(
    loadImageAsync("...original..."), // 创建期货
    loadImageAsync("...overlay...")   // 创建期货
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

使用协程，可以写成这样：

```kotlin
val future = future {
    val original = loadImageAsync("...original...") // 创建期货
    val overlay = loadImageAsync("...overlay...")   // 创建期货
    ...
    // 挂起等待图片加载
    // 图像和覆盖都加载完后执行 `applyOverlay(...)`
    applyOverlay(original.await(), overlay.await())
}
```

> 关于 `future{}` 的示例代码在[创建期货](#创建期货)一节，关于 `await()` 的示例代码在[挂起函数](#挂起函数)一节。

再一次地，协程对期货的支持减少了缩进级别、逻辑（以及异常处理，这里没有出现）更加自然，而且没有使用专门的关键字（比如 C#、JS 以及其他语言中的 `async` 和 `await`）:`future{}` 和 `await()` 都只是库函数而已。

### 生成器

协程的另一个典型用例是延时计算序列（在 C#、Python 和很多其他语言中通过 `yield`  实现）。这样的序列可以由看似连续的代码生成，但在运行时只计算真正用到的元素。

```kotlin
// 类型推断为 Sequence<Int>
val fibonacci = sequence {
    yield(1) // 斐波那契数列的首项
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // 斐波那契数列的下一项
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

代码创建里一个表示[斐波那契数列](https://zhuanlan.zhihu.com/p/26752744)的延迟序列，它可以是无限长的（类似[Haskell 中的无限长列表](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)）。我们可以计算其中一些，例如，通过 `take()`：

```kotlin
println(fibonacci.take(10).joinToString())
```

> 这会打印出 `1, 1, 2, 3, 5, 8, 13, 21, 34, 55 `。你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/fibonacci.kt)试一下。

生成器中支持任意的控制流，包括但不限于 `while`、`if`、`try`/`catch`/`finally`：

```kotlin
val seq = sequence {
    yield(firstItem) // 挂起点
    
    for (item in input) {
        if (!item.isValid()) break // 不再生成项
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // 挂起点        
    }
    
    try {
        yield(lastItem()) // 挂起点
    }
    finally {
        // 一些收尾代码
    }
} 
```

> 关于 `sequence{}` 和 `yield()` 的示例代码在[受限挂起]()一节。

注意，这种方法还允许把 `yieldAll(sequence)` 表示为库函数（像 `sequence{}` 和 `yield()` 那样），这能简化延时序列的连接操作，并允许高效的实现。

### 异步用户界面

典型的用户界面应用有一个事件调度线程，所有界面操作都发生在这个线程上。不允许在其他线程修改界面状态。所有用户界面库都提供某种原生方案，把操作挪到界面线程中执行。例如，Swing 的 [`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-)，JavaFX 的 [`Platform.runLater`](https://docs.oracle.com/javase/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-)，Android 的 [`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable)) 等等。这里有一段来自典型 Swing 应用的代码，执行一些异步操作，并把结果显示到用户界面：

```kotlin
makeAsyncRequest {
    // 异步操作完成时执行这个 λ
    result, exception ->
    
    if (exception == null) {
        // 在 UI 线程显示结果
        SwingUtilities.invokeLater {
            display(result)   
        }
    } else {
       // 异常处理
    }
}
```

这很像我们之间在[异步计算](#异步计算)用例见过的回调地狱，所以也能通过协程优雅地解决：

```kotlin
launch(Swing) {
    try {
        // 执行异步请求时挂起
        val result = makeRequest()
        // 在界面上显示结果，Swing 上下文保证了我们呆在事件调度线程上
        display(result)
    } catch (exception: Throwable) {
        // 异常处理
    }
}
```

> `Swing` 上下文的示例代码在[续体拦截器](#续体拦截器)一节。

所有的异常处理也都可以使用自然的语法结构执行。

### 更多用例

协程可以覆盖更多用例，比如下面这些：

* 基于通道的并发（就是 go 协程和通道）；
* 基于 Actor 模式的并发；
* 偶尔需要用户交互的后台进程，例如显示模式对话框；
* 通信协议：将每个参与者实现为一个序列，而不是状态机；
* Web应用程序工作流：注册用户、验证电子邮件、登录它们（挂起的协程可以序列化并存储在数据库中）。

## 协程概述

本部分概述了能够编写协程的语言机制和管理其语义的标准库。

### 术语

* *协程* —— *可挂起计算* 的*实例* 。它在概念上类似于一个线程，在这个意义上，它需要一个代码块运行，并具有类似的生命周期 —— 它可以被创建和启动，但它不绑定到任何特定的线程。它可以在一个线程中*挂起* 其执行， 并在另一个线程中*恢复* 。而且，像期货或诺言那样，它在*完成* 时可能伴随着结果（值或异常）。

* *挂起函数* —— `suspend` 修饰符标记的函数。它可能会通过调用其他挂起函数*挂起* 执行代码，而不阻塞当前执行线程。一个挂起函数不能在常规代码中被调用，只能在其他挂起函数或挂起 lambda 中（见下方）。举个例子，如用例所示的 `.await()` 和 `yield()` 是在库中定义的挂起函数。标准库提供了用于定义其他所有挂起函数所使用的基础挂起函数。

* *挂起 lambda* —— 一个必须在协程中运行的代码块。它看起来完全像一个普通的 lambda 表达式，但它的函数类型被  `suspend` 修饰符标记。就像常规 lambda 表达式是匿名局部函数的短语法形式一样，挂起 lambda 是匿名挂起函数的短语法形式。它可能会通过调用其他挂起函数*挂起* 执行代码，而不阻塞当前执行线程。举个例子，如用例所示，跟在 `launch` , `future` , 和 `BuildSequence` 函数后面花括号里的代码块，就是挂起 lambda。

  > 注意：如果允许 lambda 使用非局部的返回语句，则挂起 lambda 可以在代码的任意位置调用挂起函数。意思是说，可以在像 `apply{}` 这样的内联 lambda 中调用挂起函数，但在 `noinline` 和 `crossinline` 修饰的 lambda 中就不行。*挂起* 会被视作是一种特殊的非局部控制转移。

* *挂起函数类型*  —— 挂起函数和挂起 lambda 的函数类型。它就像一个常规的函数类型，但具有 `suspend` 修饰符。举个例子，`suspend () -> Int ` 是一个没有参数、返回 `Int` 的挂起函数的函数类型。一个像这样声明 —— `suspend fun foo(): Int` 的挂起函数符合上述函数类型。

* *协程建造者* —— 使用一些挂起 lambda 作为参数，创建一个协程，可选地提供某种形式的对其结果的访问的函数。举个例子，用例中的 `launch{}`, `future{}`, 和 `sequence{}` 是库中定义的协程建造者。 标准库提供了用于定义其他所有协程建造者所使用的基础协程建造者。

* *挂起点* —— 协程执行过程中可能会被*挂起* 的位置。从语法上说，一个挂起点是对一个挂起函数的调用，但*实际* 的挂起在挂起函数调用了标准库中的原始挂起函数时发生。

* *续体* —— 是一个挂起的协程在挂起点时的一个状态。它在概念上代表它在挂起点之后的剩余执行代码。一个例子：

  ```kotlin
  sequence {
      for (i in 1..10) yield(i * i)
      println("over")
  }  
  ```

  在这里，每次调用挂起函数 `yield()`，协程挂起时，*其执行的剩余* 被看作续体，所以我们有 10 个续体：第一次运行循环后 `i=2` ，挂起；第二次运行循环后 `i=3`，挂起……最后一个打印 "over" 并完成协程。该协程在此创建，但尚未启动，由它的初始*续体* 所表示，后者由它整个执行组成，类型为 `Continuation<Unit> ` 。

如上所述，驱动协程的要求之一是灵活性：我们希望能够支持许多现有的异步 API 和其他用例,，并将硬编码到编译器中的部分最小化。因此，编译器只负责支持挂起函数、挂起 lambda 时和相应的挂起函数类型。标准库中的原始函数很少， 其余的则留给应用程序库。

### Continuation 接口

这是标准库中接口 `Continuation` 的定义（位于 `kotlinx.coroutines` 包），代表了一个通用的回调：

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

context 将在[协程上下文](#协程上下文)一节中详细介绍，表示与协程关联的任意用户定义上下文。`resumeWIth` 函数是一个完结回调，用于报告协程完结时成功（带有值）或失败（带有异常）的结果。

为了方便，包里还定义了两个扩展函数：

```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### 挂起函数

一个典型的*挂起函数* 的某种实现，例如 `.await()` ，起来是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // 期货正常完成了
                cont.resume(result)
            else // 期货完成中产生了异常
                cont.resumeWithException(exception)
        }
    }
```

> 你可以在[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/await.kt)找到代码。这个简单的实现只要期货不完成就会永远挂起协程。[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实际实现还支持取消。

`suspend` 修饰符表明这个函数可以挂起协程的执行。这个特别的函数是作为类型 `CompletableFuture<T>` 的[扩展函数](https://kotlinlang.org/docs/reference/extensions.html)，以便其能正常地从左到右读，并且与实际执行顺序一致：

```kotlin
doSomethingAsync(...).await()
```

`suspend` 修饰符可以用于任何函数：顶层函数、扩展函数、成员函数，或操作符函数。

> 属性的取值器和设值器、构造器以及某些操作符函数（包括 `getValue`，`setValue`，`provideDelegate`，`get`，`set` 以及 `equals`）不能拥有 `suspend` 修饰符。这些限制将来可能会被消除。
>

挂起函数可以调用任何常规函数，但要真正挂起执行，必须调用一些其他的挂起函数。特别的是，这个 `await` 实现调用了在标准库中定义的顶层挂起函数 `suspendCoroutine`（位于 `kotlinx.coroutines` 包）：

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

当  `suspendCoroutine` 在一个协程中被调用时（它只可能在协程中被调用，因为它是一个挂起函数），它捕获了协程的执行状态到一个*续体* 实例，然后将其传给指定的 `block` 作为参数。为了恢复协程的执行，代码块之后需要调用 `continuation.resumeWith()`（直接调用或通过 `continuation.resume()` 或 `continuation.resumeWithException()` 调用）在该线程或其他线程中。*实际* 的协程挂起发生当 `suspendCoroutine` 代码块没有调用 `resumeWith` 就返回时。如果协程在代码块中直接被恢复，协程就不被看作已经暂停又继续执行。

传给 `continuation.resumeWith()` 的值作为调用 `suspendCoroutine` 的结果，进一步成为 `.await()` 的结果。

多次恢复同一个协程是不被允许的，并会产生  `IllegalStateException`。 

> 注意：这正是 Kotlin 协程与像 Scheme 这样的函数式语言中的顶层限定续体以及 Haskell 中的续体函子的关键区别。我们选择仅支持续体恢复一次，完全是出于实用主义考虑，因为所有这些预期的[用例](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#use-cases)都不需要多重续体。然而，还是可以在另外的库中实现多重续体，通过所谓协程本征，就是复制续体中捕获的协程状态，然后就可以再次恢复这个副本协程。

### 协程建造者

挂起函数不能够从常规函数中调用，所以标准库提供了用于在常规非挂起范围中开启协程的函数。这是简单的*协程建造者* `launch` 的实现：

```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```

> 你可以从[这里](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/run/launch.kt)获取代码。

这个实现使用了 [`Continuation(context) { ... }`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) 函数（来自 `kotlin.coroutines` 包），它提供了一个简写以实现包含其给定的 `context` 值和 `resumeWith` 函数所需的代码块的 `Continuation` 接口。这个续体作为*完成续体* 被传给 [`block.startCoroutine(...)`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html) 扩展函数（来自 `kotlin.coroutines` 包）。

协程在完成中将调用其 *完成续体*。其 `resumeWith` 函数将在协程因成功或失败达到*完成* 状态时调用。因为 `launch` 是那种“即发即弃”式的协程，它被定义成返回 `Unit` 的挂起函数，实际上是无视了其 `resume` 函数的结果。如果协程异常结束，当前线程的未捕获异常句柄将用于报告这个异常。

> 注意：这个简单实现返回了 `Unit` ，没有提供任何协程状态的访问。实际上在 [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 中的实现要更加复杂，因为它返回了一个 `Job` 实例，代表这个协程并且可以被取消。

context 在[协程上下文](#协程上下文)一节中详细介绍。`startCoroutine` 在标准库中作为无参和单参数的挂起函数类型的扩展函数：

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```

`startCoroutine` 创建协程并在当前线程中立刻启动执行（但请参阅下面的备注），直到第一个挂起点时返回。挂起点是协程中某个挂起函数的调用，由相应的挂起函数的代码来定义协程恢复的时间和方式。

> 注意：续体拦截器（来自上下文）在[后文](#续体拦截器)中会提到，它能够将协程的执行，*包括* 它的初始续体调度到另一个线程中。

### 协程上下文

协程上下文是一组可以附加到协程中的持久化用户定义对象。它可以包括负责协程线程策略的对象，日志，协程执行的安全性和事务方面，协程的标识和名称等等。下面是协程及其上下文的简单认识模型。把协程看作一个轻量线程。在这种情况下，协程上下文就像是一堆线程局部化变量。不同之处在线程局部化变量是可变的，协程上下文是不可变的，但对于协程这并不是一个严重的限制，因为他们是如此轻量以至于当需要改变上下文时可以很容易地开一个新的协程。

标准库没有包含上下文的任何具体实现，但拥有接口和抽象类，以便以可组合的方式在库中定义所有这些方面，因此来自不同库的各个方面可以和平共存在同一个上下文中。

从概念上讲，协程上下文是一组元素的索引集，其中每个元素有唯一的键。它是集合和映射的混合体。它的元素有像在映射中的那样的键，但它的键直接与元素关联，更像在集合中。标准库定义了  `CoroutineContext` 的最小接口（位于 `kotlinx.coroutines` 包）：

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

 `CoroutineContext`  本身支持四种核心操作：

* 操作符 `get` 支持通过给定键类型安全地访问元素。可以使用 `[..]` 写法，解释见 [Kotlin 操作符重载](https://kotlinlang.org/docs/reference/operator-overloading.html)。
* 函数 `fold` 类似于标准库中 [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) 扩展函数，提供迭代上下文中所有元素的方法。
* 操作符 `plus` 类似于标准库的 [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html) 扩展函数，返回两个上下文的组合, 同时加号右边的元素会替换掉加号左边具有相同键的元素。
* 函数 `minueKey` 返回不包含指定键的上下文。

协程上下文的一个 [`Element`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.html) 就是上下文本身。那是仅有这一个元素的独立上下文结构。这样就可以通过获取库定义的协程上下文元素并使用 `+` 连接它们，来创建一个复合上下文。举个例子，如果一个库定义的 `auth` 元素带着用户授权信息，其他库定义的 `threadPool` 对象带着一些协程执行信息，你就可以使用[协程建造者]() `launch{}` 建造使用组合上下文的 `launch(auth + CommonPool){...}` 调用。

> 注意： [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 提供了几个上下文元素，包括用于在一个共享后台线程池中调度协程的 `Dispatchers.Default`。

标准库提供 [`EmptyCoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html) —— 一个不包括任何元素的（空的） `CoroutineContext` 实例。

所有第三方协程元素应该继承标准库的 [`AbstractCoroutineContextElement`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-element/index.html) 类（位于 `kotlinx.coroutines` 包）。要在库中定义上下文元素，建议使用以下样式。下面的这个例子显示了一个假想的储存当前用户名的授权上下文元素：

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

> 可以在这里找到[示例](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/auth.kt)。

将上下文的 `Key` 定义为相应元素类的伴生对象能够流畅访问上下文中的相应元素。这是一个假想的的挂起函数实现，它需要检查当前用户名：

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // 做一些用户指定的事
}
```

它使用了 [`coroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html) 顶层属性（位于 `kotlinx.coroutines` 包），以在挂起函数中检索当前协程的上下文。

### 续体拦截器

让我们回想一下[异步用户界面](#异步用户界面)用例。异步界面应用程序必须保证协程程序体始终在界面线程中执行，尽管事实上各种挂起函数是在任意的线程中恢复协程执行。这是使用*续体拦截器* 完成的。首先，我们要充分了解协程的生命周期。思考一下用了协程建造者 `launch{}` 的代码片段：

```kotlin
launch(CommonPool) {
    initialCode() // 执行初始化代码
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



### 非阻塞睡眠

协程不应使用 [`Thread.sleep`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-)，因为它阻塞了线程。但是，通过 Java 的 [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html) 实现挂起的非阻塞 `delay` 函数是非常简单的。

```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
    Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont ->
    executor.schedule({ cont.resume(Unit) }, time, unit)
}
```

> 你可以从 [这里](https://github.com/Kotlin/kotlin-coroutines/blob/master/examples/delay/delay.kt) 获取到这段代码。注意：[kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)  同样提供了 `delay` 函数。





### 合作单线程多任务处理



## 异步 Sequences



### Channels



### Mutexes



## 进阶话题



### 资源管理和 GC



### 并发和线程



## 异步编程样式



## 实现细节

本节提供了协程实现细节的一小部分。它们隐藏在 [协程概述](#协程概述) 部分解释的构造代码块背后，内部类和代码的生成策略在不打破公共 API 和 ABI 的约定下是会改变的。

### Continuation 传递样式

挂起函数通过 Continuation-Passing-Style (CPS) 实现。每个挂起函数和挂起 lambda 都有一个附加的 `Continuation` 参数， continuation 隐式地在调用时传递。回想一下，[`await` 挂起函数](#挂起函数) 的定义是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

然而在 *CPS 变换* 之后，它的实际实现具有具有以下签名：

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

它的返回类型 `T` 移到了附加参数的 continuation 上。实现中的返回值 `Any?` 是用于代表挂起函数的动作。当挂起函数 *挂起* 协程时，函数返回一个特别的标识值 `COROUTINE_SUSPENDED`。当一个挂起函数没有挂起协程，协程继续执行时，它返回函数结果或者直接抛出异常。这样以来，`await` 函数实现中的返回值 `Any?` 实际上是 `T` 和  `COROUTINE_SUSPENDED` 的联合类型，这并不能在 Kotlin 的类型系统中表示出来。挂起函数的实现实际上不允许调用其栈帧中的 continuation，因为这可能导致长时间运行的协程栈溢出。标准库中的 `suspendCoroutine` 通过追踪 continuation 的调用保证了挂起函数与实际实现一致性的约定，无论 continuation 在何时怎样调用，对应用开发者隐藏它的复杂性。

### 状态机

有效的实现协程是非常重要的，即尽可能少地创建类和对象。许多语言通过 *状态机* 去实现，Kotlin 也是这样做的。在这种情况下，此方法会导致编译器只为挂起 lambda 创建一个类，在其中可能有任意数量的挂起点。

主要思想：一个将挂起函数编译为状态机，它的每个状态对应着挂起点。举个例子，我们看看具有两个挂起点的挂起代码块：

```kotlin
val a = a()
val y = foo(a).await() // 挂起点 #1
b()
val z = bar(a, y).await() // 挂起点 #2
c(z)
```

这个代码块有三个状态：

* 初始化（在挂起点之前）
* 在第一个挂起点之后
* 在第二个挂起点之后

每个状态都是这个代码块 continuation 的一个入口点（初始化 continuation 从第一行开始）。

代码会被编译为具有实现状态机方法的匿名类，一个持有状态机当前状态的字段，状态之间共享的局部变量字段，也可能有协程闭包的字段，但在这种情况下它是空的。下面的伪 Java 代码块使用 CPS 调用挂起函数 `await`：

```java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // 状态机的当前状态
    int label = 0
    
    // 协程的局部变量
    A a = null
    Y y = null
    
    void resume(Object data) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // 在此调用时, data 应为 null
        a = a()
        label = 1
        data = foo(a).await(this) // 'this' 作为 continuation 传递
        if (data == COROUTINE_SUSPENDED) return // 如果 await 执行挂起，则返回
      L1:
        // 外部代码已经恢复协程，传递 .await() 的结果作为 data
        y = (Y) data
        b()
        label = 2
        data = bar(a, y).await(this) // 'this' 作为 continuation 传递
        if (data == COROUTINE_SUSPENDED) return // 如果 await 执行挂起，则返回
      L2:
        // 外部代码已经恢复协程，传递 .await() 的结果作为 data 
        Z z = (Z) data
        c(z)
        label = -1 // 不允许执行其他步骤
        return
    }          
}    
```

请注意，这里有 `go` 运算符和标签，因为该例子描述了字节码中发生的情况，而不是源代码内容。

现在，当协程开始时，我们调用了它的 `resume()` —— `label` 是 `0`，然后我们跳去 `L0`，接着我们做一些工作，将 `label` 设为下一个状态 —— `1`，调用 `.await()`，如果协程执行挂起，则返回。当我们想继续执行时，我们再次调用 `resume()`，现在它继续到了 `L1`，做一些工作，将状态设为 `2`，调用 `.await()`，同样在挂起时返回。下一次它从 `L3` 继续，将状态设为 `-1`，这意味着 "结束了，没有更多工作要做了"。

循环内的挂起点只生成一个状态，因为循环也通过 `goto` 工作（根据条件）：

```kotlin
var x = 0
while (x < 10) {
    x += nextNumber().await()
}
```

生成为

```java
class <anonymous_for_state_machine> extends CoroutineImpl<...> implements Continuation<Object> {
    // The current state of the state machine
    int label = 0
    
    // local variables of the coroutine
    int x
    
    void resume(Object data) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        
      L0:
        x = 0
      LOOP:
        if (x > 10) goto END
        label = 1
        data = nextNumber().await(this) // 'this' is passed as a continuation 
        if (data == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await() as data 
        x += ((Integer) data).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // No more steps are allowed
        return 
    }          
}    
```



### 编译挂起函数



### 协程内部
