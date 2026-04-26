+++
author = "yuhao"
title = "Improving Thread Communication in .NET: Moving Beyond Flags and Polling"
date = "2025-04-06"
description = "In multi-threaded development, we often use flags and polling to control execution logic within threads. However, this approach reduces code readability and maintainability while lacking elegance...."
tags = [
    ".NET",
    ".NET Core",
    "C#",
    "Multithreading",
]
categories = [
    "Programming/Development",
]
+++
In multi-threaded development, we often use flags and polling to control execution logic within threads. However, this approach reduces code readability and maintainability while lacking elegance. This article explores how to use semaphores and other mechanisms to replace polling and flags, thereby improving inter-thread communication and control.

## Traditional Approach

First, let's examine the traditional approach using flags and polling with a simple example:

```csharp
class MyService
{
    private volatile bool _shouldStop;
    private Thread? _workerThread;

    public void Start()
    {
        _workerThread = new Thread(Worker);
        _shouldStop = false;
        _workerThread.Start();
    }

    public void Stop()
    {
        _shouldStop = true;
        _workerThread?.Join();
    }

    private void Worker()
    {
        while (_shouldStop)
        {
            // Do some work
            Thread.Sleep(50); // Simulate work
        }
    }
}
```

This is a typical example. Here, `_shouldStop` is a flag indicating whether the thread should stop. We set and read this flag in the `Start` and `Stop` methods, and poll it in the `Worker` method to determine whether the thread should continue executing.

>

> **Info**: The `_shouldStop` flag is marked as `volatile`, ensuring the compiler doesn't optimize it and guaranteeing each read gets the most recent value. In practice, if our polling includes operations like `Thread.Sleep`, we would still get the latest value even without the `volatile` keyword.

>

> **Warning**: Note that we're using simple code to introduce concepts, not providing a robust implementation. The example doesn't handle multiple calls to `Start` or thread exceptions.

If we want to add pause and resume functionality, we need additional flags and polling logic:

```csharp
class MyService
{
    private volatile bool _shouldStop;
    private volatile bool _isRunning;
    private Thread? _workerThread;

    public void Start()
    {
        _workerThread = new Thread(Worker);
        _shouldStop = false;
        _isRunning = true;
        _workerThread.Start();
    }

    public void Stop()
    {
        _shouldStop = true;
        _workerThread?.Join();
    }

    public void Pause()
    {
        _isRunning = false;
    }

    public void Resume()
    {
        _isRunning = true;
    }

    private void Worker()
    {
        while (_shouldStop)
        {
            if (_isRunning)
            {
                // Do some work
            }

            Thread.Sleep(50); // Sleep even when paused to avoid high CPU usage
        }
    }
}
```

## Classic Two-Flag Pattern

Now let's explore how to replace flags with semaphores.

## Using ManualResetEvent for Thread Pause and Resume

Let's consider what `_isRunning` accomplishes. We use its value to control thread execution but can't be notified immediately when it changes, so we must poll periodically. What if we could block at a specific point when it's false and continue when it becomes true, without polling?

For this requirement, we can use two `WaitHandle` subclasses: `ManualResetEvent` and `AutoResetEvent`. `ManualResetEvent` is a manually resettable semaphore that remains in a signaled state until reset. In contrast, `AutoResetEvent` automatically resets after each signal. Here, `ManualResetEvent` fits our needs better because we want continuous execution after signaling rather than requiring a set for each execution.

```csharp
class MyService
{
    private volatile bool _shouldStop;
    private ManualResetEvent _isRunningEvent = new ManualResetEvent(false); // Initially closed

    private Thread? _workerThread;

    public void Start()
    {
        _workerThread = new Thread(Worker);
        _shouldStop = false;
        _isRunningEvent.Set(); // Signal when the thread starts
        _workerThread.Start();
    }

    public void Stop()
    {
        _shouldStop = true;
        _workerThread?.Join();
    }

    public void Pause()
    {
        _isRunningEvent.Reset(); // Close the semaphore
    }

    public void Resume()
    {
        _isRunningEvent.Set(); // Signal the semaphore
    }

    private void Worker()
    {
        while (_shouldStop)
        {
            _isRunningEvent.WaitOne(); // Wait for the semaphore to signal

            // Do some work

            Thread.Sleep(50); // Sleep to avoid high CPU usage
        }
    }
}
```

## Using CancellationToken to Stop Tasks

We can further optimize by using `CancellationToken` to stop tasks. `CancellationToken` is a .NET mechanism for canceling operations that can pass a cancellation request to a task and be checked within that task. It works for both asynchronous and multi-threaded programming. Here, we use it to replace the `_shouldStop` flag:

```csharp
class MyService
{
    private ManualResetEvent _isRunningEvent = new ManualResetEvent(false); // Initially closed
    private Thread? _workerThread;
    private CancellationTokenSource _cancellationTokenSource = new CancellationTokenSource();

    public void Start()
    {
        _workerThread = new Thread(Worker);
        _isRunningEvent.Set(); // Signal when the thread starts
        _workerThread.Start();
    }

    public void Stop()
    {
        _cancellationTokenSource.Cancel(); // Cancel the operation
        _workerThread?.Join();
    }

    public void Pause()
    {
        _isRunningEvent.Reset(); // Close the semaphore
    }

    public void Resume()
    {
        _isRunningEvent.Set(); // Signal the semaphore
    }

    private void Worker()
    {
        while (!_cancellationTokenSource.Token.IsCancellationRequested)
        {
            _isRunningEvent.WaitOne(); // Wait for the semaphore to signal

            // Do some work

            Thread.Sleep(50); // Sleep to avoid high CPU usage
        }
    }
}
```

The advantages of `CancellationToken` aren't fully demonstrated in this simple example. In practice, many methods accept a `CancellationToken` parameter, allowing us to cancel long-running tasks called within the `Worker` method rather than waiting for them to finish after cancellation.

## Optimizing the Producer-Consumer Pattern

We often need to implement a producer-consumer pattern using a queue:

```csharp
class MyService
{
    private readonly Queue<int> _queue = new Queue<int>();
    private readonly object _lock = new object();

    private volatile bool _shouldStop;
    private volatile bool _isRunning;

    private Thread? _workerThread;

    public void Start()
    {
        _workerThread = new Thread(Worker);
        _shouldStop = false;
        _isRunning = true;
        _workerThread.Start();
    }

    public void Stop()
    {
        _shouldStop = true;
        _workerThread?.Join();
    }

    // Pause and Resume methods omitted

    public void Enqueue(int item)
    {
        lock (_lock)
        {
            _queue.Enqueue(item);
        }
    }

    public void Worker()
    {
        while (!_shouldStop)
        {
            lock (_lock)
            {
                if (_queue.Count > 0 && _isRunning)  // Could also use TryDequeue
                {
                    var item = _queue.Dequeue();
                    // Process item
                }
            }

            Thread.Sleep(50); // Sleep even when paused to avoid high CPU usage
        }
    }
}
```

In this example, we:

1. Use`Queue` with thread locks to ensure thread safety
2. Use`_shouldStop` and`_isRunning` to control thread execution
3. Use`lock` in the`Worker` method to access the queue safely
4. Expose an`Enqueue` method for producers to add tasks

How can we optimize this approach?

## Using Thread-Safe Collections

.NET provides thread-safe collection types like `ConcurrentQueue<T>` that can be safely used in multi-threaded environments without manual locking:

```csharp
class MyService
{
    private readonly ConcurrentQueue<int> _queue = new ConcurrentQueue<int>();

    public void Enqueue(int item)
    {
        _queue.Enqueue(item);
    }

    public void Worker()
    {
        while (!_shouldStop)
        {
            if (_isRunning && _queue.TryDequeue(out var item))
            {
                // Process item
            }

            Thread.Sleep(50); // Sleep even when paused to avoid high CPU usage
        }
    }
}
```

This eliminates the need for manual locking as `ConcurrentQueue<T>` automatically handles thread safety.

## Using Semaphores Instead of Flags

We're still polling in the code above, essentially waiting for data in the queue. We could use a semaphore that signals only when new data arrives—an `AutoResetEvent`:

```csharp
class MyService
{
    private readonly ConcurrentQueue<int> _queue = new ConcurrentQueue<int>();
    private readonly AutoResetEvent _queueEvent = new AutoResetEvent(false); // Initially closed

    public void Enqueue(int item)
    {
        _queue.Enqueue(item);
        _queueEvent.Set(); // Signal the semaphore
    }

    public void Worker()
    {
        while (!_shouldStop)
        {
            _queueEvent.WaitOne(); // Wait for the semaphore to signal

            if (_isRunning && _queue.TryDequeue(out var item))
            {
                // Process item
            }
        }
    }
}
```

However, this approach isn't ideal. If multiple data items arrive simultaneously, we call `Set` multiple times, but the semaphore only signals once, potentially delaying data processing. A better approach would be using `Semaphore`, which is like a gate with variable width—each signal makes the gate wider, unlike `AutoResetEvent`'s binary states. But we have an even better solution.

## Using BlockingCollection for the Producer-Consumer Pattern

.NET provides `BlockingCollection<T>`, a purpose-built class for the producer-consumer pattern with thread safety and blocking functionality:

```csharp
class MyService
{
    private readonly BlockingCollection<int> _queue = new BlockingCollection<int>();

    public void Enqueue(int item)
    {
        _queue.Add(item); // Add data to the queue
    }

    public void Worker()
    {
        while (!_shouldStop)
        {
            if (_isRunning && _queue.TryTake(out var item, Timeout.Infinite)) // Wait for data
            {
                // Process item
            }
        }
    }
}
```

With this approach, if the queue is empty, `TryTake` blocks the current thread until data is available. When `Add` is called, `BlockingCollection<T>` automatically signals waiting threads, eliminating the need to manually handle semaphores.

>

> **Info**: Looking at `BlockingCollection<T>`'s source code reveals it uses `ConcurrentQueue<T>` and `SemaphoreSlim` internally. The underlying collection is configurable (e.g., `ConcurrentStack<T>`, `ConcurrentBag<T>`), allowing different behaviors by passing different collection types.

## Conclusion

In this article, we explored how to use semaphores and other mechanisms to replace polling and flags for thread communication and control. Classes like `ManualResetEvent`, `AutoResetEvent`, `CancellationToken`, and `BlockingCollection<T>` enable more elegant multi-threaded programming, avoiding the problems associated with polling and flags.

In real-world development, always consider these built-in classes and tools rather than reinventing the wheel.

