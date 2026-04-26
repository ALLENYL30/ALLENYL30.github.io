+++
author = "yuhao"
title = "Why ValueTask? A Guide to C#'s Performance-Oriented Task Alternative"
date = "2025-04-23"
description = "Since C# 5.0 introduced the async and await keywords, asynchronous programming has become remarkably simple. The Task type has played a crucial role in this paradigm, becoming a ubiquitous part of..."
tags = [
    ".NET",
    ".NET Core",
    "C#",
    "async",
]
categories = [
    "Programming/Development",
]
+++
Since C# 5.0 introduced the `async` and `await` keywords, asynchronous programming has become remarkably simple. The `Task` type has played a crucial role in this paradigm, becoming a ubiquitous part of modern .NET development. However, with the release of .NET Core 2.0, Microsoft introduced a new type: `ValueTask`. What is this type, why do we need it, and in what situations should we use it? Let's explore these questions today.

## A Quick Recap of the `Task` Type

In asynchronous programming, we use the `Task` type to represent an asynchronous operation. Compared to other mainstream programming languages, C#'s tasks are incredibly lightweight. For instance, C# experts have noted that a `Task` object typically consumes only 64-136 bytes of memory, whereas a goroutine in Go starts at a minimum of 2 KB.

Furthermore, `Task` comes with numerous optimization techniques:

* **`Task.FromResult`**: To return a completed task with a specific result.
* **`Task.CompletedTask`**: To return a singleton, already-completed task.
* **`Task.FromCanceled`**: To return a task in a canceled state.
* **`Task.FromException`**: To return a task in a faulted state.

For a long time, from C# 5.0 (.NET Framework 4 era) until .NET Core 2.0, `Task` served its purpose exceptionally well.

## The Problem with the Traditional `Task` Type

As .NET evolved into a cross-platform framework, its use cases expanded dramatically. Microsoft's ambitions grew, leading to a relentless focus on performance optimization from every angle. This effort brought us innovations like `Span<T>`, `Memory<T>`, and `ref struct`, and it also led to the creation of `ValueTask`. So, what was the problem with the traditional `Task` type?

First, we need to understand that `Task` exists in both generic (`Task<TResult>`) and non-generic versions, representing async operations with and without a return value, respectively. When `ValueTask` was first introduced, it only had a generic version. This tells us that its initial design was focused exclusively on asynchronous methods that return a value.

Let's consider a classic example:

```csharp
private readonly ConcurrentDictionary<int, string> _cache = new ();

public async Task<string> GetMessageAsync(int id)
{
    if (_cache.TryGetValue(id, out var message))
    {
        return message;
    }

    message = await GetMessageFromDatabaseAsync(id);
    _cache.TryAdd(id, message);

    return message;
}
```

In the `GetMessageAsync` method above, we first try to retrieve a message from a cache. If it's not found, we fetch it from the database. The problem lies in the cache-hit scenario. Although it seems like we are returning a value directly, the method's signature is `async Task<string>`. This means that even when the result is available synchronously, the C# compiler still allocates a new `Task<string>` object on the heap to wrap that result. This leads to unnecessary memory allocation.

> This scenario, where an `async` method can return a result without ever hitting an `await` keyword, is known as **"synchronous completion."** The operation begins and ends on the same thread.

## Introducing `ValueTask`

`ValueTask` was created to solve this very problem. It was officially introduced in .NET Core 2.0 and enhanced in .NET Core 2.1 (with the addition of the `IValueTaskSource<T>` interface, enabling properties like `IsCompleted`). A non-generic `ValueTask` was also added later.

Instead of thinking of `ValueTask` as just a value type, it's more helpful to understand it as a type that can represent *either* a `Value` *or* a `Task`. This makes it perfectly suited for the caching scenario described above. We can modify the code to use `ValueTask`:

```csharp
public async ValueTask<string> GetMessageAsync(int id)
{
    if (_cache.TryGetValue(id, out var message))
    {
        return new ValueTask<string>(message); // In C# 7.2+, you can just `return message;`
    }

    message = await GetMessageFromDatabaseAsync(id);
    _cache.TryAdd(id, message);

    return message;
}
```

Now, if the data is found in the cache, the method can return a `ValueTask<T>` that wraps the result directly, avoiding any heap allocation for a `Task<T>` object. If the data is not in the cache (the "cold path"), the method proceeds to `await` the database call. In this case, `ValueTask<T>` will wrap an underlying `Task<T>`, and its performance will be roughly equivalent to (or very slightly slower than) using `Task<T>` directly, due to the overhead of the wrapper struct.

> The non-generic `ValueTask` is used even more rarely. It's only beneficial in scenarios where an operation can complete asynchronously without requiring any memory allocation. Stephen Toub, a key architect behind this feature, advises against using the non-generic `ValueTask` unless profiling has proven that the tiny allocation of a standard `Task` is a critical performance bottleneck.

At this point, you might be thinking: `ValueTask` is a `struct`, so it's allocated on the stack, which is much cheaper than heap allocation. Like `ValueTuple` versus `Tuple` or `Span<T>` versus an array slice, it seems like a clear performance win.

But is `ValueTask` really that perfect? Can it completely replace `Task`? The answer is not so simple.

## Important `ValueTask` Caveats

Now it's time to discuss the important rules and limitations you must respect when using `ValueTask`.

### A `ValueTask` Cannot Be Awaited More Than Once

A `ValueTask` may wrap a reusable underlying object (an `IValueTaskSource<T>`). Once you `await` it and the operation completes, that underlying object may be recycled and used for another operation. Awaiting it a second time could lead to reading incorrect state or other unpredictable behavior. In contrast, a `Task` represents a final state and can be awaited any number of times.

### Do Not Block on a `ValueTask`

The underlying `IValueTaskSource<T>` that a `ValueTask` may wrap is not required to support blocking until completion. This means you must **not** call methods like `.Wait()` or access the `.Result` property on a `ValueTask` that has not yet completed. Doing so is likely to cause a deadlock or throw an exception.

However, if you can confirm that a `ValueTask` is already complete (by checking its `IsCompleted` property), it is safe to access its `.Result` property to get the value.

> Microsoft has added a specific Roslyn analyzer warning for this misuse: [CA2012: Use ValueTask correctly](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2012).

### Do Not `await` a `ValueTask` Concurrently

`ValueTask` was designed as a performance optimization, not a full-featured `Task` replacement. The underlying object is intended to be awaited by a single "consumer" at a time and is not thread-safe. Attempting to await the same `ValueTask` from multiple threads concurrently can easily introduce race conditions and subtle bugs. A `Task`, on the other hand, is thread-safe and supports any number of concurrent awaiters.

## How to Work Around `ValueTask`'s Limitations

In practice, you might encounter situations where you need to overcome these limitations. Here are the recommended approaches:

* **To get the result synchronously**: If you need to block to get a `ValueTask<T>`'s result, you must first check properties like `IsCompleted` or `IsCompletedSuccessfully`. Only after confirming it's complete should you access the `.Result` property.
* **To await multiple times or from multiple threads**: Use the **`.AsTask()`** method. This will convert the `ValueTask<T>` into a standard `Task<T>`. If the `ValueTask<T>` already wraps a `Task<T>`, it returns that same instance. If it wraps a result, it will return a new, completed `Task<T>`. Once you have a `Task<T>`, you can use it with all the flexibility you're used to.

Based on its design and limitations, a widely accepted best practice has emerged:

> **Best Practice:** In the vast majority of cases, you should **`await` a `ValueTask<T>` immediately** to consume its result. Avoid assigning the returned `ValueTask<T>` to a local variable for later use. If you need to store it, pass it around, or await it multiple times, **convert it to a `Task` first using `.AsTask()`**.

## Conclusion

In summary, `ValueTask` is a powerful feature for performance-critical code, offering a way to eliminate heap allocations in "hot path" scenarios where an async method often completes synchronously. However, this performance gain comes with strict usage rules, such as the single-await limitation. Using it can feel like walking a tightrope; a misstep can lead to subtle bugs.

Fortunately, the primary use case is simple and safe: `await` the `ValueTask<T>` immediately. As long as you remember that it is a specialized tool and not a general-purpose replacement for `Task`, you can leverage its benefits effectively.

Hopefully, this article helps you use `ValueTask` correctly and confidently in your projects.

## References

* [ValueTask Source Code](https://source.dot.net/#System.Private.CoreLib/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/ValueTask.cs,77a292425839ae85)
* [Understanding the Whys, Whats, and Whens of ValueTask | .NET Blog](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
* [Working with ValueTask in C# | CodeGuru.com](https://www.codeguru.com/csharp/c-sharp-valuetask/)

