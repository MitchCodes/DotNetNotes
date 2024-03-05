# Async Programming in .NET

Asynchronous programming is a method that allows for more responsive applications by freeing up the executing thread to perform other work while waiting for an operation to complete. In .NET, this is primarily achieved using the async and await keywords. Below are summaries of useful resources and articles on asynchronous programming in .NET.

If you are familiar with async programming already and are looking for best practices / important things to remember, refer to [the Best Practice / Takeaways section](#best-practice--takeaways).

## Resources

### [Asynchronous Programming in C#](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
This article provides a comprehensive overview of asynchronous programming in C#. It covers the basics of async and await keywords, best practices, and common patterns for asynchronous programming. It emphasizes the importance of understanding how to write asynchronous code properly to avoid common pitfalls such as deadlocks and performance issues.

### [Async Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md)
David Fowler's guide offers in-depth advice on async programming. Some have called it the 'Async Bible'. It includes practical tips on how to use asynchronous programming to improve the scalability and performance of applications. The guide also discusses common mistakes and how to avoid them by giving examples of bad and good async code.

### [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
This FAQ addresses common questions and scenarios related to ConfigureAwait in .NET. This was not talked about in David Fowler's article. It explains the role of SynchronizationContext in asynchronous programming and how ConfigureAwait(false) can be used to avoid deadlocks in certain situations. The article is essential for understanding the subtleties of task continuations and synchronization contexts.

### [ExecutionContext vs SynchronizationContext](https://devblogs.microsoft.com/pfxteam/executioncontext-vs-synchronizationcontext/)
This article Stephen Toub delves into the distinctions between ExecutionContext and SynchronizationContext within .NET, particularly in relation to the async/await keywords in C# and Visual Basic. Although the post is intended for an advanced audience (as stated in the article), it is good to know for understanding async programming including ConfigureAwait(). Specifically the SynchronizationContext is important for async programming.

### [Brownfield Async Development](https://learn.microsoft.com/en-us/archive/msdn-magazine/2015/july/async-programming-brownfield-async-development)
Stephen Cleary (not to be confused with Stephen Toub) discusses strategies for introducing asynchronous programming into existing ("[brownfield](https://synoptek.com/insights/it-blogs/greenfield-vs-brownfield-software-development/)") .NET applications. The article covers techniques for transforming synchronous code to asynchronous, dealing with potential issues, and tips for maintaining readability and maintainability during the transition. The article provides multiple 'hacks', as they call it, for accomplishing this.

### [StephenCleary's AsyncEx Library](https://github.com/StephenCleary/AsyncEx/blob/master/doc/Home.md)
This library available on NuGet is designed to facilitate programming with the async and await keywords, offering a variety of tools and utilities to manage asynchronous operations more effectively. This includes helper classes that provide functionality such as allowing `await` on lock(), provide a class to `await` a lazy load and much more. It is worth checking out if you have some slightly more advanced use-cases for async programming.

## General Notes on Async

- **Understanding async and await**: The `async` modifier indicates that a method is asynchronous and may contain one or more `await` expressions. The `await` keyword is used to asynchronously wait for a task to complete without blocking the executing thread.

- **Task and Task<T>**: In asynchronous methods, `Task` represents a single operation that does not return a value, and `Task<T>` represents an asynchronous operation that returns a value of type `T`.

- **ConfigureAwait**: Proper use of `ConfigureAwait(false)` can prevent deadlocks in GUI and ASP.NET applications by not capturing the `SynchronizationContext`. However, it should be used with caution as it can lead to unexpected behavior if the context is needed after an await operation. Additionally, all `await` calls in the logic chain need `ConfigureAwait(false)` otherwise it doesn't necessarily prevent deadlocks. This includes any third-party libraries or and Microsoft libraries that are used.

- **Error Handling**: Asynchronous methods use the task-based pattern to represent errors. Exceptions thrown in an asynchronous method are captured and placed on the returned task. To observe these exceptions, await the task or access its `Exception` property.

- **I/O-bound vs. CPU-bound**: Use asynchronous programming for I/O-bound operations to free up threads to do other work while waiting for the I/O operation to complete. For CPU-bound operations, consider using parallel programming techniques like `Parallel.For` or PLINQ.

- **Async Streams**: C# 8.0 introduced asynchronous streams with `IAsyncEnumerable<T>`, which allows for asynchronous iteration over sequences of data.

- **SynchronizationContext deadlock scenarios**: As mentioned [here](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md#warning-deadlocks), there are certain environments where you will run into a deadlock in scenarios such as `Task.Result` and `Task.Wait`. These environments manage their own SyncronizationContext. These include (but are not limited to) non-core ASP.NET, WPF and Windows Forms.

## Best Practice / Takeaways 

- **async void**: As mentioned [here](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md#async-void), `async void` functions should never be created. The only time they _might_ be used is for 'fire and forget' code. They crash the process if an exception is thrown in them.

- **`ConfigureAwait(false)` usage**: If you are writing library code which could be used by multiple different kinds of apps or you do not have control of where it will be used, add `ConfigureAwait(false)` to *all* of your `await`ed calls.

- **calling async code from sync code**: Calling async code from syncronous code is tricky and dependent on the situation. The main best practice to follow is to simply not do this and go full async down the whole stack. If you need to do this, though, the secondary thing is that *you should never use `Task.Result` or `Task.Wait`* because it poses the most problems and because of the risk of a deadlock. Good resources to look at is from the ['async bible'](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md#avoid-using-taskresult-and-taskwait), learning about [ConfigureAwait(false)](https://devblogs.microsoft.com/dotnet/configureawait-faq/) and the 'hacks' mentioned and starting [in this article](https://learn.microsoft.com/en-us/archive/msdn-magazine/2015/july/async-programming-brownfield-async-development#the-blocking-hack). Generally, if you need to call an async function from sync code:
  - Use `.GetAwaiter().GetResult()` if you are sure that all async calls in the async function all the way down the stack are configured with `ConfigureAwait(false)`, including all used Microsoft or third party library calls. Alternatively, you can use this if you are *very sure* that any calling code will not have a `SynchronizationContext` that causes deadlocks. If any of the functions are missing the ConfigureAwait then a deadlock can still occur in certain app environments/scenarios mentioned in [SynchronizationContext deadlock scenarios](#general-notes-on-async). Note that `.GetAwaiter().GetResult()` is almost equivalent to simply `.Result` except that `.Result` has worse exception handling because it will throw an AggregateException wrapping the original exception.
  - Use `Task.Run(() => AsyncFunction()).GetAwaiter().GetResult()` if you are unsure of or if the async function you are calling does not implement `ConfigureAwait(false)` for every single `await`ed call all the way down the stack. Any exceptions thrown in `AsyncFunction()` will be thrown normally and not as an aggregate exception. This way is also worth considering using if you are calling async code from syncronous code high up in a complicated application with much logic below it. This is especially true if the code below it is changing often, there are a number of developers maintaining the code below, and/or the application is an existing [brownfield](https://synoptek.com/insights/it-blogs/greenfield-vs-brownfield-software-development/) application that is early in the process of introducing async. Instead of doing sync to async higher up though, you might want to consider [vertical partitions as described here](https://learn.microsoft.com/en-us/archive/msdn-magazine/2015/july/async-programming-brownfield-async-development#vertical-partitions).
  - Use the [flag argument hack](https://learn.microsoft.com/en-us/archive/msdn-magazine/2015/july/async-programming-brownfield-async-development#the-flag-argument-hack) if you are in low-level code that has both a sync and async implementations you can use such as `WebClient.DownloadString`. Doing this method provides your services with both a sync and async implementation while still preventing deadlocking and reusing code through the private function.

- **`ConfigureAwait(false)` with `GetResult()`**: As explained in the referenced [ConfigureAwait(false)](https://devblogs.microsoft.com/dotnet/configureawait-faq/) article, the `ConfigureAwait(false)` in `task.ConfigureAwait(false).GetAwaiter().GetResult()` does nothing and can/should be removed.

## ExecutionContext and SynchronizationContext

### ExecutionContext
- **What it is**: ExecutionContext is a container for various contexts, some ancillary and some vital to .NET's execution model. It's like the "air" of .NET's execution environment, usually unnoticed unless something goes wrong.
- **Its role**: It stores "ambient" information relevant to the current execution environment, traditionally maintained in thread-local storage (TLS). This is straightforward in synchronous operations but becomes complex in asynchronous scenarios where operations might not be tied to a single thread.
- **Flowing ExecutionContext**: In asynchronous operations, ExecutionContext allows the ambient state to be captured from one thread and restored onto another, maintaining logical control flow and ambient data continuity across different threads.

### SynchronizationContext
- **What it is**: An abstraction representing a particular environment for executing work, like the UI thread in Windows Forms or WPF applications.
- **Its role**: It allows for the marshaling of work back to a specific environment, crucial for operations that need to interact with UI controls or other environment-specific tasks.
- **Usage**: SynchronizationContext provides a virtual `Post` method to run a delegate in the designated environment, abstracting away the specifics of how and where the work gets executed.

### Differences and Usage
- **Flowing ExecutionContext vs. Using SynchronizationContext**: Flowing ExecutionContext involves capturing and restoring state across threads, making the state ambient during execution. In contrast, using SynchronizationContext involves capturing state and then using it to invoke a delegate, with the execution specifics determined by the `Post` method's implementation.
- **In async/await**: The async/await mechanism in .NET automatically interacts with both ExecutionContext and SynchronizationContext, ensuring ambient information and execution environment continuity across asynchronous operations.

While ExecutionContext is an underlying infrastructure component that most developers need not worry about, SynchronizationContext is more visible and requires consideration, especially when dealing with UI elements or other environment-specific tasks.