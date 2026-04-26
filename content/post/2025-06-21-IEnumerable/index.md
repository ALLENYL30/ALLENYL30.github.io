+++
author = "yuhao"
title = "Why Doesn't IEnumerable<T> Have a ForEach Method?"
date = "2025-06-20"
description = "In C#, types like List<T> (and others such as ImmutableList<T>) include a convenient ForEach method. It can be seen as a more functional-style alternative to the traditional foreach loop. For example:"
tags = [
    ".NET",
    ".NET Core",
    "C#",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
In C#, types like `List<T>` (and others such as `ImmutableList<T>`) include a convenient `ForEach` method. It can be seen as a more functional-style alternative to the traditional `foreach` loop. For example:

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };

// Traditional foreach loop
foreach (var n in list)
{
    Console.WriteLine(n);
}

// The ForEach method achieves the same result
list.ForEach(n => Console.WriteLine(n));
```


Additionally, the `Array` class provides a static `ForEach` method that serves a similar purpose. However, you'll quickly notice that this handy method is not available on the `IEnumerable<T>` interface. Why is that?

> **Note:** Let's address a common but misguided workaround: converting an `IEnumerable<T>` to a `List<T>` just to use `ForEach()` is generally a bad idea. This can trigger the premature execution of a LINQ query, involve unnecessary object creation, and incur the cost of populating a new collection.

There are good reasons for this design choice. I've summarized a few of them below.

## The Consequences of Adding `ForEach`

### Compromising Interface Purity

To put it more formally, adding a `ForEach` method would violate the **Single Responsibility Principle (SRP)** and the **Interface Segregation Principle (ISP)** from the SOLID principles.

Let's examine the definition of the `IEnumerable<T>` interface:

```csharp
namespace System.Collections.Generic
{
    public interface IEnumerable<out T> : IEnumerable
    {
        new IEnumerator<T> GetEnumerator();
    }
}
```

It's incredibly clean. Its name and definition clearly indicate its single purpose: to signify that an object is "enumerable." The `ForEach` method, however, is designed to execute logic and perform actions, which contradicts the interface's core responsibility of simple iteration.

Unlike LINQ methods such as `Select`, `Where`, and `OrderBy`, which are used for data projection, filtering, and sorting, `ForEach` is inherently about side effects. LINQ methods are designed to be pure transformations of data sequences. Imagine if LINQ queries included time-consuming logical operations (like complex CPU calculations or I/O). The reliability and predictability of LINQ would be severely compromised.

If a `ForEach` method contained such intensive operations, the better approach would be to use asynchronous programming (e.g., creating tasks and awaiting them with `Task.WhenAll`), the `Parallel` class, or PLINQ. This keeps the `IEnumerable<T>` as a pure data sequence, free from uncontrollable operational logic.

Here is a simple example using async tasks:

```csharp
IEnumerable<int> items = ...;
var tasks = items.Select(x => CalculateValueAsync(x));
await Task.WhenAll(tasks);
var results = tasks.Select(t => t.Result).ToList();
```

### Introducing Side Effects

Of course, we could write our own extension method to enable `ForEach` on `IEnumerable<T>`:

```csharp
static class EnumerableExtensions
{
    public static void ForEach<T>(this IEnumerable<T> items, Action<T> action)
    {
        foreach (var item in items)
        {
            action.Invoke(item);
        }
    }
}
```

However, this can "pollute" the established functional paradigm of LINQ. Why?

Think about the primary purpose of LINQ methods. They are for querying—projecting, filtering, and ordering data. These methods are generally understood to be "pure"; they don't modify the source data. While they accept a delegate, it's almost always a `Func<T, TResult>` that returns a new value or a basis for comparison, not an `Action<T>` that returns nothing.

The `ForEach` method is different because it accepts an `Action<T>`, implying that no value is returned. In this scenario, the operation is very likely to cause side effects by modifying the original data. For instance:

```csharp
List<Employee> employees = ...;

employees.ForEach(e => {
    if (e.IsPromoted) // If the employee is promoted, give them a raise
        e.Salary += 1000;
});
```

This shows that `ForEach` is often used with the intent to mutate the elements of the collection.

While this is a general tendency and not an absolute rule—one could misuse `Select` to modify data as well—the design of `ForEach` leans much more heavily toward encouraging side effects compared to standard LINQ methods.

### Performance and Resource Considerations

Another critical point is that collections with a built-in `ForEach` method, like `List<T>` and `Array`, have a known, finite size. This is crucial because an `IEnumerable<T>` can represent a sequence that is deferred or even **infinite**.

```csharp
IEnumerable<int> GenerateInfiniteNumbers()
{
    while (true)
    {
        yield return 0;
    }
}
```

Given this, applying a `ForEach` operation to a sequence of unknown size is inherently risky. Furthermore, unlike a `foreach` *statement*, the `ForEach` *method* doesn't allow the use of `break`, `continue`, or `return` to exit the loop early. Once it starts, it is committed to iterating over every single element in the sequence.

From this perspective, omitting a `ForEach` extension method for `IEnumerable<T>` seems like a very sensible decision. You might argue that LINQ methods also face issues with infinite sequences, which is true. However, LINQ is fundamentally designed to work with `IEnumerable<T>`, so this is an unavoidable characteristic that developers must be mindful of.

---

The benchmarks show that its performance is significantly worse than traditional `for` and `foreach` loops:

| Method | Mean | Allocation |
| - | - | - |
| `for` | 424.9 us | - |
| `foreach` | 426.4 us | - |
| `ForEach` | 1,785.0 us | 88 B |

> **Notes on the benchmark results:**
> 
> * The table above is simplified and omits some columns for clarity.
> * The test was run on a preview version of .NET 9, where `foreach` and `for` have nearly identical performance with zero memory allocations.
> * The `ForEach` method was approximately **4 times slower** and incurred memory allocation due to the delegate and its associated closure.

This provides further evidence that `ForEach` is not a high-performance option. For iterating over a collection where performance matters, traditional `for` or `foreach` loops remain the superior choice.

---

## Summary

In conclusion, while adding a `ForEach` method to `IEnumerable<T>` is technically feasible, the .NET framework designers chose not to include it due to design philosophy, code clarity, performance considerations, and the potential for unintended side effects.

However, if you find you need this functionality, you are free to implement your own extension method. Just be sure you understand the potential consequences of doing so.

