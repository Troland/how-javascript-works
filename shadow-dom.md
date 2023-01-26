# Shadow DOM 内部构造及如何构建独立组件

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-the-internals-of-shadow-dom-how-to-build-self-contained-components-244331c4de6e)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第十七章。**

![](https://user-images.githubusercontent.com/1475173/52962049-40d1fe80-33d7-11e9-84dd-089b1eb01565.png)

## 概述

网页组件指的是允许开发者使用一系列不同的技术来创建可复用的自定义元素，组件内的功能不影响其它代码，以便于开发者在网页程序中使用。

有四种网页组件标准：

* Shadow DOM
* HTML 模板
* 自定义元素
* HTML Imports

本章主要讨论 **Shadow DOM**。

Shadow DOM 是一个被设计用来构建基于组件(积木式)的网页程序的工具。它为开发者可能经常遇到过的问题提供了解决方案：

* **隔离的 DOM**：组件的 DOM 是独立的(比如 `document.querySelector()` 无法检索到组件 shadow DOM 下的f元素节点)。这样就可以简化网页程序中的 CSS 选择器，因为 DOM 组件是互不影响，这样就允许开发者可以随心所欲地使用更加通用的 id/class 命名而不用担心命名冲突。
* **局部样式**: shadow DOM 内定义的样式不会污染 shadow DOM 之外的元素。Style 样式规则不会泄漏且页面样式也不会污染 shadow DOM 内的元素样式。
* **组合**：为开发者的组件设计一个声明式，基于标签的接口。

## Shadow DOM

本篇文章假设开发者已经对 DOM 及其 API 熟拈于心。否则，可以阅读一下这方面的[详细资料](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction)。

与一般的 DOM 元素相比，Shadow DOM 有两处不同的地方：

* 与一般创建和使用 DOM 的方式相比，开发者如何创建及使用 Shadow DOM 并与页面上的其它元素相关联
* 其展现形式与页面上的其它元素的关系

一般情况下，开发者创建 DOM 节点，然后将其作为子元素挂载到其它元素下。对于 shadow DOM，开发者创建一个独立 DOM 树挂载到目标元素下，元素与其实际后代元素是分离的。该独立子树称为 **shadow 树**。shadow 树的挂载元素称为 **shadow 宿主**。包括 `<style>` 在内的所有在 shadow 树下创建的任何标签都只作用于宿主元素内部。此即 shadow DOM 如何实现 CSS 局部样式化的原理。

## 创建 Shadow DOM

一个 **shadow 根** 即是一段挂载到 "宿主" 元素下的文档碎片。挂载了 shadow 根即表示宿主元素包含 shadow DOM。调用 `element.attachShadow()` 方法来为元素创建 shadow DOM:

```
var header = document.createElement('header');
var shadowRoot = header.attachShadow({mode: 'open'});
var paragraphElement = document.createElement('p');

paragraphElement.innerText = 'Shadow DOM';
shadowRoot.appendChild(paragraphElement);
```

[规范](http://w3c.github.io/webcomponents/spec/shadow/#h-methods)定义了不能够创建 shadow 树的元素列表。

## Shadow DOM 组合功能

组合元素是 Shadow DOM 最重要的功能之一。

当书写 HTML 的时候，组合元素构建网页程序。开发者组合及嵌套诸如 `<div>`，`<header>`，`<form>` 及其它不同的构建模块来构建网页程序所需的界面。其中某些标签甚至可以互相兼容。

元素组合定义了诸如为何 `<select>`，`<form>`，`<video>` 及其它元素是灵活的且接受特定的 HTML 元素作为子元素以便用来对这些元素进行特殊处理。

比如，`<select>` 元素知道如何把 `<option>` 元素渲染成为带有预定义选项的下拉框组件。

Shadow DOM 引入如下功能，可以用来组合元素。

## Light DOM

此即组件的书写标记。该 DOM 存在于组件的 shadow DOM 之外。它是元素的实际子元素。假设开发者创建了一个名为 `<better-button>` 的自定义组件，扩展原生 button 标签及想在组件内部添加一个图片和一些文本。大概如下：

```
<extended-button>
  <!-- image 和 span 即为扩展 button 的 light DOM -->
  <img src="boot.png" slot="image">
  <span>Launch</span>
</extended-button>
```

「扩展 button」即开发者自定义组件，而其中的 HTML 即为 Light DOM 且是使用组件的用户所添加的。

这里的 Shadow DOM 即开发者创建的组件(「扩展 button」)。Shadow DOM 仅存在于组件内部且在其中定义其内部结构，局部样式及封装了组件实现细节。

## 扁平 DOM 树

浏览器分发 light DOM 的结果即，由用户在 Shadow DOM 内部创建的 HTML 内容，这些 HTML 内容构成了自定义组件的结构，渲染出最后的产品界面。扁平树即开发者在开发者工具中最终看到的内容和页面的最终渲染结果。

```
<extended-button>
  #shadow-root
  <style>…</style>
  <slot name="image">
    <img src="boot.png" slot="image">
  </slot>
  <span id="container">
    <slot>
      <span>Launch</span>
    </slot>
  </span>
</extended-button>
```

## 模板

当开发者不得不在网页上复用相同的标记结构的时候，最好使用某种模板而不是重复书写相同的页面结构。以前是可以实现的，但是现在可以使用 <template> (现代浏览器均兼容)元素轻易地实现该功能。该元素及其内容不会在 DOM 中渲染，但是可以使用 JavaScript 来引用其中的内容。

来看一个简单示例：

```
<template id="my-paragraph">
  <p> Paragraph content. </p>
</template>
```

上面的内容不会在页面中渲染，除非使用 JavaScript 来引用其中的内容，然后使用类似如下的代码来挂载到 DOM 中：

```
var template = document.getElementById('my-paragraph');
var templateContent = template.content;
document.body.appendChild(templateContent);
```

迄今为止，可以使用其它技术来实现类似的功能，但是正如之前所提到的，尽量使用原生功能来实现可能会更酷些。另外，兼容性也蛮好。

![](https://user-images.githubusercontent.com/1475173/52962058-44658580-33d7-11e9-8206-5b897a021540.png)

本身模板就很好用，但是若和自定义元素配合使用会更好哦。我们将会另外的文章中介绍自定义元素，当下开发者只需了解 customElement 接口允许开发者自定义标签内容的渲染。

让我们定义一个使用模板作为其 shadow DOM 渲染内容的网页组件。且称其为 `<my-paragraph>:`

```
customElements.define('my-paragraph',
 class extends HTMLElement {
   constructor() {
     super();

     let template = document.getElementById('my-paragraph');
     let templateContent = template.content;
     const shadowRoot = this.attachShadow({mode: 'open'}).appendChild(templateContent.cloneNode(true));
  }
});
```

这里需要注意的是使用 [Node.cloneNode()](https://developer.mozilla.org/en-US/docs/Web/API/Node/cloneNode) 方法来复制模板内容挂载到 shadow 根下。

另外，由于把模板的内容挂载到 shadow DOM 中，开发者可以在模板中使用 `<style>` 元素包含一些样式信息，该 `<style>` 元素随后会被封装进自定义元素里面。如果直接把模板挂载到标准 DOM 里面是不起作用的。

比如，可以更改模板内容为如下：

```
<template id="my-paragraph">
  <style>
    p {
      color: white;
      background-color: #666;
      padding: 5px;
    }
  </style>
  <p>Paragraph content. </p>
</template>
```

可以以如下方式使用刚才使用模板创建的自定义组件：

`<my-paragraph></my-paragraph>`

## 插槽

模板有一些不足的地方，主要的不足在于静态内容不允许开发者像一般的标准 HTML 模板那样渲染自定义的变量或者数据。

这时候 `<slot>` 就派上用场了。

可以把插槽看成是允许开发者在模板中放置自定义 HTML 的占位符的功能。这样开发者就可以创建泛型 HTML 模板并且通过引入插槽来自定义渲染内容。

让我们看一下以上模板添加一个插槽的代码如下：

```
<template id="my-paragraph">
  <p> 
    <slot name="my-text">Default text</slot> 
  </p>
</template>
```

如果在标记中引用该元素的时候没有定义插槽内容，或者浏览器不支持插槽，则 `<my-paragraph>` 只会包含默认的 "Default text" 内容。

若想要定义插槽内容，开发者得在 `<my-paragraph>` 中定义元素 HTML 结构的 [slot](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes#attr-slot) 属性值和对应填充的插槽名称保持一致即可。

如前所述，开发者可以随便写插槽内容：

```
<my-paragraph>
 <span slot="my-text">Let's have some different text!</span>
</my-paragraph>
```

所有可以被插入插槽的元素被称为[可插入元素](https://developer.mozilla.org/en-US/docs/Web/API/Slotable)；已插入插槽元素称为插槽元素。



注意以上示例中插入的 `<span>` 元素即是插槽元素。它拥有一个 `slot` 属性，属性值和模板中插槽定义的 name 属性值相等。



浏览器渲染之后，以上代码会创建如下扁平 DOM 树：

```
<my-paragraph>
  #shadow-root
  <p>
    <slot name="my-text">
      Default text
    </slot>
  </p>
  <span slot="my-text">Let's have some different text!</span>
</my-paragraph>
```

**这里原文有误，有改动。**

注意 `#shadow-root` 元素只是表示存在 Shadow DOM  而已。

## 样式化

可以在主页面样式化含有 shadow DOM 的组件，可以定义组件样式或者提供 [CSS 自定义属性的形式](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_variables)让用户覆盖掉默认样式值。

### 组件定义的样式

**局部样式** 是 Shadow DOM 极好的功能之一：

* 主页面上的 CSS 选择器不会影响到组件内部元素的样式。
* 组件内部定义的样式不会影响页面上的其它元素样式。它们只作用于宿主元素。

Shadow DOM 中的 CSS 选择器只影响组件内部的元素。实际上，这意味着开发者可以重复使用通用的 id/class 名称而不用担心和主页面上的其它样式发生冲突。简单的 CSS 选择器可以提高页面性能。

让我们看一下如下 #shadow-root 中定义的一些样式：

```
#shadow-root
<style>
  #container {
    background: white;
  }
  #container-items {
    display: inline-flex;
  }
</style>

<div id="container"></div>
<div id="container-items"></div>
```

以上示例中的样式只会作用于 #shadow-root 内部。

开发者也可以在 #shadow-root 里面使用 <link> 元素来引入样式表，也只作用于 #shadow-root 内部。

## :host 伪类

`:host` 伪类允许开发者选择和样式化包含 shadow 树的宿主元素：

```
<style>
  :host {
    display: block; /* 默认情况下, 自定义元素是内联元素 */
  }
</style>
```

只有一个地方需要注意即若主页面上定义的宿主元素样式优先级比元素里面定义的 :host 样式规则要高。这样就允许开发者从外部覆盖掉组件内部定义的顶级样式。

即当在主页面上定义了如下的样式：

```
my-paragraph {
  marbin-bottom: 40px;
}

<template id="my-paragraph">
	<style>
		:host {
      margin-bottom: 30px;/* 将不起作用，因为会被前面父页面已定义的样式覆盖 */
		}
	</style>
  <p> 
    <slot name="my-text">Default text</slot> 
  </p>
</template>
```

同理，`:host` 只在shadow 根的上下文中起作用，因此开发者不能够在 Shadow DOM 外面使用。

`:host(<selector>)` 这样的功能样式允许开发者只样式化匹配 `<selector>` 的宿主元素。这是一个绝佳的方式，开发者可以在组件内部封装响应用户交互或者状态的行为，然后基于宿主元素来样式化内部节点。

```
<style>
  :host {
    opacity: 0.4;
  }
  
  :host(:hover) {
    opacity: 1;
  }
  
  :host([disabled]) { /* 宿主元素拥有 disabled 属性的样式. */
    background: grey;
    pointer-events: none;
    opacity: 0.4;
  }
  
  :host(.pink) > #tabs {
    color: pink; /* 当宿主元素含有 pink 类时的选项卡样式. */
  }
</style>
```

## 使用 :host-context(<selector>) 伪类来定制化元素样式

`:host-context(<selector>)` 伪类找出宿主元素若宿主元素或者宿主元素任意的祖先元素匹配 `<selector>`。

常用于定制化。例如，开发者通过为 `<html>` 或者 `<body>` 添加类来进行定制化：

```
<body class="lightheme">
  <custom-container>
  …
  </custom-container>
</body>
```

或者

```
<custom-container class="lightheme">
  …
</custom-container>
```



当宿主元素的祖先元素包含有 .lightheme 类 `:host-context(.lightheme)` 将会样式化 `<fancy-tabs>`：

```
:host-context(.lightheme) {
  color: black;
  background: white;
}
```

可以使用 `:host-context()` 来进行定制化主题样式，但是更好的方法即通过使用 CSS 自定义属性来创建样式钩子。

## 从外部样式化组件宿主元素

开发者可以从外部通过把标签名作为选择器来样式化组件宿主元素，如下：

```
custom-container {
  color: red;
}
```

**外部样式比 Shadow DOM 中定义的样式拥有更高的优先级。**

例如，假设用户书写如下选择器：

```
custom-container {
  width: 500px;
}
```

将会覆盖如下组件样式规则 ：

```
:host {
  width: 300px;
}
```

组件自身样式化只能做到这么多。但如果想要样式化组件内部属性呢？这就需要 CSS 自定义属性。

## 使用 CSS 自定义属性来创建样式钩子

若组件作者使用 CSS 自定义属性提供样式钩子，用户可以用来更改内部样式。

这和 `<slot>` 思路类似只是应用到了样式。

让我们看如下示例：

```
<!-- 主页面 -->
<style>
  custom-container {
    margin-bottom: 60px;
    --custom-container-bg: black;
  }
</style>

<custom-container background>…</custom-container>
```

Shadow DOM 内部：

```
:host([background]) {
  background: var(--custom-container-bg, #CECECE);
  border-radius: 10px;
  padding: 10px;
}
```

该示例中，因为用户提供了该背景颜色值，所以组件将会把黑色作为背景颜色值。否则，默认为 `#CECECE`。

作为组件作者，需要让开发者知道可以使用的 CSS 自定义属性。可以把自定义属性看作组件的公共接口。

## 插槽 JavaScript 接口

Shadow DOM API 提供可用来操作插槽的程序。

## slotchange 事件

当一个插槽的分发元素节点发生变化的时候触发 slotchange 事件。例如，当用户从 light DOM 中添加/删除子节点。

```
var slot = this.shadowRoot.querySelector('#some_slot');
slot.addEventListener('slotchange', function(e) {
  console.log('Light DOM change');
});
```

可以在元素的构造函数中创建 `MutationObserver` 来监听 light DOM 的其它类型的修改事件。前面文章中有介绍过 [MutationObserver 的内部构造及使用指南](https://blog.sessionstack.com/how-javascript-works-tracking-changes-in-the-dom-using-mutationobserver-86adc7446401)。

## assignedNodes() 方法

了解哪些元素是和插槽相关联是很有用处的。调用 `slot.assignedNodes()` 可以找出哪些元素是由插槽渲染的。`{flatten: true}` 选项会返回插槽的默认内容(若没有分发任何节点)。

看一下如下示例：

`<slot name='slot1'><p>Default content</p></slot>`

假设以上内容包含在一个叫做 `<my-container>` 的组件内部。

让我们查看一下该组件的不同用法，然后调用 `assignedNodes()` 输出不同的结果：

第一例中，我们将往插槽中添加内容：

```
<my-container>
  <span slot="slot1"> container text </span>
</my-container>
```

调用 `assignedNodes()` 将会返回 `[<span slot="slot1"> container text </span>]`。注意结果为一个节点数组。

第二例中，将不添加内容：

`<my-container> </my-container>`

调用 `assignedNodes()` 将会返回空数组 `[]`。

但是，假设添加 `{flatten: true}` 参数将会返回默认内容：`[<p>Default content</p>]`。

同理，为了查找插槽中的元素，开发者可以调用 `assignedNodes()` 来找出元素被挂载到哪个组件插槽中。

## 事件模型

Shadow DOM 中的事件冒泡的经过是值得注意的。

事件目标被调整为维护 Shadow DOM 的封闭性。当事件被重新定位，看起来是由组件自身产生而不是作为组件一部分的 Shadow DOM 内部元素。

这里有传播出 Shadow DOM 的事件列表(还有一些只能在 Shadow DOM 内传播)：

* **Focus 事件**：blur, focus, focusin, focusout
* **鼠标事件**：click, dblclick, mousedown, mouseenter, mousemove 等.
* **滚轮事件**: wheel
* **输入事件**: beforeinput, input
* **键盘事件**: keydown, keyup
* **组合事件**: compositionstart, compositionupdate, compositionend
* **拖拽事件**: dragstart, drag, dragend, drop 等.

## 自定义事件

默认情况下，自定义事件不会传播出 Shadow DOM。开发者若想要分派自定义事件且想要传播出 Shadow DOM，需要添加 `bubbles: true` 和 `composed: true` 选项参数。

让我们瞧瞧类似这样的事件分派：

```
var container = this.shadowRoot.querySelector('#container');
container.dispatchEvent(new Event('containerchanged', {bubbles: true, composed: true}));
```

## 浏览器兼容情况

可以通过检查 attachShadow 来检查是否支持 Shadow DOM 功能：

```
const supportsShadowDOMV1 = !!HTMLElement.prototype.attachShadow;
```

![](https://user-images.githubusercontent.com/1475173/52962053-43345880-33d7-11e9-9fcd-7199da3b03ca.png)

参考资料：

* https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM
* https://developers.google.com/web/fundamentals/web-components/shadowdom
* https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots
* https://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/#toc-style-host