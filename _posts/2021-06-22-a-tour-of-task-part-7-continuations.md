---
layout: post
title: Task之旅 - Part 7：Continuations
date: 2021-06-22
comment: true
tags: [c#, Task, 异步, Task之旅系列]
categories: [.net, Task之旅系列]
post_description: Task 的等待方法，译自：https://blog.stephencleary.com/2015/01/a-tour-of-task-part-7-continuations.html
---

## 大长不看版：

- `Continuations`（延续）不阻塞线程，`async/await` 系统在处理任务时使用延续。
- 附加延续到任务的方法是 `ContinueWith` ，`ContinueWith` 也返回一个任务，表示延续本身（即延续本身也是一个任务，延续可以有自己的延续）。`ContinueWith` 方法的可选参数中，`CancellationToken` 取消延续的调度而非延续本身，已调用的延续不会被 `CancellationToken` 取消；`TaskContinuationOptions` 枚举中的条件相关的选项大致等同于在延续内检查任务状态；`TaskScheduler` 参数的默认值不是 `TaskScheduler.Default` 而是 `TaskScheduler.Current`，作者建议总是显式指定这个参数。
- 只要可能，应总是使用 `await` 代替 `ContinueWith`；总是使用 `await Task.WhenAny/WhenAll` 代替 `TaskFactory.ContinueWhenAny/ContinueWhenAll`

----

最近的几篇考量了几个等待任务完成的成员（`Wait`, `WaitAll`, `WaitAny`, `Result` 和 `GetAwaiter().GetResult()`）。所有这些成员的一个共同缺陷就是等待任务完成时会同步阻塞调用线程。 

今天这篇讨论*延续*（*continuations*）。延续是一个委托，你可以附加它到一个任务并告诉这个任务“当你完成时执行这个”。 当任务完成时，它将调度它的延续。延续附加到的任务被称为“前置”任务。 

延续很重要，因为它们不阻塞任何线程。线程只会为任务附加延续以便每当任务完成时运行，而非（同步）等待任务完成。这是异步的本质，`async/await` 系统在处理任务时使用延续。

## ContinueWith

最底层的方式是使用 `ContinueWith` 方法附加延续到一个任务。这个方法有相当多的重载，但总体思路是将一个委托作为一个延续附加到任务：

```csharp
Task ContinueWith(Action<Task>);
Task ContinueWith(Action<Task>, CancellationToken);
Task ContinueWith(Action<Task>, TaskContinuationOptions);
Task ContinueWith(Action<Task>, TaskScheduler);
Task ContinueWith(Action<Task>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task ContinueWith(Action<Task, object>, object);
Task ContinueWith(Action<Task, object>, object, CancellationToken);
Task ContinueWith(Action<Task, object>, object, TaskContinuationOptions);
Task ContinueWith(Action<Task, object>, object, TaskScheduler);
Task ContinueWith(Action<Task, object>, object, CancellationToken, TaskContinuationOptions, TaskScheduler);

Task<TResult> ContinueWith<TResult>(Func<Task, TResult>);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, CancellationToken);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, TaskContinuationOptions);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, TaskScheduler);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task<TResult> ContinueWith<TResult>(Func<Task, object, TResult>, object);
Task<TResult> ContinueWith<TResult>(Func<Task, object, TResult>, object, CancellationToken);
Task<TResult> ContinueWith<TResult>(Func<Task, object, TResult>, object, TaskContinuationOptions);
Task<TResult> ContinueWith<TResult>(Func<Task, object, TResult>, object, TaskScheduler);
Task<TResult> ContinueWith<TResult>(Func<Task, object, TResult>, object, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

噢，好多重载！让我们分析一下。首先，包含一个 `object` 参数的重载只是将该值传递给给延续委托，这只是某些情况下避免额外分配的一个优化方案，所以我们可以暂时忽略这些重载：

```csharp
Task ContinueWith(Action<Task>);
Task ContinueWith(Action<Task>, CancellationToken);
Task ContinueWith(Action<Task>, TaskContinuationOptions);
Task ContinueWith(Action<Task>, TaskScheduler);
Task ContinueWith(Action<Task>, CancellationToken, TaskContinuationOptions, TaskScheduler);

Task<TResult> ContinueWith<TResult>(Func<Task, TResult>);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, CancellationToken);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, TaskContinuationOptions);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, TaskScheduler);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

还是有三个可选参数：一个 `CancellationToken` (默认为 `CancellationToken.None`)，一系列 `TaskContinuationOptions` (默认为 `TaskContinuationOptions.None`) 和一个 `TaskScheduler` (默认为 `TaskScheduler.Current`)。所以这个重载列表能够进一步简化为：

```csharp
Task ContinueWith(Action<Task>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task<TResult> ContinueWith<TResult>(Func<Task, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

`Task<T>` 类型有它自己的匹配重载集。我就不赘述细节了 -- 另外20个方法签名，可以通过同样的方式简化为：

```csharp
Task ContinueWith(Action<Task<TResult>>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task<TContinuationResult> ContinueWith<TContinuationResult>(Func<Task<TResult>, TContinuationResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

这时候，应该很清楚了，有两种主要类型的延续委托可以传递给 `ContinueWith`: 一个有结果值（`Func<…>`）另一个没有（`Action<…>`）。延续委托总是接收一个任务作为参数。这个任务就是延续附加到的任务，所以如果你们要调用 `task.ContinueWith(t => …)`，那么 `task` 和 `t` 引用同一个前置任务实例。 

`ContinueWith` 也返回一个任务。这个任务表示延续本身。因此，每个延续自身是一个任务，并且可以有它自己的延续，依此类推。 

我们再来谈谈可选参数。 

首先是 `CancellationToken`。如果在延续被调度之前取消令牌（即`token`，`CancellationToken`），那么延续委托永远不会真的运行 -- 它被取消了。然而，注意一旦延续已经开始，令牌就不会取消它。换句话来说，`CancellationToken` 取消延续的调度，而非延续本身。因此，我认为 `CancellationToken` 参数是一种误导，我自己从不使用它。 

下一个参数是 `TaskContinuationOptions`，用于延续的一组选项。大多数选项要么与延续的条件/调度有关，要么与延续的父子关系有关。`None` 选项意味着使用默认行为，然而在现代应用程序中，这些默认行为只适用于动态任务并行，极其罕见。 

“条件选项”只在前置任务以匹配的状态完成时才会去调度延续。`OnlyOnRanToCompletion`, `OnlyOnFaulted` 和 `OnlyOnCanceled` 只会在前置任务以特定状态完成时才会调度延续。`NotOnRanToCompletion`, `NotOnFaulted` 和 `NotOnCanceled` 只会在前置任务以另外的状态完成时才会调度延续。所有这些“条件选项”大致与在延续内检查任务状态是一致的。

**2015-01-30 更新（由 [Bar Arnon](https://twitter.com/I3arnon/status/561150440960581637){:target="_blank"} 建议）**：如果先行任务满足了条件选项（比如任务以 `RanToCompletion` 状态完成而延续指定了 `OnlyOnRanToCompletion` 选项），延续会被正常调度。然而，如果条件选项不满足（比如任务以 `Faulted` 状态完成而延续指定了 `OnlyRanToCompletion` 选项），延续会被取消。延续委托永远不会执行且延续任务立即转移到取消状态。 

一些传递给 `TaskScheduler` 的“调度选项”负责调度延续。`PreferFairness` 是一个要求 FIFO 行为的提示。`LongRunning` 是延续将会长时间执行的提示。`ExecuteSynchronously` 是延续将会在完成前置任务的同一个线程被调度的请求。注意所有这些都只是提示，`TaskScheduler` 忽略它们是完全正当的，特别地，`ExecuteSynchronously` 不保证延续会同步执行。 

> 写这篇文章时，在 .NET 4.6 预览版中有另一个选项 `RunContinuationsAsynchronously` , 看起来是强制延续异步执行。当前没有方法绝对地强制延续同步或异步，强制异步延续在有些情形上确实是有用的。 

**2015-02-02 更新：**.NET小组[发表了一篇文章](https://devblogs.microsoft.com/pfxteam/new-task-apis-in-net-4-6/){:target="_blank"}描述 `RunContinuationAsynchronously` 选项。顾名思意，它事实上确实是异步运行延续的。 

还有些“调度选项”没有传递给 `TaskScheduler`。`HideScheduler` 选项（.NET 4.5 引入）将使用给定的任务调度器调度延续，但会在延续执行时假装当前是没有任务调度器，这可以用来解决不期望的默认任务调度器（下面会描述）。`LazyCancellation`（.NET 4.5 引入）是一个确保延续只在前置任务完成后完成（取消的）的选项。不使用 `LazyCancellation` 的话，如果传递给 `ContinueWith` 的取消令牌被取消，它甚至可以在原任务未完成前取消延续。 

“父子关系选项”控制延续任务如何附加到前置任务。附加的子任务改变了它们的父任务的行为，这便于一些动态任务并行的场景，但在这种（极其小范围的）使用案例之外的任何地方它都是意料外的和不合适的。`AttachedToParent` 会附加延续为前置任务的子任务。在现代代码中，你几乎从不会想要使用这个选项，更重要的是，你几乎从不会想让其它代码附加子任务到你的任务上。因此，`DenyChildAttach` 选项在 .NET 4.5 引入，阻止任何延续使用 `AttachedToParent`。 

最后一个可选参数是一个用来调度延续 `TaskScheduler`。不幸的是，这个参数的默认值不是 `TaskScheduler.Default`，而是 `TaskScheduler.Current`。这个事实多年来已经引起了相当多的困惑，因为绝大多数时间，开发者希望（并且想得到）`TaskScheduler.Default`。`Task.Factory.StartNew` 也有我[早前描述过的类似的问题](https://blog.stephencleary.com/2013/08/startnew-is-dangerous.html){:target="_blank"}。既然默认值是不期望的（而且几乎总是不合需要的），我推荐你总是传一个 `TaskScheduler` 值给 `ContinueWith`。很多公司已经碰到过这个问题并且在他们的代码库中执行类型的规则。 

总而言之，我完全不推荐使用 `ContinueWith`，除非你在做动态任务并行（极为罕见）。在现代代码中，你应该几乎总是使用 `await` 代替 `ContinueWith`。`await` 有几个好处。 

一个好处是与其它异步代码一起工作。像上面提到的，`ContinueWith` 只能使用有限数量的委托，这些委托没有一个是 [异步感知的委托](https://blog.stephencleary.com/2014/02/synchronous-and-asynchronous-delegate.html){:target="_blank"}。处理异步延续时，`ContinueWith` 会将它们看作同步的。这在使用这些延续的延续时会引起一些形式的混淆。这也意味着调度选项（比如 `LongRunning`）并不像大多数开发者期望的那样工作，它们只应用于异步委托开头的同步部分。相反的，`await` 使异步延续可以自然的工作。 

另一个好处是更好的默认任务调度器。使用 `ContinueWith` 的代码应该总是显式指定一个任务调度器来减少混淆，便 `await` 有[更合理的默认行为]([much more reasonable default behavior](https://blog.stephencleary.com/2012/02/async-and-await.html){:target="_blank"}。现代代码中几乎从不使用任务调度器，它要么使用 `SynchronizationContext.Current` 要么使用线程池调度器。 

最后一个好处是 `await` 默认使用最合适的选项。当你 `await` 一个未完成的任务时，在底层 `await` 确实使用 `ContinueWith` 来为你调度延续。尽管如此，它会自动使用合适的选项（`DenyChildAttach` 和 `ExecuteSynchronously`），而且不允许你指定不能正确工作的选项（如 `AttachedToParent` 或 `LongRunning`）。

简而言之，优先使用 `await` 而非 `ContinueWith`。`ContinueWith` 在做动态任务并行时有用，但在其它所有场景下，`await`是首选。

## TaskFactory.ContinueWhenAny

`ContinueWhenAny` 是在一系列任务的任何一个完成时执行单个延续的方法。所以，它是一种附加单个延续到多个任务的方法，并且只在第一个任务完成时运行延续。

`TaskFactory` 类型有一系列的 `ContinueWhenAny` 重载，与 `ContinueWith` 有点类似：

```csharp
Task ContinueWhenAny(Task[], Action<Task>);
Task ContinueWhenAny(Task[], Action<Task>, CancellationToken);
Task ContinueWhenAny(Task[], Action<Task>, TaskContinuationOptions);
Task ContinueWhenAny(Task[], Action<Task>, CancellationToken, TaskContinuationOptions, TaskScheduler);

Task ContinueWhenAny<TAntecedentResult>(Task<TAntecedentResult>[], Action<Task<TAntecedentResult>>);
Task ContinueWhenAny<TAntecedentResult>(Task<TAntecedentResult>[], Action<Task<TAntecedentResult>>, CancellationToken);
Task ContinueWhenAny<TAntecedentResult>(Task<TAntecedentResult>[], Action<Task<TAntecedentResult>>, TaskContinuationOptions);
Task ContinueWhenAny<TAntecedentResult>(Task<TAntecedentResult>[], Action<Task<TAntecedentResult>>, CancellationToken, TaskContinuationOptions, TaskScheduler);

Task<TResult> ContinueWhenAny<TResult>(Task[], Func<Task, TResult>);
Task<TResult> ContinueWhenAny<TResult>(Task[], Func<Task, TResult>, CancellationToken);
Task<TResult> ContinueWhenAny<TResult>(Task[], Func<Task, TResult>, TaskContinuationOptions);
Task<TResult> ContinueWhenAny<TResult>(Task[], Func<Task, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);

Task<TResult> ContinueWhenAny<TAntecedentResult, TResult>(Task<TAntecedentResult>[], Func<Task<TAntecedentResult>, TResult>);
Task<TResult> ContinueWhenAny<TAntecedentResult, TResult>(Task<TAntecedentResult>[], Func<Task<TAntecedentResult>, TResult>, CancellationToken);
Task<TResult> ContinueWhenAny<TAntecedentResult, TResult>(Task<TAntecedentResult>[], Func<Task<TAntecedentResult>, TResult>, TaskContinuationOptions);
Task<TResult> ContinueWhenAny<TAntecedentResult, TResult>(Task<TAntecedentResult>[], Func<Task<TAntecedentResult>, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

4个重载的每一组都可以简化为一个集中的方法：

```csharp
Task ContinueWhenAny(Task[], Action<Task>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task ContinueWhenAny<TAntecedentResult>(Task<TAntecedentResult>[], Action<Task<TAntecedentResult>>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task<TResult> ContinueWhenAny<TResult>(Task[], Func<Task, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
Task<TResult> ContinueWhenAny<TAntecedentResult, TResult>(Task<TAntecedentResult>[], Func<Task<TAntecedentResult>, TResult>, CancellationToken, TaskContinuationOptions, TaskScheduler);
```

带有 `TAntecedentResult` 泛型参数的重载用于当前置任务都有同样的结果类型时。有 `TResult` 的重载用于当延续会返回它自己的结果时。`TaskFactory<TResult>` 类型只有支持返回结果的延续的重载，所以它只有 `TaskFactory` 的一半重载。 

默认参数值与 `ContinueWith` 类似，除了它们是由 `TaskFactory` 的属性指定的。所以，默认的 `CancellationToken` 是 `TaskFactory.CancellationToken`，默认的 `ContinuationOptions` 值是 `TaskFactory.ContinuationOptions`，默认的 `TaskScheduler` 是 `TaskFactory.Scheduler`，这些全部都可以通过传递想要的值给 `TaskFactory` 的构造函数来设置。 

注意默认的 `TaskScheduler` 仍然很危险：任何时候一个没有显式指定 `TaskScheduler` 的 `TaskFactory` 构造时，它在 `ContinueWhenAny` 被调用时会默认被设为 `TaskScheduler.Current` 。这会导致像 `ContinueWith` 一样的令人惊讶的行为。注意静态 `TaskFactory` 实例 `Task.Factory` 确实有这个有问题的默认任务调度器。 

我推荐完全不要使用这些重载，相反，使用 `await Task.WhenAny(…)`（见下文）来异步等待一系列任务完成。

## TaskFactory.ContinueWhenAll

`ContinueWhenAll` 类似 `ContinueWhenAny`，除了它的逻辑是延续会在所有先行任务完成时执行。在 `TaskFactory` 上有16个重载，`TaskFactory<TResult>` 上有8个重载，与 `ContinueWhenAny` 一模一样。也有同样的默认参数逻辑。

并且同样的默认 `TaskScheduler` 也很危险。

并且我同样推荐完全不要使用这些重载，相反，使用 `await Task.WhenAll(…)` （见下文）。 

## Task.WhenAll

`Task.WhenAll` 返回一个当所有前置任务完成时完成的任务。概念上类似 `TaskFactory.ContinueWhenAll`，但与 `await` 工作的更好：

```csharp
Task WhenAll(IEnumerable<Task>);
Task WhenAll(params Task[]);
Task<TResult[]> WhenAll<TResult>(IEnumerable<Task<TResult>>);
Task<TResult[]> WhenAll<TResult>(params Task<TResult>[]);
```

`IEnumerable<>` 重载允许你传入一系列任务，比如一个LINQ表达式？。这个序列会被立即具体化（即复制到一个数组）。例如，这允许你直接传递一个 `Select` 表达式的结果给 `WhenAll`。就个人来说，我通常喜欢通过调用 `ToArray()` 来显式具体化序列，以便明显地知道发生了什么，但一些人喜欢直接传递序列进去的能力。 

有 `TResult` 泛型参数的重载会以数组的方式获取所有这些任务的结果。这在你有多个类似性质的操作时非常方便。例如，你可以像这样执行两个并行下载：

```csharp
var client = new HttpClient();
string[] results = await Task.WhenAll(
    client.GetStringAsync("http://example.com"),
    client.GetStringAsync("http://microsoft.com"));
// results[0] has the HTML of example.com
// results[1] has the HTML of microsoft.com
```

这在与LINQ结合使用时也很强大。下面的代码会同时下载任何源序列中的地址：

```csharp
IEnumerable<string> urls = ...;
var client = new HttpClient();
string[] results = await Task.WhenAll(urls.Select(url => client.GetStringAsync(url)));
```

## Task.WhenAny

`Task.WhenAny` 类似于 `Task.WhenAll`，它异步等待一个任务完成，而不是异步等待所有前置任务完成。它有一组相似的重载：

```csharp
Task<Task> WhenAny(IEnumerable<Task>);
Task<Task> WhenAny(params Task[]);
Task<Task<TResult>> WhenAny<TResult>(IEnumerable<Task<TResult>>);
Task<Task<TResult>> WhenAny<TResult>(params Task<TResult>[]);
```

`IEnumerable<>` 和 `TResult` 重载与它们在 `WhenAll` 中一样用于同样的目的。然而，`WhenAny` 的返回类型很有趣，`WhenAny` 返回一个在任意一个前置任务完成时完成的任务。这个任务的结果是完成了的前置任务。

这意味着用单个 `await` 调用 `WhenAny` 会返回给你那个完成的任务。这允许你做类似于同时执行两个操作并且看哪一个先完成这样的事：

```csharp
var client = new HttpClient();
Task<string> downloadExampleTask = client.GetStringAsync("http://example.com");
Task<string> downloadMicrosoftTask = client.GetStringAsync("http://microsoft.com");
Task completedTask = await Task.WhenAny(downloadExampleTask, downloadMicrosoftTask);
if (completedTask == downloadExampleTask)
  ; // example.com downloaded faster.
```

通常，当你使用 `WhenAny` 时，你其实并不关心不是第一个完成的那些任务。就是说，重要的只有第一个任务的结果。在这种场景下，你可以使用罕见但合法的“双重等待”：

```csharp
var client = new HttpClient();
string results = await await Task.WhenAny(
    client.GetStringAsync("http://example.com"),
    client.GetStringAsync("http://microsoft.com"));
// results contains the HTML for whichever website responded first.
```

如果你发现“双重等待”让人困惑，只需分解它并且指定类型。上面的代码等同于：

```csharp
var client = new HttpClient();
Task<string> firstDownloadToComplete = await Task.WhenAny(
    client.GetStringAsync("http://example.com"),
    client.GetStringAsync("http://microsoft.com"));
string results = await firstDownloadToComplete; 
// results contains the HTML for whichever website responded first.
```

> 我真心推荐使用 `await` 获取已完成任务的结果。在这种情况下，可能看起来 `await` 是不必要的，因为我们都知道任务已经完成了。然而，`await` 仍然好于 `Task.Result`，因为 `await` 不会把异常包装进一个 `AggregateException`。

----

原文链接：<https://blog.stephencleary.com/2015/01/a-tour-of-task-part-7-continuations.html>{:target="_blank"}

