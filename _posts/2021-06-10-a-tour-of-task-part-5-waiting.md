---
layout: post
title: Task之旅 - Part 5：Waiting
date: 2021-06-10
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 的等待方法，译自：https://blog.stephencleary.com/2014/10/a-tour-of-task-part-5-wait.html
---

## 大长不看版：

- `Wait/WaitAll/WaitAny` 会阻塞调用线程直到单个或多个任务完成。
- 在同步代码中偶尔会用到 `Wait/WaitAll/WaitAny` 方法， 异步代码中对应的是 `await Task/await Task.WhenAll/await Task.WhenAny`。

----

今天，我们将看一下代码在任务上阻塞的各种方式。所有这些选项都会阻塞调用线程直到任务完成，所以它们几乎从不与 Promise Task 一起使用。注意在 Promise Task 上阻塞是很常见的死锁原因（见： [Don\'t Block on Async Code](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html){:target="_blank"}），阻塞几乎是专门和 Delegate Tasks （比如 从 `Task.Run` 返回的任务）一起使用的。

## Wait

`Wait` 有5个重载：

```csharp
void Wait();
void Wait(CancellationToken);
bool Wait(int);
bool Wait(TimeSpan);
bool Wait(int, CancellationToken);
```

它们可以很好地简化为一个单独的逻辑方法：

```csharp
void Wait() { Wait(-1); }
void Wait(CancellationToken token) { Wait(-1, token); }
bool Wait(int timeout) { return Wait(timeout, CancellationToken.None); }
bool Wait(TimeSpan timeout) { return Wait(timeout.TotalMilliseconds); }
bool Wait(int, CancellationToken);  //最终都是调用这个方法
```

`Wait` 相当简单：它阻塞调用线程直到任务完成、发生超时或等待被取消。

- 如果等待被取消，那么 `Wait` 会引发一个 `OperationCanceledException`。
- 如果发生超时，`Wait` 会返回 `false`。
- 如果任务以 失败 或 取消 状态完成，`Wait` 会将任何（引发的）异常包装进一个 `AggregateException`。注意一个被取消的任务会引发一个包装在 `AggregateException` 内的 `OperationCanceledException`，而一个取消的 *等待* 则会引发一个解开了的 `OperationCanceledException`。

`Task.Wait` 偶尔有用 -- 如果它是在正确的上下文中完成的话。比如，控制台应用的 `Main` 方法如果有异步工作要做，但希望主线程同步阻塞直到工作完成，则可以使用Wait。然而，大多数时候 `Task.Wait` 十分危险，因为它有[潜在的死锁的可能性](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html){:target="_blank"}。

> 对于异步代码，使用 `await` 代替 `Task.Wait`。

## WaitAll

`WaitAll` 的重载与 `Wait` 的重载非常相似：

```csharp
static void WaitAll(params Task[]);
static void WaitAll(Task[], CancellationToken);
static bool WaitAll(Task[], int);
static bool WaitAll(Task[], TimeSpan);
static bool WaitAll(Task[], int, CancellationToken);
```

它们同样可以很好地简化为一个单独的逻辑方法：

```csharp
static void WaitAll(params Task[] tasks) { WaitAll(tasks, -1); }
static void WaitAll(Task[] tasks, CancellationToken token) { WaitAll(tasks, -1, token); }
static bool WaitAll(Task[] tasks, int timeout) { return WaitAll(tasks, timeout, CancellationToken.None); }
static bool WaitAll(Task[] tasks, TimeSpan timeout) { return WaitAll(tasks, timeout.TotalMilliseconds); }
static bool WaitAll(Task[], int, CancellationToken);  //最终都是调用这个方法
```

这些重载实际上与 `Task.Wait` 完全一致，只不过它们是等待多个任务全部完成。与 `Task.Wait` 类似，`Task.WaitAll` 如果等待被取消的话会抛出 `OperationCanceledException`，如果任意一个任务失败或被取消的话会抛出 `AggregateException`。如果发生超时 `WaitAll` 会返回 `false`。

`Task.WaitAll` 应该很少使用。在使用 Delegate Tasks 时它偶尔有用，但这种用法也很少见。编写并行代码的开发者应首先尝试数据并行，而且即使必须用到任务并行，父/子任务也会比用 `Task.WaitAll` 定义特别的依赖关系产生更整洁的代码。

> 注意 `Task.WaitAll` （用于同步代码）罕见，但 `Task.WhenAll`（用于异步代码）常见。

## WaitAny

`Task.WaitAny` 与 `WaitAll` 相似，除了它只等待第一个完成的任务（并且返回那个任务的索引），同样我们得到了相似的重载：

```csharp
static int WaitAny(params Task[]);
static int WaitAny(Task[], CancellationToken);
static int WaitAny(Task[], int);
static int WaitAny(Task[], TimeSpan);
static int WaitAny(Task[], int, CancellationToken);
```

简化为单个逻辑方法：

```csharp
static int WaitAny(params Task[] tasks) { return WaitAny(tasks, -1); }
static int WaitAny(Task[] tasks, CancellationToken token) { return WaitAny(tasks, -1, token); }
static int WaitAny(Task[] tasks, int timeout) { return WaitAny(tasks, timeout, CancellationToken.None); }
static int WaitAny(Task[] tasks, TimeSpan timeout) { return WaitAny(tasks, timeout.TotalMilliseconds); }
static int WaitAny(Task[], int, CancellationToken);  //最终都是调用这个方法
```

`WaitAny` 的语义与 `WaitAll` 和 `Wait` 有一些不同：`WaitAny` 仅仅等待第一个任务完成。它不会在 `AggregateException` 中传播那个任务的异常。反而，任何任务的失败都需要在 `WaitAny` 返回后进行检查。`WaitAny` 在超时时返回 -1，而如果等待被取消则会抛出 `OperationCanceledException`。

如果说 `Task.WaitAll` 很少使用，则 `Task.WaitAny` 根本不应该被使用。

## AsyncWaitHandle

事实上 `Task` 类型实现了 `IAsyncResult` 接口， 便于与（不幸被命名的）异步编程模型(APM) 进行互操作。这意味着 `Task` 有一个等待句柄作为它的属性：

```csharp
WaitHandle IAsyncResult.AsyncWaitHandle { get; }
```

注意这个成员是显式实现的，所以调用代码在读取它之前必须将 `Task` 转成 `IAsyncResult`。实际上底层的等待句柄是延迟分配的。

使用 `AsyncWaitHandle` 的代码应该是非常非常罕见的。当你有一大堆围绕 `WaitHandle` 构建的现有代码时它才有意义。如果你确实要读取 `AsyncWaitHandle` 属性，认真考虑[销毁任务实例](https://devblogs.microsoft.com/pfxteam/do-i-need-to-dispose-of-tasks/){:target="_blank"}。

## 结论

在一些极端情况下单个 Task.Wait 可能有用，但一般来说，代码不应该在任务上同步阻塞。

----

原文链接：<https://blog.stephencleary.com/2014/10/a-tour-of-task-part-5-wait.html>{:target="_blank"}

