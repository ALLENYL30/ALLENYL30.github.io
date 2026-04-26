+++
author = "yuhao"
title = "Fields vs. Properties: Why Not Just Use Public Fields in C#?"
date = "2025-05-12"
description = "When writing C# code, we frequently define auto-properties like this:"
tags = [
    ".NET",
    "C#",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
When writing C# code, we frequently define auto-properties like this:

```csharp
public class Person
{
    public int Age { get; set; }
}
```

This often raises a question, especially for developers new to the language: Why go through the trouble of writing an auto-property? Why not just declare a simple public field? Like this:

```csharp
public class Person
{
    public int Age;
}
```

It's a valid question, and there are several important reasons behind this convention. Let's explore them.

## A Fundamental Difference in Perspective

First and foremost, the way we perceive properties and fields—and the roles they play—is fundamentally different.

When we see a **property**, we generally expect it to represent a piece of the class's public contract. We anticipate that it has public read access, while its write access might be more restricted. When we expose a property, we are making a deliberate decision to share a member with the outside world, having fully considered the implications (for example, we can add logic to the setter or make it read-only).

In short, we typically expect a property to be:

1. **A deliberate part of the public API:** It's a member that the developer has intentionally exposed because it has meaning and value to the consumers of the class.
2. **Controlled:** Its initialization and modification can be controlled with logic. We can define when it can be set, if it's required, and whether it can be changed after initialization.
3. **Potentially computed:** It isn't necessarily backed by a simple field. Its value might be derived from other data.

> **Examples of Computed Properties:**
>
> * `List<T>.Count` is derived from the state of the internal array.
> * `AsyncRelayCommand.IsRunning` is determined by the status of an underlying `Task`.
> * `ObservableValidator.HasErrors` depends on the contents of an internal list of validation errors.
> * In UI frameworks, a `CheckBox.IsChecked` property is often connected to a more complex dependency property system.

Now, how do we typically think about **fields**? Consider this simple code snippet:

```csharp
class Manager
{
    private readonly int _uniqueId;             // A unique ID with a special purpose.
    private readonly IConfiguration _config;    // An interface object initialized via dependency injection.
    private bool _isDataProcessed = false;      // A flag for internal state tracking.
    private readonly object _syncRoot = new();  // A lock object for internal thread safety.
}
```

Does this align with your general impression of fields? If so, then the following code would probably feel a bit jarring and confusing:

```csharp
class Manager
{
    protected readonly int UniqueId;
    public bool Flag = false;
    public string ErrorMessage = "Oops!";
}
```

The reason for this discomfort should be becoming clear. Our conventional understanding of a field's role is:

* It's primarily for **internal use**, serving as a helper for properties or methods (e.g., lock objects, flags, injected dependencies).
* It's a **simple storage location** without much associated logic, lacking the sophisticated initialization options of properties.
* It's generally considered "unsafe" for public exposure, as developers consuming the class wouldn't know if they can safely modify it without breaking internal invariants.

Based on these different perspectives, it's easier to understand why we generally avoid public fields.

Of course, there are always exceptions. For example, in simple Unity game development, you might write code like this:

```csharp
public class Player : MonoBehaviour
{
    [SerializeField]
    private int health; // Unity's official convention is lowercase for serialized fields
    public int attack;
    public int defense;
}
```

Or when interacting with C/C++ DLLs (P/Invoke), you'll often define structs with public fields to match the memory layout:

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct MyData
{
    public ushort Index;
    public uint Value;
    public byte[] Data;
}
```

In these specific contexts, public fields are not just acceptable but are often the standard practice.

## Long-Standing Conventions in the Ecosystem

These differing perspectives have led to a set of strong conventions that are reinforced by the .NET standard libraries and third-party packages.

> This raises a classic "chicken or the egg" question: did the libraries adopt these conventions because it was already a best practice, or did it become a best practice because the libraries were designed that way? Regardless, the convention is firmly established.

Here are just a few examples:

* **WPF/UI Frameworks:** To bind a variable in XAML to a Model or ViewModel, that variable **must** be a property. Key features like Dependency Properties are also intrinsically linked to properties.
* **Serialization:** Libraries like **Json.NET** and **System.Text.Json** will, by default, only serialize public properties. Fields are ignored.
* **Data-Binding Controls:** UI controls like `DataGrid` or `PropertyGrid` that automatically generate their interface based on a data type do so by reflecting over its **properties**, not its fields.
* **Entity Framework Core:** In a Code-First approach, your entity's mapped members must be properties. When using a Database-First approach, the scaffolding tools generate classes with properties.
* **Interfaces:** C# interfaces can declare properties, but they cannot declare fields.
* **Records:** C#'s `record` types use properties under the hood to achieve their immutability and value-based equality features.

You'll find similar behavior in other tools, such as object-mappers (e.g., AutoMapper, Mapster), where properties are treated as the primary members for mapping.

Given how deeply ingrained this convention is, why choose to be the outlier?

## Flexibility and Encapsulation

Properties offer unparalleled flexibility. You can embed arbitrary logic within their `get` and `set` accessors. A classic example is data validation:

```csharp
public class Person
{
    private int _age;

    public int Age
    {
        get => _age;
        set
        {
            if (value < 0)
            {
                throw new ArgumentOutOfRangeException(nameof(value), "Age must be greater than 0.");
            }

            _age = value;
        }
    }
}
```

Another common use case is implementing change notification:

```csharp
class ViewModel : INotifyPropertyChanged
{
    private string _name;

    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged(nameof(Name)); // Implementation of event and method omitted
            }
        }
    }
    // ... INotifyPropertyChanged implementation
}
```

Even more important is the fine-grained encapsulation they provide. Consider the common variations for a setter:

* `public set;`: Anyone can modify the property.
* `protected set;`: Only the class and its subclasses can modify it.
* `private set;`: Only methods within the class itself can modify it.
* `internal set;`: Only code within the same assembly can modify it.
* `init;`: The property can only be set during object initialization.
* **(no setter)`: A read-only property, settable only in the constructor.

Furthermore, these can be combined with keywords like `required` and `virtual` to achieve even greater flexibility and control. Fields simply cannot compete with this level of sophistication (while some keywords can be applied to fields, their effect is usually all-or-nothing, restricting both read and write access).

## Performance Considerations

At this point, some might argue: "I know that an auto-property is just syntactic sugar. The compiler turns it into a private backing field and a pair of get/set methods. Doesn't a method call have more overhead than directly accessing a field? If so, isn't a public field more performant?"

This is an excellent question. Let's look at the compiled code. Consider this class:

```csharp
var p = new Person();

p.Age1 = 10;
p.Age2 = 20;

public class Person
{
    public int Age1 { get; set; }
    public int Age2;  
}
```

If we inspect the Intermediate Language (IL), we see a difference:

```il
IL_0000: newobj     instance void Person::.ctor()
IL_0005: dup
IL_0006: ldc.i4.s   10
IL_0008: callvirt   instance void Person::set_Age1(int32) // Method call for property
IL_000d: ldc.i4.s   20
IL_000f: stfld      int32 Person::Age2                  // Direct field store
IL_0014: ret
```

It certainly seems that `Age1` uses a method call (`callvirt`) while `Age2` uses a direct field store instruction (`stfld`). So, is the property slower? Before you jump to conclusions, let's look at the final machine code generated by the Just-In-Time (JIT) compiler:

```asm
; Program.<Main>$(System.String[])
    L0000: mov      ecx, 0x33a4ca1c
    L0005: call     0x066f300c          ; new Person()
    L000a: mov      dword ptr [eax+4], 0xa  ; p.Age1 = 10
    L0011: mov      dword ptr [eax+8], 0x14 ; p.Age2 = 20
    L0018: ret
```

As you can see, the JIT compiler is smart enough to **inline** the simple auto-property setter, converting it into a direct memory write operation. The resulting machine code for setting the property is identical to that of setting the field. This means that **at runtime, the performance of an auto-property is the same as a public field.** You can safely dismiss this concern.

## The Future is Properties: C# 13 and the `field` Keyword

If you're still on the fence, there's more good news on the horizon. .NET 9 (with C# 13) is expected to finally introduce the `field` keyword. This feature makes it much easier to add logic to properties without manually declaring a backing field.

Previously, to write a "full property," you had to do this:

```csharp
private int _age;

public int Age
{
    get => _age / 2;
    set => _age = value * 2;
}
```

With the new `field` keyword, you can write it like this:

```csharp
public int Age
{
    get => field / 2;
    set => field = value * 2;
}
```

Here, `field` acts as an implicit, compiler-generated backing field. This further streamlines the process of creating properties with custom logic, reinforcing their central role in the language.

## Summary

Through this discussion, you should now have a deeper understanding of why C# development strongly favors properties over public fields. This doesn't mean public fields are forbidden—they have their place in specific scenarios like P/Invoke or certain game development frameworks. However, for general application and library development, properties are the superior choice.

The continued evolution of C#—from `init` setters in C# 9, to records, to primary constructors in C# 12, and now the upcoming `field` keyword—is consistently aimed at improving the experience of working with properties.

This should give you the confidence to embrace properties in your C# code without hesitation.
