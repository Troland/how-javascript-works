# Deno 简介

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-introduction-to-deno-a3b1153b1855)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第二十八章。**

![](./assets/28_cover.jpeg)

Deno 是用于运行 JavaScript 和 Typescript 应用程序的安全运行时。本文中将研究 Deno 的起源，与 Node.js 在探索模块、包、异步支持、Typescript 支持、安全性和工具等各个方面进行比较。最后将深入了解 Deno 及其实现方式。首先，让我们对 Deno 进行点实操。

## 快速的 Deno Demo 

在 mac 上安装 Deno 只需要 `brew install deno`，其他操作系统可以参考[官方安装文档](https://deno.land/#installation)。

Deno 可以运行脚本，也可以是一个“读取-求值-输出”循环（英語：Read-Eval-Print Loop，简称 REPL）。

假如有个 **yeah.ts** 文件，其内容为：

`console.log('Yeah, it works!!!')`

Deno 使用命令 `deno run -q ./yeah.ts` 执行上面的脚本，得到输出 `Yeah, it works!!!`。

同样我们来试一下 REPL：
```bash
$ deno
Deno 1.9.0
exit using ctrl+d or close()
> console.log(‘Yeah, it works!’)
Yeah, it works!
undefined
```

除了 `run` 之外，Deno 还有很多其他的子命令，可以输入 `deno -h` 查看它们。

我们尝试了一些代码并输入了一些命令，现在来谈谈 Deno 的起源。

## Deno 起源

在 JSConf EU 2018，Ryan Dahl（Node.js 创造者）在其演讲 “我对 Node.js 感到遗憾的 10 件事” 中宣布了 Deno。这非常酷，创造者可以看到他的先前的项目受欢迎程度飙升，观察它被使用并从错误中吸取教训。典型的场景是创作者尝试改进和重构他们的原始项目。然而，Ryan Dahl 想要实现一些根本性的更改，而在 Node.js 中无法实现这些更改，因为它会破坏兼容性。

彻底改写显然是一个巨大的风险，但 Deno 跨越了鸿沟，势头强劲。

要知道 Deno 还很年轻；第一次提交是在 2018 年 5 月 13 日，而 1.0 版则是在两年后的 2020 年 5 月 13 日发布。

接下来让我们对比以下 Deno 与 Node.js 。

## Deno vs Node.js

以下是 Node.js 和 Deno 之间的主要区别。稍后我们也会讨论内部结构，但在这里先重点关注功能和开发者体验的差异。

### 内置包管理 vs npm

Node.js 依赖 npm 进行包管理。而 Deno 遵循 Go 和 Rust 的示例，可以通过 URL 从任何地方导入包。

### ES 模块 vs CommonJS 模块

Node 使用 CommonJS 标准:

`const module = require(‘module-name’)`

而 Deno 使用标准 ES 模块:

`import module from ‘https://some-repo/module-name.ts'`

> 请注意，Deno 模块导入需要包括文件扩展名的完整模块名。

### 基于权限的访问与完全开放访问

Node 允许你完全访问环境、文件系统和网络。这是一个严重的安全漏洞。曾经有几次攻击就是恶意 npm 模块利用了这点并获得对这些资源的访问权限。

Deno 需要明确的权限，因此在一定程度上它限制了对图谋不轨者所能造成的破坏。

### 内置 typescript 编译器 vs 需要外部的 typescript 支持

Node 不直接支持 Typescript，开发者需要使用一个重型且持续变化的工具链，包括打包器、转译器等。

Deno 直接支持 Typescript，简化了开发者不少的工作。

### Promises vs callbacks

Node 使用非阻塞 I/O 并且需要回调以在 I/O 操作完成时得到通知。

相反，Deno 使用现代 async/await 范式，它隐藏了回调链的复杂性，并使代码更清晰、更易于梳理。

> 当然现在 node 也可以轻松支持 async/await 语法。

### 死于错误 vs 未捕获的异常

在 Node 中，可以为所有未捕获的异常设置一个全局处理程序：

```js
process.on('uncaughtException', function (err) {
  console.log('忽略...');
})
```

在 Deno 中，如果不进行异常捕获，程序就会被杀掉。这是一个大胆的设计决策。

接下来我们仔细看看 Deno 的主要特性。

## Deno 的主要特性

### 模块和包管理

使用 Deno 可以通过 URL 导入模块。这意味着不再需要 `package.json` 和庞大的 `node_modules`。也就是说会有一个缓存，开发者只需下载一次包和模块。下面来一个例子：

```js
import { assertEquals } from "https://deno.land/std@0.93.0/testing/asserts.ts";
assertEquals(2 + 2, 5);
console.log('success!')
```

如果你的数学不是体育老师教的，会注意到断言是错误的（2 + 2 实际上是 4）。让我们看看 Deno 是否也这么认为：

```bash
deno run -q ./import.ts
error: Uncaught AssertionError: Values are not equal:


    [Diff] Actual / Expected

-   4
+   5

  throw new AssertionError(message);
        ^
    at assertEquals (https://deno.land/std@0.93.0/testing/asserts.ts:219:9)
    at file:///Users/chkaos/Code/demo/deno-demo/import.ts:2:1
```

修改一下，看看成功后的输出：

```js
import { assertEquals } from "https://deno.land/std@0.93.0/testing/asserts.ts";
assertEquals(2 + 2, 4);
console.log('success!')
// success
```

回到包和导入。让我们看看这一行：

`import { assertEquals } from “https://deno.land/std@0.93.0/testing/asserts.ts";`

这里 Deno 直接从 URL 导入 `assertEquals` symbol（Deno 标准库中的一个函数）。请注意，该 URL 包含了版本信息 (std@0.93.0)，因此很容易支持同一包的多个版本。

Deno 在 https://deno.land 中维护了一系列精选包，但开发者可以从任何 URL 导入包。

为了证明这一点，我们将 import 语句替换为以下等效的 import 语句，该语句使用托管 Deno 标准库的 Github URL 进行导入：

`import { assertEquals } from “https://raw.githubusercontent.com/denoland/deno_std/main/testing/asserts.ts”;`

> 但直接引用任意 URL 也不推荐，笔者自己尝试了但因为各种网络原因也没成功。

### 异步支持

Deno 从其异步 API 返回 Promise。这意味着开发者可以运行异步操作并等待结果，而无需编写出回调地狱。

这是一个疯狂的例子：

```js
const promise = Deno.run({cmd: ['deno', 'eval', 'console.log(2+3)']})
await promise.status()

/*
Output:
5
*/
```

这里发生了什么？首先使用 `Deno.run()` API 来启动一个子进程。启动的子进程是 Deno 的另一个实例求表达式 `console.log(2+3)` 的值，最后向控制台打印 5。

这种情况下子进程几乎立即返回，但对于长时间运行的进程，我们希望等待完成而不阻塞问题的其余部分，这就是 await `promise.status()` 的原因。

### Deno 与 Typescript

Deno 内置了对 Typescript 的支持。这意味着与浏览器中运行的 Node 或 Web 应用程序不同，不需要重量级和非标准的工具链。 Deno 包含一个 Typescript 编译器会将 Typescript 代码转换为 JavaScript 并稍后在 [V8 运行时](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)执行。

Deno 缓存转译的 Typescript 模块，所以只要 Typescript 文件没改变就不会被再次转译。

要查看缓存位置，请检查 `Emitted modules cache`：

```bash
deno info

DENO_DIR location: "/Users/chkaos/Library/Caches/deno"
Remote modules cache: "/Users/chkaos/Library/Caches/deno/deps"
Emitted modules cache: "/Users/chkaos/Library/Caches/deno/gen"
Language server registries cache: "/Users/chkaos/Library/Caches/deno/registries"
Origin storage: "/Users/chkaos/Library/Caches/deno/location_data"
```

### 安全性

创建 Deno 的初衷之一是安全性。 用户可以控制每个程序的访问级别。

默认情况下，网络、环境和文件系统等资源是不可访问的。例如运行一个尝试写入文件的程序，看看会得到什么结果。

将以下代码段保存到名为 write_file.ts 的文件中。

`Deno.writeTextFileSync('data.txt', 'some data')`

```bash
deno run write_file.ts

error: Uncaught PermissionDenied: Requires write access to "data.txt", run again with the --allow-write flag
Deno.writeTextFileSync('data.txt', 'some data')
     ^
    at deno:core/01_core.js:106:46
    at unwrapOpResult (deno:core/01_core.js:126:13)
    at Object.opSync (deno:core/01_core.js:140:12)
    at openSync (deno:runtime/js/40_files.js:37:22)
    at writeFileSync (deno:runtime/js/40_write_file.js:27:18)
    at Object.writeTextFileSync (deno:runtime/js/40_write_file.js:85:12)
    at file:///Users/chkaos/Code/demo/deno-demo/write_file.ts:1:6
```

正如预期那样收到了一个权限错误，一条提示告诉开发者需要添加什么标识符。让我们使用适当的权限标识符再次运行程序：

```bash
$ deno run — allow-write write_file.ts
Check file:///Users/gigi.sayfan/git/deno_test/write_file.ts
$ cat data.txt
some data
```

> 请注意 Deno REPL 拥有所有权限。因此如果在交互式 Deno 会话中运行不受信任的代码，请务必小心。

## 工具

Deno 非常重视开发者体验。它提供了许多开箱即用的工具。让我们快速回顾一下 Deno 提供的各种工具。

### 格式化

格式化并非必须，但有更好。但是，随着时间的推移，开发者社区里大少精力都浪费到空白和参数括号放置位置等无休止等争论上。

Deno 从 Go 和 Rust 吸取经验并提供了一个 `deno fmt` 命令。让我们看看官方 deno 格式是什么样的。

考虑以下文件 `fmt-test.ts`：

```ts
function foo()
{
    console.log('foo here')
  const x    = 3
         console.log('x + 2 =', x+2)
}

foo()
```

这是一个有效的 Typescript 程序。但是它的格式不是很好。 通过 `deno fmt` 运行它：

```bash
$ cat fmt_test.ts | deno fmt -
```

结果如下：

```ts
function foo() {
  console.log("foo here");
  const x = 3;
  console.log("x + 2 =", x + 2);
}

foo();
```

Deno fmt 将函数的左大括号与函数声明放在同一行，用两个空格缩进所有内容，将单引号转换为双引号，在 “+” 运算符周围放置空格，并在每行末尾添加分号。

### 测试

测试是编程的重要组成部分。 Deno 不仅依赖社区来提供测试框架，而且还配备了自己的断言模块，开发者可以使用它来编写测试。

在 `test-test.ts` 文件中定义了一个名为 `is_palindrome()` 的函数用于检查字符串是否为回文（忽略空格），然后进行一些测试，前两个测试应该通过，第三个应该失败：

```ts
import { assert } from "https://deno.land/std@0.95.0/testing/asserts.ts";

function is_palindrome(s: string) {
  const ss = s.replaceAll(' ', '')
  const a = ss.split('')
  return a.reverse().join('') == ss
} 

await Deno.test("Palindrome 1 - success", () => {
  assert(is_palindrome("tattarrattat"));
})

await Deno.test("Palindrome 2 - success", () => {
  assert(is_palindrome("never odd or even"));
})

await Deno.test("Palindrome 3 - fail", () => {
  assert(is_palindrome("this is not a palindrom"), "fail!")
})
```

使用命令 `deno test` 会得到以下结果：

```bash
$ deno test test_test.ts
Check file:///Users/gigi.sayfan/git/deno_test/$deno$test.ts
running 3 tests
test Palindrome 1 — success … ok (1ms)
test Palindrome 2 — success … ok (1ms)
test Palindrome 3 — fail … FAILED (2ms)
failures:
Palindrome 3 — fail
AssertionError: fail!
at assert (https://deno.land/std@0.95.0/testing/asserts.ts:178:11)
at file:///Users/gigi.sayfan/git/deno_test/test_test.ts:19:3
at asyncOpSanitizer (deno:runtime/js/40_testing.js:37:15)
at resourceSanitizer (deno:runtime/js/40_testing.js:73:13)
at Object.exitSanitizer [as fn] (deno:runtime/js/40_testing.js:100:15)
at TestRunner.[Symbol.asyncIterator] (deno:runtime/js/40_testing.js:272:24)
at AsyncGenerator.next (<anonymous>)
at Object.runTests (deno:runtime/js/40_testing.js:347:22)
at async file:///Users/gigi.sayfan/git/deno_test/$deno$test.ts:3:1
failures:
Palindrome 3 — fail
test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out (4ms)
```

### 打包

Bundling 使开发者可以将程序的所有模块和依赖项打包到一个单一的，可立即执行的包中。 

Deno 提供了 bundle 命令。让我们来实践以下。 `foob​​ar.ts` 模块从 `foo.ts` 导入 `foo()` 函数，从 `bar.ts` 导入 `bar()` 函数。

1. 这是 **foo.ts** 。

```ts
export function foo() {
    console.log('foo')
}
```

2. 这是 **bar.ts** ，https://gist.github.com/the-gigi/2d48187fcc1e11ad6c27299487b1a9e0

3. 这是 **foobar.ts** ：

```ts
import { foo } from "./foo.ts"
import { bar } from "./bar.ts"

foo()
bar()
```

```bash
$ deno bundle foobar.ts
Bundle file:///Users/gigi.sayfan/git/deno_test/foobar.ts
Check file:///Users/gigi.sayfan/git/deno_test/foobar.ts
```

最后打包称一个文件，结果如下：

```js
function foo() {
    console.log('foo');
}
function bar() {
    console.log('bar');
}
foo();
bar();
```

如上面所见文件里不再需要导入语句，因为 `foo()` 和 `bar()` 都包含在单个包文件中，可以直接调用。

### 调试

Deno 允许您通过 [V8 检查器协议](https://v8.dev/docs/inspector)调试程序。您可以使用 Chrome DevTools 并使用 **--inspect** 或 **--inspect-brk** 标识符运行您的程序。我个人更喜欢 JetBrains IDE 和 Deno 插件，这种方式提供了成熟的断点、监视和堆栈跟踪来调试 Deno 代码。

如果选择了 VS Code，那么推荐对应的一个 Deno 插件。同时可以使用以下 **launch.json** 配置文件手动附加调试器：

### 脚本安装

大部分情况下，使用命令 `deno run` 运行 deno 程序是没问题的，但是如果需要传递大量权限标识符并且希望能够从任何位置运行程序，那么有更好的选择。 

Deno 提供了 `deno install` 命令，该命令创建一个小 shell 脚本来调用您的 Deno 程序并将其放置在指定的位置或 **$HOME/.deno/bin**。

让我们来安装 foobar 程序：

```bash
$ deno install foobar.ts
✅ Successfully installed foobar
/Users/gigi.sayfan/.deno/bin/foobar
ℹ️ Add /Users/gigi.sayfan/.deno/bin to PATH
export PATH=”/Users/gigi.sayfan/.deno/bin:$PATH”
```

将 **$HOME/.deno/bin** 添加到我的 PATH 中，现在可以通过键入 **foobar** 从任何地方运行 **foobar**：

```bash
$ cd /tmp
$ foobar
foo
bar
```


## Deno 内部结构

我们审查了 Deno 的特性和用户体验。让我们来看看引擎盖下。 Deno 是使用 Rust 和 TypeScript 实现的。以下是 Deno 的主要组件：

- [deno](https://lib.rs/crates/deno) ：开发者与之交互的 deno 可执行文件。
- [deno_core](https://lib.rs/crates/deno_core)：负责 Javascript 执行运行时。 Deno 核心依赖 [Tokio 包](https://lib.rs/crates/tokio)来实现其异步事件循环。
- [tsc](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#tsc-the-typescript-compiler)：标准的 TypeScript 编译器。曾经是 Deno 的 TypeScript 编译器。现在，它主要负责类型检查。
- [swc](https://swc.rs/)：代表 Speedy Web Compiler，在为 Javascript 和 Typescript 代码编译为可以在任何浏览器上执行的 Javascript 方面承担着越来越多的工作。
- [rusty_v8](https://lib.rs/crates/rusty_v8)：这个包 crate 提供 Rust 到 V8 C++ API 的绑定。

## 总结

Deno 是一个充满活力的年轻项目，建立于 Node.js 的经验和教训之上。与 Node.js 相比有很多技术改进。它是使用现代技术堆栈实现的。最大的问题是它是否会成为主流 Javascript 和 Typescript 后端运行时。

现在说还为时过早，但后续关注应该会很有趣。如果开发者有一些计划使用 Node.js 实现的服务器端项目，可以考虑尝试使用 Deno。

在 Node.js、Deno 或其他一些技术之间进行选择应该主要取决于您的项目需求和您团队的专业知识。

如果已经使用 Node.js 构建了一些运行良好的东西，那么单纯为了将它用 Deno 重写而没有其他任何好处可能不是最佳决策。