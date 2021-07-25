# 正则表达式

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-regular-expressions-regexp-e187e9082913)，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理第二十七章。**

![](./assets/27_cover.png)

搜索，匹配和汇总是我们日常网络活动的重要组成部分。比如当你在浏览网页或 google 某些关键字时，就会进行大量搜寻。为了减少搜索/匹配的难度和提高精确度，诸如 Notepad 或 Sublime 等流行编辑器使用正则表达式来支持搜索和替换功能。因此当使用编辑器时你在在键盘上按 `CTRL + F` 时，就可以搜索和匹配选择的文本。

除了搜索，开发者可以使用正则表达式进行输入验证。例如，可以检查用户输入的 PIN 码是否全为数字，或者输入的密码是否具有特殊字符等。大多数开发者对正则表达式最喜欢的一点是其知识的可传递性。比如用 JavaScript 编写的正则表达式可以很轻松地迁移至 Python。

本文将解释 JavaScript 正则表达式，其重要性及特殊字符，还有如何有效地创建和编写它们，主要用例以及其不同的属性和方法。

## 什么是正则表达式

正则表达式是用于匹配字符串中字符组合的模式。在 JavaScript中，正则表达式也是对象。

正则表达式使得字符串搜索和匹配更轻松快捷。例如在搜索引擎，日志和编辑器等中，就需要轻松高效地过滤/匹配文本。这正是正则表达式的用武之地，它用一系列字符定义了搜索模式。

## 正则表达式的重要性

由于数字化转型的加速，信息成为越来越多的行业不可或缺的一部分。在本节我们将探索为什么正则表达式很重要以及它们在数据管理中的作用。

### 搜索/匹配字符串

大多数正则表达式的使用是用于执行字符串的搜索和匹配。 正则表达式允许开发者在大量文本池搜索。搜索文本时，如果找到匹配文本则返回 `true` ，反之为 `false`。从一组文本中匹配一个文本时，开发者会得到一个包含预期文本（即与搜索模式匹配的文本）的数组。

### 输入验证

对于大多数软件开发者，输入验证是一个重要功能。如果希望用户输入的 PIN 码全为数字，或正确使用 `@xx.com` 后缀输入电子邮件，实现类似功能时大多数开发者会使用到正则表达式。

下面是一个 RegExp 示例，以验证用户的输入确保他们的输入仅包含数字：

```js
let num = 1;
let regex = new RegExp('[0-9]');
console.log(regex.test(num)); // 返回值为 true
```

执行上面的代码会返回 `true`，因为 `num` 是介于0到9之间的数字。但是，如果将 `num` 的值更改为文本，则输出 `false`。


```js
let num = 'me';
let regex = new RegExp('[0-9]');
console.log(regex.test(num)); // 返回值为 false
```

### 网络爬虫

网络爬虫涉及从网站提取数据。使用正则表达式，开发者可以轻松地执行此任务。例如可以通过指向网页并提取与其模式匹配的数据来。

### 数据预处理

从网页抓取上的数据还有其他用处。比如基于决策目的，你可以评估来自网络的数据并整理成所需的格式。借助正则表达式可以聚合和映射数据以将其用于分析目的。

经过预处理的信息可以存储起来以备不时之需，使检索变得更加容易。

## JavaScript 创建 RegExp 对象

JavaScript 正则表达式是使用 RegExp 对象创建的。因此，正则表达式主要是JavaScript对象。前文使我们对正则表达式有了更好的了解，接下来我们一下如何在 JavaScript 中创建它们。

### 字面量 

字面量是在 JavaScript 中创建 RegExp 对象的一种方法。此方法涉及RegExp文字语法的使用。 RegExp 字面量将表达式用斜杠 `/` 包括而不是引号。

因为涉及 JavaScript 字面量的使用，也意味着这是运行时不能更改的固定值，使用字面量创建的正则表达式是一个常量。例如开发者不会想在循环中使用字面量。因为如果每次迭代后没有重新编译，则循环中的由字面量创建的值将不会改变。

以下代码是使用字面量创建 JavaScript 正则表达的语法：

```js
let re = /hello/
```

让我们再看一个简单例子，它会在字符串中寻找完全匹配的字符。这将执行区分大小写的搜索：

```js
let re = "Hello Studytonight";
let result = /hello/.test(re);
console.log(result); // 输出 false
```

上面代码的执行结果是 `false`，因为 `hello` 不等于 `Hello`，这是区分大小写的搜索。上面的命令执行的操作是在文本 `String Hello Studytonight` 中搜索 `hello`。我们可以使用不区分大小写的 `i` 修饰符执行不区分大小写的搜索。让我们使用 `i` 修饰符对上面的示例进行调整:

```js
let re = "Hello Studytonight";
let result = /hello/i.test(re);
console.log(result); // 输出 true
```

这次程序将输出true，因为没有执行区分大小写的搜索，所以 `Hello`等同于 `hello`。

### 构造函数

另一种创建正则表达式的方法是使用构造函数。此方法将正则表达式文本作为函数参数接收。从ECMAScript 6开始，构造函数现在可以接受正则表达式字面量。

需要注意的是使用构造函数创建的正则表达式，它们的模式可以在运行时发生改变。例如，在验证用户输入或执行循环迭代时。以下代码是使用构造函数创建 JavaScript 正则表达式的语法示例：

```js
let re = new RegExp('hello', 'g'); // 基于字符串构建
let re = new RegExp(/hello/, 'g'); // 基于正则表达式字面量构建 (ES6)
```

就像字面量示例一样，我们将使用 RegExp 构造函数创建区分大小写的搜索：

```js
let str = "Hello Studytonight";
let regex = new RegExp('hello');
console.log(regex.test(str)); // 输出 false
```

接下来，我们将 `i` 修饰符添加到函数参数中，以忽略搜索中的区分大小写。

```js
let str = "Hello Studytonight";
let regex = new RegExp('hello', 'i');
console.log(regex.test(str)); // 输出 true
```

现在由于区分大小写的忽略，因此代码将输出 `true`。

## 正则表达式方法

正则表达式有两个主要方法，分别是 `exec()` 和 `test()`。但还有其他用于正则表达式的 String 方法，如 `match()`，`matchAll()`，`replace()`，`replaceAll()`，`search()`和 `split()`。本节中我们将探讨用于 JavaScript 正则表达式的不同方法：

### exec()

此方法执行搜索并返回结果数组或 null，可用于获取字符串中多个匹配项。 比如下面示例展示了 `exec()` 分别使用循环和不使用的代码。

```js
//without iteration
let regex1 = RegExp('fam*', 'g');
let str1 = 'make family everything familiar';

str1 = regex1.exec(str1);
console.log(str1); //output fam, index:5

//with iteration
const regex1 = RegExp('fam*', 'g');
const str1 = 'make family everything familiar';
let array1;

while ((array1 = regex1.exec(str1)) !== null) {
  console.log(`Found ${array1[0]}. Next starts at ${regex1.lastIndex}.`);
  // outputs "Found fam. Next starts at 8."
  // outputs "Found fam. Next starts at 26."
}
```

> 注意在没有迭代的情况下仅获得第一个匹配项的索引。但是通过迭代可以获得所有（多个）匹配项的结果。

### test()

此 RegExp 方法搜索正则表达式和字符串之间的匹配项。如果找到匹配项，则返回 `true` 或 `false`。还可以对此方法使用全局修饰符 `g`。让我们看一个示例，在带和不带全局修饰符 `g` 的字符串中搜索正则表达式。

```js
const str = 'in a space of time spark';
const regex = new RegExp('spa');
const globalRegex = new RegExp('spa', 'g');
console.log(regex.test(str));
// output: true
console.log(globalRegex.test(str));
// output: true
```

在上面示例中，`regex.test(str)` 和 `globalRegex.test(str)` 均输出 `true`，因为可以在文本（`space` 和 `spark`内）找到匹配的 `spa`。

但全局修饰符允许开发者在搜索中进行迭代，以确定目标字符串中存在 `spa` 的次数及不同出现位置的索引。没有全局修饰符的话是无法实现的，因为 `test()`方法将在字符串中运行，从而确定 `spa` 是否存在，而不考虑它是否出现过一次或多次。下面代码对此进行了更好的解释：

```js
const str = 'in a space of time spark';
const regex = new RegExp('spa');
const globalRegex = new RegExp('spa', 'g');
console.log(regex.test(str));
//output: true
console.log(regex.lastIndex);
// doesn’t matter what position. spa is present so it outputs 0
console.log(regex.test(str));
//output: true
console.log(regex.lastIndex);
// doesn’t matter what position. spa is present so it outputs 0
console.log(regex.test(str));
//output: true
console.log(regex.lastIndex);
// doesn’t matter what position. spa is present so it outputs 0
console.log(globalRegex.test(str));
// output: true
console.log(globalRegex.lastIndex);
// output: 8
console.log(globalRegex.test(str));
// output: true
console.log(globalRegex.lastIndex);
// output: 22
console.log(globalRegex.test(str));
// output: false
console.log(globalRegex.lastIndex);
// output: 0 (because spa only occurs in position 8 and 22)
```

`test` 方法的语法为 `test(str)`。`str` 是要匹配正则的字符串。与 `search()` 方法不同，此方法返回一个布尔值而不像 `search()` 返回匹配项的索引；如果找不到匹配项，则返回 -1。

### match()

此方法返回与正则表达式与文本匹配的结果数组而不是布尔值。让我们看一个示例，该示例将使用全局修饰符查找字符串中的所有匹配字母。全局修饰符用于遍历文本每个字母进行匹配。

```js
const paragraph = 'TheGirl Fakesa Smile.';
const regex = /F(a)[a-z]/g;
const found = paragraph.match(regex);
console.log(found); // Should return ["Fak"]
```

> 如果输入正则对象作为参数直接使用 `match()` 方法，则会得到一个空字符串数组。如果在此方法中不使用全局修饰符，则会得到与 `exec()`方法相同的结果。另外此方法还支持使用其他属性，比如[`groups`, `index`, `input`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/match#return_value)等。

### matchAll()

`matchAll()` 方法必须配合全局修饰符来使用。与 `match()` 方法的区别在于它返回具有所有匹配组和捕获组的迭代器。在 `match()` 方法中，使用`g`修饰符不会返回捕获组，没有`g`修饰符的情况下将返回第一个匹配项以及相关捕获组。

全局修饰符对于 `matchAll()` 很重要，否则会报错。让我们看一下用`matchAll()`方法来执行 `match` 方法中的例子。

```js
const paragraph = "TheGirl Fakesa Smile.";
const regex = /F(a)[a-z]/g;
const found = paragraph.matchAll(regex);

//should print (2) ["Fak", "a", index: 8, input: "TheGirl Fakesa Smile.", groups: undefined].
Array.from(found, (res) => console.log(res)); 
```

上面的示例中返回了捕获组 `a`。在 `match()` 示例中则未返回此值。另外二者的语法是相同的。

### replace()

```js
const p = 'The girl is a beautiful girl';
console.log(p.replace('girl', 'lady'));
//output: "The lady is a beautiful girl"
 
const regex = /girl/i;
console.log(p.replace(regex, 'woman'));
// output: "The woman is a beautiful girl"
```

上面例子中可以看到初始字符串 `p` 不变。产生变化的是返回结果。另外，第二个字符串 `girl` 也保持不变。 此方法语法如下所示：

```js
// for RegExp pattern
replace(regexp, newSubstr)
replace(regexp, replacerFunction)
// for String Pattern
replace(substr, newSubstr)
replace(substr, replacerFunction)
```

### replaceAll()

此方法适用于使用正则替换文本中所有匹配文本。在 `replace()` 方法中，仅模式匹配的第一个文本被替换。但在 `replaceAll()` 会将所有匹配文本替换掉。让我们来看一下 `replace()` 方法中的相同示例。 二者的语法相同：

```js
const p = 'The girl is a beautiful girl';
console.log(p.replaceAll('girl', 'lady'));
//output: "The lady is a beautiful lady"
 
const regex = /girl/g;
console.log(p.replaceAll(regex, 'woman')); //output: "The woman is a beautiful woman"
```

### search()

搜索方法 `search()` 用于搜索正则表达式和字符串间的匹配项。此方法不输出布尔值或结果数组。而是输出一个数字，显示第一个匹配项的索引。

例如，下面的示例输出 4，为第一个大写 `S` 的索引。

```js
let str = "the Soup Is Sour"
let re = /[A-Z]/g
console.log(str.search(re)) //This should output 4
```

### split()

> 非正则对象的方法

`split` 方法用于从字符串中提取子字符串。此方法的作用是根据我们的模式将字符串分为子字符串。然后，它将返回一个包含所有子字符串的数组。比如可以使用该方法将字符串分割为单词，字符等。

```js
const str = 'The house is beautiful and spacious';
const words = str.split(' ');
console.log(words[3]);
//output: "beautiful"
 
const chars = str.split('');
console.log(chars[8]);
//output: "e"
 
const strCopy = str.split();
console.log(strCopy);
//output: Array ["The house is beautiful and spacious"]
```

`split` 方法的相关语法如下：

```js
split()
split(separator)
split(separator, limit)
```

分隔符描述每个拆分应该在哪里发生。分隔符可以是字符串或正则表达式。开发者传入参数 `limit`，此参数规定了包含数组中的子字符串数量。例如，如果 `limit` 为 0，则将返回一个空数组 `[]`。

## 正则编写

在 JavaScript 中，可以使用简单的模式、特殊字符和来编写正则表达式。在本节中，我们将探索编写正则表达式的不同方法，同时重点关注简单模式、特殊字符和修饰符。

### 简单模式

有时在搜索文本，会希望获得完全匹配。例如，如果您想在“Blessing made good fries by frying fr yosh's potatoes” 这句话中搜索 “fry”。你不会想要得到诸如 “fr yosh's” 或“fries” 之类的结果，而是像“frying”这样的精确匹配。这就是简单模式的内容，使用简单的模式，您可以创建模式以获得精确匹配。它们大多仅由字符组成。

下面是一个简单的例子。它允许我们创建一个精确匹配字符串“fry”的搜索。

```js
const paragraph = 'Blessing makes good fries by frying fr yosh’s potatoes.';
// characters f-r-y
const regex = /fry/;
console.log(paragraph.search(regex));
// 29
```

### 特殊字符

有时候我们要求搜索不必精确。例如，我们可能想进行范围搜索。您可能想要搜索字母 a — c，无论字符串中是否有空格。为此，开发人员需要使用特殊字符。 JavaScript 中 RegExp 的特殊字符分为以下几类：断言、字符类、分组和范围、量词和 Unicode 属性转义。让我们看看如何在这些类别中使用特殊字符。

#### 断言

RegExp 中的断言表示模式边界。使用断言可以指明单词的开头和结尾。还可以使用以下表达式为匹配编写模式：前瞻、后瞻和条件语句。

对于边界类型断言，您可以使用 `^`、`$`、`\b` 或 `\B` 等字符。

- **^** — 此字符用于匹配输入的开头。如果将 multiline 修饰符设置为 true，则该字符可以在换行符后立即匹配。
- $ - 匹配输入的结尾。如果将 multiline 修饰符设置为 true，则此字符可以在换行符之前立即匹配。
- **\b** — 此字符匹配单词边界。也就是说，在一个单词字符之后或之前没有另一个单词字符的地方。
- **\B** — 此字符匹配非单词边界。也就是说，前一个和下一个字符的类型相同：两者都必须是单词或非单词。例如，字母后不能跟空格。

对于前瞻断言和后瞻断言之类的表达式，请使用以下字符：

- `x(?=y)` — 此字符语法用于前瞻断言。仅当 x 后跟 y 时，语法才会匹配 x。将 x 和 y 替换为您选择的值以执行断言。例如，`/Man(?=Money)/` 仅在后跟 “money” 时才匹配 “man”。
- `x(?!y)` — 用于否定前瞻断言。只有当它后面没有 y 时，它才会匹配 x。例如，`/Man(?=Money)/` 只会在后面没有跟 “money” 的情况下匹配 “man”。
- `(?<=y)x` — 此语法用于后瞻断言。仅当 x 前面有 y 时，它才会匹配 x。例如，`/Man(?=Money)/` 仅当它前面有 “money” 时才会匹配 “man”。
- `(?<!y)x` — 用于否定后瞻断言。只有当它前面没有 y 时，它才会匹配 x。例如，`/Man(?=Money)/` 仅当前面没有 “money” 时才会匹配 “man”。

让我们用讨论过的特殊字符和断言来看看下面的例子。

```js
let str = `let the river dry up`;
 
// 1) Use ^ to fix the matching at the begining of the string, and right after newline.
str = str.replace(/^l/,'h');
console.log(1, str); //This will change the l in "let to h"
// 输出 1 "het the river dry up"
 
//2) Use $ to fix the matching at the end of the string, and right before a newline.
let str1 = `let the river dry up`;
str1 = str1.replace(/p$/,'n');
console.log(2, str1); //This will change the p in "up" to n
// 输出 2 "let the river dry un"
 
//2) Use \b to match a word boundary
let str2 = `let the river dry up`;
str2 = str2.replace(/\bl/,'n');
console.log(3, str2); //This will change the l in "let to n"
// 输出 3 "net the river dry up"
 
//2) Use \B to match a non-word boundary
let str3 = `let the river dry up`;
str3 = str3.replace(/\Bt/,'y');
console.log(4, str3); //This will change the t in "let" to y
// 输出 4 "ley the river dry up"
 
//look ahead assertion
let str4 = "let usmake light"
str4 = str4.replace(/us(?=make)/, 'them');
console.log(str4);
// 输出 let themmake light
 
//negative look ahead assertion
let str5 = "let us make light"
str5 = str5.replace(/us(?!let)/, 'everyone');
console.log(str5);
// 输出 let everyone make light
 
//look behind assertion
let str6 = "letus make light"
str6 = str6.replace(/(?<=let)us/, 'them');
console.log(str6);
// 输出 letthem make light
```

#### 字符类

字符类用于区分不同的字符。比如可以使用字符类区分字母和字母。让我们看看带有字符类的特殊字符以及它们是如何工作的。

- `\d` — 该字符匹配一个数字。即 `0-9` 之间的数字。您可以使用此字符 `/\d/` 或 `/[0–9]/` 来匹配数字。
- `\D` — 用于匹配任何非数字字符。 `/\D/` 等价于 `/[^0-9]/`。
- `\w` — 此字符用于匹配基本拉丁字符中的字母数字。 /\w/ 等价于 [A-Za-z0–9_]。
- `\W` — 用于匹配非字母数字字符。即非基本拉丁字母数字。 /\W/ 等价于 [^A-Za-z0–9_]。
- `\s` — 用于匹配单个空白字符。即空格、制表符、换页、换行和其他 Unicode 空格。 `/\s/` 等价于 `[ \f\n\r\t\v\u00a0\u1680\u2000-\u200a\u2028\u2029\u202f\u205f\u3000\ufeff]`。
- `.` — 点号用于匹配除 `\n`、`\r`、`\u2028` 和 `\u2029` 等换行符外的单个字符。
- `[\b]` — 此字符用于匹配退格。
- `\0` — 这匹配一个空字符。
- `\xhh` — 此语法用于匹配具有两个十六进制数字的字符 (x)。
- `\uhhhh` — 此语法用于匹配带有十六进制数字的 UTF-16 代码单元。
- `\cX` — 这用于匹配使用[脱字符表示法](https://en.wikipedia.org/wiki/Caret_notation)的一个控制字符。

还有其他特殊字符如 `\t`、`\r`、`\n`、`\v`、`\f`，它们分别匹配水平制表符、回车、换行、垂直制表符和换页符。现在让我们看一个简单的例子，展示字符类中这些特殊字符的用法：

```js
const chess = 'She played the Queen in C4 and he moved his King in c2.';
const coordinates = /\w\d/g;
console.log(chess.match(coordinates));
// expected output: Array [ 'c4', 'c2']

const mood = 'happy 🙂, confused 😕, sad 😢';
const emoji = /[\u{1F600}-\u{1F64F}]/gu;
console.log(mood.match(emoji));
// expected output: Array ['🙂', '😕', '😢']
```

#### 分组和范围

如果您想对表达式字符进行分组或指定范围，则可以使用以下特殊字符。

- `x|y` — 此语法用于匹配 x 或 y。例如，表达式 man|woman 将匹配字符串中的 “man” 或 “woman”。
- `[xbz]` — 用于匹配括号内的任何字符。例如，[xbz] 将匹配字符串中的 “x”、 “b”、 “z”。
- `[a-c]` — 用于匹配括号中字符范围内的任何字符。例如，[a-c] 将匹配 “a”、 “b” 和 “c”。但是，如果连字号位于括号的开头或结尾，则将其视为普通字符。因此，[-ac] 将匹配 “non-profit” 中的连字符。
- `[^xyz]` — 这将匹配未在括号中的任何字符。例如，[^xyz] 不会匹配 “Lazy” 中的 “y” 和 “z”，但会匹配 “L” 和 “A”。
- `[^a-c]` — 这将匹配括号中包含的字符范围以外的任何内容。例如，[^a-c] 不会匹配 “bank” 中的 “b” 和 “a”，但会匹配 “n” 和 “k”。
- `(x)` — 此字符用于捕获组。例如，`(x)` 将匹配字符 “x” 并记住匹配的字符以供后续引用使用。例如 `/(family)/` 像在捕获组中一样匹配并记住 “make family familiar” 中的 “family”。因此，如果在 “make family familiar” 中替换 “family”，则文本将更改为在所有代码中替换 “family” 的内容。

- `\n` — 此语法用作最后一个子字符串的反向引用，匹配正则表达式中的组号 “n”，其中 “n” 是正整数。
- `\k<Name>` — 此语法是对最后一个子字符串的反向引用，匹配由 <Name> 指定的命名捕获组。
- `(?<Name>x)` — 此语法用于名称捕获组。它匹配 x 并将其存储在 <Name> 指定的名称下的返回匹配项的组属性中。
- `(?:x)` — 适用于非捕获组。在这种情况下，模式匹配 x 但不记得匹配。因此开发者无法从结果数组中调用匹配的子字符串。

我们已经讨论了用于组和范围的不同特殊字符。现在，让我们看看他们的实际作用。

```js
let str1 = `let the river dry up`;
str1 = str1.replace(/let|the/, 'm');
console.log(1, str1); 
//This will output m the river dry up

let str2 = `let the river dry up`;
str2 = str2.replace(/[abcde]/, 'o');
console.log(2, str2); 
//This will output lot the river dry up

let str3 = `let the river dry up`;
str3 = str3.replace(/[^abcde]/, 'o');
console.log(3, str3); 
//This will output oet the river dry up

let str4 = "Sir, yes Sir in Do you copy? Sir, yes Sir!";
str4 = str4.replace(/(?<title>\w+), yes \k<title>/, "Hello");
console.log(4, str4); 
//This will output Hello in Do you copy? Sir, yes Sir!
```

#### 量词

匹配字符时，有时需要指定要匹配的表达式或字符的数量。量词允许开发者指明想要匹配的表达式或字符的数量。让我们看看正则表达式中用作量词的特殊字符。

- `x*` — 此语法用于匹配前面的项 x 零次或多次。例如，`/bo*/` 匹配 “bird” 中的 “b” 而单词 “goat” 中则没有匹配。
- `x+` — 此语法与前面的项 x 匹配一次或多次。 `/x+/` 等价于 {1,}。
- `X？` — 此语法将匹配前一项 x 零次或一次。
- `x{n}` — 此语法允许精确匹配前项 x 出现 “n” 次，其中 “n” 是正整数。
- `x{n,}` — 您可以匹配前面项 x 的所有出现次数等于或大于 “n”，而不是精确匹配前一项 x 的 “n” 次出现。其中 “n” 是正整数。
- `x{n,m}` - 其中 n 是正整数或零，m 是正整数，并且 m 大于“n”，用于匹配前一项的至少 “n” 和最多 “m” 次出现x 。

默认情况下，像 * 和 + 这样的量词尝试在字符串中尽可能多地匹配。因此，他们被称为贪婪。而 ？字符使量词变为非贪婪模式，因此会在第一次遇到匹配后停止。

让我们看一个如何在正则表达式中使用量词的例子：

```js
let str = `let the river dry up`;
str = str.replace(/et*/,'a');
console.log(1, str); 
// This will change "let" to la"
 
let str1 = `let the river dry up`;
str1 = str1.replace(/e+/,'a');
console.log(2, str1); 
// This will change "let" to lat"
 
let str2 = `let the river dry up`;
str2 = str2.replace(/e?et?/,'a');
console.log(3, str2); 
// This will change "let" to la"
 
let str3 = `let the riveer dry up`;
str3 = str3.replace(/e{2}/,'a');
console.log(4, str3); 
// This will change "riveer" to "rivar"
 
let str4 = `let the riveer dry up`;
str4 = str4.replace(/e{2,}/,'a');
console.log(5, str4); 
// This will change "riveer" to "rivar"
 
let str5 = `let theee riveer dry up`;
str5 = str5.replace(/e{1,3}/,'a');
console.log(6, str5); 
// This will change the "let" to lat"
```

####  Unicode 属性转义

可以根据字符的 Unicode 属性匹配字符。使用 Unicode 属性转义，您可以匹配表情符号、标点符号、来自特定语言的字母或脚本。 Unicode 属性的正则表达式必须具有 u 修饰符。此外，您可以为二进制和非二进制值编写 Unicode 属性。

编写 Unicode 属性转义的语法如下：

```js
//Syntax for non-binary values
\p{UnicodePropertyValue}
\p{UnicodePropertyName=UnicodePropertyValue}
 
// Syntax for binary and non-binary values
\p{UnicodeBinaryPropertyName}
 
// Syntax for negation: \P is negated \p
\P{UnicodePropertyValue}
\P{UnicodeBinaryPropertyName}
```

下面的例子展示了如何在正则表达式中使用 Unicode 属性转义：

```js
const paragraph = 'A ticket to 大阪 costs ¥2000 👌.';
 
const emoji = /\p{Emoji_Presentation}/gu;
console.log(paragraph.match(emoji));
// expected output: Array ["👌"]
 
const nonLatin = /\P{Script_Extensions=Latin}+/gu;
console.log(paragraph.match(nonLatin));
// expected output: Array [" ", " ", " 大阪 ", " ¥2000 👌."]
 
const currencyOrPunctuation = /\p{Sc}|\p{P}/gu;
console.log(paragraph.match(currencyOrPunctuation));
// expected output: Array ["¥", "."]
```

### 修饰符

JavaScript 中的正则表达式有七个修饰符。这些修饰符增强了正则表达式模式。例如， `i` 修饰符用于不区分大小写的搜索。您可以单独或一起使用修饰符，它们可以作为正则表达式的一部分包含在内。让我们看看这些修饰符以及其用途：

- `d` — 此修饰符用于为子字符串匹配生成索引。
- `g` — 用于指示全局搜索。
- `i` — 用于不区分大小写的搜索。如果要在不强制区分大小写的情况下执行搜索，请使用此修饰符。
- `m` — 用于执行多行搜索。
- `u` — 表示 Unicode；它将模式视为一系列 Unicode 代码点。
- `y` — 用于执行粘性搜索。
- `s` - 允许 . 字符匹配换行符。

要在正则表达式中使用修饰符，请使用以下语法 :

```js
//for literal notation
var re = /pattern/flags;
// or
//for constructor function
var re = new RegExp('pattern', 'flags');
```

## 什么时候不该用正则表达式

到目前为止，我们已经探索了正则表达式，它们在 JavaScript 中是如何工作的以及为什么要使用它们。但某些情况下，最好使用其他工具而不是 RegExp。在以下场景中使用正则表达式是一种不好的做法：

1. 使用 RegExp 解析 HTML 不是一个好习惯，因为 HTML 不是常规语言。通常，源代码不是常规语言，不应使用 RegExp 进行解析。

2. 使用比 RegExp 更好的工具或内置 [URL 解析器](https://developer.mozilla.org/en-US/docs/Web/API/URL_API)来解析 URL 的路径和查询参数。因为使用 RegExp 无法获得标记化输出。

3. 尽管开发人员可以使用 RegExp 来查找或验证电子邮件，但尝试这样做时事情会变得非常复杂，[这是一个示例](http://emailregex.com/)。

尽管正则表达式的知识是“可转移的”，但学习并非一蹴而就。如果想更深入地了解 RegExp，本文档是不错的选择。此外，在 JavaScript 中还有不错的工具，例如 [RegExr](https://regexr.com/)、[Regex tester](https://regex101.com/) 和 [Regex Visualizer](https://extendsclass.com/regex-tester.html) 等。

积累了更深入的正则表达式知识后，开发人员可以更好地决定应用的时机。有许多现实生活中的例子表明正则表达式是解决某个问题的最佳方法。这不仅适用于 JavaScript 和前端，同样也适用于后端。