# WebAssembly 对比 JavaScript 及其使用场景

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-a-comparison-with-webassembly-why-in-certain-cases-its-better-to-use-it-d80945172d79)，略有改动，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是  JavaScript 工作原理的第六章。**

现在，我们将会剖析 WebAssembly 的工作原理，而最重要的是它和 JavaScript 在性能方面的比对：加载时间，执行速度，垃圾回收，内存使用，平台 API 访问，调试，多线程以及可移植性。

我们构建网页程序的方式正面临着改革－这只是个开始而我们对于网络应用的思考方式正在发生改变。

## 首先，认识下 WebAssembly 吧

WebAssembly（又称 wasm） 是一种用于开发网络应用的高效，底层的字节码。

WASM 让你在其中使用除 JavaScript 的语言以外的语言（比如 C, C++, Rust 及其它）来编写应用程序，然后编译成（提早） WebAssembly。

构建出来的网络应用加载和运行速度都会非常快。

## 加载时间

为了加载 JavaScript，浏览器必须加载所有文本格式的 js 文件。

浏览器会更加快速地加载 WebAssembly，因为 WebAssembly 只会传输已经编译好的 wasm 文件。而且 wasm 是底层的类汇编语言，具有非常简洁的二进制格式。

## 执行速度

如今 Wasm 运行速度只比**原生代码**慢 20%。无论如何，这是一个令人惊喜的结果。它是这样的一种格式，会被编译进沙箱环境中且在大量的约束条件下运行以保证没有任何安全漏洞或者使之强化以对抗漏洞。和真正的原生代码比较，执行速度的下降微乎其微。另外，未来将会更加快速。

更让人高兴的是，它具备很好的浏览器兼容特性－所有主流浏览器引擎都支持 WebAssembly 且运行速度相关无几。

为了理解和 JavaScript 对比，WebAssembly 的执行速度有多快，你应该首先阅读之前的 [JavaScript 引擎工作原理](https://github.com/Troland/how-javascript-works/blob/master/v8.md)的文章。

让我们快速浏览下 V8 的运行机制：

![](https://user-images.githubusercontent.com/1475173/40056004-8a3ec756-587c-11e8-87e5-23f21b3e4805.png)

<center>V8 技术：懒编译</center>

左边是 JavaScript 源码，包含 JavaScript 函数。首先，源码先把字符串转换为记号以便于解析，之后生成一个[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)。

抽象语法树是 JavaScript 程序逻辑在内存中的图示。一旦生成图示，V8 直接进入到机器码阶段。你基本上是遍历树，生成机器码然后获得编译后的函数。这里没有做任何真正的尝试来加快速度。

现在，让我们看一下下一阶段 V8 管道的工作内容：

![](https://user-images.githubusercontent.com/1475173/40056005-8a7d6920-587c-11e8-8b0b-2665efdd722a.png)

<center>V8 管道设计</center>

现在，我们拥有 [TurboFan](https://github.com/v8/v8/wiki/TurboFan) ，它是 V8 的优化编译程序之一。当 JavaScript 运行的时候，大量的代码是在 V8 内部运行的。TurboFan  监视运行得慢的代码，引起性能瓶颈的地方及热点（内存使用过高的地方）以便优化它们。它把以上监视得到的代码推向后端即优化[即时编译器](https://en.wikipedia.org/wiki/Just-in-time_compilation)，编译器把消耗大量 CPU 资源的函数转换为性能更优的代码。

它解决了性能的问题，但是缺点即是分析代码及辨别哪些代码需要优化的过程也是会消耗 CPU 资源的。这也即意味着更多的耗电量，特别是在手机设备。

但是，wasm 并不需要以上的全部步骤－它如下所示插入到执行过程中：

![](https://user-images.githubusercontent.com/1475173/40056008-8b0416dc-587c-11e8-9d5f-9263ae8a8d53.png)

<center>V8 管道设计 + WASM</center>

wasm 在编译阶段就已经通过了代码优化。总之，解析也不需要了。你拥有优化后的二进制代码可以直接插入到后端（即时编译器）并生成机器码。编译器在前端已经完成了所有的代码优化工作。

由于跳过了编译过程中的不少步骤，这使得 wasm 的执行更加高效。

## 内存模型

![](https://user-images.githubusercontent.com/1475173/40056007-8ac584b2-587c-11e8-9006-49c58276279f.png)

<center>WebAssembly 可信和不可信状态</center>

举个栗子，一个 C++ 的程序的内存被编译为 WebAssembly，它是整段连续的没有空洞的内存块。wasam 中有一个可以用来提升代码安全性的功能即执行栈和线性内存隔离的概念。在 C++ 程序中，你有堆，从堆底部进行分配，然后从其顶部来增加执行栈的大小。你可以获得一个指针然后在堆栈内存中遍历以操作你不应该接触到的变量。

这是大多数可疑软件可以利用的漏洞。

WebAssembly 采用了完全不同的内存模型。执行栈和 WebAssembly 程序本身是隔离开来的，所以你无法从里面修改和诸如修改变量。同样地，函数使用整数位移而不是指针。函数指向一个间接函数表。之后，这些直接的计算出的数字进入模块中的函数。它就是这样构建的，这样你就可以同时引入多个 wasm 模块，偏移所有索引且每个模块都运行良好。

更多关于 JavaScript 内存模型和管理的文章详见[这里](https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec)。

## 内存垃圾回收 

你已经知晓 JavaScript 的内存管理是由内存垃圾回收器处理的。

WebAssembly 的情况有点不太一样。它支持手动操作内存的语言。你也可以在 wasm 模块中内置自己的内存垃圾回收器，但这是一项复杂的任务。

目前，WebAssembly 是围绕 C++ 和 RUST 的使用案例来设计的。由于 wasm 是非常底层的语言，这意味着只比汇编语言高一级的编程语言会容易被编译成 WebAssembly。C 语言可以使用 malloc，C++ 可以使用智能指针，Rust 使用完全不同的模式（一个完全不同的话题）。这些语言没有使用内存垃圾回收器，所以他们不需要所有复杂运行时的东西来追踪内存。WebAssembly 自然就很适合于这些语言。

另外，这些语言并不能够 100% 地应用于复杂的 JavaScript 使用场景比如更改 DOM 。用 C++ 来写整个的 HTML 程序是毫无意义的因为 C++ 并不是为此而设计的。大多数情况下，工程师用使用 C++ 或 Rust 来编写 WebGL 或者高度优化的库（比如大量的数学运算）。

然而，将来 WebAssembly 将会支持不带内存垃圾回功能的的语言。

## 平台接口访问

依赖于执行 JavaScript 的运行时环境，可以通过 JavaScript 程序来直接访问特定平台所暴露出的接口。比如，当你在浏览器中运行 JavaScript，网络应用可以调用一系列的[网页接口](https://developer.mozilla.org/en-US/docs/Web/API)来控制浏览器／设备的功能且访问 [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model)，[CSSOM](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model)，[WebGL](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API)，[IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)，[Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) 等等。

然而，WebAssembly 模块不能够访问任何平台的接口。所有的这一切都得由 JavaScript 来进行协调。如果你想在 WebAssembly 模块内访问一些特定平台的接口，你必须得通过 JavaScript 来进行调用。

举个栗子，如果你想要使用 `console.log`，你就得通过 JavaScript 而不是 C++ 代码来进行调用。而这些 JavaScript 调用会产生一定的性能损失。

情况不会一成不变的。规范将会为在未来为 wasm 提供访问特定平台的接口，这样你就可以不用在程序中书写 JavaScript。

## 源码映射

当压缩了 JavaScript 代码的时候，你需要有合适的方法来进行调试。

这时候[源码映射](https://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)就派上用场了。

大体上，源码映射就是把合并／压缩了的文件映射到未构建状态的一种方式。当为生产环境进行代码构建的时候，与压缩和合并 JavaScript 一起，会生成源码映射用来保存原始文件信息。当你想在生成的 JavaScript 代码中查询特定的行和列的代码的时候，可以在源码映射中进行查找以获得代码的原始位置。

由于没有规范定义源码映射，所以目前 WebAssembly 并不支持，但最终会有的（可能快了）。

当在 C++ 代码中设置了断点，就会看到 C++ 代码而不是 WebAssembly。至少，这是 WebAssembly 源码映射的目标吧。

## 多线程

JavaScript 是单线程的。有很多方法来利用事件循环和使用在之前的[文章](https://github.com/Troland/how-javascript-works/blob/master/event-loop.md)中有提到的异步编程。

JavaScript 也使用 Web Workers 但是只有在非常特殊的情况下－大体上，可以把任何可能阻塞 UI 主线程的密集的 CPU 计算移交给 Web Worker 执行以获得更好的性能。但是，Web Worker 不能够访问 DOM。

目前 WebAssembly 不支持多线程。但是，这有可能是接下来 WebAssembly 要实现的。Wasm 将会接近实现原生的线程（比如，C++ 风格的线程）。拥有真正的线程将会在浏览器中创造出很多新的机遇。那么当然，会增加滥用的可能性。

## 可移植性

现在 JavaScript 几乎可以运行于任意的地方，从浏览器到服务端甚至在嵌入式系统中。

WebAssembly 设计旨在安全性和可移植性。正如 JavaScript 那样。它将会在任何支持 wasm 的环境（比如每个浏览器）中运行。

WebAssembly 拥有和早年 Java 使用 Applets 来实现可移植性的同样的目标。

## WebAssembly 使用场景

WebAssembly 的最初版本主要是为了解决大量计算密集型的计算的（比如处理数学问题）。最为主流的使用场景即游戏－处理大量的像素。

你可以使用你熟悉的 OpenGL 绑定来编写 C++/Rust 程序，然后编译成 wasm。之后，它就可以在浏览器中运行。

浏览下（在火孤中运行）－<http://s3.amazonaws.com/mozilla-games/tmp/2017-02-21-SunTemple/SunTemple.html>。这是运行于[Unreal engine](https://www.unrealengine.com/en-US/what-is-unreal-engine-4)（这是一个可以用来开发虚拟现实的开发套件）中的。

另一个合理使用 WebAssembly （高性能）的情况即实现一些处理计算密集型的库。比如，一些图形操作。

正如之前所提到的，wasm 可以有效减少移动设备的电力损耗（依赖于引擎），这是由于大多数的处理步骤已经在编译阶段提前处理完成。

未来，你可以直接使用 WASM 二进制库即使你没有编写可编译成它的代码。你可以在 NPM 上面找到一些开始使用这项技术的项目。

针对操作 DOM 和频繁使用平台接口的情况 ，使用 JavaScript 会更加合理，因为它不会产生额外的性能开销且拥有原生支持的接口。

在 [SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=blog&utm_content=Post-6-webassembly-outro) 我们一直致力于持续提升 JavaScript 的性能以编写高质量和高效的代码。我们的解决方案必须拥有闪电般的性能因为我们不能够影响用户程序的性能。一旦把 SessionStack 整合进网络应用或网站的生产环境，它会开始记录所有的一切：所有的 DOM 变化，用户交互，JavaScript 异常，堆栈追踪，失败的网络请求和调试数据。所有的这一切都是在生产环境中产生且没有影响到产品的任何交互和性能。我们必须极大地优化我们的代码并且尽可能地让它异步执行。

我们不仅仅有库，还有其它功能！当你在 SessionStack 中重放用户会话，我们必须渲染问题产生时用户的浏览器所发生的一切，而且我们必须重构整个状态，允许你在会话时间线上来回跳转。为了使之成为可能，我们大量地使用异步操作，因为  JavaScript 中没有比这更好的替代选择了。

有了 WebAssembly，我们就可以把大量的数据计算和渲染的工作移交给更加合适的语言来进行处理而把数据收集和 DOM 操作交给 JavaScript 进行处理。

## 番外篇

打开 [webassembly](https://webassembly.org/) 官网就可以在头部醒目地看到显示它兼容的浏览器。分别是火孤，Chrome，Safari，IE Edge。点开 learn more 可以查看到这是于 2017／2／28 达成一致推出浏览器预览版。现在各项工作开始进入实施阶段了，相信在未来的某个时刻就可以在生产环境使用它了。官网上面介绍了一个 JavaScript 的子集 [asm.js](http://asmjs.org/)。另外，这里有一个 WebAssembly 和 JavaScript 进行性能比对的[测试网站](https://takahirox.github.io/WebAssembly-benchmark/)。