# 函数式编程及与其他范式对比

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-functional-style-and-how-it-compares-to-other-approaches-8a1398c73919)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第二十五章。**

![](./assets/1_LGWkRvJovafFbY-txmy57Q.jpeg)

## 概述

在此之前，已经有程序员使用逻辑式，过程式以及最常见的面向对象范式来编写代码。这些范式涵盖了用于解决计算问题的编码风格，框架，特性和模式，也有助于各种编程语言的发展。

比如过程式语言（COBOL）是基于过程式原理，即每个语句可被解释为子程序或函数。

另一方面，函数式编程是本文的基石，涉及到项目构建，问题解决或使用数学函数来处理计算。我们将使用数据作为“输入”的函数来编写代码，该数据经过函数计算并作为“输出”返回。

函数式编程的美丽之处在于这些函数无法更改我们的数据或输入（数据不可变性）并可以观察到任何副作用。语句以函数来表示：接受输入，产生输出。

本文将讨论更多有关函数式编程的内容，包括它如何在 JavaScript 中工作以及一些重要概念，以便更好理解 JavaScript 中的函数式编程。

## 更多关于函数式编程的内容

- [Why Functional programming matters](https://www.youtube.com/watch?v=XrNdvWqxBvA)
- [An introduction to functional programming in JavaScript](https://opensource.com/article/17/6/functional-javascript)
- [Master the Coding interview: What is functional programming](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-functional-programming-7f218c68b3a0)
