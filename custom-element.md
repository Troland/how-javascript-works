# 自定义元素探秘及构建可复用组件最佳实践

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-under-the-hood-of-custom-elements-best-practices-on-building-reusable-e118e888de0c)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理第十九章。**

## 概述

在 [前述文章](https://blog.sessionstack.com/how-javascript-works-the-internals-of-shadow-dom-how-to-build-self-contained-components-244331c4de6e)中，我们介绍了 Shadow DOM 接口和一些其它概念，而这些都是网页组件的组成部分。网页组件背后的思想即通过创建颗粒化，模块化和可复用的元素来扩展 HTML 内置功能。这是一个已经被所有主流浏览器兼容的相对崭新的 W3C 标准且可以被用在生产环境之中，虽然不兼容的浏览器需要使用兼容库(将在随后的章节中进行讨论)。

正如开发者所知，浏览器为构建网站和网页程序提供了一些重要的开发工具。我们所说的 HTML，CSS 和 JavaScript 即开发者使用 HTML 来构建结构，CSS 使其美观，然后使用 JavaScript 来让页面动起来。然而，在网页组件出现之前，把 JavaScript 脚本行为和 HTML 结构组合起来并非易事。

本文将阐述网页组件的基石-自定义元素。总之，开发者可以使用自定义元素接口来创建包含 JavaScript 逻辑和样式的自定义元素(正如名称的字面意思)。许多开发者会把自定义元素和 shadow DOM 混为一谈。但是，他们是完全不同的概念且它们互补而不是可以相互替代的。

一些框架(比如 Angular，React) 试图通过引进其自有概念来解决同样的问题。开发者可以把自定义元素和 Angular 的指令或者 React 组件进行对比。然而，自定义元素是浏览器原生的且只需要原生 JavaScript，HTML 和 CSS。当然了，这并不意味着它可以取代一个典型的 JavaScript 框架。现代框架不仅仅为开发者提供模仿自定义元素行为的能力。因此，可以同时使用框架和自定义元素。

## 接口

在深入了解之前，让我们先大概快速浏览一下接口的内容。全局 `customElements` 对象为开发者提供了一些方法：

* `define(tagName, constructor, options)` －创建一个新的自定义元素。

  包含三个参数：自定义元素的可用标签名称，自定义元素类定义及选项参数对象。目前仅支持一个选项参数：`extends` 指定想要扩展的 HTML 内置元素名称的字符串。用来创建定制化内置元素。

* `get(tagName)` －若元素已经定义则返回自定义元素的构造函数否则返回 undefined。只有一个参数：自定义元素的有效标签名称。

* `whenDefined(tagName)`－返回一个 promise 对象，当定义自定义元素即解析。若元素已定义则立即进行解析。若自定义元素标签名称无效则摒弃 promise。只有一个参数：自定义元素的有效标签名称。

## 如何创建自定义元素

创建自定义元素实际上就是小菜一碟。开发者只需要做两件事：创建扩展 `HTMLElement` 类元素的类定义，然后以合适的名称注册元素。

```
class MyCustomElement extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
}

customElements.define('my-custom-element', MyCustomElement);
```

或者如你所愿，可以使用匿名类以防止弄乱当前作用域

```
customElements.define('my-custom-element', class extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
});
```

从以上例子可见，使用 `customElements.define(...)` 方法注册自定义元素。

## 自定义元素所解决的问题

实际上，问题是啥？**嵌套 DIV** 是问题之一。嵌套 Div 是啥？在现代网页程序中这是一个非常常见的现象，开发者会使用多个嵌套块状元素(div 互相嵌套之类)。

```
<div class="top-container">
  <div class="middle-container">
    <div class="inside-container">
      <div class="inside-inside-container">
        <div class="are-we-really-doing-this">
          <div class="mariana-trench">
            …
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

因为浏览器可以在页面上正常进行渲染，所以使用了这样的嵌套结构。但是，这会使得 HTML 不具可读性且难以维护。

因此，例如假设有如下组件：

![](https://user-images.githubusercontent.com/1475173/53549992-a446f380-3b70-11e9-9327-69d7b5458e4d.png)

那么传统 HTML 结构类似如下：

```
<div class="primary-toolbar toolbar">
  <div class="toolbar">
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-undo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-redo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-print">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-toggle-button toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-paint-format">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

但想象下如果可以使用类似如下代码：

```
<primary-toolbar>
  <toolbar-group>
    <toolbar-button class="icon-undo"></toolbar-button>
    <toolbar-button class="icon-redo"></toolbar-button>
    <toolbar-button class="icon-print"></toolbar-button>
    <toolbar-toggle-button class="icon-paint-format"></toolbar-toggle-button>
  </toolbar-group>
</primary-toolbar>
```

要我说，第二个示例清爽多了。第二个示例更具可维护性，可读性且对于浏览器和开发者更加合理。更加简洁。

另一个问题即可复用性。作为开发者，不仅仅要书写可运行的代码还得写出可维护代码。书写可维护代码即能够轻易地复用代码片段而不是重复地复制粘贴。

我将会给出一个简单的示例而你就会明白。假设有如下元素：

```
<div class="my-custom-element">
  <input type="text" class="email" />
  <button class="submit"></button>
</div>
```

若需要在其它地方使用这段代码，开发者需要再次书写相同的 HTML 结构。现在，想象 一下需要稍微修改一下这些元素。开发者需要找出每个代码需要修改的地方，然后一遍遍地做出同样的修改。太恶心了。。。

若使用如下码岂不会更好？

```
<my-custom-element></my-custom-element>
```

现代网页程序不仅仅只有静态 HTML。开发者需要做交互。这就需要 JavaScript。一般来说，开发者需要做的即创建一些元素然后在上面监听事件以响应用户输入。点击，拖拽或者悬浮事件等等。

```
var myDiv = document.querySelector('.my-custom-element');

myDiv.addEventListener('click', () => {
  myDiv.innerHTML = '<b> I have been clicked </b>';
});
```

```
<div class="my-custom-element">
  I have not been clicked yet.
</div>
```

使用自定义元素接口可以把所有的逻辑封装进元素自身。以下代码可以实现和上面代码一样的功能：

```
class MyCustomElement extends HTMLElement {
  constructor() {
    super();

    var self = this;

    self.addEventListener('click', () => {
      self.innerHTML = '<b> I have been clicked </b>';
    });
  }
}

customElements.define('my-custom-element', MyCustomElement);
```

```
<my-custom-element>
  I have not been clicked yet
</my-custom-element>
```

咋一看上去，自定义元素技术需要书写更多的 JavaScript 代码。但是在实际程序中，创建不需复用的单一组件的情况是很少见的。一个典型的现代网页程序的重要特征即大多数元素都是动态创建的。那么，开发者就需要分别处理使用 JavaScript 动态添加元素或者使用 HTML 结构中预定义内容。那么可以使用自定义元素来实现这些功能。

总之，自定义元素让开发者的代码更易理解和维护，并分割为小型，可复用及可封闭的模块。

## 要求

在创建自定义元素之前，开发者需要遵守如下特殊规则：

* 名称必须包含一个破折号 - 。这样 HTML 解析器就可以把自定义元素和内置元素区分开来。这样可以保证不会和内置元素出现命名冲突的问题(不管是现在或者将来当添加其它元素的时候)。比如，`<my-custom-element>` 是正确的而 `myCustomElement` 和 `<my_custom_element>` 则不然。
* 不允许重复注册标签名称。重复注册标签名称会导致浏览器抛出 `DOMException` 错误。不可以覆盖已注册自定义元素。
* 自定义元素不可以自我闭合。HTML 解析器只允许一小撮内置元素可以自我闭合(比如 `<img>`，`<link>`，`<br>`)。

## 功能

那么究竟自定义元素可以实现哪些功能？答案是很多。

最好用的功能之一即元素的类定义指向 DOM 元素自身。这意味着开发者可以直接使用 `this` 来直接监听事件，访问 DOM 属性，访问 DOM 元素子节点等等。

```
class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    this.addEventListener('mouseover', () => {
      console.log('I have been hovered');
    });
  }

  // ...
}
```

当然，这样开发者就可以使用新内容来覆盖元素的子节点。但一般不推荐这样做，因为这可能会导致不可预期的行为。作为自定义元素的使用者，因为不是使用者开发的，当元素里面的标记被其它内容所取代，用户会觉得很奇怪。

在元素生命周期的特定阶段，开发者可以在一些生命周期钩子中执行代码。

`constructor`

每当创建或者更新元素会触发构造函数(随后再详细讲解下)。一般情况会在该阶段初始化状态，监听事件，创建 shadow DOM 树等等。需要记住的是必须总是在构造函数中调用 `super()`。

`connectedCallback`

每当在 DOM 中添加元素的时候会调用 `connectedCallback` 方法。可以用来(推荐)延迟执行某些代码直到元素实际显示在页面(比如获取一个资源)。

`disconnectedCallback`

与 `connectedCallback `相反，当元素被从 DOM 删除时调用 `disconnectedCallback` 方法。一般用于释放资源的时候调用。需要注意的是若用户关闭选项卡不会调用 `disconnectedCallback` 方法。因此，首先开发者需要注意初始化代码。

`attributeChangedCallback`

每当添加，删除，更新或者替换元素的某一属性的时候调用。当解析器创建该元素的时候也会调用。但是，请注意只有在 `observedAttributes` 属性白名单中的属性才会触发。

`addoptedCallback`

当使用 `document.adoptNode(...)` 来把元素移动到另一个文档的时候会触发 `addoptedCallback`方法。

请注意以上所有的回调都是同步。例如，当把元素添加进 DOM 的时候只会触发连接回调。

## 属性反射

内置 HTML 元素提供了一个非常方便的功能:属性反射。这意味着直接修改某些属性值会直接反射到 DOM 的属性中。例如 `id` 属性：

`myDiv.id = 'new-id';`

将会更新 DOM 为

`<div id="new-id"> ... </div>`

反之亦然。这是非常有用的因为这样就使得开发者可以使用声明式书来写元素。

自定义元素自身没有该功能，但是有办法可以实现。为了在自定义元素中实现该相同的功能，开发者需要定义属性的 getters 和 setters 方法。

```
class MyCustomElement extends HTMLElement {
  // ...

  get myProperty() {
    return this.hasAttribute('my-property');
  }

  set myProperty(newValue) {
    if (newValue) {
      this.setAttribute('my-property', newValue);
    } else {
      this.removeAttribute('my-property');
    }
  }

  // ...
}
```

## 扩展元素

开发者不仅仅可以使用自定义元素接口创建新的 HTML 元素还可以用来扩展现有的 HTML 元素。而且该接口在内置元素和其它自定义元素中工作得很好。仅仅只需要扩展元素的类定义即可。

```
class MyAwesomeButton extends MyButton {
  // ...
}

customElements.define('my-awesome-button', MyAwesomeButton);
```

或者当扩展内置元素时，开发者需要为 `customElements.define(...)` 函数添加第三个 `extends` 的参数，参数值为需要扩展的元素标签名称。由于许多内置元素共享相同的 DOM 接口，`extends` 参数会告诉浏览器需要扩展的目标元素。若没有指定需要扩展的元素，浏览器将不会知道需要扩展的功能类型 。

```
class MyButton extends HTMLButtonElement {
  // ...
}

customElements.define('my-button', MyButton, {extends: 'button'});
```

一个可扩展原生元素也被称为可定制化内置元素。

开发者需要记住的经验法则即总是扩展已经存在的 HTML 元素。然后，渐进式添加功能。这样就可以保留元素之前的功能(只读属性，动态属性，函数)。

请注意现在只有 Chrome 67+ 才支持定制化内置元素。以后，其它浏览器也会实现，但是 Safari 完全没有实现该功能。

## 更新元素

如上所述，可以使用 `customElements.define(...)` 方法注册自定义元素。但这并不意味着，开发者必须首先注册元素。可以推迟在之后某个时间注册自定义元素。甚至可以在元素已经添加进 DOM 后再注册也是可以的。这一过程称为更新元素。开发者可以使用 `customElements.whenDefined(...)` 方法获取元素的定义时间。开发者传入元素标签名，返回一个 promise 对象，然后当元素注册后进行解析。

```
customElements.whenDefined('my-custom-element').then(_ => {
  console.log('My custom element is defined');
});
```

例如，开发者也许想要延迟执行代码直到元素内所有子元素均已定义。若内嵌自定义元素，这将会非常有用。

有时候，父元素有可能会依赖于其子元素的实现。在这种情况下，开发者需要确保子元素在其父元素之前定义。

## Shadow DOM

如前所述，需要把自定义元素和 shadow DOM 一起使用。前者用来把 JavaScript 逻辑封装进元素而后者用来为一小段  DOM 创建一个不为外部影响的隔绝环境。建议查看之前专门介绍 [shadow DOM](https://blog.sessionstack.com/how-javascript-works-the-internals-of-shadow-dom-how-to-build-self-contained-components-244331c4de6e) 的文章以便更好地理解 shadow DOM 概念。

只需调用 `this.attachShadow` 就可以在自定义元素内使用 shadow DOM

```
class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    let shadowRoot = this.attachShadow({mode: 'open'});
    let elementContent = document.createElement('div');
    shadowRoot.appendChild(elementContent);
  }

  // ...
});
```

## 模板

我们在之前的[文章](https://blog.sessionstack.com/how-javascript-works-the-internals-of-shadow-dom-how-to-build-self-contained-components-244331c4de6e)中简单介绍了下模板，需要单独一篇文章来专门介绍模板。这里，我们将会给出一个简单的示例来介绍如何在自定义元素中使用模板。

通过使用 `<template>`来声明一个 DOM 片段标签，该标签内容只会被解析而不会在页面上渲染。

```
<template id="my-custom-element-template">
  <div class="my-custom-element">
    <input type="text" class="email" />
    <button class="submit"></button>
  </div>
</template>
```

```
let myCustomElementTemplate = document.querySelector('#my-custom-element-template');

class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    let shadowRoot = this.attachShadow({mode: 'open'});
    shadowRoot.appendChild(myCustomElementTemplate.content.cloneNode(true));
  }

  // ...
});
```

那么现在，我们在自定义元素里面使用了 shadow DOM 和 模板，创建了一个元素，该元素作用域和其它元素隔绝且把 HTML 结构和 JavaScript 逻辑完美地隔离开来。

## 样式化

那么，我们讲解了 HTML 和 JavaScript，现在还剩下 CSS。显然，需要样式化元素。开发者可以在 shadow DOM 中添加样式但是用户如何从外部样式化元素呢？答案很简单－只需要和一般的内置元素一样写样式即可。

```
my-custom-element {
  border-radius: 5px;
  width: 30%;
  height: 50%;
  // ...
}
```

请注意外部定义的样式比元素内部定义的样式优先级高，外部样式会覆盖掉元素内定义的样式。

开发者需要明白有时候页面渲染，然后会在某些时刻会发现无样式内容闪烁(FOUC)。开发者可以通过为未定义组件定义样式及当元素开始显示的时候使用一些动画过渡效果。使用 :defined 选择器来达成这一效果。

```
my-button:not(:defined) {
  height: 20px;
  width: 50px;
  opacity: 0;
}
```

## 未知元素对比未定义自定义元素

HTML 规范非常灵活且允许开发者任意声明标签。若不被浏览器识别则会解析为 `HTMLUnknownElement`。

```
var element = document.createElement('thisElementIsUnknown');

if (element instanceof HTMLUnknownElement) {
  console.log('The selected element is unknown');
}
```

但是这并不适用于自定义元素。还记得讨论定义自定义元素时候的特殊命名规则吗？原因是因为当浏览器发现一个自定义元素的名称有效的时候，浏览器会把它解析为 `HTMLElement` ，然后浏览器会把它看作一个未定义的自定义元素。

```
var element = document.createElement('this-element-is-undefined');

if (element instanceof HTMLElement) {
  console.log('The selected element is undefined but not unknown');
}
```

在外观上， HTMLElement  和 HTMLUnknownElement 可能没啥不同，但是需要注意其它地方。解析器会区别对待这两种元素。具有有效自定义名称的元素会被看作拥有自定义元素实现。在定义实现细节之前该自定义元素会被看成一个空 div 元素。而一个未定义元素没有实现任何内置元素的任何方法或属性。

## 浏览器兼容

custom elements 第一版是在 Chrome 36+ 中引入的。被称为自定义元素接口 v0，虽然现在仍然可用，但是已经被弃用并被认为是糟糕的实现。若想要学习 v0 版，可以阅读这篇[文章](https://www.html5rocks.com/en/tutorials/webcomponents/customelements/)。从 Chrome 54 和 Safari 10.1(虽然只有部分支持)  开始支持自定义元素接口 v1，微软 Edge 还处于其原型设计阶段而 Mozilla 从 v50 开始支持，但默认不支持需要显式启用。目前只有 webkit 浏览器完全支持。然而，如上所述，可以使用兼容库兼容到包括 IE11 在内的所有浏览器。

## 检测可用性

通过检查 `window` 对象中的 `customElements` 属性是否可用来检查浏览器是否支持自定义元素。

```
const supportsCustomElements = 'customElements' in window;

if (supportsCustomElements) {
  // 可以使用自定义元素接口
}
```

否则需要使用兼容库：

```
function loadScript(src) {
  return new Promise(function(resolve, reject) {
    const script = document.createElement('script');

    script.src = src;
    script.onload = resolve;
    script.onerror = reject;

    document.head.appendChild(script);
  });
}

// Lazy load the polyfill if necessary.
if (supportsCustomElements) {
  // 浏览器原生支持自定义元素
} else {
  loadScript('path/to/custom-elements.min.js').then(_ => {
    // 加载自定义元素兼容
  });
}
```

总之，网页组件标准中的自定义元素为开发者提供了如下功能：

* 把 JavaScript 和 CSS 样式整合进一个单一的 HTML 元素
* 允许开发者扩展已有的 HTML 元素(内置和其它自定义元素)
* 不需要其它库或者框架的支持。只需要原生 JavaScript，HTML 和 CSS 还有可选的兼容库来支持旧浏览器。
* 可以和其它网页组件功能无缝衔接(shadow DOM，模板，插槽等)。
* 深度集成于浏览器开发者工具中。
* 使用现存的可访问功能

总之，自定义元素和开发者已经使用过的组件技术并没有什么大的不同。它只是让开发网页程序过程更加方便的另一种方式。因此，它使得更快地构建非常复杂的程序成为可能。

参考资料：

* <https://developers.google.com/web/fundamentals/web-components/customelements>
* <https://www.html5rocks.com/en/tutorials/webcomponents/customelements/>
* <https://github.com/w3c/webcomponents/>