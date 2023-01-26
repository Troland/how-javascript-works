# 迭代器及生成器高阶控制技巧

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-iterators-tips-on-gaining-advanced-control-over-generators-41dc3eb3bc20)，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理第二十三章。**

![](./assets/0_qelaH45ZWG3UybLO.jpg)

## 概述

无论在什么编程语言中，处理集合中的每个项是很常见的操作。JavaScript 也不例外，它提供了许多迭代集合的方法，从简单的 for 循环到更复杂的 `map()` 和 `filter()` 方法等。

迭代器和生成器将迭代的概念直接带入语言核心，并提供了一种自定义 `for...of` 循环的行为的机制。

## 迭代器

在 JavaScript 中，迭代器是一个对象，它定义一个序列，并在终止时可能返回一个返回值。 

迭代器可以是实现 `Iterator` 接口的任意对象。这意味着它必须有一个 `next()` 方法，该方法会返回一个具有以下两项属性的对象：

更具体地说，迭代器是通过使用 `next()` 方法实现 Iterator protocol 的任何一个对象，该方法返回具有两个属性的对象： value，这是序列中的 next 值；和 done ，如果已经迭代到序列中的最后一个值，则它为 true 。如果 value 和 done 一起存在，则它是迭代器的返回值。

* `value`： 序列中的 `next` 值
* `done`：如果已经迭代到序列中的最后一个值，则它为 `true` 。如果 `value` 和 `done` 同时存在，则它是迭代器的返回值。

一旦创建，迭代器对象可以通过重复调用 `next()`显式地迭代。在终止值产生后，对 `next()`的额外调用应该继续返回 `{done：true}`。

### 迭代器的使用

可能有时需要很多资源才能为数组分配值并遍历每个值。迭代器则只在必要时使用。这为迭代器遍历无限大小的序列提供了可能性。

这是一个创建简单的迭代器的示例，该迭代器生成[斐波那契数列](https://en.wikipedia.org/wiki/Fibonacci_number)：

```js
function makeFibonacciSequenceIterator(endIndex = Infinity) {
  let currentIndex = 0;
  let previousNumber = 0;
  let currentNumber = 1;

  return {
    next: () => {
      if (currentIndex >= endIndex) { 
          return { value: currentNumber, done: true }; 
      }

      let result = { value: currentNumber, done: false };
      let nextNumber = currentNumber + previousNumber;
      previousNumber = currentNumber;
      currentNumber = nextNumber;
      currentIndex++;

      return result;
    }
```
`makeFibonacciSequenceIterator` 开始生成斐波那契数，在到达 `endIndex` 时停止。迭代器在每次迭代时返回当前的斐波那契数，并在完成后继续返回最后生成的数。

这是通过上面的迭代器生成斐波那契数的输出：

```js
let fibonacciSequenceIterator = makeFibonacciSequenceIterator(5); // Generates the first 5 numbers.
let result = fibonacciSequenceIterator.next();
while (!result.done) {
    console.log(result.value); // 1 1 2 3 5 8
    result = fibonacciSequenceIterator.next();
}
```

### 定义可迭代目标

上述例子中迭代器的创建可能会产生某些问题因为没有办法事先校验这是否是一个有效迭代器。你可能会说返回的对象不是包含 `next()` 方法可以用来验证，但也存在一些非迭代对象本身就定义了  `next()` 。

这也是为什么 `JavaScript` 在定义一个可迭代对象时有更多要求的原因。

上面的斐波那契例子不会被 `JavaScript` 判断为一个可迭代对象。开发者可以用 `for...of` 循环对其进行遍历来测验。

```js
let fibonacciSequenceIterator = makeFibonacciSequenceIterator(5);

for (let x of fibonacciSequenceIterator) {
    console.log(x);
}
```

上面代码会抛出异常 `Uncaught TypeError: fibonacciSequenceIterator is not iterable` 。

一些内置类型，如 `Array` 或 `Map` 拥有默认的迭代行为，而其他类型（比如`Object`）则没有。

为了实现可迭代，一个对象必须实现 `@@iterator` 方法，这意味着这个对象（或其原型链中的任意一个对象）必须具有一个带 `Symbol.iterator` 键（key）的属性。属性的定义是一个返回待迭代元素的函数

让我们来来看看在斐波那契例子中应该进行如何的改造：

```js
function makeFibonacciSequenceIterator(endIndex = Infinity) {
  let currentIndex = 0;
  let previousNumber = 0;
  let currentNumber = 1;

  let iterator = {};
  iterator[Symbol.iterator] = () => {
    return {
      next: () => {
        if (currentIndex >= endIndex) { 
            return { value: currentNumber, done: true }; 
        }
        
        const result = { value: currentNumber, done: false };
        const nextNumber = currentNumber + previousNumber;
        previousNumber = currentNumber;
        currentNumber = nextNumber;
        currentIndex++;

        return result;
      }
    }
  };
```

现在我们可以用 `for...of` 循环对其进行遍历了。

```js
let fibonacciSequenceIterator = makeFibonacciSequenceIterator(5);

for (let x of fibonacciSequenceIterator) {
    console.log(x); //1 1 2 3 5 8
}
```

## 生成器

自定义迭代器非常有用，在某些场景下能极大提高效率，但由于需要显式地维护其内部状态，因此需要谨慎地创建及维护代码。

生成器函数提供另一种强大的思路：它允许你定义一个包含自有迭代算法的函数，同时它可以自动维护自己的状态。生成器函数使用 `function*` 语法。

调用时，生成器函数在一开始不会执行其代码。相反，它们返回一种特殊的迭代器类型，称为生成器。当通过调用生成器的 `next` 方法消耗一个值时，`Generator` 函数将一直执行直到遇到 `yield` 关键字。

一个生成器可以视为连续调用的生成一系列值的函数而返回单个值的函数。

生成器的语法包括一个称为 `yield` 的运算符，该运算符允许函数暂停直到请求下一个值。

```js
function* makeFibonacciSequenceGenerator(endIndex = Infinity) {
    let previousNumber = 0;
    let currentNumber = 1;
    
    for (let currentIndex = 0; currentIndex < endIndex; currentIndex++) {
        yield currentNumber;
        let nextNumber = currentNumber + previousNumber;
        previousNumber = currentNumber;
        currentNumber = nextNumber;
    }
}

let fibonacciSequenceGenerator = makeFibonacciSequenceGenerator(5);

for (let x of fibonacciSequenceGenerator) {
    console.log(x);
}
```

可以看出生成器语法更易于实现和维护。

### 生成器高阶控制

迭代器显式定义 `next()` 函数，以便通过 JavaScript 实现需要的接口。使用生成器时，将隐式添加 `next()` 函数，但该函数仍然存在。这就是生成器生成有效的可迭代对象的方式。

生成器隐式定义的 `next()` 函数接受可用于修改内部状态的参数。传递给 `next()` 的值将被 `yield` 语句接收。

让我们进一步修改斐波那契示例，以便控制在遍历序列的每个步骤中可以控制跳过多少个数字：

```js
function* makeFibonacciSequenceGenerator(endIndex = Infinity) {
    let previousNumber = 0;
    let currentNumber = 1;
    let skipCount = 0;
    
    for (let currentIndex = 0; currentIndex < endIndex; currentIndex++) {
        if (skipCount === 0) {
            skipCount = yield currentNumber; // skipCount is the parameter passed through the invocation of `fibonacciSequenceGenerator.next(value)` below.
            skipCount = skipCount === undefined ? 0 : skipCount; // makes sure that there is an input
        } else if (skipCount > 0){
            skipCount--;
        }
        
        let nextNumber = currentNumber + previousNumber;
        previousNumber = currentNumber;
        currentNumber = nextNumber;
    }
}

let fibonacciSequenceGenerator = makeFibonacciSequenceGenerator(50);

console.log(fibonacciSequenceGenerator.next().value);  // prints 1
console.log(fibonacciSequenceGenerator.next(3).value); // prints 5 since 1, 2, and 3 are skipped.
```

请注意，传递给 `next()` 首次调用的参数始终会被忽略。

另一个重要功能是能够通过调用生成器的 `throw()` 方法传递应抛出的异常值来强制生成器抛出异常。调用后将从生成器的当前挂起上下文中抛出此异常，就好像当前挂起的 `yield` 语句会被 `throw` 代替。

如果异常未在生成器中被捕获，它将通过 `throw()` 的外部调用向上传播，随后对 `next()` 的调用将导致 `done` 属性设为 `true`。让我们看下面的例子：

```js
function* makeFibonacciSequenceGenerator(endIndex = Infinity) {
    let previousNumber = 0;
    let currentNumber = 1;
    let skipCount = 0;
    
    try {
      for (let currentIndex = 0; currentIndex < endIndex; currentIndex++) {
          if (skipCount === 0) {
              skipCount = yield currentNumber;
              skipCount = skipCount === undefined ? 0 : skipCount;
          } else if (skipCount > 0){
              skipCount--;
          }
 
          let nextNumber = currentNumber + previousNumber;
          previousNumber = currentNumber;
          currentNumber = nextNumber;
      }
    } catch(err) {
    	console.log(err.message); // will print ‘External throw’ on the fourth iteration.
    }
}
 
let fibonacciSequenceGenerator = makeFibonacciSequenceGenerator(50);

console.log(fibonacciSequenceGenerator.next(1).value);
console.log(fibonacciSequenceGenerator.next(3).value);
console.log(fibonacciSequenceGenerator.next().value);
fibonacciSequenceGenerator.throw(new Error('External throw'));
console.log(fibonacciSequenceGenerator.next(1).value); // undefined will be printed since the generator is done.
```

也可以通过调用返回给定值的 `return(value)` 方法来终止生成器：

```js
let fibonacciSequenceGenerator = makeFibonacciSequenceGenerator(50);

console.log(fibonacciSequenceGenerator.next().value); // 1
console.log(fibonacciSequenceGenerator.next(3).value); // 5
console.log(fibonacciSequenceGenerator.next().value);   // 8
console.log(fibonacciSequenceGenerator.return(374).value); // 374
console.log(fibonacciSequenceGenerator.next(1).value); // undefined
```

### 异步生成器

可以在异步上下文中定义并使用生成器。异步生成器可以异步生成一系列值。

语法非常直接。关键字 `async` 需要位于生成器的 `function*` 前。

当遍历生成的序列时，需要在 `for…of` 构造中使用 `await` 关键字。

我们将再次修改斐波那契示例，使其根据预定义超时来生成序列：

```js
async function* makeFibonacciSequenceGenerator(endIndex = Infinity) {
    let previousNumber = 0;
    let currentNumber = 1;
    
    for (let currentIndex = 0; currentIndex < endIndex; currentIndex++) {
        await new Promise(resolve => setTimeout(resolve, 1000)); // a simple timeout as an example.
        yield currentNumber;
        let nextNumber = currentNumber + previousNumber;
        previousNumber = currentNumber;
        currentNumber = nextNumber;
    }
}

(async () => {
  const fibonacciSequenceGenerator = makeFibonacciSequenceGenerator(6);
  for await (let x of fibonacciSequenceGenerator) {
    console.log(x); // 1, then 1, then 2, then 3, then 5, then 8 (with delay in between).
  }
```

由于生成器是异步的，因此我们可以内部使用 `await`，依赖 `promise` 执行网络请求等异步操作。这里生成器的 `next()` 方法返回一个 `Promsie`。

如果出于某种原因不想使用生成器但又想定义一个可迭代的对象，则必须使用 `Symbol.asyncIterator` 而不是上面的 `Symbol.iterator`。

尽管与迭代器相比，生成器更易于创建和维护，但与普通函数相比，它们会更难调试。在异步上下文中尤其如此。可能有很多原因。一个例子是假如在外部调用 `throw()` 方法时，栈跟踪作用可能非常有限。在这种情况下几乎不可能正常调试，开发者可能需要要求用户提供更多的信息。

## 参考资源
* <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators>
* <https://blog.bitsrc.io/explore-iterators-and-generators-in-javascript-ea4102015377>