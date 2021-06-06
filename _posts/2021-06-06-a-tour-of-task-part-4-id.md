---
layout: post
title: Task之旅 - Part 4：Id
date: 2021-06-06
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 类的 Id 和 CurrentId 属性，译自：https://blog.stephencleary.com/2014/06/a-tour-of-task-part-4-id.html
---

## 大长不看版：

- `Task` 的实例属性 `Id` 在需要时才会分配值，不宜作为 `Task` 的标识，这种情况应使用 `AsyncLocal`。
- `Task` 的静态属性 `CurrentId` 表示当前正在运行的 Delegate Task 的 `Id`。

----

## Id

```csharp
int Id { get; }
```

我之前提过 [task identifiers ](https://blog.stephencleary.com/2013/03/a-few-words-on-taskid-and.html){:target="_blank"}，所以我这里只会介绍重点。

首先，无论文档是如何说的，`Id`标识实际上并非唯一。它非常接近，但实际上并不唯一。标识在需要时生成，绝不会为0。 

任务标识在你 [读取ETW事件](https://msdn.microsoft.com/en-us/library/ee517329.aspx){:target="_blank"} 或 [使用任务窗格调试](https://msdn.microsoft.com/en-us/library/dd998369.aspx){:target="_blank"} 时有用，但它们在诊断和调试之外没有任何用武之地。

有时开发者们会尝试使用任务标识作为集合的键，来为任务关联“额外数据”。这是个错误的方式，通常他们所要找的是 `AsyncLocal`。

## CurrentId

```csharp
static int? CurrentId { get; }
```

`CurrentId` 属性返回当前在执行的任务的标识，或者在没有任务执行时返回 `null` 。这里的关键词是 **执行** -- `CurrentId` 只适用于 Delegate Tasks，不用于 Promise Tasks。

特别地，通过 `async` 方法（异步方法）返回的任务是 Promise Task，它*逻辑*上表示 `async` 方法，但它并非 Delegate Task，且实际上并没有异步代码作为它的委托。 在 `async` 方法内 `CurrentId` 可能是也可能不是 `null`，依赖于底层 `SynchronizationContext` 或 `TaskScheduler` 的实现细节。

> 要看更多信息，包括示例代码，请看原作者的文章 [async 方法中的 CurrentId ](https://blog.stephencleary.com/2013/03/taskcurrentid-in-async-methods.html){:target="_blank"}

在并行代码中，可以使用当前任务标识作为键加入到集合中来存储任务本地值或结果，但在我看来那是个糟糕的方法。通常使用 PLINQ/Parallel 内置的本地值 和 结果聚合支持 要好得多。

----

原文链接：<https://blog.stephencleary.com/2014/06/a-tour-of-task-part-4-id.html>{:target="_blank"}

