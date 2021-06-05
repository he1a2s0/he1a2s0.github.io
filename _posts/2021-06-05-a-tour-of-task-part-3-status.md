---
layout: post
title: Task之旅 - Part 3：Status
date: 2021-06-05
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 类的 Status 属性和状态转换，译自：https://blog.stephencleary.com/2014/06/a-tour-of-task-part-3-status.html
---

## 要点：

- Delegate Task 的三种初始状态及状态转换
- Promise Task 的状态转换
- 几乎不会直接使用 Status，但应了解任务的状态和转换逻辑

----

## Status

```csharp
TaskStatus Status { get; }
```

若你将 Task 视为一个状态机，则 Status 属性就表示当前的状态。 不同类型的任务用不同的路径通过状态机。

> 像往常一样，这篇文章只是取 [Stephen Toub 所述](https://devblogs.microsoft.com/pfxteam/the-meaning-of-taskstatus/){:target="_blank"} 内容，加了一点详细描述，并且画了一些丑图。 :)	

## Delegate Tasks

Delegate Tasks 遵循下图中的基本模式：

![Delegate Tasks](/images/posts/2021/06/05/TaskStates.Delegate.png)

一般来说，Delegate Tasks 通过 `Task.Run` (或 `Task.Factory.StartNew`）创建，并且以 `WaitingToRun` 状态进入状态机。`WaitingToRun` 意味着任务已被关联到一个任务调度器，只等着轮到它运行。

- 在 `WaitingToRun` 状态进入（状态机）是 Delegate Tasks 的常规路径，但也有一些其它的可能性。

- 如果 Delegate Tasks 是从 `Task` 构造函数 启动的，那么它开始会处在 `Created` 状态，只有当你通过 `Start` 或 `RunSynchronously` 将它分配给一个任务调度器时，才会转移到 `WaitingToRun` 状态。

- 如果 Delegate Tasks 是另一个任务的延续，那么它开始会处于 `WaitingForActivation` 状态，并且当另外那个任务完成时它会自动转移到 `WaitingToRun` 状态。

当 Delegate Task 的委托实际执行时，任务会处于 `Running` 状态。当委托执行完毕时，任务前进到 `WaitingForChildrenToComplete` 状态，直到它的子任务全部完成。最终，任务以三种状态之一结束：`RanToCompletion`(成功), `Faulted` 或 `Canceled`。

记住，因为 Delegate Tasks表示运行中的代码，很可能你看不到这些状态中的一个或多个。例如：有可能将一些非常快速的工作放入线程池的队列，在（调用流程）返回到你的代码之前任务就已经完成了。

## Promise Tasks

Promise Tasks 的状态机简单得多：

![Promise Tasks](/images/posts/2021/06/05/TaskStates.Promise.png)

> 图有点简化，严格来说，Promise Tasks能进入到 `WaitingForChildrenToComplete` 状态。然而，这相当反直觉，因此为 `async` 而创建的任务通常指定 `DenyChildAttach` 标志。

将基于I/O的操作说成运行或执行是很正常的，比如，“HTTP下载当前正在运行”。然而，（这个操作）没有实际的CPU代码在运行，所以 Promise Tasks（比如 Http 下载任务） 永远不会进入到 **WaitingToRun** 或是 **Running** 状态。是的，这意味着 Promise Task 可以不实际运行而以 **RanToCompletion** 状态 结束。好吧，它就是这样的。。

所有 Promise Tasks 创建后是 “热状态”的，就是说操作已经在进行。令人困惑的部分是，“正在进行”状态实际上叫做 **WaitingForActivation**.

> 为此，我尝试在谈及 Promise Tasks 时避免使用术语 "运行" 或 "执行"；我反而更原意说”这个操作正在进行中“。

## Status 属性

Task 有几个方便的属性用于决定 task 的最终状态。

```csharp
bool IsCompleted { get; }
bool IsCanceled { get; }
bool IsFaulted { get; }
```

`IsCanceled` 和 `IsFaulted` 直接映射到 `Canceled` 和 `Faulted` 状态，但 `IsCompleted` 就有点麻烦了。 `IsCompleted` 不映射到 `RanToCompletion`，而是任务在任何最终状态时它都为`true`。也就是说：

| Status          | IsCompleted | IsCanceled | IsFaulted |
| --------------- | ----------- | ---------- | --------- |
| other           | ❌           | ❌          | ❌         |
| RanToCompletion | ✔️           | ❌          | ❌         |
| Canceled        | ✔️           | ✔️          | ❌         |
| Faulted         | ✔️           | ❌          | ✔️         |

## 结论

同所有这些状态属性一样有趣的是：几乎不会实际用到它们（除了调试）。异步和并行代码一般都不使用 `Status` 或那3个便捷的属性，相反，通常的用法是等待任务完成然后提取结果。

----

原文链接：<a href ="https://blog.stephencleary.com/2014/06/a-tour-of-task-part-3-status.html" target="_blank">https://blog.stephencleary.com/2014/06/a-tour-of-task-part-3-status.html</a>

