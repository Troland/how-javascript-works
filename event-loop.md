# 事件循环及异步编程的出现和 5 种更好的 async/await 编程方式

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5)，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是  JavaScript 工作原理的第四章。**

现在，我们将会通过回顾单线程环境下编程的弊端及如何克服这些困难以创建令人惊叹的 JavaScript 交互界面来展开第一篇文章。老规矩，我们将会在本章末尾分享 5 条利用 async/await 编写更简洁代码的小技巧。

## 单线程的局限性

在第一篇[文章](https://github.com/Troland/how-javascript-works/blob/master/overview.md)开头，我们考虑了一个问题即当调用栈中含有需要长时间运行的函数调用的时候会发生什么。

譬如，试想下，在浏览器中运行着一个复杂的图片转换算法。

恰好此时调用栈中有函数需要执行，此时浏览器将会被阻塞，它不能够做其它任何事情。这意味着，浏览器会没有响应，不能够进行渲染和运行其它代码。这将会带来问题－程序界面将不再高效和令人愉悦。

程序没有响应。

在某些情况下，这或许没什么大不了的。但是，这可能会造成更加严重的问题。一旦浏览器在调用栈中同时运行太多的任务的时候，浏览器会很长时间停止响应。到了那个时候，大多数浏览器会抛出一个错误，询问你是否关闭网页。

这很丑陋且它完全摧毁了程序的用户体验。

![未响应](https://user-images.githubusercontent.com/1475173/40287991-b76a14b8-5ce3-11e8-9808-242e1c6501ba.jpeg)

## JavaScript 程序的构成模块

你可能会在单一的 .js 文件中书写 JavaScript 程序，但是程序是由多个代码块组成的，当前只有一个代码块在运行，其它代码块将在随后运行。最常见的块状单元是函数。

大多数 JavaScript 菜鸟有可能需要理解的问题即**之后运行**表示的是并不是必须严格且立即在现在之后执行。换句话说即，根据定义，**现在**不能够运行完毕的任务将会异步完成，这样你就不会下意识地预期或者希望会遇到以上提及的 UI 阻塞行为。

看下如下代码：

```
// ajax 为一个库提供的任意 ajax 函数
var response = ajax('https://example.com/api');

console.log(response);
// `response` 将不会有数据返回
```

可能你已经知道标准的 ajax 请求不会完全同步执行完毕，意即在代码运行阶段，ajax(..) 函数不会返回任何值给 response 变量。

获得异步函数返回值的一个简单方法是使用回调函数。

```
ajax('https://example.com/api', function(response) {
    console.log(response); // `response` 现在有值
});
```

只是要注意一点：实际上你可以发起同步 ajax 请求。不过，永远不要那么做。如果发起同步 ajax 请求，JavaScript 程序的 UI 将会被阻塞－用户不能够点击，输入数据，跳转或者滚动。这将会冻结任何用户交互体验。这是非常糟糕的实践。

以下示例代码，但请别这样做-不要毁掉网页：

```
// 假设你使用 jQuery
jQuery.ajax({
    url: 'https://api.example.com/endpoint',
    success: function(response) {
        // 成功回调.
    },
    async: false // 同步
});
```

我们以 Ajax 请求为例。你可以异步执行任意代码块。

你可以使用 `setTimeout(callback, milliseconds)` 函数来异步执行代码。`setTimeout` 函数会在之后的某个时刻触发事件(定时器)。如下代码：

```
function first() {
    console.log('first');
}
function second() {
    console.log('second');
}
function third() {
    console.log('third');
}
first();
setTimeout(second, 1000); // 1 秒后调用 second 函数
third();
```

控制台输出如下：

```
first
third
second
```

## 事件循环详解

我们将会以一个有些让人奇怪的声明开始－尽管允许异步 JavaScript 代码(比如之前讨论的 `setTimetout`)，但是直到 ES6，实际上 JavaScript 本身并没有集成任何直接的异步编程概念。JavaScript 引擎只允许在任意时刻执行单个的程序片段。

可以查看之前的[文章](https://github.com/Troland/how-javascript-works/blob/master/overview.md)来了解 JavaScript 引擎的工作原理。

那么， JS 引擎是如何执行程序片段的呢？实际上，JS 引擎并不是隔离运行的－它运行在一个宿主环境中，对大多数开发者来说，一般指的是 web 浏览器或者 Node.js。实际上，现在 JavaScript 广泛嵌入从机器到电灯泡的各种设备之中。每个设备代表了不同类型的 JS 引擎宿主环境。

所有宿主环境都含有一个被称为**事件循环**的内置机制，随着时间的推移，事件循环会执行程序中多个代码片段，每次都会调用 JS 引擎。

这意味着 JS 引擎只是任意 JS 代码的按需执行环境。这是一个周围环境，在其中进行事件的调度(运行JS 代码)。

<b>这里的周围环境根据 ecma 规范，即是说 LexicalEnvironment (词法环境)和VariableEnvironment（变量环境）。</b>

所以，打个比方，当 JavaScript 程序发起 Ajax 请求来从服务器获得数据，你在回调函数中书写 "response" 代码，JS 引擎会告诉宿主环境：

"嘿，我现在要挂起执行了，现在当你完成网络请求的时候且返回了数据，请执行回调函数。"

之后浏览器会监听从网络中返回的数据，当有数据返回的时候，它会通过把回调函数插入事件循环来调度执行。

让我们看下如下图示：

![内存图示](https://user-images.githubusercontent.com/1475173/40288048-fc615fc2-5ce3-11e8-9f1e-e96489238538.png)

你可以在之前的[文章](https://github.com/Troland/how-javascript-works/blob/master/overview.md)中阅读更多关于动态内存管理和调用栈的信息。

什么是网页 API ？本质上，你没有权限访问这些线程，你只能够调用它们。它们是浏览器的一部分，在其中引入了并发操作。如果你是个 Node.js 开发者，这些是 C++ APIs。

说了那么多，事件循环到底是啥？

![事件循环图示](https://user-images.githubusercontent.com/1475173/40288117-31e5b440-5ce4-11e8-98fa-d4c10c1723f7.png)

事件循环只有一项简单的工作－监测调用栈和回调队列。如果调用栈是空的，它会从回调队列中取得第一个事件然后入栈，并有效地执行该事件。

栈的操作是 FIFO 即先进先出，即是说当有两个 setTimeout 执行的时候，那么，第一个会先执行，再执行后一个。

譬如：

```
setTimeout(function () {

 console.log('first timeout')

}, 0)

setTimeout(function () {

 console.log('second timeout')

}, 0)

// output
'first timeout'
'second timeout'
```

事件循环中的这样一次遍历被称为一个 tick。每个事件就只是一个回调函数。

```
console.log('Hi');
setTimeout(function cb1() { 
    console.log('cb1');
}, 5000);
console.log('Bye');
```

让我们执行这段代码，然后看看会发生什么：

1.空状态。浏览器控制台是空的，调用栈也是空的。

![空状态图例](https://user-images.githubusercontent.com/1475173/40288154-62498904-5ce4-11e8-9396-1aa895d305b9.png)

2.`console.log('Hi')` 入栈。

![入栈图例](https://user-images.githubusercontent.com/1475173/40288182-88b8845a-5ce4-11e8-8a5c-98a5f4cd47a4.png)

3.执行 `console.log('Hi')`。

![](https://user-images.githubusercontent.com/1475173/40288225-b032e75a-5ce4-11e8-824c-97df3bc9c73d.png)

4.`console.log('Hi')` 出栈

![](https://user-images.githubusercontent.com/1475173/40288716-1d16588c-5ce7-11e8-82c9-193086d00bbc.png)

5. `setTimeout(function cb1() { ... })` 入栈。

![](https://user-images.githubusercontent.com/1475173/40288795-88ca548e-5ce7-11e8-9469-9b6e01dca718.png)

6.执行 `setTimeout(function cb1() { ... })`，浏览器创建定时器作为网页 API 的一部分并将会为你处理倒计时。

![](https://user-images.githubusercontent.com/1475173/40288812-a1b4dc3a-5ce7-11e8-9556-30dbb52a6aa0.png)

7.`setTimeout(function cb1() { ... })` 执行完毕并出栈。

  ![](https://user-images.githubusercontent.com/1475173/40288835-b9e7cf92-5ce7-11e8-8874-15c17676a82e.png)

8.`console.log('Bye')` 入栈。

  ![](https://user-images.githubusercontent.com/1475173/40288846-da4959fe-5ce7-11e8-8248-3cc7759a68e8.png)

9.执行 `console.log('Bye')`。

![](https://user-images.githubusercontent.com/1475173/40288872-fcfc690a-5ce7-11e8-83fb-79ed9f560531.png)

10.`console.log('Bye')` 出栈。

![](https://user-images.githubusercontent.com/1475173/40288907-2a75d696-5ce8-11e8-94bd-8550da83aaac.png)

11.至少 5 秒之后，定时器结束运行并把 `cb1` 回调添加到回调队列。为什么说至少五秒呢？

<b>因为 setTimeout 设定的 timer 并不一定会真的在 5 秒后执行，期间需要考虑是否有其它任务在执行，比方说有 microTask  在执行，如  promise 等，根据官方 event loop 文档即可知。</b>

   ![](https://user-images.githubusercontent.com/1475173/40288953-57f94c9c-5ce8-11e8-8adc-2a2ea2e1f5f1.png)

12.事件循环从回调队列中获得 `cb1` 函数并且将其入栈。

![](https://user-images.githubusercontent.com/1475173/40288902-289c42ba-5ce8-11e8-973b-2af55f8688e2.png)

13.运行 `cb1` 函数并将 `console.log('cb1')` 入栈。

![](https://user-images.githubusercontent.com/1475173/40288904-297d47ba-5ce8-11e8-86fd-0c3ef759f41c.png)

14.执行 `console.log('cb1')`。

![](https://user-images.githubusercontent.com/1475173/40288906-2a2b8bfe-5ce8-11e8-970a-2419ebf96971.png)

15.`console.log('cb1')` 出栈。

![](https://user-images.githubusercontent.com/1475173/40288901-2850356e-5ce8-11e8-906b-3102bb29496c.png)

16.`cb1` 出栈

![](https://user-images.githubusercontent.com/1475173/40289120-2a506694-5ce9-11e8-8de6-0518f8278059.png)

录像快速回放：

![](https://user-images.githubusercontent.com/1475173/40289266-e9afffc2-5ce9-11e8-9377-2acafe329f55.gif)

令人感兴趣的是，ES6 规定事件循环如何工作的，这意味着从技术上讲，它在 JS 引擎负责的范围之内，而不再只是扮演着宿主环境的角色。ES6 中 Promise 的出现是导致改变的主要原因之一，因为 ES6 要求有权直接细粒度地调度操作事件循环队列(之后会深入探讨)。

## setTimeout(…) 工作原理

需要注意的是 `setTimeout(…)` 并没有自动把回调添加到事件循环队列。它创建了一个定时器。当定时器过期，宿主环境会把回调函数添加到事件循环队列中，然后，将会在未来的某个 tick 取出并执行该回调。查看如下代码：

```
setTimeout(myCallback, 1000);
```

这并不意味着 1 秒之后会执行 `myCallback` 回调而是在 1 秒后将其添加到回调队列。然而，该队列有可能在之前就添加了其它的事件－所以回调就会被阻塞。

有相当一部分的文章和教程开始会建议你使用 `setTimeout(callback, 0)` 来书写 JavaScript 异步代码。那么，现在你明白了事件循环和 setTimeout 的原理：调用 `setTimeout` 把其第二个参数设置为 0 表示延迟执行回调直到调用栈被清空。

查看如下代码：

```
console.log('Hi');
setTimeout(function() {
    console.log('callback');
}, 0);
console.log('Bye');
```

虽然定时时间设定为 0 毫秒， 但是控制台中的结果将会如下显示：

```
Hi
Bye
callback
```

## ES6 作业概念

ES6 介绍了一个被称为『作业队列』的概念。它位于事件循环队列的顶部。你极有可能在处理 Promises(之后会介绍) 的异步行为的时候接触到这一概念。

现在我们将会接触这个概念，以便当讨论 Promises 的异步行为之后，理解如何调度和处理这些行为。

像这样想象一下：作业队列是附加于事件循环队列中每个 tick 末尾的队列。事件循环的一个 tick 中所发生的某些异步操作不会导致将全新的事件添加到事件循环队列中，但是反而会在当前 tick 的作业队列末尾添加一个作业项（即作业）。

这意味着，你可以添加延时运行其它功能且你可以确保它会在其它任何功能之前立刻执行。

一个作业也可以在同一队列末尾添加更多的作业。理论上讲，存在着作业循环的可能性(比如作业不停地添加其它作业)。

为了无限循环，就会饥饿程序所需要的资源直到下一个事件循环 tick。从概念上讲，这类似于在代码里面书写长时间运行或者无限循环(类似 `while(true)`)。

作业是有些类似于 `setTimeout(callback, 0)` 的小技巧，但是是以这样的方式实现的，它们拥有明确定义和有保证的执行顺序：之后而尽快地执行。

## 回调

正如你已知的那样，回调函数是 JavaScript 程序中用来表示和进行异步操作的最常见方法。的确，回调是 JavaScript 语言中最为重要的异步模式。无数的 JS 程序，甚至非常复杂的那些，都是建立在回调函数之上的。

回调并不是没有缺点。许多开发者试图找到更好的异步模式。然而，如果你不理解底层的原理而想要高效地使用任何抽象化的语法这是不可能的。

在接下来的章节中，我们将会深入探究这些抽象语法并理解更复杂的异步模式的必要性及是推荐的。

## 嵌套回调

查看以下示例：

```
listen('click', function (e){
	setTimeout(function(){
  	ajax('https://api.example.com/endpoint', function (text){
    	if (text == "hello") {
	    	doSomething();
	    } else if (text == "world") {
	    	doSomethingElse();
       }
      });
    }, 500);
});
```

我们有三个链式嵌套函数，每个函数代表一个异步操作。

这类代码通常被称为『回调地狱』。但是，实际上『回调地狱』和代码嵌套及缩进没有任何关系。这是一个更加深刻的问题。

首先，我们等待点击事件，然后，等待定时器执行，最后等待 Ajax 返回数据，在 Ajax 返回数据的时候，可能会重复执行这一过程。

乍一眼看上去，可以上把以上具有异步特性的代码拆分为按步骤执行的代码，如下所示：

```
listen('click', function (e) {
	// ..
});
```

之后：

```
setTimeout(function(){
    // ..
}, 500);
```

再后来：

```
ajax('https://api.example.com/endpoint', function (text){
    // ..
});
```

最后：

```
if (text == "hello") {
    doSomething();
}
else if (text == "world") {
    doSomethingElse();
}
```

因此，以这样顺序执行的方式来表示异步代码看起来一气呵成，应该有这样的方法吧？

## Promises

查看如下代码：

```
var x = 1;
var y = 2;
console.log(x + y);
```

这很直观：计算出 x 和 y 的值然后在控制台打印出来。但是，如果 x 或者 y 的初始值是不存在的且不确定的呢？假设，在表达式中使用 x 和 y 之前，我们需要从服务器得到 x 和 y 的值。想象下，我们拥有函数 `loadX` 和 `loadY` 分别从服务器获取 x 和 y  的值。然后，一旦获得 `x` 和 `y` 的值，就可以使用 `sum` 函数计算出和值。

类似如下这样：

```
function sum(getX, getY, callback) {
    var x, y;
    getX(function(result) {
        x = result;
        if (y !== undefined) {
            callback(x + y);
        }
    });
    getY(function(result) {
        y = result;
        if (x !== undefined) {
            callback(x + y);
        }
    });
}
// 同步或异步获取 `x` 值的函数
function fetchX() {
    // ..
}


// 同步或异步获取 `y` 值的函数
function fetchY() {
    // ..
}

sum(fetchX, fetchY, function(result) {
    console.log(result);
});
```

这里需要记住的一点是－在代码片段中，`x` 和 `y` 是未来值，我们用 `sum(..)`(从外部)来计算和值，但是并没有关注 `x` 和 `y` 是否马上同时有值。

当然喽，这个粗糙的基于回调的技术还有很多需要改进的地方。这只是理解考虑未来值而不用担心何时有返回值的好处的一小步。

## Promise 值

让我们简略地看一下如何用 Promises 来表示 `x+y` ：

```
function sum(xPromise, yPromise) {
	// `Promise.all([ .. ])` 包含一组 Promise,
	// 并返回一个新的 Promise 来等待所有 Promise 执行完毕
	return Promise.all([xPromise, yPromise])

	// 当新 Promise 解析完毕，就可以同时获得 `x` 和 `y` 的值并相加。
	.then(function(values){
		// `values` 是之前解析 promises 返回的消息数组
		return values[0] + values[1];
	} );
}

// `fetchX()` and `fetchY()` 返回 promise 来取得各自的返回值，这些值返回顺序不确定。
sum(fetchX(), fetchY())

// 获得一个计算两个数和值的 promise，现在，就可以链式调用 `then(...)` 来处理返回的 promise。
.then(function(sum){
    console.log(sum);
});
```

以上代码片段含有两层 Promise。

 `fetchX()` 和 `fetchY()` 都是直接调用，它们的返回值(promises!)都被传入 `sum(…)` 作为参数。虽然这些 promises 的 返回值也许会在现在或之后返回，但是无论如何每个 promise 都具有相同的异步行为。我们可以不关心返回时间顺序的方式来考虑 `x` 和 `y` 的值。暂时称他们为未来值。

第二层次的 promise 是由 `sum(…)` (通过 Promise.all([ ... ]))所创建和返回的，然后通过调用 `then(…)` 来等待 promise 的返回值。当 `sum(…)` 运行结束，返回 sum 未来值然后就可以打印出来。我们在 `sum(…)` 内部隐藏了等待未来值 `x` 和 `y` 的逻辑。

**注意：**在 `sum(…)` 内部，`Promise.all([ … ])`创建了一个 promise(在等待 `promiseX` 和 `promiseY` 解析之后)。链式调用 `.then(…)` 创建了另一个 promise，该 promise 会由代码 `values[0] + values[1]` 立刻进行解析(返回相加结果)。因此，在代码片段的末尾即 `sum(…)` 的末尾链式调用 `then(…)`－实际上是在操作第二个返回的 promise 而不是第一个由 `Promise.all([ ... ])` 创建返回的 promise。同样地，虽然我们没有在第二个`then(…)` 之后进行链式调用，但是它也创建了另一个 promise，我们可以选择观察／使用该 promise。我们将会在本章的随后内容中进行详细地探讨 promise 的链式调用相关。

在 Promises 中，实际上 `then(…)` 函数可以传入两个函数作为参数，第一个函数是成功函数，第二个是失败函数。

```
sum(fetchX(), fetchY())
.then(
    // 成功句柄
    function(sum) {
        console.log( sum );
    },
    // 拒绝句柄
    function(err) {
    	console.error( err ); // bummer!
    }
);
```

当获取 `x` 或者 `y` 出现错误或者计算和值的时候出现错误，`sum(…)` 返回的 promise 将会失败，传入 `then(…)` 作为第二个参数的回调错误处理程序将会接收 promise 的返回值。

因为 Promise 封装了时间依赖性的状态－等待外部的成功或者失败的返回值，Promise 本身是与时间无关的，这样就能够以可预测的方式组成(合并) Promise 而不用关心定时或者底层结果。

除此之外，一旦 Promise 解析完成，它就会一直保持不可变的状态且可以被随意观察。

链式调用 promise 真的很管用：

```
function delay(time) {
    return new Promise(function(resolve, reject){
        setTimeout(resolve, time);
    });
}

delay(1000)
.then(function(){
    console.log("after 1000ms");
    return delay(2000);
})
.then(function(){
    console.log("after another 2000ms");
})
.then(function(){
    console.log("step 4 (next Job)");
    return delay(5000);
})
// ...
```

调用 `delay(2000)` 创建一个将在 2 秒后返回成功的 promise，然后，从第一个 `then(…)` 的成功回调函数中返回该 promise，这会导致第二个 `then(…)` 返回的 promise 等待 2 秒后返回成功的 promise。

**Note：**因为一个 promise 一旦解析其状态就不可以从外部改变，由于它的状态不可以被随意修改，所以可以安全地把状态值随意分发给任意第三方。当涉及多方观察 Promise 的返回结果时候更是如此。一方影响另一方观察 Promise 返回结果的能力是不可能。不可变性听起来像是个晦涩的科学课题，但是，实际上这是 Promise 最根本和重要的方面，你得好好研究研究。

## Promise 使用时机

Promise 的一个重要细节即确定某些值是否是真正的 Promise。换句话说，这个值是否具有 Promise 的行为。

我们知道可以利用 `new Promise(…)` 语法来创建 Promise，然后，你会认为使用 `p instanceof Promise` 来检测某个对象是否是 Promise 类的实例。然而，并不全然如此。

主要的原因在于你可以从另一个浏览器窗口(比如 iframe)获得 Promise 实例，iframe 中的 Promise 不同于当前浏览器窗口或框架中的 Promise，因此，会导致识别 Promise 实例失败。

除此之外，库或框架或许会选择使用自身自带的 Promise 而不是原生的 ES6 实现的 Promise。实际工作中，你可以使用库自带的 Promise 来兼容不支持 Promise 的老版本浏览器。

## 异常捕获

如果在创建 Promise 或者是在观察解析 Promise 返回结果的任意时刻，遇到了诸如 `TypeError` 或者 `ReferenceError` 的  JavaScript 错误异常，这个异常会被捕获进而强制进行中的 Promise 为失败状态。

比如：

```
var p = new Promise(function(resolve, reject){
    foo.bar();	  // `foo` 未定义，产生错误!
    resolve(374); // 永不执行 :(
});

p.then(
    function fulfilled(){
        // 永不执行 :(
    },
    function rejected(err){
        // `err` 会是一个 `TypeError` 异常对象
	     // 由于 `foo.bar()` 代码行.
    }
);
```

但是，如果 Promise 成功解析了而在成功解析的监听函数(`then(…)` 注册回调)中抛出 JS 运行错误会怎么样？仍然可以捕捉到该异常，但或许你会发现处理这些异常的方式有些让人奇怪。直到深入理解其中原理：

```
var p = new Promise( function(resolve,reject){
	resolve(374);
});

p.then(function fulfilled(message){
    foo.bar();
    console.log(message);   // 永不执行
},
    function rejected(err){
        // 永不执行
    }
);
```

看起来 `foo.bar()` 抛出的错误异常真的被捕获到了。然而，事实上并没有。然而，深入理解你会发现我们没有监测到其中一些错误。`p.then(…)` 调用本身返回另一个 promise，该 promise 会返回 `TypeError` 类型的异常失败信息。

**拓展一下以上的说明，这是原文没有的。**

```
var p = new Promise( function(resolve,reject){
	resolve(374);
});

p.then(function fulfilled(message){
    foo.bar();
    console.log(message);   // 永不执行
},
    function rejected(err){
        // 永不执行
    }
).then(
function() {},
function(err) { console.log('err', err);}
);
```

如上代码所示就可以真正捕获到 promise 成功解析回调函数里面的代码错误。

## 处理未捕获的异常

有其它许多据说更好的处理异常的技巧。

普遍的做法是为 Promises 添加 `done(..)` 回调，本质上这会标记 promise 链的状态为 "done."。`done(…)` 并不会创建和返回 promise，因此，传入 `done(..)` 的回调显然并不会抛出错误到一个不存在的链式 Promise。

和未捕获的错误状况一样：任何在 `done(..)` 失败处理函数中的异常都将会被抛出为全局未捕获错误(基本上是在开发者控制台)。

```
var p = Promise.resolve(374);

p.then(function fulfilled(msg){
    // 数字没有字符类的函数，所以会报错
    console.log(msg.toLowerCase());
})
.done(null, function() {
    // 若发生错误，将会抛出全局错误
});
```

## ES8 中的 Async/await

JavaScript ES8 中介绍了 `async/await`，这使得处理 Promises 更加地容易。我们将会简要介绍 `async/await` 的所有可能姿势并利用其来书写异步代码。

那么，让我们瞧瞧 async/await 工作原理。

使用 `async` 函数声明定义一个异步函数。该函数会返回[异步函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncFunction)对象。`AsyncFunction` 对象表示在异步函数中运行其内部代码。

当调用异步函数的时候，它会返回一个 `Promise`。异步函数返回值并非 `Promise`，在函数过程中会自动创建一个 `Promise` 并使用函数的返回值来解析该 `Promise`。当 `async` 函数抛出异常，`Promise` 失败回调会获取抛出的异常值。

`async` 函数可以包含一个 `await` 表达式，这样就可以暂停函数的执行来等待传入的 Promise 的解析结果，之后重启异步函数的执行并返回解析值。

你可以把 JavaScript 中的 `Promise` 看作 Java 中的 `Future` 或 `C#` 中的 Task。

> `async/await` 本意是用来简化 promises 的使用。

看下如下代码：

```
// 标准 JavaScript 函数
function getNumber1() {
    return Promise.resolve('374');
}
// 和 getNumber1 一样
async function getNumber2() {
    return 374;
}
```

类似地，抛出异常的函数等价于返回失败的 promises。

```
function f1() {
    return Promise.reject('Some error');
}
async function f2() {
    throw 'Some error';
}
```

`await` 关键字只能在 `async` 函数中使用并且允许你同步等待 Promise。如果在 `async` 函数外使用 promises，我们仍然必须使用 `then` 回调。

```
async function loadData() {
    // `rp` 是个发起 promise 的函数。
    var promise1 = rp('https://api.example.com/endpoint1');
    var promise2 = rp('https://api.example.com/endpoint2');
   
    // 现在，并发请求两个 promise，现在我们必须等待它们结束运行。
    var response1 = await promise1;
    var response2 = await promise2;
    return response1 + ' ' + response2;
}
// 因为不再使用 `async function`，所以必须使用 `then`。
loadData().then(() => console.log('Done'));
```

你也可以使用异步函数表达式来定义异步函数。异步函数表达式拥有和异步函数语句相近的语法。异步函数表达式和异步函数语句的主要区别在于函数名，异步函数表达式可以忽略函数名来创建匿名函数。异步函数表达式可以被用作 IIFE(立即执行函数表达式)，可以在定义的后立即运行。

像这样：

```
var loadData = async function() {
    // `rp` 是个发起 promise 的函数。
    var promise1 = rp('https://api.example.com/endpoint1');
    var promise2 = rp('https://api.example.com/endpoint2');
   
    // 现在，并发请求两个 promise，现在我们必须等待它们结束运行。
    var response1 = await promise1;
    var response2 = await promise2;
    return response1 + ' ' + response2;
}
```

更为重要的是，所有的主流浏览器都支持 async/await。

![](https://user-images.githubusercontent.com/1475173/40289389-98f3ab96-5cea-11e8-9e81-a8a3c1eec6d3.png)

如果该兼容性不符合你的需求，你可以使用诸如 [Babel](https://babeljs.io/docs/plugins/transform-async-to-generator/) 和 [TypeScript](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-3.html) 的 JS 转译器来转换为自己需要的兼容程度。

最后要说的是，不要盲目地使用最新的技术来写异步代码。理解 JavaScript 中 async 的内部原理是非常重要的，学习为什么深入理解所选择的方法是很重要的。正如编程中的其它东西一样，每种技术都有其优缺点。

## 书写高可用，强壮的异步代码的 5 条小技巧

1.简洁：使用 async/await 可以让你写更少的代码。每次书写 async/await 代码，你都可以跳过书写一些不必要的步骤： 比如不用写 `.then` 回调，创建匿名函数来处理返回值，命名回调返回值。

```
// `rp` 是个发起 promise 的工具函数。
rp(‘https://api.example.com/endpoint1').then(function(data) {
 // …
});
```

对比：

```
// `rp` 是个发起 promise 的工具函数
var response = await rp(‘https://api.example.com/endpoint1');
```

2.错误处理：Async/await 允许使用日常的 try/catch 代码结构体来处理同步和异步错误。看下和 Promise 是如何写的：

```
function loadData() {
    try { // 捕获同步错误.
        getJSON().then(function(response) {
            var parsed = JSON.parse(response);
            console.log(parsed);
        }).catch(function(e) { // 捕获异步错误.
            console.log(e); 
        });
    } catch(e) {
        console.log(e);
    }
}
```

对比：

```
async function loadData() {
    try {
        var data = JSON.parse(await getJSON());
        console.log(data);
    } catch(e) {
        console.log(e);
    }
}
```

3.条件语句：使用 `async/await` 来书写条件语句会更加直观。

```
function loadData() {
  return getJSON()
    .then(function(response) {
      if (response.needsAnotherRequest) {
        return makeAnotherRequest(response)
          .then(function(anotherResponse) {
            console.log(anotherResponse)
            return anotherResponse
          })
      } else {
        console.log(response)
        return response
      }
    })
}
```

对比：

```
async function loadData() {
  var response = await getJSON();
  if (response.needsAnotherRequest) {
    var anotherResponse = await makeAnotherRequest(response);
    console.log(anotherResponse)
    return anotherResponse
  } else {
    console.log(response);
    return response;    
  }
}
```

4.栈桢：和 `async/await` 不同的是，从链式 promise 返回的错误堆栈中无法得知发生错误的地方。看如下代码：

```
function loadData() {
  return callAPromise()
    .then(callback1)
    .then(callback2)
    .then(callback3)
    .then(() => {
      throw new Error("boom");
    })
}
loadData()
  .catch(function(e) {
    console.log(err);

// Error: boom at callAPromise.then.then.then.then (index.js:8:13)
});
```

对比：

```
async function loadData() {
  await callAPromise1()
  await callAPromise2()
  await callAPromise3()
  await callAPromise4()
  await callAPromise5()
  throw new Error("boom");
}
loadData()
  .catch(function(e) {
    console.log(err);
    // output
    // Error: boom at loadData (index.js:7:9)
});
```

5.调试：如果使用 promise，你就会明白调试它们是一场噩梦。例如，如果你在 .then 代码块中设置一个断点并且使用诸如 "stop-over" 的调试快捷键，调试器不会移动到下一个 .then 代码块，因为调试器只会步进同步代码。

使用 `async/await` 你可以就像一般的同步函数那样逐句通过 await 调用。

不仅是程序本身还有库，书写异步 JavaScript 代码都是相当重要的。

参考资源：

* <https://github.com/getify/You-Dont-Know-JS/blob/master/async%20%26%20performance/ch2.md>
* <https://github.com/getify/You-Dont-Know-JS/blob/master/async%20%26%20performance/ch3.md>
* <http://nikgrozev.com/2017/10/01/async-await/>

