# 引擎，运行时，调用堆栈

> 原文请查阅[这里](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

**这是  JavaScript 工作原理 的第一章。本章会对语言引擎，运行时，调用栈做一个概述。**

随着 JavaScript 越来越流行，团队也利用其在他们诸如前端，后端，混合 apps，嵌入设备以及更多设备等开发栈中的不同层面的支持。

本章系列的第一章，本系列旨在深入 JavaScript  并理解它是如何运行的：我们认为在了解 JavaScript 的构建模块和它们是如何捏合在一起工作之后你将会写出更好的代码和 apps。我们将会分享一些当我们在创建 [SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-post1-intro) 时候的经验法则，[SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-post1-intro) 是一个轻量级的 JavaScript 程序它拥有强壮性和高性能的优点以保持竞争力。

正如 GitHut stats(http://githut.info/) 所显示的那样，JavaScript 的活跃库和总推送数在 Github 排名第一。其它方面的表现也不会比其它语言落下太多。

![](./assets/1_Zf4reZZJ9DCKsXf5CSXghg.png)

([点击查看最新 Github 语言统计](https://madnight.github.io/githut/))

如果工程非常依赖于 JavaScript，那么这意味着开发者不得不使用 JavaScript 和其语言生态提供的一切事物，为了能够创造出很酷的软件，就得更加深入地了解 JavaScript 语言的内部工作机制。

事实上，有很多开发者在每天日常开发中都会使用 JavaScript 但是却不了解其底层的知识。

## 概述

几乎所有人都已经听说过 V8 引擎的概念，并且很多人知道 JavaScript 是单线程的或者说是使用回调队列的。

在本章中，我将会详细地过一下这些概念并解释 JavaScript 实际上是如何工作的。有赖于了解这些细节，通过合理地使用提供的 APIs 你将可能写出更好的，非阻塞的程序。

如果你是新手，本文将会帮助你理解为什么和其它语言比较 JavaScript 是不可思议的。

如果你是一个经验丰富的 JavaScript 开发者，但愿，它将会给你日常使用的 JavaScript 运行时实际上是如何工作的提供一些崭新的深刻见解。

## JavaScript 引擎

谷歌 V8 引擎是流行的 JavaScript 引擎之一。V8 引擎在诸如 Chrome 和 Node.js 内部使用。这里有一个简单的视图来描绘其大概。

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_OnH_DlbNAPvB9KLxUCyMsA.png)

引擎包括两个主要组件：

* 动态内存管理 － 在这里分配内存
* 调用栈－这里你的代码执行即是你的堆栈结构

## 运行时

几乎每个 JavaScript 开发者都使用过一些浏览器 API(比如 setTimeout)。然而这些 API并不是引擎所提供的。

那么它们从何而来？

事实上这个情况有点复杂呃。。

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_4lHHyfEhVB0LnQ3HlhSs8g.png)

所以，我们除了引擎但是实际上还有更多其它方面的东西。我们拥有被称为 Web API 的东西，这些 Web API 是由浏览器提供的，比如 DOM,AJAX,setTimeout 以及其它。

于是乎，我们拥有了流行的事件循环和回调队列。

## 调用栈

JavaScript 只是一个单线程的编程语言，这意味着它只有一个调用栈。这样它只能一次做一件事情。

调用栈是一种数据结构，里面会记录我们在程序中的大概位置。当我们执行进入一个函数，我们把它置于栈的顶部。如果我们从函数中返回则从栈顶部移除函数。这就是调用栈所能够做的事情。

举个栗子。查看如下代码：

```
function multiply(x, y) {
  return x * y;
}

function printSquare(x) {
  var s = multiply(x, x);
  console.log(s);
}

printSquare(5);
```

当引擎开始执行这段代码的时候，调用栈会被清空。之后，产生如下步骤：

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_Yp1KOt_UJ47HChmS9y7KXw.png)

调用栈中的每个入口被称为堆栈结构。

当抛出异常的时候这正好是堆栈追踪是如何被构造出来的－当发生异常的时候这基本上是调用栈的状态。看下如下代码：

```
function foo() {
  throw new Error('SessionStack will help you resolve crashes:)');
}

function bar() {
  foo();
}

function start() {
  bar();
}

start();
```

如果在 Chrome 中执行（假设代码在 foo.js 的文件中），将会产生如下的堆栈追踪：

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_T-W_ihvl-9rG4dn18kP3Qw.png)

"堆栈溢出"－当你达到最大调用栈大小的时候发生。这种情况相当容易发生，特别是当你使用递归而没有仔细地检查你的代码的时候。查看下如下代码：

```
function foo() {
  foo();
}

foo();
```

当引擎开始执行这段代码的时候，它开始调用 foo 函数。这个函数，然而，会递归并开始调用其自身而没有任何结束条件。所以在每步执行过程中，调用堆栈会反复地添加同样的函数。执行过程如下所示：

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_AycFMDy9tlDmNoc5LXd9-g.png)

在某一时刻，然而，调用堆栈中的函数调用次数超过了调用堆栈的实际大小，这样浏览器决定抛出错误的动作，如下所示：

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_e0nEd59RPKz9coyY8FX-uw.png)

在单线程中运行代码会相当轻松因为你不用处理多线程环境中产生的一些复杂情况，比如死锁。

但是在单线程运行代码也会有相当的限制。由于 JavaScript 只有一个调用栈，如果运行很慢会发生什么？

## 并发和事件循环

当你在调用栈中有函数为了完成运行需要消耗大量的时间的时候会发生什么？例如，想象一下你想要在浏览器用 JavaScript 来执行一些复杂的图像转化。

你或许会问－为什么这也是个问题？问题是这样的当调用栈有函数需要执行，浏览器实际上不能做其它任何事－它被阻塞了。这意味着浏览器不能够执行渲染，它不能够运行其它代码，它卡住了。如果你想要在你的 app 中拥有酷炫的流畅 UI 体验，这将会是个问题。

这不会是仅有的问题。一旦你的浏览器开始在调用栈中执行如此多的任务，浏览器将会在相当一段时间内停止交互。大多数浏览器会抛出一个错误，询问你是否关闭网页。

![](/Users/Troland/repos/iwork/how-javascript-works/assets/1_WlMXK3rs_scqKTRV41au7g.jpeg)

现在，这并不是最好的用户体验，难道不是吗？

因此，我们如何不阻塞 UI 且不让浏览器停止响应来执行运行缓慢的代码呢？使用异步回调。

这将会在 『JavaScript 工作原理』 第二章：『在V8 引擎中如何写最佳代码的 5 条小技巧』中进行详细阐述。

