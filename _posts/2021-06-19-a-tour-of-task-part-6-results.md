---
layout: post
title: Task之旅 - Part 6：Results
date: 2021-06-19
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 的等待方法，译自：https://blog.stephencleary.com/2014/12/a-tour-of-task-part-6-results.html
---

## 大长不看版：

- `Task<T>` 有 `Result` 属性而 `Task` 没有；`Task<T>.Result` 与 `Wait`类似，会阻塞调用线程直到任务完成。
- `Task/Task<T>` 的 `Exception` 属性不阻塞，但在任务抛出异常前访问它只会返回 `null`。
- `GetAwaiter().GetResult()` 像 `Wait/Result` 一样会同步阻塞，不同的是有异常时会抛出原始异常而非包装在 `AggregateException` 中。
- 绝大多数情况下应首选使用 `await` 。

----

这篇文章讨论的任务成员与从任务中检索结果有关。一旦任务完成，调用代码必定会取到任务的结果。即使任务没有结果，它对调用代码检验任务是否有错误以便知道任务是否成功地完成或失败是很重要的。

## Result

`Result` 成员只存在于 `Task<T>` 类型，在 `Task` 类型（表示没有结果值的任务）上不存在。

```csharp
T Result { get; }
```

与 `Wait` 类似，`Result` 会同步阻塞调用线程直到任务完成。这通常不是个好主意，与 `Wait` 不是一个好主意的原因相同：容易造成死锁。

此外，`Result` 会将任务异常包裹在一个 `AggregateException` 内。通常只会使错误处理变得复杂。

## Exception

说到异常，有个特定成员专门用来从任务获取异常：

```csharp
AggregateException Exception { get; }
```

不像 `Result` 和 `Wait`，`Exception` 不会一直阻塞到任务完成。如果 `Exception` 在任务还在进行中时被调用，它只会返回 `null`。如果任务成功完成或被取消，那么 `Exception` 仍会返回 `null`。如果任务失败，`Exception` 会返回包装在 `AggregateException` 中的任务异常。同样，这通常只会使错误处理变得复杂。

## GetAwaiter().GetResult()

`GetAwaiter()` 成员在 .NET 4.5 被加到 `Task` 和 `Task<T>` 中。在 .NET 4.0 中可以通过使用 Nuget 包 `Microsoft.Bcl.Async` 让它作为一个扩展方法使用。正常情况下，`GetAwaiter` 方法仅仅被 `await` 使用，但你可能会自己调用它：

```csharp
Task<T> task = ...;
T result = task.GetAwaiter().GetResult();
```

上述代码会同步阻塞直到任务完成。因此，它也受与 `Wait` 和 `Result` 一样的 老式死锁问题 的影响。然而，它不会将任务异常包裹进一个`AggregateException`。

上述代码会从 `Task<T>` 取得结果值。同样的代码模式也能被应用到（无返回值的）`Task` 上，这时 "`GetResult`" 实际上意味着“检查任务是否有错误”：

```csharp
Task task = ...;
task.GetAwaiter().GetResult();
```

一般而言，我会尽量避免在异步任务上同步阻塞。然而，在部分情形下我确实会违反这个指导原则。在这些罕见的情况下，我更喜欢的方法是 `GetAwaiter().GetResult()`，因为它保留了任务异常，而非将它们包裹到 `AggregateException` 中。

## await

当然，`await` 不是 `Task` 类型的一个成员。但是，我认为有必要提醒现在的读者从 Promise Task 获取结果最好的方式是仅使用 `await`。`await` 以最友好的方式获取任务结果；`await` 会异步等待（不阻塞）; `await` 会为（任何）成功的任务返回结果；`await` 会为失败的任务（重新）抛出异常而不包装在 `AggregateException` 内。

简而言之，`await` 应当是你获取任务结果的首选项。绝大多数时候 `await` 应该用来取代 `Wait`, `Result`, `Exception` 或是 `GetAwaiter().GetResult()`。

----

原文链接：<https://blog.stephencleary.com/2014/12/a-tour-of-task-part-6-results.html>{:target="_blank"}

