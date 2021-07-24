---
layout: post
title: Task之旅 - Part 9：Delegate Tasks
date: 2021-07-25
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 的等待方法，译自：https://blog.stephencleary.com/2015/03/a-tour-of-task-part-9-delegate-tasks.html
---

## 大长不看版：

基本上都是通过 `TaskFactory.StartNew` 和 `Task.Run` 方法创建 Delegate Task，区别：

- `TaskFactory.StartNew` 不支持异步委托，而 `Task.Run` 支持。
- `TaskFactory.StartNew` 和 `Task.Run` 的 `CancellationToken` 参数仅取消委托的调度而非取消委托本身，用处并不大。
- `TaskFactory.StartNew` 有一个非最优的默认选项 `TaskCreationOptions.None`。`Task.Run` 使用更恰当的默认值 `TaskCreationOptions.DenyChildAttach` 。
- `Task.Factory.StartNew` 有个令人困惑的默认调度器 `TaskScheduler.Current`。`Task.Run` 则总是会使用恰当的默认值 `TaskScheduler.Default`。
- `TaskFactory.StartNew` 不能自动处理异步委托。要在自定义任务调度器上运行异步代码，你需要使用 `Unwrap`。
- 对大多数把工作放入线程池队列的代码来说 `Task.Run` 是最佳的现代方式。

----

上一篇我们看了一些旧的启动 Delegate Tasks 的方法。今天我们将会着眼于一些在更现代的代码中创建 Delegate Tasks 的成员。这些方法不像任务的构造函数，它们返回一个已经运行的（或者至少已经调度运行的） Delegate Task。 

## TaskFactory.StartNew

首先是经常被过度使用的 `TaskFactory.StartNew` 方法。它有一些可用的重载： 

```csharp
Task StartNew(Action);
Task StartNew(Action, CancellationToken);
Task StartNew(Action, TaskCreationOptions);
Task StartNew(Action, CancellationToken, TaskCreationOptions, TaskScheduler);

Task StartNew(Action<object>, object);
Task StartNew(Action<object>, object, CancellationToken);
Task StartNew(Action<object>, object, TaskCreationOptions);
Task StartNew(Action<object>, object, CancellationToken, TaskCreationOptions, TaskScheduler);

Task<TResult> StartNew<TResult>(Func<TResult>);
Task<TResult> StartNew<TResult>(Func<TResult>, CancellationToken);
Task<TResult> StartNew<TResult>(Func<TResult>, TaskCreationOptions);
Task<TResult> StartNew<TResult>(Func<TResult>, CancellationToken, TaskCreationOptions, TaskScheduler);

Task<TResult> StartNew<TResult>(Func<object, TResult>, object);
Task<TResult> StartNew<TResult>(Func<object, TResult>, object, CancellationToken);
Task<TResult> StartNew<TResult>(Func<object, TResult>, object, TaskCreationOptions);
Task<TResult> StartNew<TResult>(Func<object, TResult>, object, CancellationToken, TaskCreationOptions, TaskScheduler);
```

包含一个 `object` 参数的重载就是简单化的传递它的值给延续委托，这只是一个在某些情况下避免额外分配的优化方案，所以我们可以暂时忽略这些重载。剩下两组重载，表现得像两个核心方法的默认参数：

```csharp
Task StartNew(Action, CancellationToken, TaskCreationOptions, TaskScheduler);
Task<TResult> StartNew<TResult>(Func<TResult>, CancellationToken, TaskCreationOptions, TaskScheduler);
```

`StartNew` 可以接收一个没有返回值（`Action`）或者有返回值（`Func<TResult>`）的委托，并基于委托是否有返回值而返回一个恰当的任务类型。注意这两个委托类型都不是异步感知的委托，在开发者尝试使用 `StartNew` 启动异步任务时会让事情变得复杂。

>`TaskFactory.StartNew` 不支持异步感知的委托。而 `Task.Run` 支持。

`StartNew` 重载的默认值来自于 `TaskFactory` 实例。`CancellationToken` 参数默认为 `TaskFactory.CancellationToken`。`TaskCreationOptions` 参数默认为 `TaskFactory.CreationOptions`。`TaskScheduler` 参数默认为 `TaskFactory.Scheduler`。让我们依次来看一下这些参数。

 

## CancellationToken

首先是 `CancellationToken`。这个参数常被误解。我看到过很多（聪明的）开发人员向 `StartNew` 传递一个 `CancellationToken`，坚信这个信息（token）可以用来在执行期间随时取消委托。然而，事情并非如此。传递给 `StartNew` 的 `CancellationToken` 只在委托开始执行前有效。也就是说，它取消委托的启动，而非委托本身。一旦委托开始执行，`CancellationToken` 参数就不能用来取消委托。委托本身必须观察 `CancellationToken`（例如，使用 `CancellationToken.ThrowIfCancellationRequested`）以便支持在它开始执行后取消。

 ![](/images/posts/2021/07/25/i-do-not-think-it-means.jpg)

*你一直在用那个取消令牌。 我不认为它跟你想的是一个意思。* 

尽管如此，如果你确实传了一个 `CancellationToken` 给 `StartNew`，它的行为上是有细微区别的。如果委托自己监视 `CancellationToken`，那么它会引发一个 `OperationCanceledException`。如果 `StartNew` 调用不含 `CancellationToken`，则返回的任务会因该异常而出错。然而，如果委托引发的 `OperationCanceledException` 来自于传给 `StartNew` 的同一个 `CancellationToken`，返回的任务会被取消而非出错，并且 `OperationCanceledException` 也会被 `TaskCanceledException` 取代。 

好吧，有点难以用语言来描述。要是你想看用代码表达的同样的细节，见[这个 gist ](https://gist.github.com/StephenCleary/37d95619f7803f444d3d)里的单元测试。

然而，只要你使用这些常用模式中的一种来检测取消，这种行为上的区别并不影响您的代码。对异步代码来说，你应该对任务使用 `await` 并捕获 `OperationCanceledException`（更多完整示例见[这个 gist ](https://gist.github.com/StephenCleary/dfd2a8b0a50ea3040695)里的单元测试）： 

```csharp
try
{
  // "task" was started by StartNew, and either StartNew or
  // the task delegate observes a cancellation token.
  await task;
}
catch (OperationCanceledException ex)
{
  // ex.CancellationToken contains the cancellation token,
  // if you need it.
}
```

对同步代码，你应该对任务调用 `Wait`（或 `Result`）并期望一个 `InnerException` 是 `OperationCanceledException` 的 `AggregateException`（更多完整示例见[这个 gist ](https://gist.github.com/StephenCleary/6674ae30974f478a4b7f)里的单元测试）

```csharp
try
{
  // "task" was started by StartNew, and either StartNew or
  // the task delegate observes a cancellation token.
  task.Wait();
}
catch (AggregateException exception)
{
  var ex = exception.InnerException as OperationCanceledException;
  if (ex != null)
  {
    // ex.CancellationToken contains the cancellation token,
    // if you need it.
  }
}
```

总之，`StartNew` 的 `CancellationToken` 参数几乎没什么用。它在行为上引入了一些微妙的改变，也令许多开发者困惑。我自己从不使用它。

## TaskCreationOptions

有几个只是传递 `TaskScheduler` 调度任务的“调度选项”。`PreferFairness` 是请求 FIFO 行为的提示。`LongRunning` 是任务会长时间执行的提示。这这篇文章时，任务调度器 `TaskScheduler.Default` 会为带有 `LongRunning` 标志的任务创建一个单独的线程（在线程池之外），然而，这种行为无法保证。注意这个选项都只是提示，`TaskScheduler` 忽略它们是完全正当的。

还有些“调度选项”没有传递给 `TaskScheduler`。`HideScheduler` 选项（.NET 4.5 引入）将使用给定的任务调度器调度延续，但会在延续执行时假装当前是没有任务调度器，这可以用来解决不期望的默认任务调度器（下面会描述）。`RunContinuationsAsynchronously` 选项（.NET 4.6 引入）会强制这个任务的任何延续异步执行。

“父子关系选项”控制任务如何附加到当前正在执行的任务上。附加的子任务改变了它们的父任务的行为，这便于一些动态任务并行的场景，但在这种（极其小范围的）使用案例之外的任何地方它都是意料外的和不合适的。`AttachedToParent` 会附加任务为正在执行的任务的子任务。在现代代码中，你几乎从不会想要使用这个选项，更重要的是，你几乎从不会想让其它代码附加子任务到你的任务上。因此，`DenyChildAttach` 选项在 .NET 4.5 引入，阻止任何其它任务使用 `AttachedToParent`。 

> `Task.Factory.StartNew` 有一个非最优的默认选项 `TaskCreationOptions.None`。`Task.Run` 使用更恰当的默认值 `TaskCreationOptions.DenyChildAttach`。

## TaskScheduler

`TaskScheduler` 用于调度延续。`TaskFactory` 可以定义它自己的默认使用的 `TaskScheduler` 。注意 `Task.Factory` 实例的默认 `TaskScheduler` 不是 `TaskScheduler.Default`，而是 `TaskScheduler.Current`。这个事实多年来已经引起了相当多的困惑，因为绝大多数时间，开发者希望（并且想得到）`TaskScheduler.Default`。我之前详细描述过这个问题，不过稍作回顾并无坏处。

下面的代码先创建一个 UI TaskFactory 来在 UI 线程上调度工作。接着，作为这个工作的一部分，它启动了一些在后台运行的工作。 

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    var ui = new TaskFactory(TaskScheduler.FromCurrentSynchronizationContext());
    ui.StartNew(() =>
    {
        Debug.WriteLine("UI on thread " + Environment.CurrentManagedThreadId);
        Task.Factory.StartNew(() =>
        {
            Debug.WriteLine("Background work on thread " + Environment.CurrentManagedThreadId);
        });
    });
}
```

在我的系统上输出为： 

```
UI on thread 9
Background work on thread 9
```

问题在于当外部的 `StartNew` 运行时，`TaskScheduler.Current` 是 UI 任务调度器。这被内部的 `StartNew` 获得作为 `TaskScheduler` 参数的默认值，导致后台工作在 UI 线程上而非线程池线程上被调度。可以通过传递 `HideScheduler` 给外部的 `StartNew` 任务，或是传递一个明确的 `TaskScheduler.Default` 给内部的 `StartNew` 来避免这种情形。

> `Task.Factory.StartNew` 有个令人困惑的默认调度器 `TaskScheduler.Current`。`Task.Run` 则总是会使用恰当的默认值 `TaskScheduler.Default`。

总之，我完全不推荐使用 `Task.Factory.StartNew`，除非你在做动态任务并行（这种情况极为罕见）。在现代代码中，你应该总是使用 `Task.Run` 代替。如果你确实有一个自定义的 `TaskScheduler`（例如，`ConcurrentExclusiveSchedulerPair` 里的一个调度器），那么创建自己的 `TaskFactory` 实例并且对它使用 `StartNew` 是合适的，然而，应避免使用 `Task.Factory.StartNew` 。

**2015****-03-04** **更新（由** [**Bar Arnon**](https://twitter.com/I3arnon/status/561150440960581637) **建议）：**如果你确实选择使用 `StartNew`（即，如果你需要使用一个自定义 `TaskScheduler`）,记住 `StartNew` 不能自动处理异步委托。要在自定义任务调度器上运行异步代码，你需要使用 `Unwrap`。 

## Task.Run

`Task.Run` 是现代的、首选的将工作加入线程池队列的方法。它不能同自定义调度器一起工作，但提供了一个比 `Task.Factory.StartNew` 更简单的 API，且支持异步启动： 

```csharp
Task Run(Action);
Task Run(Action, CancellationToken);

Task Run(Func<Task>);
Task Run(Func<Task>, CancellationToken);

Task<TResult> Run<TResult>(Func<TResult>);
Task<TResult> Run<TResult>(Func<TResult>, CancellationToken);

Task<TResult> Run<TResult>(Func<Task<TResult>>);
Task<TResult> Run<TResult>(Func<Task<TResult>>, CancellationToken);
```

这里有三个方向的重载：是否有 `CancellationToken`，委托是否返回一个 `TResult` 值以及委托是同步（`Action/Func<TResult>`）还是异步（`Func<Task>/Func<Task<TResult>>`）的。从技术上来讲，`Task.Run` 并不总是会创建一个 Delegate Task；当它接收到一个异步委托时，它实际上会返回一个 Promise Task。但从概念上来讲，`Task.Run` 是专门用来在线程池上执行委托的，所以我和 `StartNew` 一起讲这一系列重载（`StartNew`总是创建 Delegate Tasks）。 

令人悲伤的是 `CancellationToken` 参数有上面对 `StartNew` 描述过的同样的问题。即它事实上只取消委托的调度，而调度几乎在一瞬间发生。`CancellationToken` 参数的存在确实稍稍改变了语义，类似于 `StartNew`。完整的单元测试[在这份 gist 中](https://gist.github.com/StephenCleary/37d95619f7803f444d3d)，它只有一个可能令人惊讶的结果：如果一个异步委托明确地监视 `CancellationToken`，返回的任务会被取消而不是出错。就像 `TaskFactory.StartNew`，如果消费的代码使用检测取消的标准模式，那么这一点点语义上的不同并无影响。 

所以，我的结论是 `Task.Run` 的 `CancellationToken` 参数并没有什么用处。 

然而，其它重载是很有用的，并且对大多数把工作放入线程池队列的代码来说， `Task.Run` 是最佳的现代方式。

----

原文链接：<https://blog.stephencleary.com/2015/03/a-tour-of-task-part-9-delegate-tasks.html>{:target="_blank"}

