# 渲染引擎及性能优化小技巧

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-the-rendering-engine-and-tips-to-optimize-its-performance-7b95553baeda)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第十一章。**

迄今为止，之前的 JavaScript 工作原理系列文章集中于关注 JavaScript 语言本身的功能，在浏览器中的执行过程，如何优化等等。

然而，当在构建网络应用的时候，不仅仅只是编写能够独自运行的 JavaScript 代码。所编写的 JavaScript 代码与运行环境息息相关。理解 JavaScript 运行环境，它的运行原理及其组成会让你构建出更好的应用并且一旦让应用程序运行于各种环境下的时候，让你更加胸有成竹地应对潜在的问题。

![](https://user-images.githubusercontent.com/1475173/41496898-b37e0f3c-717c-11e8-9905-ae84e2d2839a.png)

那么，让我们一探浏览器主要组件吧：

* **用户界面:** 包括地址栏，后退和前进按钮，书签菜单等等。本质上，这里包含了除用户看到的网页窗口外的浏览器所显示的每个部分。
* **浏览器引擎:** 处理用户界面和渲染引擎的交互
* **渲染引擎:** 负责显示网页。渲染引擎解析 HTML 和 CSS 并在屏幕上显示解析的内容。
* **网络:** 使用各个平台的不同实现的诸如 XHR 请求的网络调用，这些网络调用是基于跨平台的接口实现的。
* **UI 后端:** 负责绘制诸如复选框和窗口的核心部件。它暴露出一个平台无关的泛型接口。底层使用操作系统的 UI 方法。
* **JavaScript 引擎**: 我们在之前的系列[文章](https://github.com/Troland/how-javascript-works/blob/master/v8.md)中有详细介绍过。基本上，这是 JavaScript 代码执行的地方。
* **数据存储:** 网络应用可能需要本地存储所有数据。支持的存储机制类型包括 [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), [indexDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API), [WebSQL](https://en.wikipedia.org/wiki/Web_SQL_Database) 以及 [FileSystem](https://developer.mozilla.org/en-US/docs/Web/API/FileSystem)。

本文将专注介绍渲染引擎，因为它是用来处理 HTML 和 CSS 的解析和可视化的，而这些是大多数的 JavaScript 应用需要持续进行交互的方面。

## 渲染引擎概述

渲染引擎的主要职责即在浏览器屏幕上显示请求的页面。

渲染引擎可以显示 HTML，XML 文档以及图片。如果使用额外的插件，就可以显示诸如 PDF 的不同类型的文档。

## 渲染引擎

与 JavaScript 引擎类似，不同浏览器也使用不同的渲染引擎。以下为比较流行的引擎：

* **Gecko**－Firefox
* **WebKit**－Safari
* **Blink**－Chrome, Opera(从版本 15 开始)

## 渲染过程

渲染引擎从网络层获取到请求的文档内容。

![](https://user-images.githubusercontent.com/1475173/41500213-28cb163e-71c0-11e8-9c98-0b28d7760a44.png)

## 构建 DOM 树

渲染引擎的第一步即解析 HTML 文档和转化解析的元素为 DOM 树 上的实际 [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction) 节点。

假设有如下的文本输入框：

```
<html>
  <head>
    <meta charset="UTF-8">
    <link rel="stylesheet" type="text/css" href="theme.css">
  </head>
  <body>
    <p> Hello, <span> friend! </span> </p>
    <div> 
      <img src="smiley.gif" alt="Smiley face" height="42" width="42">
    </div>
  </body>
</html>
```

HTML 的 DOM 树类似这样：

![](https://user-images.githubusercontent.com/1475173/41500209-27148924-71c0-11e8-99a3-b413fc987442.png)

基本上，每个元素包含有若干元素。然后依次类推。

## 构建 CSSOM 树

CSSOM 即 **CSS Object Model**。当浏览器构建页面的 DOM 树的时候，它在 `head` 标签部分遇到一个引用外部 `theme.css` 样式表的 link 标签。希望它可能需要样式表来渲染页面，于是便马上分派一个请求来获取样式表。假设以下为 `theme.css` 文件内容：

```
body { 
  font-size: 16px;
}

p { 
  font-weight: bold; 
}

span { 
  color: red; 
}

p span { 
  display: none; 
}

img { 
  float: right; 
}
```

与 HTML 一样，渲染引擎需要把 CSS 转化为浏览器可以操作的东西－即 CSSOM。以下为 CSSOM 的大概模样：

![](https://user-images.githubusercontent.com/1475173/41500210-2774ec9c-71c0-11e8-90ed-1d7bec6d30dc.png)

想知道为什么 CSSOM 是树状结构的吗？当为页面上的任意对象计算其最终的样式集的时候，浏览器先把最为通用的样式规则应用于该节点（比如，它是 body 的子节点，会先应用 body 的所有样式）然后通过应用更为具体的样式规则来递归重定义计算的样式。

让我们看下具体的例子吧。`body` 中的 `span` 标签中的任何文字样式为字体大小 16 像素且字体颜色为红色。这些样式继承自 `body` 元素。`p` 元素的子元素 `span` 由于应用了更为具体的样式从而不会显示其内容(`display:none`)。

还有，请注意以上 CSSOM 树并不完整而且只显示了样式表中指定的重写样式。每个浏览器提供了一份默认的样式集即 『用户代理样式』－ 这即当没有显式提供任何样式的时候的默认显示样式。我们的样式只是简单地重写了这些默认样式。

## 构建渲染树

HTML 中的可视化指令和 CSSOM 树的样式数据结合起来创建渲染树。

你可能为问渲染树是什么？它是按顺序构建可视化元素并显示在屏幕上的树。它带有相应的样式的 HTML 的视觉表现。该树旨在按正确的顺序绘制内容。

在 Webkit 中渲染树中的每个节点即是一个渲染器或者渲染器对象。

以下为以上的 DOM 和 CSSOM 树合成的渲染器树的大概模样：

![](https://user-images.githubusercontent.com/1475173/41500212-286aacd6-71c0-11e8-808c-aa2dd859bb51.png)

为了创建渲染树，浏览器大概做了几下几件事：

* 从 DOM 树的根节点开始，遍历每个可见节点。一些节点是不可见的（比如，script 标签，meta 标签等等），然后会被忽略，因为它们并不会显示在渲染结果中。一些节点通过样式隐藏然后也会被忽略。比如以上例子中的 span 节点，因为为其显式设置了 `display: none` 的样式。
* 浏览器为每个可见节点应用相对应的 CSSOM 规则并应用这些样式规则。
* 释放出包含有内容及其经过计算的样式的可见节点。

可以浏览下 RenderObject 的源码(Webkit 中)：<https://github.com/WebKit/webkit/blob/fde57e46b1f8d7dde4b2006aaf7ebe5a09a6984b/Source/WebCore/rendering/RenderObject.h>

看一下这个类的一些核心构件吧：

```
class RenderObject : public CachedImageClient {
  // 重绘整个对象。当边框颜色改变或者边框样式更改的时候调用。
  
  Node* node() const { ... }
  
  RenderStyle* style;  // 计算的样式
  const RenderStyle& style() const;
  
  ...
}
```

每个渲染器对象代表一个矩形区域通常是和一个节点的 CSS 盒模型相对应。它包括诸如宽度，高度以及定位的几何信息。

## 渲染树布局

当创建了渲染器并添加到渲染树的时候，它并没有定位和大小的信息。计算这些值即称为布局。

HTML 使用了流式布局模型，意即大多数情况下可以一次性计算出渲染器的几何位置信息。坐标系统是相对于根渲染器的。这里使用 Top 和 left 坐标。

布局是一个递归的过程－它从根渲染器开始进行渲染，根渲染器即 HTML 文档的 `html` 元素。布局继续通过一部分或者整个渲染器层级结构递归进行，为每个需要计算几何信息的渲染器提供其信息。

根渲染器的定位为 `0,0` 和大小即为浏览器窗口的可视化部分的大小(比如 viewport)。

进行布局的过程即计算出每个节点在屏幕上显示的准确位置。

## 绘制渲染树

该阶段，遍历渲染器树然后调用渲染器的 `paint()` 方法来在屏幕上显示其内容。

绘制可以是全局或增量式的(类似于布局)：

* **全局**－重绘整个树。
* **增量**－只更改部分渲染器在某种程度上不会影响到整颗树。渲染器作废其在屏幕上的矩形区域。这会导致操作系统把它看成是一个需要重绘的区域并生成一个 `paint` 事件。操作系统会智能地把几个区域合并成一个以提升渲染性能。

总之，理解绘制是个渐进式的过程是很重要的。为了更好的交互体验，渲染引擎会试图尽快在屏幕上显示内容。它不会等待所有的 HTML 解析完成才开始构建和布局渲染树。会先解析和显示部分内容，与此同时持续处理从网络接收的剩下的内容项。

## 脚本和样式的处理顺序

当解析器遇到 `<script>` 标签的时候会立即解析和执行该标签里面的代码。整个文档的解析会停止直到脚本执行完毕。意即该过程是同步的。

*当 script 引用的是一个外部资源，必须首先获取该资源(也是同步的)。所有的解析会停止直到获取该脚本资源。*

HTML5 添加了一个选项来异步加载该资源，这样就可以使用另外的线程来解析和执行该资源。IE 可以使用 `defer `属性，其它可以使用 async 属性。IE10 以下使用 defer 属性，IE10 以上也可以使用 async 属性。

这里有一个需要注意的地方即 IE10 以下对于 defer 的支持，打开 https://caniuse.com 查找即可发现对于 IE10 以下的支持是一些需要注意的地方即 defer 的脚本有可能会在 DOMContentLoaded 事件之后才开始运行，参见[这里](https://bugzilla.mozilla.org/show_bug.cgi?id=688580)，这里就不做试验了，有兴趣可以点击[这里](https://www.browserling.com/internet-explorer-testing)测试下 IE 下的表现。

这里稍微做一下引申，在 jQuery 源码中，ready.js 有一段如下的代码：

```
// Catch cases where $(document).ready() is called
// after the browser event has already occurred.
// Support: IE <=9 - 10 only
// Older IE sometimes signals "interactive" too soon
if ( document.readyState === "complete" ||
	( document.readyState !== "loading" && !document.documentElement.doScroll ) ) {

	// Handle it asynchronously to allow scripts the opportunity to delay ready
	window.setTimeout( jQuery.ready );

} else {

	// Use the handy event callback
	document.addEventListener( "DOMContentLoaded", completed );

	// A fallback to window.onload, that will always work
	window.addEventListener( "load", completed );
}
```

里面的 `window.setTimeout( jQuery.ready );` 是允许脚本有机会延迟执行 ready 事件。大概是为 IE script 标签的 defer 属性准备的吧？

## 优化渲染性能

若想要优化网络应用的性能，需要关注五个主要的方面。这些方面是你可以进行控制的：

1. **JavaScript**－之前的文章中有介绍了编写不阻塞 UI ，高效的代码等等。谈到渲染时候，需要考虑 JavaScript 代码是如何和页面上的 DOM 元素进行交互的。JavaScript 会在界面上做很多的更改，特别是在单页应用中。
2. **样式计算**－这个过程即应用样式规则到匹配选择器的元素上。一旦定义了样式规则，它们会应用于对应的元素，然后计算每个元素的最终样式。
3. **布局**－一旦浏览器了解元素应用的样式规则，它会开始计算元素所占用的空间和其在浏览器屏幕上的显示位置。根据网页的布局模型定义，元素的布局能够影响到其它的元素。比如，`<body>` 标签的宽度会影响到其子孙元素的宽度等等。这即意味着布局过程是相当耗时的。绘制是在多个图层完成的。
4. **绘制**－该阶段即开始填充实际像素。这一过程包括绘制文字，颜色，图片，边框，阴影等所有每个元素的可见部分。
5. **合成**－因为页面部分可能会被绘制成多层，它们必须在屏幕以正确的顺序进行绘制，这样页面渲染才会正常。这是至关重要的，特别是对于那些重叠元素。

## 优化 JavaScript 代码

JavaScript 经常会在浏览器端触发视觉改变。尤其是在构建 SPA 的过程中会更多。

这里有一些优化 JavaScript 中部分代码来提升渲染效率的建议：

* 避免使用 `setTimeout` 或者 `setInterval` 来进行视觉的更改。这些会在帧的某个时间点调用 `callback`，有可能是在帧的末尾。这样就会造成卡顿。必须在帧的开始触发视觉更改。

* 把耗时的 JavaScript 移入之前提到的[网页工作线程](https://github.com/Troland/how-javascript-works/blob/master/worker.md)。

* 使用微任务来处理跨多个帧的 DOM 更改。这是为了预防当任务需要访问 DOM 时，而网络工作线程无法办到的情况的。

  大概意即需要把一个大型的任务分割为多个小任务然后根据不同的任务性质在 `requestAnimationFrame` ，`setTimeout` 或 `setInterval` 中执行。

## 优化样式

通过添加和移除元素及更改属性等等修改 DOM 会导致浏览器重新计算元素样式及大多数情况整个页面或者部分页面的布局。

使用以下方法来优化渲染：

* 减少选择器的复杂度。选择器复杂度会占用超过计算所需元素样式的 50% 的时间，剩余时间即构建样式本身。
* 减少必须进行样式计算的元素的个数。本质上，直接更改少数元素的样式而不是使整个页面的样式失效。

## 优化布局

重新计算布局是很耗费浏览器性能的。考虑以下优化方案：

* 尽可能减少布局的数量。当更改样式的时候，浏览器检查样式更改是否需要重新计算布局。一般而言更改诸如 width, height, left, top 等和几何学相关的属性会需要布局。所以，尽可能避免修改它们。
* 尽可能使用 `flexbox` 来进行布局而不是老式的布局模型。它会渲染得更快并且会极大地提升网络应用的性能。
* 避免强制同步布局。需要记住的是当运行 JavaScript的时候，上一帧的老的布局值是已知的且可以被查询得到。当访问 `box.offsetHeight` 这并不会造成性能问题。然而，如果在访问它之前更改它的样式(比如为元素动态添加样式类)，浏览器将不得不首先应用样式更改然后计算布局样式。这将会非常耗时和耗资源，所以尽力避免这样做。

## 优化绘制

这经常会是所有任务中最耗时的，所以尽量避免触发绘制。优化方案：

* 更改除 transfroms 或者 opacity 外的属性会触发绘制。所以省着点用啊。
* 当触发一个布局也会触发绘制，因为更改元素的几何信息会更改元素的展示效果。
* 通过提升层和动画编排来减少绘制区域。

## 扩展

参考谷歌官方关于性能的[文档](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count#use_transform_and_opacity_changes_for_animations)，提升元素使用如下的代码：

```
.moving-element {
  will-change: transform;
}
```

使用 [FASTDOM](https://github.com/wilsonpage/fastdom) 来避免强制同步布局和抖动。

另外关于 JavaScript 代码的优化方面，避免去处理一些微优化，比如使用 `offsetTop` 比用 `getBoundingClientRect` 速度更快，但这得基于所创建的网络应用而言，假设创建一个游戏，对性能要求非常高而且调用这些方法的地方多，那么性能的提升将会很可观的。还记得以前经常会去使用诸如 [jsperf]( https://jsperf.com/) 来测试某个方法的速度，千万别钻牛角尖，因地制宜，避免掉入去计较那些微小的优化而付出过大的精力。

关于渲染可以使用一些骨架图来提升用户体验。

## 一些想法

* 关于性能的体验，其实你可以想象成造房子吧，假如是整个翻修当然是会更加耗时，但是如果装修某个区域就会提升性能。
* 然后有其中的某个属性会提升性能，这可能理解为『工欲善其事必先利其器』。
* 关于任务的切分可以理解为，建设设计的哲学，小技巧。

## 参考资源

* <https://developers.google.com/web/fundamentals/performance/critical-rendering-path/constructing-the-object-model>
* <https://developers.google.com/web/fundamentals/performance/rendering/reduce-the-scope-and-complexity-of-style-calculations>
* <https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/#The_parsing_algorithm>