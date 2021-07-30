# 使用 MutationObserver 监测 DOM 变化

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-tracking-changes-in-the-dom-using-mutationobserver-86adc7446401)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第十章。**

网络应用在客户端日益复杂，这是由很多因素的造成的，比如需要更加丰富的界面交互以提供更加复杂的应用功能，实时计算等等。

网络应用的日益复杂导致无法知晓其生命周期中指定时刻准确的交互界面状态。

如果你正在构建一些框架或者一个库，这会更加的困难，比如，你无法通过监测 DOM 来响应并执行一些特定的操作。

## 概述

[MutationObserver ](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) 是现代浏览器提供的用来检测 DOM 变化的网页接口。你可以使用这个接口来监听新增或者删除的节点，属性更改，或者文本节点的内容更改。

可以干点啥好呢？

你可以在以下几种情况信手拈来 MutationObserver 接口。比如：

* 通知用户当前所在的页面所发生的一些变化。
* 通过使用一些新的很棒的 JavaScript 框架来根据 DOM 的变化来动态加载 JavaScript 模块。
* 可能当你在开发一个所见即所得编辑器的时候，使用 MutationObserver 接口来收集任意时间点上的更改，从而轻松地实现撤消/重做功能。

![](https://user-images.githubusercontent.com/1475173/41054347-b4ad6ecc-69f0-11e8-9ecb-dfe18497b393.png)

这只是几个 MutationObserver 的使用场景。

## 如何使用 MutationObserver

在应用中集成 MutationObserver 是相当简单的。通过往构造函数 `MutationObserver` 中传入一个函数作为参数来初始化一个 MutationObserver 实例，该函数会在每次发生 DOM 发生变化的时候调用。`MutationObserver` 的函数的第一个参数即为单个批处理中的 所有的 DOM 变化集合。每个变化包含了变化的类型和所发生的更改。

```
var mutationObserver = new MutationObserver(function(mutations) {
  mutations.forEach(function(mutation) {
    console.log(mutation);
  });
});
```

创建的实例对象拥有三个方法：

* `observe`－开始进行监听 DOM 更改。接收两个参数－要观察的 DOM 节点以及一个配置对象。
* `disconnect`－停止监听变化。
* `takeRecords`－触发回调前返回最新的批量 DOM 更改。

以下为开始监听的代码片段：

```
// 开始监听页面根元素 HTML 变化。
mutationObserver.observe(document.documentElement, {
  attributes: true,
  characterData: true,
  childList: true,
  subtree: true,
  attributeOldValue: true,
  characterDataOldValue: true
});
```

现在，假设你写了一个简单的 `div` 元素：

```
<div id="sample-div" class="test"> Simple div </div>
```

可以使用 jQuery 来移除 div 的 `class` 属性：

```
$("#sample-div").removeAttr("class");
```

当调用 `mutationObserver.observe(…)` 就可以开始监听 DOM 变化。

当每次发生 DOM 变化的时候，会打印出各个 [MutationRecord](https://developer.mozilla.org/en-US/docs/Web/API/MutationRecord) 日志信息：

![](https://user-images.githubusercontent.com/1475173/41054356-b715da3c-69f0-11e8-9248-cf5171efa6f0.png)

这一更改是由移除 `class` 属性所引起的。

最后，如果想停止监听 DOM 变化可以使用如下方法：

```
// MutationObserver 停止监听 DOM 变化
mutationObserver.disconnect();
```

现在，`MutationObserver` 浏览器兼容情况很好：

![](https://user-images.githubusercontent.com/1475173/41054350-b5bedb8e-69f0-11e8-9a13-2a34c469aecf.png)

## 替代方法

然而，之前 `MutationObserver` 并没有被广泛使用。那么，当没有 `MutationObserver` 的时候，开发者是如何解决监听 DOM 变化的呢？

有几下几种可用的方法：

* **轮询**
* **MutationEvents**
* **CSS 动画**

## 轮询

最简单且粗糙的方法即使用轮询。使用浏览器内置的 setInterval 网页接口你可以创建一个定时任务来定时检查是否 DOM 发生变化。当然了，这个方法会显著地减弱网络应用/网站的性能。

其实，这是可以理解为脏检查，如果有使用过 AngularJS 应该会有看过其脏检查所导致的性能问题。在我的另一个系列里面有稍微介绍了下，具体可以查看[这里](https://github.com/Troland/writing-a-javascript-framework/wiki/4.%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A%E7%AE%80%E4%BB%8B)。

## MutationEvents

早在 2000 年，就推出了 [MutationEvents API](https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Mutation_events) 。虽然挺管用的，但是每个单一的 DOM 变化都会触发 mutation 事件，结果又会造成性能问题。现在，`MutationEvents` 接口已经被废弃，不久的将来，现代浏览器将全都停止支持该接口。

以下是 `MutationEvents` 的浏览器兼容情况：

![](https://user-images.githubusercontent.com/1475173/41054352-b63b227a-69f0-11e8-8208-c009ab04b428.png)

## CSS 动画

依靠 [CSS 动画](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Animations/Using_CSS_animations) 是一个有点令人感到新奇的替代方案。这听起来会让人有些困惑。大体上，实现思路是这样的，创建一个动画，一旦在 DOM 中添加一个元素就会触发该动画。开始执行 CSS 动画的时候就会触发 `animationstart` 事件：假设为该事件添加事件监听器，就可以准确知晓 DOM 中添加元素的时机。动画的运行时间周期必须非常的短几乎让用户感知不到，即体验更佳。

首先，需要一个父级元素，在里面监听节点添加事件：

```
<div id=”container-element”></div>
```

为了处理节点的添加，需要创建关键帧序列动画，该序动画在添加节点的时候启动：

```
@keyframes nodeInserted { 
 from { opacity: 0.99; }
 to { opacity: 1; } 
}
```

创建好关键帧之后，在需要监听的元素上应用动画。注意到那个短暂的持续时间-在浏览器端动画痕迹会非常平滑（即用户会感觉不到有动画发生）：

```
#container-element * {
 animation-duration: 0.001s;
 animation-name: nodeInserted;
}
```

这样会为 `container-element` 的所有后代节点添加动画。当动画结束，触发 insertion 事件。

我们需要创建一个函数作为事件监听器。在函数内部，开始必须使用 `event.animationName` 代码进行检查，确保是我们所监听的动画。

```
var insertionListener = function(event) {
  // 确保是所监听的动画
  if (event.animationName === "nodeInserted") {
    console.log("Node has been inserted: " + event.target);
  }
}
```

为父元素绑定事件监听器：

```
document.addEventListener(“animationstart”, insertionListener, false); // standard + firefox
document.addEventListener(“MSAnimationStart”, insertionListener, false); // IE
document.addEventListener(“webkitAnimationStart”, insertionListener, false); // Chrome + Safari
```

**这里采用了事件委托。**

CSS 动画浏览器支持情况：

![](https://user-images.githubusercontent.com/1475173/41054354-b6bbc696-69f0-11e8-8782-051c474bcf54.png)

相比以上几种替代方案 `MutationObserver` 有几点优势。本质上，它会监听 DOM 可能发生的每个变化并且性能更优，因其会在批量 DOM 变化之后才触发回调事件。总之，`MutationObserver` 的兼容性很好，并且还有一些垫片，这些垫片底层使用 `MutationEvents` 。

