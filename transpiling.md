# 类和继承及 Babel 和 TypeScript 代码转换探秘

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-the-internals-of-classes-and-inheritance-transpiling-in-babel-and-113612cdc220)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第十五章。**

如今使用类来组织各种软件工程代码是最常用的方法。本章将会探索实现 JavaScript 类的不同方法及如何构建类继承。我们将深入理解原型继承及分析使用流行的类库模拟实现基于类继承的方法。接下来，将会介绍如何使用转换器为语言添加非原生支持的语法功能和如何在 Babel 和 TypeScript 中运用以支持  ECMAScript 2015 类。最后介绍几个 V8 原生支持实现类的例子。

## 概述

JavaScript 没有原始类型且一切皆对象。比如，如下字符串：

```
const name = "SessionStack";
```

可以立即调用新创建对象上的不同方法：

```
console.log(a.repeat(2)); // 输出 SessionStackSessionStack
console.log(a.toLowerCase()); // 输出 sessionstack
```

JavaScript 和其它语言不一样，声明一个字符串或者数值会自动创建一个包含值的对象及提供甚至可以在原始数据类型上运行的不同方法。

另外一个有趣的事实即诸如数组的复杂数据类型也是对象。当使用 typeof 来检查一个数组实例的时候会输出 `object`。数组中每个元素的索引值即对象的属性。所以通过数组索引来访问元素的时候，实际上是在访问一个数组对象的属性然后获得属性值。当涉及到数据存储方式的时候，以下两种定义是相同的：

```
let names = [“SessionStack”];

let names = {
  “0”: “SessionStack”,
  “length”: 1
}
```

因此，访问数组元素和对象属性的速度是一样的。我走了很多弯路才发现该事实。以前有段时间，我得对项目中某段至关重要的代码进行大量的性能优化。当试验过其它简单的办法之后，我把所有的对象替换为数组。按理说，访问数组元素会比访问哈希表的键值更快。然而，我惊奇地发现没有半点性能的提升。在 JavaScript 中，所有数据元素的操作都是由访问哈希表中的键来实现的且耗时相同。

## 使用原型模拟类

当谈到对象的时候，脑海中首先出现的即类。开发人员习惯于使用类和类之间的关联来组织程序。虽然 JavaScript 中一切皆对象，但是并没有使用经典的基于类的继承。而是使用[原型](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)来实现继承。

![](https://user-images.githubusercontent.com/1475173/43037673-5c0635c2-8d42-11e8-99be-5bc3ec88f1ce.png)

在 JavaScript 中，每个对象关联其原型对象。当访问对象的一个方法或属性的时候，首先在对象自身进行搜索。如果没有找到，则在对象原型上进行查找。

让我们以定义基础类的构造函数为例：

```
function Component(content) {
  this.content = content;
}

Component.prototype.render = function() {
    console.log(this.content);
}
```

在原型上添加 render 函数，这样 Component 的实例就可以使用该方法。当调用该 Component 类实例的方法的时候，首先在实例上查询该方法。然后在原型上找到该渲染方法。

![](https://user-images.githubusercontent.com/1475173/43037678-5d436ce8-8d42-11e8-9b8a-005b904fa6c9.png)

现在，尝试扩展 component 类，引入新的子类。

```
function InputField(value) {
    this.content = `<input type="text" value="${value}" />`;
}
```

如果想要 InputField 扩展 component 类的方法且可以调用其 render 方法，就需要更改其原型。当调用子类的实例方法的时候，肯定不希望在一个空原型上进行查找(*这里其实所有对象都一个共同的原型，这里原文不够严谨*)。该查找会延续到 Component 类上。

```
InputField.prototype = Object.create(new Component());
```

这样，就可以在 Component 类的原型上找到 render 方法。为了实现继承，需要把 InputField 的原型设置为Component 类的实例。大多数库使用 [Object.setPrototypeOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf) 来实现继承。

![](https://user-images.githubusercontent.com/1475173/43037674-5c431e74-8d42-11e8-91df-1772608dc942.png)

然而，还有其它事情需要做。每次扩展类，所需要做的事如下：

* 设置子类的原型为父类的实例
* 在子类的构建函数中调用父类构造函数，这样才可以执行父类构造函数的初始化逻辑。
* 引入访问父类的方法。当重写父类方法的时候，会想要调用父类方法的原始实现。

正如你所见，当想要实现所有基于类继承的功能的时候，每次都需要执行这么复杂的逻辑步骤。当需要创建这么多类的时候，即意味着需要把这些逻辑封装为可重用的函数。这就是开发者当初通过各种类库来模拟从而解决基于类的继承的问题。这些解决方案是如此流行，以至于迫切需要语言集成该功能。这就是为什么 ECMAScript 2015 的第一个重要修订版中引入了支持基于类继承的创建类的语法。

## 类转换

当在 ES6 或者 ECMAScript 2015 中提议新功能时，JavaScript 开发者社区就迫不及待想要引擎和浏览器实现支持。一种好的实现方法即通过代码转换。它允许使用 ECMAScript 2015 来进行代码编写然后转换为任何浏览器均可以运行的 JavaScript 代码。这包括使用基于类的继承来编写类并转换为可执行代码。

![](https://user-images.githubusercontent.com/1475173/43037679-5d819b12-8d42-11e8-82f4-3443714ced84.png)

Babel 是最为流行的转换器之一。让我们通过 babel 转换 component 类来了解代码转换原理。

```
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
  	console.log(this.content)
  }
}

const component = new Component('SessionStack');
component.render();
```

以下为 Babel 是如何转换类定义的：

```
var Component = function () {
  function Component(content) {
    _classCallCheck(this, Component);

    this.content = content;
  }

  _createClass(Component, [{
    key: 'render',
    value: function render() {
      console.log(this.content);
    }
  }]);

  return Component;
}();
```

如你所见，代码被转换为可在任意环境中运行的 ECMAScript 5 代码。另外，引入了额外的函数。它们是 Babel 标准库的一部分。编译后的文件中引入了 `_classCallCheck` 和 `_createClass` 函数。第一个函数保证构造函数永远不会被当成普通函数调用。这是通过检查函数执行上下文是否为一个 Component 对象实例来实现的。代码检查 this 是否指向这样的实例。第二个函数 `_createClass`  通过传入包含键和值的对象数组来创建对象(类)的属性。

为了理解继承的工作原理，让我们分析一下继承自 Component 类的 InputField 子类。

```
class InputField extends Component {
    constructor(value) {
        const content = `<input type="text" value="${value}" />`;
        super(content);
    }
}
```

这里是使用 Babel 来处理以上示例的输出：

```

var InputField = function (_Component) {
  _inherits(InputField, _Component);

  function InputField(value) {
    _classCallCheck(this, InputField);

    var content = '<input type="text" value="' + value + '" />';
    return _possibleConstructorReturn(this, (InputField.__proto__ || Object.getPrototypeOf(InputField)).call(this, content));
  }

  return InputField;
}(Component);
```

本例中，在 _inherits 函数中封装了继承逻辑。它执行了前面所说的一样的操作即设置子类的原型为父类的实例。

为了转换代码，Babel 执行了几次转换。首先，解析 ES6 代码并转化成被称为抽象语法树的中间展示层，抽象语法树在之前的[文章](https://github.com/Troland/how-javascript-works/blob/master/ast.md)有讲过了。该树会被转换为一个不同的抽象语法树，该树上每个节点会转换为对应的 ECMAScript 5 节点。最后，把抽象语法树转换为 ES5 代码。

过程即：ES6 代码，语法解析等 => AST，进行 AST 相关操作 => 转换为 ES5 代码。

## Babel 中的抽象语法树

AST 由节点组成，每个节点只有一个父节点。Babel 中有一种基础类型节点。该节点包含节点的内容及在代码中的位置的信息。有各种不同类型的节点比如字面量表示字符串，数值，空值等等。也有控制流(if) 和 循环(for, while)的语句节点。另外，还有一种特殊类型的类节点。它是基础节点类的子类，通过添加字段来存储对基础类的引用和把类的正文作为单独的节点来拓展自身。

转化以下代码片段为抽象语法树：

```
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
    console.log(this.content)
  }
}
```

以下为该代码片段的抽象语法树的大概情况：

![](https://user-images.githubusercontent.com/1475173/43037675-5ca41d32-8d42-11e8-906f-7354ddd19741.png)

创建抽象语法树后，每个节点转换为其对应的 ECMAScript 5 节点然后转化为遵循 ECMAScript 5 标准规范的代码。这是通过寻找离根节点最远的节点然后转换为代码。然后，他们的父节点通过使用为每个子节点生成的代码片段来转化为代码，依次类推。该过程被称为 [depth-first traversal](https://en.wikipedia.org/wiki/Depth-first_search) 即深度优先遍历。

以上示例，首先生成两个 MethodDefinition 节点，之后类正文节点的代码，最后是 ClassDeclaration 节点的代码。

## 使用 TypeScript 进行转换

TypeScript 是另一个流行的框架。它引入了一种编写 JavaScript 程序的新语法，然后转换为任意浏览器或引擎可以运行的 EMCAScript 5 代码。以下为使用 Typescript 实现 component 类的代码：

```
class Component {
    content: string;
    constructor(content: string) {
        this.content = content;
    }
    render() {
        console.log(this.content)
    }
}
```

以下为抽象语法树示意图：

![](https://user-images.githubusercontent.com/1475173/43037672-5b7623f6-8d42-11e8-82f4-f42810032c4d.png)

同样支持继承。

```
class InputField extends Component {
    constructor(value: string) {
        const content = `<input type="text" value="${value}" />`;
        super(content);
    }
}
```

代码转换结果如下：

```
var InputField = /** @class */ (function (_super) {
    __extends(InputField, _super);
    function InputField(value) {
        var _this = this;
        var content = "<input type=\"text\" value=\"" + value + "\" />";
        _this = _super.call(this, content) || this;
        return _this;
    }
    return InputField;
}(Component));
```

类似地，最后结果包含了一些来自 TypeScript 的类库代码。`__extends` 中封装了和之前第一部分讨论的一样的继承逻辑。

随着 Babel 和 TypeScript 的广泛使用，标准类和基于类的继承渐渐成为组织 JavaScript 程序的标准方式。这就推动了浏览器原生支持类。

## 类的原生支持

2014 年，Chrome 原生支持[类](https://www.chromestatus.com/feature/4633745457938432)。这就可以不使用任意库或者转换器来实现声明类的语法。

![](https://user-images.githubusercontent.com/1475173/43037671-5b382b28-8d42-11e8-9098-93bacea9625b.png)

类的原生实现的过程即被称为语法糖的过程。这只是一个优雅的语法可以被转换为语言早已支持的相同的原语。使用新的易用的类定义，归根结底也是要创建构造函数和原型赋值。

![](https://user-images.githubusercontent.com/1475173/43037680-5dbe8d1a-8d42-11e8-972a-7505ce1001e5.png)

## V8 引擎支持情况

让我们了解下 V8 是如何原生支持 ES6 类的。如前面[文章](https://github.com/Troland/how-javascript-works/blob/master/ast.md)所讨论的那样，首先解析新语法为可运行的 JavaScript 代码并添加到 AST 树中。类定义的结果即在抽象语法树中添加一个 [ClassLiteral](https://github.com/v8/v8/blob/a86fa968136f0ec6237f51a0d535fbd932868d4d/src/ast/ast.h#L2421) 类型的新节点。

该节点包含了一些信息。首先，它把构造函数当成单独的函数且包含类属性集。这些属性可以是一个方法，一个 getter, 一个 setter, 一个公共变量或者私有字段。该节点还储存了指向其拓展的父类的指针引用，并储存了构造函数，属性集和及父类，依次类推。

一旦把新的 ClassLiteral [转换为字节码](https://github.com/v8/v8/blob/be3a1df9008ee78d1101855d3044db54a203f515/src/interpreter/bytecode-generator.cc#L1818)，再将其转化为各种函数和原型。