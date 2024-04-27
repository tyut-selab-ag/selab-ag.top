# C++20协程详解 -- 协程概念（转）

本文转自 [知乎 - 孙咖啡](https://www.zhihu.com/people/sun-ming-zhi-91)

[原文链接1 知乎ID:孙咖啡](https://zhuanlan.zhihu.com/p/693105590?utm_campaign=shareopn&utm_medium=social&utm_psn=1765768206265159680&utm_source=wechat_session)

[原文链接2 知乎ID:孙咖啡](https://zhuanlan.zhihu.com/p/693149493)

最近看了Lewis Baker写的协程文章，感觉写的非常非常好，之前看到的各种材料都没有从底层把协程的概念讲清楚，导致学的时候不明不白的，直到看完了这5篇文章，我才算是把协程彻底搞明白了，遂打算搬运过来。这个专栏打算写6篇，包括5篇理论再加一篇额外的协程实践。

本文的主体内容翻译自[Coroutine Theory](https://link.zhihu.com/?target=https%3A//lewissbaker.github.io/2017/09/25/coroutine-theory)，在翻译的过程中对原文中过时的概念进行了更新，并对复杂的概念进行了额外的更详细的阐述。在这篇文章中，我将介绍函数和协程之间的区别，并提供一些关于它们所支持操作的理论。这篇文章的目的是介绍一些基础概念，帮助你构建思考 C++ 协程的方式。

## 协程是函数，函数是协程

协程是函数的泛化，它允许函数暂停，之后再恢复。我将详细解释它的含义，但在此之前，我想先回顾一下 "普通 "C++ 函数是如何工作的。

## C++"普通 "函数

一个普通函数可以被认为有两个操作： 调用（Call）和返回（Return）（注意，我在这里将 "抛出异常 "笼统地归入返回操作）。调用操作会创建一个激活帧，暂停调用函数的执行，并将执行转移到被调用函数的起始位置。返回 操作将返回值传递给调用者，销毁激活帧，然后在调用者调用函数的位置之后恢复调用者的执行。下面让我们进一步分析一下这些语义。

### 激活帧

那么 "激活帧 "是什么东西呢？你可以把激活帧看作是保存函数的特定调用的当前状态的内存块。这个状态包括传递给它的任何参数的值和任何局部变量的值。

对于一般的函数，激活帧还包括返回地址（从函数返回时将执行转移到该地址）和调用函数的激活帧地址。我们可以把这些信息看作是函数调用的 "继续"，也就是说，它们描述了当函数调用完成时，哪个函数的哪个调用应该在哪个位置继续执行。

对于 "正常 "函数，所有激活帧都有严格的嵌套生命周期。这种严格的嵌套允许使用高效的内存分配数据结构来分配和释放每个函数调用的激活帧。这种数据结构通常称为 "栈"。在栈数据结构上分配的激活帧通常称为 "栈帧"。这种堆栈数据结构非常常见，以至于大多数（全部）CPU 架构都有一个专用寄存器来存放指向栈顶部的指针（例如，在 x64 中是 rsp 寄存器）。

### 调用（Call）操作

当一个函数调用另一个函数时，调用者必须首先做好暂停的准备。这个 "暂停 "步骤通常包括将当前保存在 CPU 寄存器中的任何值保存到内存中，以便以后函数恢复执行时可以根据需要恢复这些值。根据函数的调用约定，调用者和被调用者可能会就谁保存这些寄存器值进行协调，但不论如何这都是**调用**操作的一部分。调用者还将传递给被调用函数的任何参数值保存到新的激活帧中，以便函数访问。最后，调用者将恢复点地址写入新的激活帧，并将执行转移到被调用函数的起点。

在 X86/X64 体系结构中，这一最后操作有自己的指令，即`call`指令，它将下一条指令的地址写入堆栈，递增栈寄存器，然后跳转到指令操作数中指定的地址。

### 返回（return）操作

当函数通过`return`语句返回时，函数首先会将返回值（如果有的话）存储在调用者可以访问的地方。这可以是在调用者的激活帧中，也可以是在被调函数的激活帧中（参数和返回值跨越两个激活帧的边界时，区别可能会有点模糊）。

然后，函数通过以下方式销毁激活帧：

- 销毁被调用函数作用域（Scope）内的所有局部变量。
- 销毁所有的参数对象。
- 释放激活帧的内存。

最后，通过以下方式恢复调用者的执行：

- 通过设置栈寄存器，使其指向调用程序的激活帧，并恢复任何可能被之前函数调用占用的寄存器，从而恢复调用程序的激活帧。
- 跳转到 "调用 "操作中存储的调用者恢复点。请注意，与 "调用 "操作一样，某些调用约定可能会将 "返回 "操作的责任分担给调用者或被调用者函数的某些指令。

## 协程

协程通过将调用和返回操作中执行的一些步骤分离出来，将函数的操作泛化出三个额外的操作： 暂停（Suspend）、恢复（Resume）和销毁（Destroy）。

暂停（Suspend）操作在函数内的当前点暂停执行协程，并在不销毁激活帧的情况下将执行转回调用者。暂停执行后，暂停点范围内的任何对象都将保持存活。需要注意的是，与函数的 `return` 操作一样，协程只能在明确定义的暂停点从协程内部暂停。

恢复（Resume）操作可在暂停点恢复已暂停的协程的执行。这将重新激活协程的激活帧。

销毁（Destroy）操作会销毁激活帧，但不会恢复协程的执行。任何在挂起点所在作用域内的对象都将被销毁。用于存储激活帧的内存也将被释放。

### 协程激活帧

由于协程可以在不破坏激活帧的情况下被暂停，我们无法再保证激活帧的生命周期将严格嵌套。这意味着激活帧一般不能使用栈进行分配，因此可能需要存储在堆上。

C++20 协程中有一些规定，如果编译器能证明协程的生命周期确实严格嵌套在调用者的生命周期内，则允许从调用者的激活帧中分配协程需要的内存。如果编译器足够智能，这在很多情况下都可以避免堆分配。

在协程中，激活帧的某些部分需要在协程暂停时保留，而有些部分只需要在协程执行时保留。例如，范围不跨越任何协程暂停点的变量的生命周期有可能被存储在栈中。

从逻辑上讲，你可以认为协程的激活帧由两部分组成："协程帧 "和 "栈帧"。

协程帧（coroutine frame）保存着协程的部分激活帧，在协程被挂起时仍然存在，而栈帧（stack frame）部分只在协程执行时存在，并在协程挂起并将执行转回调用者/用户时释放。

### 暂停（Suspend）操作

协程的暂停（Suspend）操作允许协程在函数执行过程中暂停执行，并将执行转回到协程的调用者或转发者。

协程主体中的某些点被指定为暂停点。在 C++20 中，这些暂停点通过使用 `co_await` 或 `co_yield` 关键字来标识。

当协程遇到这些暂停点之一时，它首先会通过以下方式为恢复协程做好准备：

- 确保将寄存器中的所有值写入协程帧.
- 向协程帧写入一个值，指示协程在哪个挂起点挂起。这样，后续的恢复（Resume）操作就能知道在哪里恢复协程的执行，或者后续的销毁（Destroy）操作就能知道哪些值在作用域内，需要销毁。

一旦协程为恢复做好了准备，该协程就被视为 "**暂停**"。

这种在协程进入“暂停”状态后执行逻辑的能力允许协程被安排恢复，而无需同步，否则如果协程在进入“暂停”状态之前被安排恢复，则需要同步，因为存在暂停和恢复协程进行数据竞争（data race）的可能性。我将在以后的文章中详细介绍这一点。

然后，协程可以选择立即恢复/继续执行协程，也可以选择将执行转移回调用者/用户。如果将执行转移给调用者/用户，则会释放协程激活帧的栈帧部分，并将其从栈中弹出。

### 恢复（Resume）操作

可以对当前处于 "暂停 "状态的协程执行恢复操作。

当函数想要恢复协程时，它需要调用协程的特定方法。调用者恢复协程的方式是：在协程帧句柄上调用 `void resume()` 方法。就像普通函数调用一样，调用 `resume()` 会分配一个新的栈，并将调用者的返回地址存储在栈中，然后将执行转移到相应的协程。

不过，它不会将执行转移到协程的起点，而是将执行转移到协程中最后一次暂停的位置。为此，它会从协程帧中加载恢复点并跳转到该点。当协程下一次暂停或运行完成时，`resume()`的调用将返回并恢复调用函数的执行。

### 销毁（Destroy）操作

销毁操作会销毁协程帧，而不会恢复协程的执行。该操作只能在暂停的协程上执行。销毁操作的作用与恢复操作类似，都是重新激活协程的激活帧，包括分配新的栈帧和存储销毁操作调用者的返回地址。

然而，在释放协程帧使用的内存之前，它不会将执行转移到上一个挂起点的协程主体，而是将执行转移到另一个代码路径，在此将调用挂起点作用域内所有局部变量的析构函数。

与恢复操作类似，销毁操作通过调用暂停操作中提供的协程帧句柄上的 `void destroy()` 方法来销毁的特定激活帧。

### 协程的调用（Call）操作

协程的调用操作与普通函数的调用操作基本相同。事实上，从调用者的角度来看，两者并无区别。

不过，在协程中，调用操作不会在函数运行完成后才返回调用者，而是会在协程到达第一个挂起点（暂停点）时恢复调用者的执行。

在对协程执行调用操作时，调用者会分配一个新的栈帧、将参数写入栈帧、将返回地址写入栈帧并将执行转移到协程。这与调用普通函数完全相同。

然后，协程所做的第一件事就是在堆上分配一个协程帧，并将参数从栈帧复制/移动到协程帧中，这样参数的生命周期就会超过第一个挂起点。

### 协程的返回（return）操作

协程的返回操作与普通函数的返回操作略有不同。

当协程执行返回语句时，它会将返回值存储在某个地方（具体存储在哪里可以由协程自定义），然后销毁作用域内的所有局部变量（但不包括参数）。

然后，协程就有机会执行一些附加逻辑，然后再将执行转回调用者/用户。

这些附加逻辑可能会执行一些操作来发布（publish）返回值，也可能会恢复另一个正在等待结果的协程。这是完全可定制的。

然后，协程执行暂停操作（保持协程帧存活）或销毁操作（销毁协程帧）。

然后，根据 Suspend/Destroy 操作语义，将执行转回调用者/用户，并将激活帧的栈帧组件从栈中弹出。

值得注意的是，传递给返回操作的返回值与调用操作返回的返回值不同，因为返回操作可能在调用者从初始调用操作恢复后很久才执行。

## 演示说明

为了帮助大家理解这些概念，我想通过一个简单的例子来说明协程被调用、暂停和恢复时会发生什么。假设我们有一个函数（或协程）`f()`，它调用了一个协程 `x(int a)`。在调用之前，我们的情况看起来有点像这样：

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-a110bbaab6ba42d919df88e119799dd2_1440w.webp)

那么当调用 `x(42)` 时，首先会为 `x()` 创建一个栈帧，就像普通函数一样。

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-facdb3a97ee85e0088b04f338f505db2_1440w.webp)

然后，一旦协程 `x()` 在堆上为协程帧分配了内存，并将参数值复制/移动到协程帧中，我们就会得到下图所示的结果。请注意，编译器通常会将协程帧的地址保存在栈指针之外的另一个寄存器中（例如，MSVC 将其保存在 `rbp`寄存器中）。

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-414936172c40a31db115424eef65006d_1440w.webp)

然后如果协程 `x()` 调用另一个普通函数 `g()`，就会出现这样的结果：

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-cf32cbab2cfa1ed26b798888951cf622_1440w.webp)

当`g()`返回时，它将销毁其激活帧，并恢复`x()`的激活帧。比方说，我们将`g()`的返回值保存在一个局部变量 `b` 中，该变量存储在协程帧中：

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-24c4f96b481c7087c75afa43ae3810ff_1440w.webp)

如果 `x()` 现在遇到挂起点，并在不破坏其激活帧的情况下暂停执行，那么执行将返回到 `f()`。

这将导致 `x()` 的栈帧部分从栈中弹出，而协程帧留在堆上。协程第一次暂停时，会向调用者返回一个返回值。该返回值通常包含一个与暂停的协程帧相关的句柄，可用于以后恢复该协程帧。当 `x()` 暂停时，它也会将 `x()` 的恢复点地址（RP）存储在协程帧中。

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-f09bcb556cb6be352910b7075d4e73d9_1440w.webp)

现在，这个句柄可以作为一个普通值在函数之间传递。在以后的某个时刻，可能来自不同的调用栈，甚至在不同的线程上，某些函数（比如 h()）将决定恢复该协程的执行。例如，当异步 I/O 操作完成时。

恢复协程的函数会调用 `void resume(handle)` 函数来恢复协程的执行。对于调用者来说，这看起来就像其他任何对带有单个参数的`void`返回函数的正常调用一样。这将创建一个新的栈帧，记录调用者对 `resume()` 的返回地址。通过将协程的地址加载到寄存器来激活协程帧，并在协程帧中存储的恢复点恢复 `x()` 的执行。

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-dae7f1d4dd6f2e8baa67156586ad6474_1440w.webp)

## 总结

我将协程描述为函数的泛化，除了 "普通 "函数提供的 "调用 "和 "返回 "操作外，它还具有 "暂停"、"恢复 "和 "销毁 "三种额外操作。我希望这能为如何思考协程帧及其控制流提供一些有用的思维框架。在下一篇文章中，我将介绍 C++20 协程的机制，并解释编译器如何将你编写的代码转化为协程。

# C++20协程详解 -- 协程概念（二）

本文的主体内容翻译自[C++ Coroutines: Understanding operator co_await](https://link.zhihu.com/?target=https%3A//lewissbaker.github.io/2017/11/17/understanding-operator-co-await)，在翻译的过程中对原文中过时的概念进行了更新，并对复杂的概念进行了额外的更详细的阐述。

在上一篇关于协程理论的文章中（**[C++20协程详解（1） -- 协程概念](https://zhuanlan.zhihu.com/p/693105590)**），我描述了函数和协程之间的高层次区别，但没有详细介绍 C++20 协程（N4680）所描述的协程的语法和语义。C++20协程提案为 C++ 语言添加的关键新功能是暂停协程的能力，并允许稍后再恢复协程。该提案为此提供的机制是新的 **co_await** 运算符。

了解 **co_await** 运算符的工作原理有助于揭开协程行为的神秘面纱，并了解它们是如何暂停和恢复的。在本篇文章中，我将解释 **co_await** 运算符的机制，并介绍相关的 **Awaitable** 和 **Awaiter** 类型概念。但在深入研究 **co_await** 之前，我想先简要介绍一下C++20的协程，以提供一些背景知识。

## C++20 协程给我们带来了什么？

- 三个新语言关键字：**co_await**、**co_yield** 和 **co_return。**

- `std`命名空间中的几种新类型：

- - `coroutine_handle<P>`
  - `coroutine_traits<Ts...>`
  - `suspend_always`
  - `suspend_never`

- 库编写者可用于与协程交互并自定义其行为的通用机制。

- 一种让编写异步代码变得更容易的语言工具！

C++20协程提供的设施可以看作是协程的低级汇编语言。这些设施很难直接安全地使用，主要供库编写者用来构建高层封装，以便后续的应用程序开发人员可以安全使用的。不过在后续的C++标准中也将提供一些安全的高层封装（比如C++23的std::generator）。

## 编译器 <-> 库交互

有趣的是，C++20协程实际上并没有定义协程的语义。它没有定义如何产生返回值给调用者；它没有定义如何处理传递给 **co_return** 语句的返回值；也没有定义如何处理从协程传播出去的异常；甚至也没有定义应在哪个线程上恢复协程。相反，C++20协程为库代码指定了一种通用机制，通过自定义特定的接口类来定制协程的行为。然后，编译器会生成代码，调用上述的自定义接口。这种方法类似于库编写者可以通过定义 `begin()/end()` 方法来定制基于范围的 for 循环。

事实上，C++20协程并不对协程的机制规定任何特定语义，这使其成为一个强大的工具。它允许库编写者定义多种不同类型的协程，用于各种不同的目的。例如，你可以定义一个异步生成单个值的协程，或者定义一个懒生成一系列值的协程，或者定义一个在遇到 `nullopt` 值时提前退出以简化消耗`optional<T>`值的控制流的协程。

C++20协程定义了两个接口类：**Promise** 接口类和 **Awaitable** 接口类。

**Promise** 接口类指定了用于自定义协程本身行为的方法。库编写者可以自定义调用协程时发生的情况、协程返回时发生的情况（通过正常方式或未处理异常），以及自定义协程中任何 **co_await** 或 **co_yield** 表达式的行为。

**Awaitable** 接口类指定了一些方法，用来控制 **co_await** 表达式的语义。当一个值被 **co_await**ed 时，代码会被转换成一系列对**Awaitable**对象上的方法的调用，这些方法允许它指定：是否暂停当前协程；在暂停后执行一些逻辑以调度协程的后续恢复；以及在协程恢复后执行一些逻辑以产生 **co_await** 表达式的结果。

我将在以后的文章中介绍 **Promise** 接口的细节，现在先让我们来看看 **Awaitable** 接口。

## Awaiters 和 Awaitables: 理解 co_await 运算符

**co_await** 运算符是一个新的一元运算符，可以应用于一个值。例如：`co_await someValue`。**co_await** 运算符只能在协程中使用。这有点像同义反复，因为根据定义，任何包含 **co_await** 运算符的函数体都将被编译为协程。

支持 **co_await** 运算符的类型称为 **Awaitable**（可等待类型）。请注意，**co_await** 运算符能否应用于某个类型取决于 **co_await** 表达式出现的上下文。协程使用的 Promise 接口类可以通过其 `await_transform` 方法改变协程中 **co_await** 表达式的含义（稍后详述）。

为了在必要时更加具体的描述**Awaitable**，我喜欢使用术语 Normally Awaitable 来描述支持 **co_await** 运算符的类型，并且该类型的 promise 类型没有 `await_transform` 成员。相反的，使用术语 Contextually Awaitable 来描述：由于协程的 promise 类型中存在 `await_transform` 方法，使得该类型仅在特定的上下文中支持 **co_await** 运算符。

**Awaiter** 类型实现了三个特殊方法：`await_ready`、`await_suspend` 和 `await_resume`，这些方法将作为 **co_await** 表达式的一部分被调用。请注意，一个类型既可以是 **Awaitable** 类型，也可以是 **Awaiter** 类型。

当编译器看到一个 `co_await <expr>` 表达式时，根据所涉及的类型，会有许多可能的翻译方式，下面我们深入研究一下这个过程。

### 获取 Awaiter 对象

编译器首先要做的是生成代码，以获取co_await表达式作用于的**Awaiter**对象。假设协程的 Promise 对象的类型为 P，并且 `promise` 是对当前协程的 Promise 对象的左值引用。如果 Promise 类型 P 有一个名为` await_transform` 的成员，那么 `<expr>` 将首先被传递到 `promise.await_transform(<expr>)` 的调用中，以获得 **Awaitable** 对象，记作`awaitable`。否则，如果 Promise 类型中没有 `await_transform` 成员，我们就直接使用 `<expr>` 的求值结果作为 **Awaitable** 对象，同样记作`awaitable`。

然后，如果 `awaitable` 有适用的 `operator co_await()` 重载，则调用该重载以获得 **Awaiter** 对象。否则，`awaitable` 将被用作**Awaiter**对象。

如果我们将这些规则编码成函数 `get_awaitable()` 和 `get_awaiter()`，它们可能看起来像这样：

```cpp
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
```

### 等待Awaiter

假设我们已将把 `<expr>` 转化为一个**Awaiter**对象的逻辑封装到上述函数中，那么 `co_await <expr>` 的语义可以（大致）翻译如下：

```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::experimental::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(p));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
```

当 `await_suspend()` 函数返回时，`void` 返回版本的 `await_suspend()` 会无条件地将执行流转回协程的调用者/恢复者，而 `bool` 返回版本的 `await_suspend()` 则允许 awaiter 对象根据条件选择立即恢复协程还是返回调用者/恢复者。

`await_suspend()` 的 `bool` 返回版本在 awaiter 启动一个同步操作的情况下非常有用。在同步操作完成时，`await_suspend()` 方法可以返回 `false`，表示应立即恢复协程并继续执行。

在 `<suspend-coroutine>` 处，编译器会生成一些代码来保存协程的当前状态，并为恢复执行做好准备。这包括存储 `<resume-point>` 的位置，以及将当前寄存器中的值保存到协程帧中。

`<suspend-coroutine>` 操作完成后，当前协程将被视为暂停。第一个可以观察到暂停的协程位置是在`await_suspend()`函数内。一旦协程被暂停，它就可以被恢复或销毁。

`await_suspend()` 方法在完成操作后有责任安排协程在将来的某个时刻恢复（或销毁）。请注意，从`await_suspend()` 返回`false` 会调度协程以在当前线程上立即恢复。

`await_ready()` 方法存在的目的是：在已知操作将同步完成而无需挂起的情况下避免 `<suspend-coroutine>`操作的成本。

在 `<return-to-caller-or-resumer>` 点，执行被转回调用者/恢复者，弹出本地栈帧，但保持协程帧的存活。

当被挂起的协程最终被恢复时，执行会在`<resume-point>`处继续。也就是在`await_resume()`方法调用之前立即恢复执行。

`await_resume()`方法调用的返回值成为**co_await**表达式的结果。`await_resume()`方法也可以抛出异常，此时异常会从**co_await**表达式中传播出去。请注意，如果`await_suspend()`调用中发生异常，则协程会自动恢复，并且异常会从**co_await**表达式中传播出去，而不会调用`await_resume()`方法。

## 协程句柄

你可能注意到了在**co_await**表达式的`await_suspend()`调用中传递的`coroutine_handle<P>`类型。该类型表示对协程帧的非拥有式句柄，可用于恢复协程的执行或销毁协程帧。它还可以用于访问协程的promise对象。

`coroutine_handle`类型具有以下接口：

```cpp
namespace std
{
  template<typename Promise>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void>
  {
    bool done() const;

    void resume();
    void destroy();

    void* address() const;
    static coroutine_handle from_address(void* address);
  };

  template<typename Promise>
  struct coroutine_handle : coroutine_handle<void>
  {
    Promise& promise() const;
    static coroutine_handle from_promise(Promise& promise);

    static coroutine_handle from_address(void* address);
  };
}
```

在实现**Awaitable**类型时，你将会使用的`coroutine_handle`的关键方法`.resume()`，当操作完成并且你想要恢复等待中的协程时，应该调用它。在`coroutine_handle`上调用`.resume()`会在`<resume-point>`处重新激活挂起的协程。调用`.resume()`的操作会在协程下次到达`<return-to-caller-or-resumer>`时返回。

`.destroy()`方法会销毁协程帧，调用所有在作用域内的变量的析构函数，并释放协程帧使用的内存。通常情况下，你不需要（并且确实应该避免）调用`.destroy()`，除非你是实现协程Promise类型的库编写者。通常，协程帧将由从协程的调用返回的某种RAII类型所拥有。如果没有与RAII对象的协作，调用`.destroy()`可能会导致双重销毁的bug。

`.promise()`方法返回对协程的promise对象的引用。然而，与`.destroy()`一样，它通常只在编写协程Promise类型时有用。你应该将协程的promise对象视为协程的内部实现细节。对于大多数常规的**Awaitable**类型，你应该在`await_suspend()`方法的参数类型中使用`coroutine_handle<void>`，而不是`coroutine_handle<Promise>`。

`coroutine_handle<P>::from_promise(P& promise)`函数允许从promise对象的引用重新构造协程句柄。请注意，你必须确保类型P与协程的Promise类型完全匹配。尝试从Promise类型为Derived的协程构造`coroutine_handle<Base>`可能会导致未定义行为（其中Derived是Base的子类）。

`.address()` / `from_address()`函数允许将协程句柄转进行`void*`指针的转换。这主要用于将其作为上下文参数传递给现有的C风格API，因此在某些情况下，你可能会发现在实现**Awaitable**类型时它很有用。然而，在大多数情况下，我发现有必要通过这个上下文参数传递额外的信息给回调函数，因此我通常会将`coroutine_handle`存储在一个结构体中，并将结构体的指针作为上下文参数传递，而不使用`.address()`的返回值。

## 无同步的异步代码

**co_await**运算符的一个强大特性是：允许协程被挂起后但在执行返回给调用者/恢复者之前执行代码的能力。这使得**Awaiter**对象能够在协程已经挂起后启动异步操作，将挂起的协程的`coroutine_handle`传递给这个操作，并在操作完成时（可能在另一个线程上）安全地恢复协程，而无需任何额外的同步。例如，在协程已经挂起时在`await_suspend()`中启动异步读取操作意味着我们只需在操作完成时恢复协程，而无需进行任何线程同步来协调启动操作的线程和完成操作的线程。

![img](https://raw.githubusercontent.com/Ar-Gas/Ar-Gas.github.io/main/photo/v2-a78f97ff704a622ebb39e4b66a1f4e17_1440w.webp)

在利用这种方法时要非常小心的一点是，一旦你把协程句柄传递给了其他的线程，那么在`await_suspend()`返回之前，另一个线程可能已经启动了恢复协程的操作，并可能与剩下的`await_suspend()`代码并行执行。

协程在恢复时首先要做的是调用`await_resume()`来获取结果，然后通常会立即销毁**Awaiter**对象（即`await_suspend()`调用的this指针）。协程甚至可能会在`await_suspend()`返回之前完成运行，并销毁协程和promise对象。

因此，在`await_suspend()`方法中，一旦协程有可能在另一个线程上并发恢复，你需要确保避免访问`this`或协程的`.promise()`对象，因为两者都可能已经被销毁。通常情况下，在操作启动并安排协程恢复后，只有在`await_suspend()`内部的局部变量是可以安全访问的。

## 与有栈协程的比较

让我们简单比较一下C++20中的无栈协程与一些现有的常见有栈协程框架（如Win32 fibers或boost::context）。

C++20中的无栈协程具有在协程挂起后执行逻辑的能力。然而在许多有栈协程框架中，协程的暂停操作与另一个协程的恢复操作合并为一个“上下文切换”操作。在这个“上下文切换”操作中，通常没有机会在挂起当前协程之后、在将执行转移给另一个协程之前执行代码。

这意味着，如果我们想要在有栈协程之上实现类似的异步文件读取操作，那么我们必须在暂停协程之前启动操作。因此，有可能在协程暂停并准备恢复之前，操作已经在另一个线程上完成。另一个线程上完成操作和协程暂停之间的潜在竞争需要一些线程同步机制来协调。

可能有一些方法可以解决这个问题，比如使用一个跳板上下文（trampoline context）。然而，这将需要额外的基础设施和额外的上下文切换来实现，而且可能引入的开销可能比它试图避免的同步开销还要大。

## 避免动态内存分配

异步操作通常需要存储一些操作状态的数据，用于跟踪操作的进度。这种状态通常需要在整个操作的持续时间内保持，并且只有在操作完成后才能释放。例如，调用异步的Win32 I/O函数需要分配并传递一个指向`OVERLAPPED`结构的指针。调用者负责确保该指针在操作完成之前保持有效。

在传统的基于回调的API中，为了确保状态具有适当的生命周期，通常需要在堆上分配这种状态。如果执行许多操作，可能需要为每个操作分配和释放这种状态。如果性能是一个问题，可以使用自定义分配器从内存池中分配这些状态对象。

然而，当我们使用协程时，我们可以通过利用协程挂起时保持协程帧内的局部变量存活的特性，避免为操作状态分配堆存储空间。

通过将每个操作的状态放置在**Awaiter**对象中，我们可以有效地从协程帧中“借用”内存，用于存储**co_await**表达式的整个操作状态。一旦操作完成，协程恢复执行，**Awaiter**对象被销毁，该内存将被释放给协程帧中的其他局部变量使用。

最终，协程帧可能仍然在堆上分配。然而，一旦分配完成，一个协程帧可以用于执行许多异步操作，只需要进行一次堆分配。仔细思考一下，协程帧就像一个高性能的内存分配器。编译器在编译时计算出所有局部变量所需的总体内存大小，并能够根据需要将这块内存分配给局部变量，而没有任何额外开销！

## **示例：实现简单的线程同步原语**

现在我们已经探讨了**co_await**运算符的许多机制，我想展示如何将这些知识应用于实践，下面我将实现一个基本的可等待同步原语：异步手动复位事件。

这个事件的基本要求是它可以被多个并发执行的协程等待，并且在等待时需要暂停等待的协程，直到某个线程调用`.set()`方法，此时任何等待中的协程都会被恢复。如果某个线程已经调用了`.set()`，那么协程应该继续执行而不暂停。理想情况下，我们希望它是**noexcept**的，不需要堆分配，并且具有无锁的实现。

使用示例如下：

```cpp
T value;
async_manual_reset_event event;

// A single call to produce a value
void producer()
{
  value = some_long_running_computation();

  // Publish the value by setting the event.
  event.set();
}

// Supports multiple concurrent consumers
task<> consumer()
{
  // Wait until the event is signalled by call to event.set()
  // in the producer() function.
  co_await event;

  // Now it's safe to consume 'value'
  // This is guaranteed to 'happen after' assignment to 'value'
  std::cout << value << std::endl;
}
```

让我们首先思考一下这个事件可能的状态：'未设置'和'已设置'。当处于'未设置'状态时，可能有一个（可能为空）的协程等待列表，它们正在等待事件变为'已设置'。

当处于'已设置'状态时，不会有任何等待中的协程，因为在该状态下，**co_await**该事件的协程可以继续执行而不需要挂起。

实际上，这个状态可以用一个`std::atomic<void*>`来表示。

- 为'已设置'状态保留一个特殊的指针值。在这种情况下，我们将使用事件的`this`指针，因为我们知道它不可能与等待列表中的任何地址相同。
- 否则，事件处于'未设置'状态，其值是指向等待协程结构体链表头部的指针。

我们可以通过将节点存储在awaiter对象中（awaiter对象在协程帧内），从而避免额外在堆上分配链表节点。

让我们从一个类接口开始，如下所示：

```cpp
class async_manual_reset_event
{
public:

  async_manual_reset_event(bool initiallySet = false) noexcept;

  // No copying/moving
  async_manual_reset_event(const async_manual_reset_event&) = delete;
  async_manual_reset_event(async_manual_reset_event&&) = delete;
  async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
  async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

  bool is_set() const noexcept;

  struct awaiter;
  awaiter operator co_await() const noexcept;

  void set() noexcept;
  void reset() noexcept;

private:

  friend struct awaiter;

  // - 'this' => set state
  // - otherwise => not set, head of linked list of awaiter*.
  mutable std::atomic<void*> m_state;

};
```

在这里，我们有一个相当直接且简单的接口。需要注意的是，它具有一个`operator co_await()`方法，该方法返回一个尚未定义的类型——awaiter。现在让我们定义awaiter类型。

### 定义Awaiter

首先，它需要知道它将等待哪个`async_manual_reset_event`对象，因此它需要引用该事件并提供一个构造函数来进行初始化。它还需要作为一个awaiter链表中的节点，因此需要保存指向链表中下一个awaiter对象的指针。

它还需要存储执行**co_await**表达式的协程的`coroutine_handle`，以便在事件变为'已设置'时可以恢复协程。我们不关心协程的promise类型，因此我们将使用`coroutine_handle<>`（这是`coroutine_handle<void>`的简写）。

最后，它需要实现**Awaiter**接口，因此需要这三个特殊方法：`await_ready`、`await_suspend`和`await_resume`。由于我们不需要从**co_await**表达式返回值，因此`await_resume`可以返回`void`。

将所有这些组合在一起，awaiter的基本类接口如下所示：

```cpp
struct async_manual_reset_event::awaiter
{
  awaiter(const async_manual_reset_event& event) noexcept
  : m_event(event)
  {}

  bool await_ready() const noexcept;
  bool await_suspend(std::coroutine_handle<> awaitingCoroutine) noexcept;
  void await_resume() noexcept {}

private:

  const async_manual_reset_event& m_event;
  std::coroutine_handle<> m_awaitingCoroutine;
  awaiter* m_next;
};
```

现在，当我们对事件进行**co_await**时，如果事件已经被设置，我们不希望等待的协程被挂起。因此，我们可以定义`await_ready()`方法，在事件已经设置时返回`true`。

```cpp
bool async_manual_reset_event::awaiter::await_ready() const noexcept
{
  return m_event.is_set();
}
```

接下来，让我们看一下`await_suspend()`方法。这通常是**Awaiter**类型中最神奇的地方。

首先，它需要将等待的协程的协程句柄存储到`m_awaitingCoroutine`成员中，以便`async_manual_reset_event`类稍后可以在其上调用`.resume()`。然后，一旦完成上述操作，我们需要尝试原子地将awaiter排队到等待者的链表中。如果成功排队，我们返回`true`，表示我们不希望立即恢复协程；否则，如果我们发现事件已经被并发地更改为'已设置'状态，则返回`false`，表示协程应立即被恢复。

```cpp
bool async_manual_reset_event::awaiter::await_suspend(
  std::coroutine_handle<> awaitingCoroutine) noexcept
{
  // Special m_state value that indicates the event is in the 'set' state.
  const void* const setState = &m_event;

  // Remember the handle of the awaiting coroutine.
  m_awaitingCoroutine = awaitingCoroutine;

  // Try to atomically push this awaiter onto the front of the list.
  void* oldValue = m_event.m_state.load(std::memory_order_acquire);
  do
  {
    // Resume immediately if already in 'set' state.
    if (oldValue == setState) return false; 

    // Update linked list to point at current head.
    m_next = static_cast<awaiter*>(oldValue);

    // Finally, try to swap the old list head, inserting this awaiter
    // as the new list head.
  } while (!m_event.m_state.compare_exchange_weak(
             oldValue,
             this,
             std::memory_order_release,
             std::memory_order_acquire));

  // Successfully enqueued. Remain suspended.
  return true;
}
```

请注意，我们在加载旧状态时使用acquire内存顺序，这样如果我们读取到特殊的"已设置"值，就可以看到在调用`set()`之前发生的写操作。

如果比较交换成功，我们需要release语义，这样后续对`set()`的调用将会看到我们对`m_awaitingCoroutine`的写操作以及协程状态之前的写操作。

### 完善事件类的其余部分

现在我们已经定义了`awaiter`类型，让我们回过头来看一下`async_manual_reset_event`类的方法的实现。

首先是构造函数。它需要初始化为'未设置'状态，即等待者列表为空（`nullptr`）；或者初始化为'已设置'状态（`this`）。

```cpp
async_manual_reset_event::async_manual_reset_event(
  bool initiallySet) noexcept
: m_state(initiallySet ? this : nullptr)
{}
```

接下来，`is_set()` 方法就非常简单了：如果它具有特殊值 `this`，那么它就是'已设置'状态：

```cpp
bool async_manual_reset_event::is_set() const noexcept
{
  return m_state.load(std::memory_order_acquire) == this;
}
```

接下来是 `reset()` 方法。如果它处于 "已设置 "状态，我们要将其转换回空的 "未设置 "状态，否则保持原样。

```cpp
void async_manual_reset_event::reset() noexcept
{
  void* oldValue = this;
  m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
}
```

对于 `set()` 方法，我们希望通过将当前状态与`this`值交换，过渡到 "已设置 "状态，然后检查旧值是什么。如果有任何等待的协程，那么我们要依次恢复每个协程，最后再返回。

```cpp
void async_manual_reset_event::set() noexcept
{
  // Needs to be 'release' so that subsequent 'co_await' has
  // visibility of our prior writes.
  // Needs to be 'acquire' so that we have visibility of prior
  // writes by awaiting coroutines.
  void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
  if (oldValue != this)
  {
    // Wasn't already in 'set' state.
    // Treat old value as head of a linked-list of waiters
    // which we have now acquired and need to resume.
    auto* waiters = static_cast<awaiter*>(oldValue);
    while (waiters != nullptr)
    {
      // Read m_next before resuming the coroutine as resuming
      // the coroutine will likely destroy the awaiter object.
      auto* next = waiters->m_next;
      waiters->m_awaitingCoroutine.resume();
      waiters = next;
    }
  }
}
```

最后，我们需要定义 `operator co_await()` 方法。这只需要构造一个`awaiter`对象。

```cpp
async_manual_reset_event::awaiter
async_manual_reset_event::operator co_await() const noexcept
{
  return awaiter{ *this };
}
```

这样我们就完成了一个可等待的异步手动复位事件，并且是无锁、无额外内存分配、无异常的实现。你还可以在 [cppcoro](https://link.zhihu.com/?target=https%3A//github.com/lewissbaker/cppcoro)库中找到该类的实现，以及其他一些有用的可等待类型，如 `async_mutex` 和 `async_auto_reset_event`。

### 测试

我们将所有代码整合到一起，并做一个简单的测试：

```cpp
#include <coroutine>
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>
class async_manual_reset_event {
    friend struct awaiter;
public:
    async_manual_reset_event(bool initiallySet = false) noexcept : m_state(initiallySet ? this : nullptr) {};
    async_manual_reset_event(const async_manual_reset_event&) = delete;
    async_manual_reset_event(async_manual_reset_event&&) = delete;
    async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
    async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;
    bool is_set() const noexcept;
    struct awaiter;
    awaiter operator co_await() const noexcept;
    void set() noexcept;
    void reset() noexcept;
private:
    mutable std::atomic<void*> m_state;
};
struct async_manual_reset_event::awaiter {
    friend class async_manual_reset_event;
    awaiter(const async_manual_reset_event& event) noexcept
        : m_event(event) {}
    bool await_ready() const noexcept;
    bool await_suspend(std::coroutine_handle<> awaitingCoroutine) noexcept;
    void await_resume() noexcept {}
private:
    const async_manual_reset_event& m_event;
    std::coroutine_handle<> m_awaitingCoroutine;
    awaiter* m_next;
};
bool async_manual_reset_event::awaiter::await_ready() const noexcept {
    return m_event.is_set();
}
bool async_manual_reset_event::awaiter::await_suspend(std::coroutine_handle<> awaitingCoroutine) noexcept {
    // Special m_state value that indicates the event is in the 'set' state.
    const void* const setState = &m_event;
    // Stash the handle of the awaiting coroutine.
    m_awaitingCoroutine = awaitingCoroutine;
    // Try to atomically push this awaiter onto the front of the list.
    void* oldValue = m_event.m_state.load(std::memory_order_acquire);
    do {
        // Resume immediately if already in 'set' state.
        if (oldValue == setState) return false;
        // Update linked list to point at current head.
        m_next = static_cast<awaiter*>(oldValue);
        // Finally, try to swap the old list head, inserting this awaiter
        // as the new list head.
    } while (!m_event.m_state.compare_exchange_weak(
        oldValue,
        this,
        std::memory_order_release,
        std::memory_order_acquire));
    // Successfully enqueued. Remain suspended.
    return true;
}
bool async_manual_reset_event::is_set() const noexcept {
    return m_state.load(std::memory_order_acquire) == this;
}
void async_manual_reset_event::reset() noexcept {
    void* oldValue = this;
    m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
}
void async_manual_reset_event::set() noexcept {
    // Needs to be 'release' so that subsequent 'co_await' has
    // visibility of our prior writes.
    // Needs to be 'acquire' so that we have visibility of prior
    // writes by awaiting coroutines.
    void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
    if (oldValue != this) {
        // Wasn't already in 'set' state.
        // Treat old value as head of a linked-list of waiters
        // which we have now acquired and need to resume.
        auto* waiters = static_cast<awaiter*>(oldValue);
        while (waiters != nullptr) {
            // Read m_next before resuming the coroutine as resuming
            // the coroutine will likely destroy the awaiter object.
            auto* next = waiters->m_next;
            waiters->m_awaitingCoroutine.resume();
            waiters = next;
        }
    }
}
async_manual_reset_event::awaiter async_manual_reset_event::operator co_await() const noexcept {
    return awaiter{ *this };
}

struct task { // A simple task-class for void-returning coroutines.
    struct promise_type {
        task get_return_object() noexcept { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() noexcept {}
    };
};
async_manual_reset_event event{ false };
void producer() {
    using namespace std::literals;
    std::this_thread::sleep_for(300ms);
    // Publish the value by setting the event.
    std::cout << "Producer: " << std::this_thread::get_id() << std::endl;
    event.set();
}
task consumer() {
    co_await event;
    std::cout << std::this_thread::get_id() << std::endl;
}
int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(consumer);
        using namespace std::literals;
        std::this_thread::sleep_for(10ms);
    }
    /*
    * 所有的consumer线程都可以在producer调用前完成：
    * 1. 如果event==true，那么consumer线程可以直接执行完成。
    * 2. 如果event==false，那么consumer线程会直接返回，等producer完成后，在producer中完成相应任务。
    */
    for (auto& thread : threads) {
        thread.join();
    }
    producer();
    return 0;
}
```

在上面的代码里，我们先开启consumer线程，再开启producer线程。

- 如果事件的初始值为`true`，那么无需任何的等待操作，所有的任务都会在不同的线程上直接完成（输出不同的`this_thread::get_id()`）。
- 反之，如果事件的初始值为`false`，那么所有的consumer都会在**co_await**处暂停，然后返回调用者（这会使得当前线程直接结束）。当producer设置事件之后，所有的等待任务都会在producer线程上完成（输出相同的`this_thread::get_id()`）。

## 结语

本文介绍了`operator co_await`如何根据**Awaitable**和**Awaiter**的概念进行实现和定义。同时，它还详细介绍了如何实现一个可等待的异步线程同步原语，并充分利用了awaiter对象在协程帧上分配内存的特性，避免了额外的堆内存分配。希望本文能帮助你更好地理解新的**co_await**运算符。

在下一篇文章中，我将探讨Promise接口以及协程类作者如何自定义协程的行为。