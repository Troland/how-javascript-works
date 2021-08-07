# CSS 和 JS 动画底层原理及如何优化其性能

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-under-the-hood-of-css-and-js-animations-how-to-optimize-their-performance-db0e79586216)，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是  JavaScript 工作原理的第十三章。**

## 概述

正如你所知，动画在创建令人叹服的网络应用中扮演着一个关键角色。由于用户越来越注重用户体验，企业开始意识到完美，令人愉悦的用户体验的重要性，结果网络应用变得越来越臃肿并且拥有更多动态交互的功能。这就要求网络应用于用户使用过程中提供更加复杂的动画来实现平滑的状态过渡。现在，这已经司空见惯。用户变得越来越挑剔，他们默认期许可以体验高度响应能力和交互良好的用户界面。

然而，让界面具有动画效果不一定必须的。哪些需要使用动画，动画的时机，方面及采用何种动画效果都是复杂的问题。

## JavaScript 和 CSS 动画比较

JavaScript 和 CSS 是创建网页动画的两条主要途径。两种不分好赖，看情况用吧。

## CSS 动画

使用 CSS 动画是让元素在屏幕上移动的最简单方法。

我们将会以如何让元素在 X 和 X 座标上移动元素 50 像素作为小示例开始。通过持续 1 秒的 CSS 过渡来移动元素。

```
.box {
  -webkit-transform: translate(0, 0);
  -webkit-transition: -webkit-transform 1000ms;

  transform: translate(0, 0);
  transition: transform 1000ms;
}

.box.move {
  -webkit-transform: translate(50px, 50px);
  transform: translate(50px, 50px);
}
```

当为元素添加 `move` 类的时候，改变 `transform` 的值然后开发发生过渡效果。

除了过渡持续时间，还有 **easing** 参数，它主要负责动画体验。该参数会在之后详细介绍。

如果通过以上的代码片段可以创建单独的样式类来操作动画，那么也可以使用 JavaScript 来切换每个动画。

如下元素：

```
<div class="box">
  Sample content.
</div>
```

然后，使用 JavaScript 来切换每个动画。

```
var boxElements = document.getElementsByClassName('box'),
    boxElementsLength = boxElements.length,
    i;

for (i = 0; i < boxElementsLength; i++) {
  boxElements[i].classList.add('move');
}
```

以上代码片段为每个包含 `box` 类的元素添加 `move` 类来触发动画。

这样做可以很好为你的网络应用提供很好的平衡。你就可以专注于使用 JavaScript 来操作应用状态，然后只需为目标元素设置合适的类，让浏览器来处理动画。如若你选择这么处理，就可以监听元素的 `transitionend` 事件，不过须放弃支持 IE 老版本浏览器。

![](https://user-images.githubusercontent.com/1475173/42127232-1d296300-7cc7-11e8-8842-e9a7b92eb183.png)

如下监听 `transitioned` 事件，该事件会在动画结束时触发。

```
var boxElement = document.querySelector('.box'); // 获取第一个包含 box 类的元素
boxElement.addEventListener('transitionend', onTransitionEnd, false);

function onTransitionEnd() {
  // Handle the transition finishing.
}
```

除了使用 CSS 过渡，还可以使用 CSS 动画，CSS 动画可以让你更好地控制单独的动画关键帧，持续时间以及循环次数。

> 关键帧是用来通知浏览器在规定的时间点上应有的 CSS 属性值，然后填充空白。

看一个例子：

```
/**
 * 该示例是没有包含浏览器前缀的精简版。加上以后会更加准确些。
 *
 */
.box {
  /* 选择动画名称 */
  animation-name: movingBox;

  /* 动画时长 */
  animation-duration: 2300ms;

  /* 动画循环次数 */
  animation-iteration-count: infinite;

  /* 每次奇数次循环时反转动画方向 */
  animation-direction: alternate;
}

@keyframes movingBox {
  0% {
    transform: translate(0, 0);
    opacity: 0.4;
  }

  25% {
    opacity: 0.9;
  }

  50% {
    transform: translate(150px, 200px);
    opacity: 0.2;
  }

  100% {
    transform: translate(40px, 30px);
    opacity: 0.8;
  }
}
```

效果示例－<https://sessionstack.github.io/blog/demos/keyframes/>

通过使用 CSS 动画定义独立于目标元素的动画本身，然后设置元素的 animation-name 属性来使用想要的动画效果。

CSS 动画仍然是需要加浏览器前缀的，在 Safari, Safari 移动浏览器和 Android 端添加 `-webkit-` 前缀。Chrome, Opera, Internet Explorer, and Firefox 端全部不需要添加前缀。有很多工具可以用来创建包含任意前缀的样式，这样就不需要在源文件中带样式前缀。

**可以使用 autoprefixer 或者 cssnext 来自动为样式添加前缀。**

## JavaScript 动画

和 CSS 过渡或者 CSS 动画相比，使用 JavaScript 来创建动画要更加复杂些，但是一般而言，它会为开发者提供引人注目且强大的功能。

一般情况下，可以内联 JavaScript 动画作为代码的一部分。也可以把它们封装在其它对象之中。以下为复现之前描述的 CSS 过渡的 JavaScript 代码：

```
var boxElement = document.querySelector('.box');
var animation = boxElement.animate([
  {transform: 'translate(0)'},
  {transform: 'translate(150px, 200px)'}
], 500);
animation.addEventListener('finish', function() {
  boxElement.style.transform = 'translate(150px, 200px)';
});
```

默认情况下，网页动画只是修改了元素的展示效果。如果想要让元素停留在指定的移动位置，那么就得在动画结束的时候修改其底层样式。这也是为什么在以上的示例中监听 `finish` 事件然后设置` box.style.transform` 属性为 `translate(150px, 200px)` 的原因，该属性值和 CSS 动画执行的第二个样式转换是一样的。

通过使用 JavaScript 动画，开发者可以完全控制每一步元素的样式。这意味着可以随心所欲地减速，暂停，停止或者翻转动画以控制目标元素。由于可以适当地封装动画行为，所以当在构建复杂的、面向对象的应用程序的时候会特别有用。

## Easing 定义

自然平滑地移动会让网络应用拥有更好的用户交互体验。

自然条件下，没有事物可以线性地从一个点运动到另一个点。现实生活中，在我们周围的物理世界中物体在移动的时候会加速或减速，因为我们并不生活在真空状态下且有不同的因素来影响事物的运行状态。人类的大脑会本能地感受到这样的移动，所以当为网络应用制作动画的时候，利用此类知识会有好处。

这是你所应该理解的术语：

* 『ease in』－开始移动缓慢而后加速
* 『ease out』－开始移动迅速而后减速

可以合并两个动画，比如 『ease in out』。

Easing 可以使得动画更加自然平滑。

### Easing 关键字

可以为 CSS 过渡和动画选择任意的 easing 方法。不同的关键字会影响动画的 easing。你也可以完全自定义 easing 方法。

以下为可以选择用来控制 easing 的 CSS 关键字：

* `linear`
* `ease-in`
* `ease-out`
* `ease-in-out`

让我们深入了解并查看他们的效果。

### Linear 动画

不使用任何的 easing 方法的动画即为 **linear**。

以下为 linear 过渡效果的图示：

![](https://user-images.githubusercontent.com/1475173/42127233-1d6c06a6-7cc7-11e8-8036-34c69bc6c478.png)

值随着时间流逝，值等比增加。使用 linear 动效，会让动画不自然。一般来说，避免使用 linear 动效。

使用如下代码实现一个简单的线性动画：

`transition: transform 500ms linear;`

### Ease-out 动画

正如前所述，和 linear 对比，easing out 让动画快速启动，结束时会减速。以下为图示：

![](https://user-images.githubusercontent.com/1475173/42127238-1ecfce6a-7cc7-11e8-9c99-9b9ccece2841.png)

总之，easing out 是最适合做界面体验的，因为快速地启动会给人以快速响应的动画的感觉，而结束时由于不一致的移动速度让人感觉平滑。

**打个比喻，比如那些跑车，首先启动速度相当的快，这就给人以愉悦的感觉。这个就比较符合人类对于动画的感知。**

有很多的方法来实现 ease out 动画效果，而最简单的即为 CSS 中的 `ease-out` 关键字。

`transition: transform 500ms ease-out;`

### Ease-in 动画

和 ease-out 动画相反－其启动慢然后结束时变快。图示如下：

![](https://user-images.githubusercontent.com/1475173/42127234-1dbac69c-7cc7-11e8-803d-db8d6b34109c.png)

和 ease-out 动画比较，由于他们启动缓慢给人以反应卡顿的感觉，所以 ease-in 让人感觉动画不自然。动画结束时很快给人一种奇怪的感觉，因为整个动画一直在加速，而现实世界中当事物突然停止运动的时候会减速而不是加速。

和 ease-out 和 linear 动画类似，使用 CSS 关键字来实现 ease-in 动画：

`transition: transform 500ms ease-in;`

### Ease-in-out 动画

该动画为 ease-in 和 ease-out 的合集。图示如下：

![](https://user-images.githubusercontent.com/1475173/42127239-1f1ff37c-7cc7-11e8-876e-84d06ccbc88a.png)

不要设置动画持续时间过长，否则会给人一种界面不响应的感觉。

使用 `ease-in-out` CSS 关键字来实现 ease-in-out 动画：

`transition: transform 500ms ease-in-out;`

### 自定义 easing

你可以自定义自己的 easing 曲线，这样就更有效地控制项目中的动画。

实际上， `ease-in`，`linear` 及 `ease` 关键字映射到预定义[贝塞尔曲线 ](https://en.wikipedia.org/wiki/B%C3%A9zier_curve)，可以在 [CSS transitions specification](http://www.w3.org/TR/css3-transitions/) 和 [Web Animations specification](https://w3c.github.io/web-animations/#scaling-using-a-cubic-bezier-curve) 中查找更多关于贝塞尔曲线的内容。

## 贝塞尔曲线

让我们看一下贝塞尔曲线的运行原理。一条贝塞尔曲线包含四个点，或者准确地说是包含两组数值。每一对数值内包含表示三次贝塞尔曲线控制点的 X 和 Y 坐标。贝塞尔曲线的起点坐标为 (0, 0) ，终点坐标为 (1, 1)。可以设置两组数值对。每个控制点的 X 轴值必须在 [0, 1] 之间，而 Y 轴值可以超过 [0, 1]，虽然规范并没有明确允许超过的数值。



即使每个控制点的 X 和 Y 值的微小差异都会输出完全不同的贝塞尔曲线。

查看维基百科关于[贝塞尔曲线](https://en.wikipedia.org/wiki/B%C3%A9zier_curve)的说明，通俗一点讲即，现在所说的即三次贝塞尔曲线，该曲线由四个点组成，P0, P1, P2, P3 组成，那么，P0 和 P1 组成一对，P2 和 P3 组成一对，P1 和 P2 即为控制点，P0 和 P3 即为起始和结束节点。如下图所示：

![](https://user-images.githubusercontent.com/1475173/42132929-7d8a85e4-7d53-11e8-85bb-20a96954809c.png)

看下两张拥有相近但不同坐标的控制结点的贝塞尔曲线图。

![](https://user-images.githubusercontent.com/1475173/42127235-1dff5262-7cc7-11e8-9625-40aa3326b9df.png)

和

![](https://user-images.githubusercontent.com/1475173/42127236-1e45015e-7cc7-11e8-81e9-9fa6728dea26.png)

如你所见，两张图有很大不同。第一个控制点向量差为 (0.045, 0.183)，而第二个控制点向量差为 (-0.427, -0.054)。

第二条曲线的样式为：

`transition: transform 500ms cubic-bezier(0.465, 0.183, 0.153, 0.946);`

第一组数值为起始控制点的 X 和 Y 坐标而第二组数值为第二个控制点的 X 和 Y 坐标。

## 性能优化

你得维持动画帧数为 60 帧每秒，否则会影响到用户体验。

和世界上其它事物一样，动画会有性能开销。一些属性的动画性能开销相比其它属性要小。比如，为元素的 `width` 和 `height` 做动画会更改其几何结构并且可能会造成页面上的其它元素移动或者大小的改变。这一过程被称为布局。之前的[文章](https://github.com/Troland/how-javascript-works/blob/master/rendering.md)中有详细介绍过布局和渲染。

总之，应该尽量避免为会引起布局和绘制的属性做动画。对于大多数现代浏览器而言，即把动画局限于 `opacity` 和 `transform` 属性。

## Will-change

可以使用 [will-change](https://dev.w3.org/csswg/css-will-change/) 来通知浏览器将会更改某个元素的属性。这会允许浏览器当更改某个元素属性的时候，事先进行最合理的优化。但不要滥用 `will-change`，因为这样做会适得其反，使得浏览器浪费更多的资源，从而造成更多的性能问题。

为 transforms 和 opacity 添加 `will-change` 代码如下：

```
.box {
  will-change: transform, opacity;
}
```

该属性在 Chrome, Firefox，Opera 得到很好的兼容。

![](https://user-images.githubusercontent.com/1475173/42127237-1e8970a0-7cc7-11e8-9f96-c4b441bdbef3.png)

## 如何选择 JavaScript 和 CSS 来执行动画

这个问题是无解的。只需谨记以下原则：

* 基于 CSS 的动画和原生支持的网页动画一般都是由被称为『合成线程』的线程来处理的。这不同于浏览器的主线程，主线程是用来执行计算样式，布局，绘制及 JavaScript 代码的。这即意味着如果浏览器在主线程上运行耗时的任务，不会中断动画的运行。
* 很多时候，也可以由合成线程来处理 `transforms` 和 `opacity` 属性值的更改。
* 如果有任何动画触发绘制，布局或者同时触发两者，『主线程』将不得不来进行处理。事实是基于 CSS 和 JavaScript 的动画和布局或者绘制的性能开销将很有可能会阻塞所有和 CSS 或者 JavaScript 运行相关的工作，致命动画问题变得毫无意义。

## 正确使用动画

良好的动画为项目添加一层令人愉快和互动的用户体验。你可以随意使用动画，不管是宽度，调试，定位，颜色或背景色，但必须注意潜在的性能瓶颈。糟糕的动画选择会影响用户体验，所以动画必须是高效且合理的。尽可能减少使用动画。只使用动画来让用户体验流畅自然而不是滥用。

## 使用动画进行交互

不要因为只是为了用而去使用动画。相反，有策略性地使用动画来加强用户交互体验。避免使用不必要的动画来打断或者阻碍用户的使用。

## 避免为性能开销大的属性做动画

比糟糕的动画使用更糟的是那些会引起页面卡顿的动画。这类动画让用户感到懊丧和不快。

## 引用资源

- <https://developers.google.com/web/fundamentals/design-and-ux/animations/css-vs-javascript>
- <https://developers.google.com/web/fundamentals/design-and-ux/animations/>
- <https://developers.google.com/web/fundamentals/design-and-ux/animations/animations-and-performance>

## 疑问

可以看下网上的这篇介绍[贝塞尔曲线](http://www.html-js.com/article/1628)的文章，那么可以如何使用贝塞尔曲线来做出令人惊叹的动画呢？