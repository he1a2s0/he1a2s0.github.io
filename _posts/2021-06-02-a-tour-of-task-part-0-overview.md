---
layout: post
title: Task之旅：part 0
date: 2021-06-02
comment: true
tags: [c#, Task, 异步]
categories: [c#, .net]
post_description: Task 类的历史和分类，译自：https://blog.stephencleary.com/2014/04/a-tour-of-task-part-0-overview.html
---

## 太长不看版

- TPL Task 和 async Task 使用同一个Task类型，但意义和用法不同（正常来说，async Task应该区别于 TPL Task，可以叫 Future；而 TPL Task 的部分成员并不用于 async）
- 作者将Task分两类：Delegate 和 Promise，基本上分别对应 TPL Task 和 async Task
	- Delegate Task：有代码要运行
	- Promise Task：表示某类事件或信号

----

## 一点Task的历史

对学习 async 的开发者来说，一个最大的阻碍其实是 Task 类型本身。大部分开发者是这两类人之一：

- 从 Task 和 TPL 被引入到 .NET 4.0 之后就使用过它的开发者。这些开发者熟悉 Task 以及 [它是如何在并行处理中使用的](https://msdn.microsoft.com/en-us/library/ff963553.aspx){:target="_blank"}。这些开发者面临的危险是 **(TPL 使用的）Task 几乎完全不同于（async 使用的）Task**
- 在 async 出来前从未听过 Task 的开发者。对他们来说，Task 只是 async 的一部分 -- 另一个需要学习的（相当复杂的）东西。“Continuation”是一个外来词. 这些开发者面临的危险是 **假设 Task 的每个成员都是适用于 async 编程的，而实际上并非如此**。

	Continuation 借用自旧法语，来源于拉丁文 continuātiō.

微软的 async 小组确实考虑编写他们自己的 "**Future**" 类型来表示异步操作，但 Task 类型太有诱惑力了。 事实上，甚至在 .NET 4.0 Task 就支持 promise 风格的 futures（有点尴尬），要让它完全支持 async 也只需要少量的扩展。而且，通过合并 "Future" 和 已有的 Task 类型，我们达成了很好的统一：在后台线程上开始一些操作并异步地处理它是非常容易的。无需从 Task 转换为 Future.

使用同一类型的缺点是它确实造成了一些开发者的困惑。像上面说的那样，过去使用过 Task 的开发人员倾向于尝试用同样的方式在 async 环境中使用它（这是错误的）；过去未使用过 Task 的则会面临令人迷惑的对 Task 成员的选择，而它们几乎全部都不应在 async 环境下使用。

这就是今天我们要讲的内容。这个博客系列会浏览 Task 各种各样的成员（是的，所有），并解释每个成员背后的目的。我们会看到，绝大多数 Task 成员在 async 代码中没有立足之地。

## 两类 Task

有两种类型的 Task。第一种是 Delegate Task（委托型Task），是有代码要运行的任务。第二种是 Promise Task（承诺型Task），是一种表示某些类型的事件或信号的任务。 Promise Tasks 通常是基于 I/O 的信号（比如，“HTTP下载已完成”），但事实上它们可以表示任何东西（比如，“10秒的计时器已到期”）

	1.Promise 与 ES 中 Promise 含义类似，翻译时应当作特有名词不翻译，这里为了与前面委托型Task对应，将其译为“承诺”	
	2.原文的评论中有人提到 Task 对应 Future，TaskCompletionSource 对应 Promise（即表示第二种类型的 Task 命名不妥），作者回复表示同意，但也说明这两个术语是来源于官方 Task 源码，Promise Task 意指 “来源于 Promise 的 Task”

TPL中，大部分任务是 Delegate Tasks（带有对 Promise Task 的一些支持）。当代码进行并行处理时，各种 Delegate Task 被拆分到不同的线程，然后实际上由这些线程来执行这些 Delegate Tasks 中的代码。在 async 环境中，大部分 task 是 Promise Tasks（带有对 Delegate Task 的部分支持）。 代码在 Promise Task 上执行 await 时，[没有绑定的线程](https://blog.stephencleary.com/2013/11/there-is-no-thread.html){:target="_blank"}来等待那个任务完成。

以前，我曾使用“基于代码的Task”和“基于事件的Task”两个术语来描述这两种类型的任务。在这个系列中，我将尝试使用 "Delegate Task" 和 "Promise Task" 来区分它们。


----

原文链接：<a href ="https://blog.stephencleary.com/2014/04/a-tour-of-task-part-0-overview.html" target="_blank">https://blog.stephencleary.com/2014/04/a-tour-of-task-part-0-overview.html</>
