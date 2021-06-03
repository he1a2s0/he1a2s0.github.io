---
layout: post
title: Task之旅 - Part 1：构造函数
date: 2021-06-03
comment: true
tags: [c#, Task, 异步]
categories: [c#, .net]
post_description: Task 类的构造函数，译自：https://blog.stephencleary.com/2014/05/a-tour-of-task-part-1-constructors.html
---

## 太长不看版：不要使用 Task 或 Task<T> 的构造函数
- `new Task/Task<T>(..)` 创建的任务处于 `Created` 状态（任务状态将在 Part 3 中详述），需要手动调度。
- 绝大部分情况下，对数据并行和静态任务并行应使用 `Parallel` 类和 PLINQ；对动态任务并行应使用 `Task.Run` 或 `Task.Factory.StartNew`

----

## 介绍

事实上我很是仔细考虑了一番如何开始这个系列！最终决定从 构造函数 开始，尽管 Task 的构造函数就是***红鲱鱼***（转移焦点的/不相干的事物）。

> 可参考 [ 一条遗臭万年的红鲱鱼 - 知乎](https://zhuanlan.zhihu.com/p/101324869) 和 [短语红鲱鱼（Red herring）是如何发展的？](https://mp.weixin.qq.com/s/F_eRjp8gTq2yzq1HaM-_-w)  

Task 类型有多达8个构造函数：	

```csharp
Task(Action);
Task(Action, CancellationToken);
Task(Action, TaskCreationOptions);
Task(Action<Object>, Object);
Task(Action, CancellationToken, TaskCreationOptions);
Task(Action<Object>, Object, CancellationToken);
Task(Action<Object>, Object, TaskCreationOptions);
Task(Action<Object>, Object, CancellationToken, TaskCreationOptions);
```
BCL（Base Class Library）没有使用 默认参数，因为它们和 [版本控制](http://haacked.com/archive/2010/08/10/versioning-issues-with-optional-arguments.aspx/) 和 反射 不能很好的一起工作。尽管如此，我仍会使用可选参数去重写一些成员来减少我要讲到的重载的数量。

我称这个8个构造函数为“实际成员”，因为它们真实存在。不过它们可以被简化为一个“逻辑成员”：

```csharp
Task(Action action, CancellationToken token = new CancellationToken(), TaskCreationOptions options = TaskCreationOptions.None)
    : this(_ => action(), null, token, options) { }
Task(Action<Object>, Object, CancellationToken = new CancellationToken(), TaskCreationOptions = TaskCreationOptions.None);
```

> 上方两个构造函数只是`Action`有参与无参的区别，且前一个直接调用了后一个，这应该作者列出了两个构造函数却用“only one”来形容的原因。

同样，`Task<T>` 有8个真实的构造函数：

```csharp
Task<TResult>(Func<TResult>);
Task<TResult>(Func<TResult>, CancellationToken);
Task<TResult>(Func<TResult>, TaskCreationOptions);
Task<TResult>(Func<Object, TResult>, Object);
Task<TResult>(Func<TResult>, CancellationToken, TaskCreationOptions);
Task<TResult>(Func<Object, TResult>, Object, CancellationToken);
Task<TResult>(Func<Object, TResult>, Object, TaskCreationOptions);
Task<TResult>(Func<Object, TResult>, Object, CancellationToken, TaskCreationOptions);
```

可以简化为一个逻辑构造函数：

```csharp
Task<TResult>(Func<TResult> action, CancellationToken token = new CancellationToken(), TaskCreationOptions options = TaskCreationOptions.None)
    : base(_ => action(), null, token, options) { }
Task<TResult>(Func<Object, TResult>, Object, CancellationToken, TaskCreationOptions);
```

如此一来，我们有了16个真实构造函数和2个逻辑构造函数。

## 用来做什么？

`Task` 构造函数的使用案例极其少。

记住有两类任务：Promise Task 和 Delegate Task. `Task`构造函数无法创建 Promise Task，只能创建 Delegate Task。

> 任务类型的定义在本系列[开篇](/2021/06/a-tour-of-task-part-0-overview/)

`Task` 构造函数不应与 `async` 一同使用，也只应在极稀有的情况下在并行编程时使用。

并行编程可以分为两类：**数据并行** 和 **任务并行**，大多数并行情况下要求数据并行。任务并行可以进一步分成两类：**静态任务并行**（工作项的数量在并行处理开始时就已知）和**动态任务并行**（工作项的数量在它们被处理时会改变）。Task Parallel Library 中的 `Parallel` 类 和 PLINQ 类型 提供了高层次的构造来处理**数据并行**和**静态任务并行**。你为并行代码创建一个Delegate Task的唯一理由是你正在执行动态任务并行。但是尽管那样，你也几乎从不想要去使用 Task构造函数。Task构造函数会创建一个没有准备好运行的任务；它必须先被调度。这几乎从无必要；在真实情况中，大多数任务应该立即被调度。你想要创建一个任务并且不调度它的唯一理由就是如果你希望由调用者来决定task实际上运行在哪个线程上。而且就算在那样的场景下，我也推荐使用 Func<Task> 代替返回未调度的Task。

> [数据并行（任务并行库）- Microsoft Docs](https://docs.microsoft.com/zh-cn/dotnet/standard/parallel-programming/data-parallelism-task-parallel-library){:target="_blank"} 中数据并行的定义为 “对源集合或数组的元素同时（即，并行）执行相同操作的场景”。
>
> [基于任务的异步编程 - .NET - Microsoft Docs](https://docs.microsoft.com/zh-cn/dotnet/standard/parallel-programming/task-based-asynchronous-programming){:target="_blank"} 中任务并行的定义为“一个或多个独立的任务同时运行”。
>
> > 上述定义为 MSDN 文档中的描述，其语境仅限于 .NET 的并行编程环境。可以简单地认为：数据并行即划分数据后并行处理多份数据；任务并行则是分解任务并行运行多个任务。

让我换一种方式来说明：如果你正执行动态任务并行，需要构造一个可以运行在任意线程上的 `Task`，并将调度的决定留给另一部分代码，而且（无论出于什么原因）不能使用 `Func<Task>` 代替，能且只能使用 `Task` 构造函数。我写过无数的异步和并行应用，但我**从未**遇到过这种情况。

简而言之：**不要使用！**

## 用什么代替？

如果你在写 `async` 代码，最简单的方法是使用 `async` 关键字创建 Promise Task。如果你在包装另一个异步API或事件，使用 `Task.Factory.FromAsync` 或 `TaskCompletionSource<T>`。 如果你需要运行一些 CPU 密集代码并且异步地处理它，使用 `Task.Run`。我们将会在后面的文章中看到所有这些以及更多的选项。

如果你在写并行代码，先尝试使用 Parallel 或 LINQ。如果你确实在做动态任务并行，使用 Task.Run 或 Task.Factory.StartNew。后面的文章中我们也会考虑这些选项。

## 结论

抱歉第一篇博文仅仅归结为“不要使用”，但它就是如此。随后介绍 `Task.Factory.StartNew` 时我会介绍诸如 `CancellationToken` 的所有构造函数参数。

----

原文链接：<a href ="https://blog.stephencleary.com/2014/05/a-tour-of-task-part-1-constructors.html" target="_blank">https://blog.stephencleary.com/2014/05/a-tour-of-task-part-1-constructors.html</a>
