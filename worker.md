# Web Workers 分类及 5 个使用场景

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them-a547c0757f6a)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第七章。**

现在，我们将会剖析 Web Workers：我们将会综合比较不同类型的 workers，如何组合运用他们的构建模块来进行开发以及不同场景下各自的优缺点。最后，我们将会介绍 5 个适用 Web Workder 的场景。

在[前面的详细介绍的文章中](https://github.com/Troland/how-javascript-works/blob/master/overview.md)你已经清楚地了解到 JavaScript 是单线程的事实。然而，JavaScript 也允许开发者编写异步代码。

## 异步编程的局限性

前面我们了解到[异步编程](https://github.com/Troland/how-javascript-works/blob/master/event-loop.md)及其使用时机。

异步编程通过调度部分代码使之在事件循环中延迟执行，这样就允许优先渲染程序界面，从而让程序运行流畅。

AJAX 请求是一个很好的异步编程的使用场景 。因为请求可能会花很长的时间，所以可以异步执行它们，然后在客户端等待数据返回的同时，运行其它代码。

```
// 假设使用 jQuery
jQuery.ajax({
    url: 'https://api.example.com/endpoint',
    success: function(response) {
    		// 当数据返回时候的代码
    }
});
```

然而，这里会产生一个问题－AJAX 请求是由浏览器网页 API 进行处理的，可以异步执行其它代码吗？比如，假设成功回调的代码是 CPU 密集型的：

```
var result = performCPUIntensiveCalculation();
```

如果 `performCPUIntensiveCalculation` 不是一个 HTTP 请求而是一个会阻塞界面渲染的代码（比如大量的 `for` 循环），这样就没有办法释放事件循环和非阻塞浏览器 UI－浏览器会被冻结住且失去响应。

这意味着，异步函数只是是解决了一部分 JavaScript 的单线程限制。

在某些情况下，你可以通过使用 `setTimeout` 来很好地解决由于长时间计算所造成的 UI 阻塞。比如，通过把一个复杂的计算批量拆分为若干独立的`setTimeout` 调用 ，把它们放在事件循环的不同位置执行，然后这样就可以使得 UI 有时间进行渲染及响应。

让我们看一个计算数值数组平均值的简单函数。

```
function average(numbers) {
    var len = numbers.length,
        sum = 0,
        i;

    if (len === 0) {
        return 0;
    } 
    
    for (i = 0; i < len; i++) {
        sum += numbers[i];
    }
   
    return sum / len;
}
```

可以把以上代码重写为模拟异步：

```
function averageAsync(numbers, callback) {
    var len = numbers.length,
        sum = 0;

    if (len === 0) {
        return 0;
    } 

    function calculateSumAsync(i) {
        if (i < len) {
            // 在事件循环中调用下一个函数
            setTimeout(function() {
                sum += numbers[i];
                calculateSumAsync(i + 1);
            }, 0);
        } else {
             // 到达数组末尾，调用回调
            callback(sum / len);
        }
    }

    calculateSumAsync(0);
}
```

这里利用 `setTimeout` 函数在事件循环中循序添加每一次计算。在每一次计算之间，将会有充足的时间来进行其它的计算和解冻浏览器。

## Web Workders 来救场

[HTML5](https://www.w3schools.com/html/html5_intro.asp) 给我们带了很多开箱即用的好用的功能，包括：

* SSE（之前[文章](https://github.com/Troland/how-javascript-works/blob/master/http.md)中提到过并且和 WebSockets 进行了比较）
* Geolocation
* Application cache
* Local Storage
* Drag and Drop
* **Web Workers**

**Web Workers** 是浏览器内置的线程所以可以被用来执行非阻塞事件循环的 JavaScript 代码。

太棒了。整个 JavaScript 是基于单线程环境的而 Web Workers （部分）可以突破这方面的限制。

Web Workers 允许开发者把长时间运行和密集计算型的任务放在后台执行而不会阻塞 UI，这会使得应用程序运行得更加流畅。另外，这样就不用再使用 `setTimeout` 的黑科技来防止阻塞事件循环了。

这里有一个展示使用和未使用 Web Workers 来进行数组排序的区别的[示例](http://afshinm.github.io/50k/)。

## Web Workers 概览

Web Workers 允许你做诸如运行处理 CPU 计算密集型任务的耗时脚本而不会阻塞 UI 的事情。事实上，所有这些操作都是并行执行的。Web Workers 是真正的多线程。

你或许会有疑问－『难道 JavaScript 不是单线程的吗？』。

当你意识到 JavaScript 是一门没有定义线程模型的语言的时候，或许你会感觉非常的惊讶。Web Workers 并不是 JavaScript 的一部分，他们是可以通过 JavaScript 进行操作的浏览器功能之一。以前，大多数的浏览器是单线程的（当然，现在已经变了），而且大多数的 JavaScript 功能是在浏览器端实现完成的。Node.js 没有实现 Web Workers －它有 『cluster』和 『child_process』的概念，这两者和 Web Workers 有些许差异。

值得注意的是，规范中有三种类型的 Web Workers：

* [Dedicated Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
* [Shared Workers](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)
* [Service workers](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorker_API)

## Dedicated Workers

Dedicated Web Workers 是由主进程实例化并且只能与之进行通信

![](https://user-images.githubusercontent.com/1475173/40270200-7324ac2e-5bbb-11e8-80d6-654e0e0b5dfb.png)

<center>Dedicated Workers 浏览器支持情况</center>

## Shared Workers

Shared workers 可以被运行在同源的所有进程访问（不同的浏览的选项卡，内联框架及其它shared workers）。

![](https://user-images.githubusercontent.com/1475173/40270199-72b94baa-5bbb-11e8-9755-541047f8e93a.png)

<center>Shared Workers 浏览器支持情况</center>

## Service Workers

Service Worker 是一个由事件驱动的 worker，它由源和路径组成。它可以控制其关联的网页，拦截及修改导航，资源的请求，以及一种非常细粒度的方式来缓存资源以让你非常灵活地控制程序在某些情况下的行为（比如网络不可用）。

![](https://user-images.githubusercontent.com/1475173/40270197-7267d41e-5bbb-11e8-9bd3-978cab346084.png)

<center>Service Workers 浏览器支持情况</center>

本篇文章，我们将会专注于 Dedicated Workers 并以 『Web Workers』或者 『Workers』来称呼它。

## Web Workers 运行原理

Web Workers 是以加载 `.js` 文件的方式实现的，这些文件会在页面中异步加载。这些请求会被 [Web Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) 完全隐藏。

Workers 使用类线程的消息传输来实现并行通讯。它们非常适合于为用户提供最新的 UI ，高性能及流畅的体验。

Web Workers 运行于浏览器的一个隔离线程之中。因此，他们所执行的代码必须被包含在一个单独的文件之中。请谨记这一特性。

让我们看如何创建一个基础的 worker：

```
var worker = new Worker('task.js');
```

如果 『task.js』文件存在且可访问，浏览器会生成一个线程来异步下载文件。当下载完成的时候，文件会立即执行然后 worker 开始运行。万一文件不存在，worker 会运行失败且没有任何提示。

为了启动创建的 worker，你需要调用 `postMessage` 方法：

```
worker.postMessage();
```

## Web Worker 通信

为了在 Web Worker 和 创建它的页面间进行通信，你得使用 `postMessage` 方法或者一个[广播信道](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)。

## postMessage 方法

最新的浏览器支持方法的第一参数为一个 `JSON` 对象而旧的浏览器只支持字符串。

让我们通过往 worker 的方法传入`JSON` 对象参数作为复杂示例来理解页面所创建的 worker 是如何与之进行来回通信的。传入字符串与之类似。

让我们看下以下的 HTML 页面（或者更准确地说是 HTML 页面的一部分）

```
<button onclick="startComputation()">Start computation</button>

<script>
  function startComputation() {
    worker.postMessage({'cmd': 'average', 'data': [1, 2, 3, 4]});
  }
  var worker = new Worker('doWork.js');
  worker.addEventListener('message', function(e) {
    console.log(e.data);
  }, false);
  
</script>
```

worker 的脚本如下：

```
self.addEventListener('message', function(e) {
  var data = e.data;
  switch (data.cmd) {
    case 'average':
      var result = calculateAverage(data); // 某个数值数组中计算平均值的函数
      self.postMessage(result);
      break;
    default:
      self.postMessage('Unknown command');
  }
}, false);
```

当点击按钮，会在主页面调用 `postMessage`  方法。

`worker.postMessage` 行代码会把包含 `cmd` 和 `data` 属性及其各自属性值的 `JSON` 对象传入 worker。worker 通过定义监听 `message` 事件句柄来处理传过来的消息。

当接收到消息的时候，worker 会执行实际的计算而不会阻塞事件循环。worker 会检查传进来的 `e` 事件，然后像一个标准的 JavaScript 函数那样运行。当运行结束，传回主页面计算结果。

在 worker 的上下文中，`self` 和 `this` 都指向 worker 的全局作用域。

> 有两种方法来中断 woker 的执行：主页面中调用 `worker.terminate()` 或者在 workder 内部调用 `self.close()`

## 广播信道

[Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) 是更为普遍的通信接口。它允许我们向共享同一个源的所有上下文发送消息。同一个源下的所有的浏览器选项卡，内联框架或者 workers 都可以发送和接收消息：

```
// 连接到一个广播信道
var bc = new BroadcastChannel('test_channel');

// 发送简单信息示例
bc.postMessage('This is a test message.');

// 一个在控制台打印消息的简单事件处理程序示例
// logs the message to the console
bc.onmessage = function (e) { 
  console.log(e.data); 
}

// 关闭信道
bc.close()
```



视觉上看，你可以通过广播信道的图例以更加深刻的理解它。

![](https://user-images.githubusercontent.com/1475173/40270201-736e77d2-5bbb-11e8-9aa8-c8706a15f0e6.png)

<center>所有的浏览器上下文都是同源的</center>

然而，广播信道浏览器兼容性不太好：

![](https://user-images.githubusercontent.com/1475173/40270202-73c75668-5bbb-11e8-8793-6f48bf059294.png)

## 消息大小

有两种向 Web Workers 发送消息的方法：

* 复制消息：消息被序列化，复制，然后发送出去，接着在接收端反序列化。页面和 worker 没有共享一个相同的消息实例，所以在每次传递消息过程中最后的结果都是复制的。大多数浏览器是通过在任何一端自动进行 JSON 编码/解码消息值来实现这一功能。正如所预料的那样，这些对于数据的操作显著增加了消息传送的性能开销。消息越大，传送的时间越长。
* 消息传输：这意味着最初的消息发送者一发送即不再使用（<!--和导弹的发射后不管一样-->）。数据传输非常的快。唯一的限制即只能传输 [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) 数据对象。

## Web Workers 的可用功能

由于 Web Workers 的多线程特性，它只能使用一部分 JavaScript 功能。以下是可使用的功能列表：

* `navigator` 对象
* `location` 对象（只读）
* `XMLHttpRequest`
* `setTimeout()/clearTimeout()` 和 `setInterval()/clearInterval()`
* [Application Cache](https://www.html5rocks.com/tutorials/appcache/beginner/)
* 使用 `importScripts` 来引用外部脚本
* [创建其它 web workers](https://www.html5rocks.com/en/tutorials/workers/basics/#toc-enviornment-subworkers)

## Web Worker 的局限性

令人沮丧的是，Web Workers 不能够访问一些非常关键的 JavaScript  功能：

* DOM（非线程安全的）
* `window` 对象
* `document` 对象
* `parent` 对象

这意味着 Web Worker 不能够操作 DOM（因此不能更新 UI）。有时候，这会很棘手，不过一旦你学会如何合理地使 Web Workers，你就会把它当成单独的『计算器』来使用而用其它页面代码来操作 UI。Workers 将会为你完成繁重的计算任务，然后一旦任务完成，会把结果传到页面中并对界面进行必要的更新。

## 错误处理

和任何 JavaScript 代码一样，你会想要处理 Web Workers 中的任何错误。当在 worker 执行过程中有错误发生的时候，会触发 `ErrorEvent` 事件。这个接口包含三个有用的属性来指出错误的地方：

* **filename**－引起错误的 worker 脚本名称
* **lineno**－引起错误的代码行数
* **message**－错误描述

示例：

```
function onError(e) {
  console.log('Line: ' + e.lineno);
  console.log('In: ' + e.filename);
  console.log('Message: ' + e.message);
}

var worker = new Worker('workerWithError.js');
worker.addEventListener('error', onError, false);
worker.postMessage(); // 启动 worker 而不带任何消息
```

```
self.addEventListener('message', function(e) {
  postMessage(x * 2); // 意图错误. 'x' 未定义
};
```

这里，你可以看到我们创建了一个 worker 然后开始监听 `error` 事件。

在 worker 中（在 `workerWithError` 中），我们通过未在作用域中定义的 `x` 乘以 2 来创建一个有意的错误。异常会传播到初始化脚本（即主页面中）然后调用 onError 并传入关于错误的信息。

## Web Workers 最佳使用场景

迄今为止，我们列举了 Web Workers 的长处及其限制。让我们看看他们的最佳使用场景：

* **射线追踪**：射线追踪是一项通过追踪[光线](https://en.wikipedia.org/wiki/Light)的路径作为像素来生成图片的渲染技术。Ray tracing 使用 CPU 密集型数学计算来模仿光线的路径。思路即模仿一些诸如反射，折射，材料等的效果。所有的这些计算逻辑可以放在 Web Worker 中以避免阻塞 UI 线程。甚至更好的方法即－你可以轻易地把把图片的渲染拆分在几个 workers 中进行（即在各自的 CPU 中进行计算，意思是说利用多个 CPU 来进行计算，可以参考下 nodejs 的 api）。这里有一个使用 Web Workers 来进行射线追踪的简单示例－<https://nerget.com/rayjs-mt/rayjs.html>。

* 加密：端到端的加密由于对保护个人和敏感数据日益严格的法律规定而变得越来越流行。加密有时候会非常地耗时，特别是如果当你需要经常加密大量数据的时候（比如，发往服务器前加密数据）。这是一个使用 Web Worker 的绝佳场景，因为它并不需要访问 DOM 或者利用其它魔法－它只是纯粹使用算法进行计算而已。一旦在 worker 进行计算，它对于用户来说是无缝地且不会影响到用户体验。

* 预取数据：为了优化网站或者网络应用及提升数据加载时间，你可以使用 Workers 来提前加载部分数据以备不时之需。不像其它技术，Web Workers 在这种情况下是最棒哒，因为它不会影响程序的使用体验。

* 渐进式网络应用：即使在网络不稳定的情况下，它们必须快速加载。这意味着数据必须本地存储于浏览器中。这时候 [IndexDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) 及其它类似的 API 就派上用场了。大体上说，一个客户端存储是必须的。为了不阻塞 UI 线程的渲染，这项工作必须由 Web Workers 来执行。呃，当使用 IndexDB的时候，可以不使用 workers 而使用其异步接口，但是之前它也含有同步接口（可能会再次引入 ），这时候就必须在 workers 中使用。

  **这里需要注意的是在现代浏览器已经不支持同步接口了，具体可查看[这里](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API)。**

* 拼写检查：一个基本的拼写检测器是这样工作的－程序会读取一个包含拼写正确的单词列表的字典文件。字典会被解析成一个搜索树以加快实际的文本搜索。当检查器检查一个单词的时候，程序会在预构建搜索树中进行检索。如果在树中没有检索到，则会通过替换可选单词并测试单词是否可用-如果是用户所需的单词来用户提供替代的拼写。这个检索过程中的所有工作都可以交由 Web Worker 来完成，这样用户就只需输入单词和语句而不会阻塞 UI，与此同时 worker 会处理所有的搜索和供给建议。

在 [SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-web-workers-outro) 中对于我们来说性能和可靠性是至关重要的。之所以这么重要的原因是一旦把 SessionStack 整合进网络应用，它就会开始收集从 DOM 变化，用户交互到网络请求，未处理异常和调试信息的所有一切信息。所有的数据都是即时传输到我们的服务器的，这样就允许你以视频的方式重放网络应用中的所有问题以及观察用户端产生的一切问题。所有的一切都只会给程序带来极小的延迟且没有任何的性能开销。

这就是为什么我们使用 Web Workers 来处理监视库和播放器的逻辑的原因，因为 Web Workers 会帮我们处理诸如使用哈希来验证数据完整性，渲染等 CPU 密集型的任务。

在这个网络技术日新月异的时代，我们更加努力地保证 SessionStack 轻巧且不会给用户程序带来任何性能影响。

## 扩展

实际工作过程会遇到用户需要通过解析远程图片来获得图片 base64 的案例，那么这时候，如果图片非常大，就会造成 canvas 的 `toDataURL` 操作相当的耗时，从而阻塞页面的渲染。

所以解决思路即把这里的处理图片的操作交由 worker 来处理。以下贴出主要的代码：

```
<!DOCTYPE html>
<html lang="zh-cn">
<head>
  <meta charset="UTF-8">
  <title>Canvas to base64</title>
</head>
<body>
  <script>
    function loadImageAsync(url) {
      if (typeof url !== 'string') {
        return Promise.reject(new TypeError('must specify a string'));
      }

      return new Promise(function(resolve, reject) {
        const image = new Image();
        // 允许 canvas 跨域加载图片
        image.crossOrigin="anonymous";
        image.onload = function() {
          const $canvas = document.createElement('canvas');
          const ctx = $canvas.getContext('2d');
          const width = this.width;
          const height = this.height;
          let imageData;
          
          $canvas.width = width;
          $canvas.height = height;
          ctx.drawImage(image, 0, 0, width, height);
          imageData = ctx.getImageData(0, 0, $canvas.width, $canvas.height);
          resolve({image, imageData});
        };

        image.onerror = function() {
          reject(new Error('Could not load image at ' + url));
        };

        image.src = url;
      });
    }
    
    function blobToDataURL(blob) {
      return new Promise((fulfill, reject) => {
        let reader = new FileReader();
        reader.onerror = reject;
        reader.onload = (e) => fulfill(reader.result);
        reader.readAsDataURL(blob);
      })
    }

    document.addEventListener("DOMContentLoaded", function () {
      loadImageAsync('https://cdn-images-1.medium.com/max/1600/1*4lHHyfEhVB0LnQ3HlhSs8g.png')
        .then(function (image) {
          // jpeg-web-worker.js https://github.com/kentmw/jpeg-web-worker
          const worker = new Worker('jpeg-web-worker.js');
          worker.postMessage({
            image: image.imageData,
            quality: 50
          });
          worker.onmessage = function(e) {
            // e.data is the imageData of the jpeg. {data: U8IntArray, height: int, width: int}
            // you can still convert the jpeg imageData into a blog like this:
            const blob = new Blob( [e.data.data], {type: 'image/png'} );
            blobToDataURL(blob).then((imageURL) => {
              console.log('imageUrl:', imageURL);
            })
          }
        })
        .catch(function (err) {
          console.log('Error：', err.message);
        });
    });
  </script>
</body>
</html>
```

以上是通过 canvas 来获取图片数据，那么是否有其它方法呢？肯定有的啦，动下脑筋吧少年。