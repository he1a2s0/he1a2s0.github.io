---
layout: post
title: Task之旅 - Part 2：AsyncState 和 CreationOptions 属性
date: 2021-06-04
comment: true
tags: [c#, Task, 异步]
categories: [.net, Task之旅系列]
post_description: Task 类的 AsyncState 和 CreationOptions 属性，译自：https://blog.stephencleary.com/2014/05/a-tour-of-task-part-2-asyncstate-and-creationoptions.html
---

## 太长不看版：

- `Task` 的 `AsyncState` 和 `CreationOptions` 在 `async` 场景下没有用处。

----

我跳过了上周的文章（因为我忙于网站的重新设计），所以今天买1赠1！😄。

## AsyncState

```csharp
object AsyncState { get; } // 实现了 IAsyncResult.AsyncState
```

`Task` 的`AsyncState` 属性实现了 `IAsyncResult.AsyncState`。这个成员以前很有用，但在现代应用中不怎么有用了。	

在异步编程经历它尴尬的青涩时期时，`AsyncState` 是 `Asynchronous Programming Model(APM)` 的一个重要部分。`Begin*` 方法会接受一个 `state` 参数，这个参数会被赋给 `IAsyncResult.AsyncState`。然后，当应用程序代码的回调被调用时，它能够通过访问 `AsyncState` 的值来确定哪一个异步操作完成了。

>`IAsyncResult.AsyncState`（和其它状态类的参数）现在已经不需要了；`lambda` 回调能够很容易的以类型安全的方式捕获任意数量的本地变量。我喜欢 lambda 方式，相比单个对象的 `state` 参数来说，它花费更多、也不那么脆弱而且更加灵活。然而，`state`参数的方式避免了内存分配，所以有时还是会被用在性能敏感的代码中。

在现代的代码中，`Task.AsyncState` 成员主要用于 `Task` 到 `APM` 的互操作。只有在你编写必须存在于一个旧的异步框架里面的 `async/await` 代码时才需要它（现在已非常罕见）。在那种情形下，你实现 `Begin*/End*` 方法并且使用一个 `Task` 实例作为 `IAsyncResult` 的实现。标准的方法是使用 `TaskCompletionSource<T>` 创建一个 `Task<T>`，并把 `state` 参数传递给 `TaskCompletionSource<T>` 的构造函数：

```csharp
public static IAsyncResult BeginOperation(AsyncCallback callback, object state)
{
    var tcs = new TaskCompletionSource<TResult>(state);
    ... // 开始执行操作，然后在操作完毕时完成 "tcs"。（指的是调用 tcs.SetResult 或其它类似方法）
    return tcs.Task;
}
```

现代代码中并不真的需要读取 `AsyncState`，它之所以重要是因为它实现了 `IAsyncResult.AsyncState`。

## CreationOptions

```csharp
TaskCreationOptions CreationOptions { get; }
```

`CreationOptions` 仅允许你读取用来创建这个任务的创建选项。你可以在使用`Task`构造函数、`Task.Factory.StartNew` 或 `TaskCompletionSource<T>` 创建任务时指定这些选项。我会在后面讲到 `Task.Factory.StartNew` 时讲这些选项的含义。

尽管如此，一旦任务被创建，几乎没有任何理由再去读取任务的创建选项。只有在你用父/子任务或任务调度做一些有趣的工作时才有需要 -- 与异步或并行任务的正常场景毫不相关。


## 结论

再说一次，这两个成员在真实世界的代码中没有用处。

----

原文链接：<a href ="https://blog.stephencleary.com/2014/05/a-tour-of-task-part-2-asyncstate-and-creationoptions.html" target="_blank">https://blog.stephencleary.com/2014/05/a-tour-of-task-part-2-asyncstate-and-creationoptions.html</a>
