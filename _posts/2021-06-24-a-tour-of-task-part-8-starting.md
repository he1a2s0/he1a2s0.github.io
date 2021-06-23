---
layout: post
title: Task之旅 - Part 8：Starting
date: 2021-06-24
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 的等待方法，译自：https://blog.stephencleary.com/2015/02/a-tour-of-task-part-8-starting.html
---

## 大长不看版：

- `Task.Start/RunSynchronously` 只能对 `Created` 状态的任务（即使用构造函数创建的任务）调用。
- 现代代码中不应使用 `Task.Start/RunSynchronously`，确实需要进行类似操作时可以使用 `Task.Run` 代替。

----

使用构造函数创建一个任务后，它最初是 `Created` 状态。这是一种“待机”状态，任务在开始前不会做任何事情。今天的文章探讨启动已创建任务的方法。

这篇文章里没有任何现代代码推荐的东西。如果你在找最佳实践，可以跳过，这儿没什么可看的。

## Start

 最基础的启动一个任务的方法只需调用它的 `Start` 方法。听起来很简单，对吗？ 

```csharp
void Start();
void Start(TaskScheduler);
```

到现在为止你可能猜到了，默认的 `TaskScheduler` 不是 `TaskScheduler.Default`，是 `TaskScheduler.Current`。再说一次，这带来了[与 `Task.Factory.StartNew` 类似的处理 `TaskScheduler` 时遇到的所有相同的问题](https://blog.stephencleary.com/2013/08/startnew-is-dangerous.html){:target="_blank"}。

原先，`Start`（和任务构造函数）为了让开发者可以定义要执行的任务，类似于一个花式委托。然后，其它代码可以按它们觉得适合的任何方式执行这些任务（例如，在UI线程或在后台线程）。但在现实世界中，这几乎没用处，因为委托所做的通常确定了它需要的上下文（例如访问UI元素的委托必须在UI线程上运行）。所以这种分隔对大多数代码并无意义，即使有，开发者只需直接使用委托代替任务。

`Start` 只能在使用构造函数创建的任务上被调用，即它只适用于处于 `Created` 状态的 Delegate Tasks。一旦 `Start` 被调用，任务会转移到 `WaitingToRun` 状态（并且再也不会返回到 `Created` 状态），因此 `Start` 不能在任务上被调用多次。`Start` 绝不能对 Promise Tasks 调用，因为它们从不会处于 `Created` 状态。

> 要了解更多任务状态、Delegate Tasks、Promise Tasks 的更多信息，见 [这个系列的 Part 3 (Status)](/2021/06/a-tour-of-task-part-3-status/){:target="_blank"}。

在现代代码即使是动态任务并行代码中，`Start` 毫无用处。使用 `Task.Run` 创建和调度任务，而非任务构造函数和 `Start`。 

## RunSynchronously

`RunSynchronously` 很像 `Start`，有相同的重载： 

```csharp
void RunSynchronously();
void RunSynchronously(TaskScheduler);
```

`RunSynchronously` 会尝试立即启动任务并在当前线程上执行它。事情并非总是如此，然而，最终的决定依赖于传递给 `RunSynchronously` 的任务调度器。比如，UI线程的任务调度器不会允许任务在线程池线程上运行。如果任务调度器拒绝同步执行任务，那么 `RunSynchronously` 与 `Start` 的行为一致，即任务进入到任务调度器的队列以便将来执行。同样类似于 `Start`，`RunSynchronously` 只能对处于 `Created` 状态的任务调用，且对一个任务只能调用一次。

再说一遍，默认的 `TaskScheduler` 是 `TaskScheduler.Current`。尽管如此，这次这个行为没有意义，因为 `RunSynchronously` 会尝试在当前线程上执行任务委托，可以合理的假设当前任务调度器是正确的那个。

类似于 `Start`，`RunSynchronously` 在现代代码中没有任何用处。 

## IAsyncResult.CompletedSynchronously

`Task` 有一个叫 `CompletedSynchronously` 的显式接口实现的成员：

```csharp
bool IAsyncResult.CompletedSynchronously { get; }
```

如果你相信 MSDN 文档，这个成员应该在任务同步完成时返回 `true`。不幸的是，这个成员总是返回 `false`，即便是像从 `Task.FromResult` 返回的同步完成的 Promise Tasks。

`IAsyncResult.CompletedSynchronously` 被用于一些基于 `IAsyncResult` 的遗留代码。但一般一说，这个成员不应该在现代代码中使用。特别是，对于任务，你不能指望它会是除 `false` 外的任何值。

----

原文链接：<https://blog.stephencleary.com/2015/02/a-tour-of-task-part-8-starting.html>{:target="_blank"}

