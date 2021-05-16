# 函数式编程及与其他范式对比

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-functional-style-and-how-it-compares-to-other-approaches-8a1398c73919)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第二十五章。**

![](./assets/25_cover.jpeg)

## 概述

在此之前，已经有程序员使用逻辑式，过程式以及最常见的面向对象范式来编写代码。这些范式涵盖了用于解决计算问题的编码风格，框架，特性和模式，也有助于各种编程语言的发展。

比如过程式语言（COBOL）是基于过程式原理，即每个语句可被解释为子程序或函数。

另一方面，函数式编程是本文的基石，涉及到项目构建，问题解决或使用数学函数来处理计算。我们将使用数据作为“输入”的函数来编写代码，该数据经过函数计算并作为“输出”返回。

函数式编程的美丽之处在于这些函数无法更改我们的数据或输入（数据不可变性）并可以观察到任何副作用。语句以函数来表示：接受输入，产生输出。

本文将讨论更多有关函数式编程的内容，包括它如何在 JavaScript 中工作以及一些重要概念，以便更好理解 JavaScript 中的函数式编程。

## JavaScript 面向对象概述 

毋庸置疑 JavaScript 是基于原型的语言，而不是基于类，并且 JavaScript 开发者经常会混淆或误解此模型。基于类的语言（例如C＃，Java等）在两个主要概念：类和实例。类定义了该对象在实例化时应该具有的所有属性。

在 JavaScript 中，引入的 OOP 语法使其看起来是面向对象，但实际上并非如此。这意味着当开发者使用类语法创建对象时，它会自动获取其原型。另外，如果有不存在的方法或属性，JavaScript 将检查原型以查看其是否存在（原型链）。

让我们看一个代码示例，该例子说明 JavaScript 中的 `class` 语法：

```js
let Person = class {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  speak() {
    return `Hello, my name is  ${this.name} and I am ${this.age}`
  }
}
```

请注意，`Person` 类具有一个有两个参数的构造函数。`this` 关键字指向属性 `Person`。

方法 `speak()` 可以在以下创建新示例的例子中一样使用：

```js
let victor = new Person('Victor', 23);
console.log(victor.speak());
```

`new Person` 调用构造函数，并传入将设置至 `victor` 对象的参数。在底层方法 `speak()` 将被添加至构造函数的原型中。 

同样，我们可以基于 `Person` 类创建一个新的类，以扩展其中已存在的属性和方法。如下：

```js
let Work = class extends Person {
  constructor(name, age, work) {
    super(name, age);
    this.work = work
  }
  
  getInfo() {
    return `Hey! It's ${this.name}, I am age ${this.age} and work for ${this.work}.`
  }
}

let alex = new Work('Alex', 30, 'SessionStack');
console.log(alex.getInfo());
```

这样我们就创建了一个对象并将其扩展以定义更多属性。在基于类的编程语言中，这种概念称为子类。

那么在 JavaScript 旧语法中是应该实现呢？ 记住 `class` 语法在底层仍然只是制作原型。一起来看下面这个例子：

```js
let Person = function(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.speak = function() {
  return `Hello, my name is  ${this.name} and I am ${this.age}`
}

let victor = new Person(`Victor`, 23)
console.log(victor.speak())
```

上面的代码中，我们调用方法 `Person()`, 注意方法前的 `new` 关键字改变了函数的上下文从而调用了构造函数。基于 `Person` 对象创建的示例返回并赋值给 `victor`。

就像在上面 `Work` 扩展 `Person` 类语法那样扩展对象，可以与以原型方法完成：

```js
let Work = function(name, age, work) {
  Person.call(this, name, age);
  this.work = work;
}
```

注意我们并没有使用 `super()` 关键字因为它并不存在与 JavaScript 原型上下文中。`call()` 方法接收 `this` 和参数，并将 `Person` 的属性添加到 `Work` 。`call()` 方法将构造函数从 `Person` 链接到 `Work`，换句话说，我们从 `Person` 借用属性并将其添加到 `Work` 上。

同样的，`class` 语法里的 `extend` 关键字也不是原型的一个属性。然而 `Person` 和 `Work` 可以通过以下方式连接：

```js
Object.setPropertyOf(Work.prototype, Person.prototype);
```

`Work` 原型现在将使用 `Person` 原型。下面的例子是向新原型`Work`中添加方法：

```js
Work.prototype.getInfo = function() {
       return `Hey! It's ${this.name}, I am age ${this.age} and work for ${this.work}.`
}

let alex = new Work("Alex", 40, 'SessionStack');
console.log(alex.getInfo());
```

现在我们应当更了解基于类和基于原型的概念。应该清楚的是 JavaScript 是一种基于对象的语言，它通过原型链接起来。大多数开发者使用的类语法只是 ES6 的语法糖。

通过这基本概述，现在我们可以继续介绍 JavaScript 中的函数式编程了。

## 什么是 JavaScript 函数式编程？

对于大多数开发者来说，在 JavaScript 中使用函数式编程的思想似乎会更轻松。为什么？众所周知，JavaScript 是一门基于原型的语言，而这些 `prototype inheritance`，`this`，`setPropertyOf`和其他确实令人困惑，而且大多都被被误解。

不过，与在基于原型的编码中使用错误的 `this` 绑定相比，我们有 JavaScript 的函数式方法使工作变得更简单，更少的 bug且易于维护。

使用函数式编程思想的 JavaScript 开发者社区很大，并且大多数库都允许在项目中使用这种范式。这意味着在遇到问题时，StackOverflow 或其他任何地方都将提供足够的帮助。

JavaScript 中的函数式代码如下：

```js
const sayHello = function(name) {
  return `Hello ${name}`;
}

sayHello('Victor');

# => Hello Victor
```

我们声明了一个带有参数 `name` 的函数，当该函数被调用时，如果参数值是 `Victor`，则返回字符串 `Hello Victor`。这是使用函数编写代码时一种更简洁的自解释方法。

要了解 JavaScript 函数式编程的更多信息，需要了解某些重要的概念比如：纯函数，高阶函数，不变性，一等函数等。我们还将在本文中讨论它们。

## JavaScript 函数式编程中的概念

### 纯函数

函数编程式的一个主要目标是 -- 避免副作用并使用纯函数。避免副作用意味着函数应仅通过接受参数（输入）并对其进行处理来进行计算。函数应该是纯粹的。让我们看一个具有副作用的非纯函数示例。

```js
let surname = 'Jonah';

const sayHi = function() {
  return `Hi ${surname}`;
}
```

上面代码中的函数没有输入（参数），但从函数外部接收到一个全局变量 `surname`。这可能会产生副作用。

纯函数代码实例应当如下：

```js
const sayHi = function(surname) {
  return `Hi ${surname}`;
}
```

函数仅关心输入的 `surname` 并将对进行计算。这就是纯函数。当谈到函数式编程时，这是主要且最重要的概念。

### 高阶函数

围绕这个概念，我们将更好地理解函数式编程的风格。高阶函数是将函数作为参数或返回函数的函数。请记住，函数是值，这意味着可以传递这些值。例如：

```js
const getSum = function(num) {
  return num + num;
}

getSum(9);
```

我们创建了一个函数并将其分配给变量 `getSum`，现在我们可以将变量传递给另一个值（变量）。像这样：

```js
const addNum = getSum;

addNum(9);
```

我们可以继续将函数（值）传递到另一个函数中，以帮助我们编写或将许多较小的函数引入较大的函数。这就产生了组合。

```js
function a(x) {
  return x * 2;
}

function b(x) {
  return x + 1;
}

function c(x) {
  return x * 3;
}

const d = c(b(a(2)));
console.log(d) // 15
```

我们可以传递每个函数返回的值，并将其传递给下一个函数。这就是我们使用高阶函数的原因，因为它可以进行组合，使我们的代码更整洁且具有鲁棒性。

数组的 `filter()` 方法是最常用的高阶函数示例之一。当另一个函数作为参数传递时，此法为我们生成一个新数组。例如：

```js
function isLarge(value) {
  return value > 10;
}

const dataArray = [10, 11, 12, 3, 4];


const filteredArray = dataArray.filter(isLarge);
console.log(filteredArray); // [11, 12]
```

`filter()` 方法遍历数组 `dataArray`, 每个元素都将作为参数传入回调函数 `isLarge`。 回调函数应当返回一个布尔值。如果为 `true`, 值就会被添加至新数组。这简单又不失优雅。其他受欢迎的示例还有是 `map()` 和 `reduce()`。我们不需要在这里任何地方使用 for 循环。在界山下一个概念之前，让我们看一下 `map()` 的一个例子：

```js
function squareRoot(value) {
  return Math.sqrt(value)
}

const dataArray = [4, 9, 16];


const mappedArray = dataArray.map(squareRoot);
console.log(mappedArray); // [2, 3, 4]
```

### 不变性

函数式编程中的另一个概念是强调了避免改变的重要性。当我们说 `mutation` 时，我们的意思是改变状态或数据。因此，当某些东西是不可变的，一旦它被设置，它就保持不变，而当我们要进行更改时，我们使用新更改的数据来设置一个新对象。让我们来看下面的代码：

```js
let data = [1, 2, 3, 4, 4];

data[4] = 5;

console.log(data); // [1, 2, 3, 4, 5]
```

请注意，我们如何将数据从 `1、2、3、4、4` 更改为 `1、2、3、4、5`，这看起来并不好，因为我们可能在代码中引入错误。想象一下我们在代码库中的某个位置计算了数据数组，并且在这时已经被更改，使用函数式编程是可以避免这种情况的。那么，不变性如何在 JavaScript 中起作用？下面我用代码来解释一下。

```js
const names = ["Alex", "Victor", "John", "Linda"]

const newNamesArray = names.slice(1, 3) // ["Victor", "John"]
```

原来的数组 `name` 没有被修改，并且返回了新数组。

对象也可以使用方法 `Object.freeze()` 变成不可变。此方法冻结对象并使其不允许删除数据或添加到对象属性。比如：

```js
const employee = {
  name: "Victor",
  designation: "Writer",
  address: {
    city: "Lagos"
  }
};

Object.freeze(employee);

employee.name = "Max"
//Outputs: Cannot assign to read-only property 'name'

//Checks if our object is immutable or not
Object.isFrozen(employee); // === true
```

这样对象变得不可变并且不会受到干扰，这绝对是我们期待的结果。

不变性一直存在一个问题，即当需要更改时，必须一遍又一遍地复制数组。比如有一个包含 1000 个子元素的 `names` 数组，并且我们需要不断对其进行修改并创建新数组，那么最终可能会占用过多内存，从而出现时间复杂度或持久化的问题。当然社区也存在解决方案，并有对应的 JavaScript 库比如[immutable.js](https://immutable-js.github.io/immutable-js/)和[Mori](https://swannodette.github.io/mori/#immutability)，但本文不对此进行拓展。

通过这几个概念应该可以了解到 JavaScript 函数式编程的思想和美。我们可以看到代码更具可读性和简洁性，可用于执行快速和可操作的流程如数据处理和并发。

## 声明式与命令式 JavaScript

JavaScript 不仅可以进行函数式编程，还可以采用声明式和命令式方法编写代码。我们直接来命令式方法的代码风格。命令式方法像是陈述解决问题所需的所有步骤。例如，如果想吃意大利面，那么必要的步骤如下：

- 煮意大利面
- 混合原料
- 蒸意大利面及原料
- 混合酱料

我确信使用这些步骤来会得到差劲的面食，但重点是使用了命令式方法来实现。

而声明式方法只是声明或说出想做什么。比如要煮意大利面，仅此而已。当涉及到声明式方法时，不需要表达控制流。这是两种方法的简单描述用于对比它们之间的差异，而不要指导开发者该使用其中的某一种。

###　JavaScript　中的命令式方法：

这里我们将直接告诉计算机明确的对应操作。

```js
// Function to filter an array; return greater than 5 numbers

const filterArray = (array) => {
  let filteredArray = [];
  for(let i = 0; i < array.length; i++) {
    if(array[i] > 5) {
      filteredArray.push(array[i]);
     }
  }
  return filteredArray;
}

const array = [1, 2, 3, 4, 5, 6, 7, 8]
filterArray(array)
```

与其告诉计算机想要什么，我们只是逐步说明要实现的目标。步骤包括：

- 声明一个空数组
- 遍历给定数组
- if/else，如果数值大于5
- 将通过测试的元素推入先前声明的空数组中
- 显示新数组

###　JavaScript　中的声明式方法：

```js
// Filter method to give us a new array

const filterArray = array => array.filter(x => x > 5);

const array = [1, 2, 3, 4, 5, 6, 7, 8];

console.log(filterArray(array)); // [6, 7, 8]
```

## 函数式编程的优势

开发人员选择函数式编程的常见原因有几个。虽然对初学者来说会比较棘手，但是当掌握这些概念时，它也会变得更加容易。主要原因如下：

- 为我们的代码库提供了模块化；功能之间可以结合甚至分离。它将函数视为值，使我们可以将函数作为参数传递。这也提供了代码可复用性，其中函数是可组合的（被视为组件），如 React 一样。
- 调试更加容易：更少的 bug 和更轻松的维护。高阶函数可以更安全地进行编程，因为它更容易调试和维护更少的代码。
- 任何开发人员都可以快速阅读和理解您的代码。因为编写的是你直观的想法，而不是计算机该如何替你思考。
- 函数式编程中开发者可以更快地编写整洁的代码。用高阶函数 `map()，filter()` 替换 for 循环之类的迭代代码会带来简洁性。

## 结论

本文通过一些示例对 JavaScript 面向对象的工作方式进行了基本概述，及 `class` 语法（语法糖）和基于原型的 JavaScript 间的区别。还了解了 ES6 类语法仍然在底层实现了基于原型的语法。

JavaScript 中的一些函数式编程概念，可以帮助开发者更好地判断是否选择这种范式。纯函数是这种范式的重中之重。我们还介绍了声明式和命令式方法及对应的代码示例。

编程范式没有孰是孰非，本文也不是对此的辩论而旨在介绍函数式编程的概念以助你更好的理解。

## 更多关于函数式编程的内容

- [Why Functional programming matters](https://www.youtube.com/watch?v=XrNdvWqxBvA)
- [An introduction to functional programming in JavaScript](https://opensource.com/article/17/6/functional-javascript)
- [Master the Coding interview: What is functional programming](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-functional-programming-7f218c68b3a0)
