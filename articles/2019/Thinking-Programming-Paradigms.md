﻿# 对编程范式的简单思考

> 2019/1/16
>
> 编程最重要的事情，其实是让写出来的符号，能够简单地对实际或者想象出来的“世界”进行建模。一个程序员最重要的能力，是直觉地看见符号和现实物体之间的对应关系。 —— 王垠

## 为什么要写这篇文章 TL;DR

一位朋友曾经曾经和我说：“千万别和不知道回调函数的人，解释什么是回调函数”。（见 [如何浅显的解释回调函数](../2017/Callback-Explained.md)）我本以为 **回调函数** _(callback function)_ 是一个非常简单的概念，但和许多刚入门编程的人解释这个概念的时候，他们都觉得很 **费解**。

直到现在，我才发现，原来要理解回调函数，就需要先接受 [函数是一等公民 _(first-class function)_](https://en.wikipedia.org/wiki/First-class_function) 的事实（函数和数据一样都可以被存储、传递），然后理解 [高阶函数 _(higher-order function)_](https://en.wikipedia.org/wiki/Higher-order_function) 的概念（函数可以作为参数传递到另一个函数里）。

这对于没有接触过 **函数式** 的人来说，简直是 **世界观的颠覆**：

- **面向过程** 里，数据是数据、操作是操作
- **面向对象** 里，操作必须放到对象里，操作属于对象

为了批判 **面向对象** 里 “操作必须放到对象里” 的回调思想，写了一篇文章 [回调 vs 接口](../2017/Callback-vs-Interface.md)（后来读到 陈硕 也有一篇类似的文章 [以boost::function和boost:bind取代虚函数](https://blog.csdn.net/Solstice/article/details/3066268)），但境界还不够，一直没有发现这个问题的 **本质** —— **函数式 vs 面向对象**。

最近终于读懂了几篇 [王垠的博客](http://www.yinwang.org/)，大概能理解了文章的思想（虽然比较偏激，但论述非常严谨）：

- [解密“设计模式”](http://www.yinwang.org/blog-cn/2013/03/07/design-patterns)：批判（面向对象）设计模式（[备份](Thinking-Programming-Paradigms/解密设计模式.html)）
- [Purely functional languages and monads](https://yinwang0.wordpress.com/2013/11/16/pure-fp-and-monads/)：批判“纯”函数式（[备份](Thinking-Programming-Paradigms/Purely-functional-languages-and-monads_Surely-I-Am-Joking.html)）
- [编程的宗派](http://www.yinwang.org/blog-cn/2015/04/03/paradigms)：批判“纯”面向对象、“纯”函数式 和 说“各有各的好处”的“好好先生”（[备份](Thinking-Programming-Paradigms/编程的宗派.html)）

本文总结一下自己的一些体会。

## 三种范式

本文主要分析我所接触过的三种 [编程范式 _(programming paradigm)_](https://en.wikipedia.org/wiki/Programming_paradigm)：

- [面向过程 _(procedural programming)_](https://en.wikipedia.org/wiki/Procedural_programming)
- [面向对象 _(object-oriented programming)_](https://en.wikipedia.org/wiki/Object-oriented_programming)
- [函数式 _(functional programming)_](https://en.wikipedia.org/wiki/Functional_programming)

| 范式 | 数据 | 计算 | 计算数据 |
|-|---|---|---|
| 面向过程 | 数据结构 | 函数 | 用数据调用函数 |
| 面向对象 | 对象的字段 | 对象的操作 | 调用对象的操作，派发消息 |
| 函数式 | 数据结构/闭包 | 函数 | 用数据/函数调用函数 |

### 面向过程

> Algorithms + Data Structures = Programs（算法 + 数据结构 = 程序）—— Niklaus Wirth

数据和计算是两个不同的角色：

- 数据 **存储** 了程序运行的 **状态**
- 计算通过修改数据，**变换** 运行的 **状态**

> 参考：[图灵机 _(Turing machine)_](https://en.wikipedia.org/wiki/Turing_machine)

### 面向对象

- [封装 _(encapsulation)_](https://en.wikipedia.org/wiki/Encapsulation_%28computer_programming%29)：将数据和计算放到一起，并引入访问控制
- [继承 _(inheritance)_](https://en.wikipedia.org/wiki/Inheritance_%28object-oriented_programming%29)：共享数据和计算，避免冗余
- [多态 _(polymorphism)_](https://en.wikipedia.org/wiki/Polymorphism_%28computer_science%29)：用同一个消息（调用），实现不同的操作

> 参考：[浅谈面向对象编程](../2018/Object-Oriented-Programming.md)

### 函数式

由于数据是 **有状态的** _(stateful)_，而计算是 **无状态的** _(stateless)_；所以需要将数据 **绑定** _(bind)_ 到函数上，得到“有状态”的函数，即 [闭包 _(closure)_](https://en.wikipedia.org/wiki/Closure_%28computer_programming%29)。通过传递闭包，实现更复杂的功能的组合。

> 参考：[λ 演算 _(lambda calculus)_](https://en.wikipedia.org/wiki/Lambda_calculus)

## 例子：运行时动态选择计算操作

### 面向过程

### 面向对象

### 函数式

## 写在最后

如果有什么问题，**欢迎交流**。😄

Delivered under MIT License &copy; 2019, BOT Man