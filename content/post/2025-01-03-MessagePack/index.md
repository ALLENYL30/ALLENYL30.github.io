+++
author = "yuhao"
title = "BSON vs MessagePack: Differences and How to Choose"
date = "2025-01-03"
description = "BSON (Binary JSON) is a binary-encoded JSON format developed by MongoDB, primarily used for data storage and transmission in MongoDB databases. It extends the JSON format by adding support for..."
tags = [
    ".NET",
    "C#",
    "Database",
]
categories = [
    "Programming/Development",
]
+++
## Basic Information

**BSON** (Binary JSON) is a binary-encoded JSON format developed by MongoDB, primarily used for data storage and transmission in MongoDB databases. It extends the JSON format by adding support for additional data types such as dates, timestamps, and binary data.

**MessagePack** is an efficient binary serialization format designed to achieve the smallest possible size and fastest possible processing speed. It supports multiple programming languages and is commonly used for network communication and data storage.

## Similarities

BSON and MessagePack share several similarities. First, both encode data in binary form, which can reduce data volume and improve transmission efficiency compared to text-based JSON. While JSON has the advantage of readability, its text nature can lead to slower parsing and greater storage space when handling large amounts of data. Binary formats avoid these issues by storing data in a way that computers can process more easily, thereby improving efficiency.

Second, they both support various data types, including basic types (such as integers, floating-point numbers, strings, boolean values) and complex types (such as arrays, binary data, dates). This allows them to flexibly handle various data structures and meet the needs of different application scenarios.

Furthermore, both provide implementations in multiple programming languages, enabling data exchange between different systems and platforms. Whether using Python, Java, C#, Go, or other programming languages, we can find corresponding libraries to use them. Finally, they both offer efficient serialization and deserialization mechanisms, making it convenient for developers to perform data-related operations.

So, being both binary data formats, do they both have similarly high space efficiency and performance?

## Differences

However, there are some key design differences between BSON and MessagePack. These differences lead to variations in their space efficiency and specialized functionalities.

BSON's design goal is primarily for MongoDB's data storage and transmission, rather than providing a superior alternative to JSON. This difference in design philosophy directly affects their performance in terms of space efficiency and performance. Since BSON includes some additional metadata (such as field lengths), its space efficiency is relatively lower; MessagePack uses compact representation methods to encode data, such as using fewer bytes to represent smaller integers, resulting in very high space efficiency. MessagePack's design philosophy is "as small as possible," making it an ideal choice in scenarios with strict data size requirements.

In terms of performance, BSON's serialization and deserialization speed is relatively slower, while MessagePack's is very fast. This is because MessagePack's encoding method is simpler and more efficient, reducing the computational load required for processing data. However, although BSON is relatively less efficient, it has many optimizations in MongoDB. For example, in terms of indexing, BSON's structure allows MongoDB to efficiently traverse and query data, which compensates for some of its shortcomings in general performance. Specifically, MongoDB can utilize information such as field lengths stored in BSON to quickly locate needed data, without needing to scan character by character as when parsing JSON.

## Code Comparison

Here, we use a simple example in C# to compare BSON and MessagePack. We use the Newtonsoft.Json.Bson and MessagePack libraries for implementation.

First, we design a class with some properties and populate it with data:

```csharp
public class MyModel
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
    public decimal Price { get; set; }
    public double Rating { get; set; }
    public Guid UniqueId { get; set; }
    public byte[] Data { get; set; }
    public List<string> Tags { get; set; }
    public Dictionary<string, int> Counts { get; set; }
    public long BigNumber { get; set; }
    public short SmallNumber { get; set; }
    public char Initial { get; set; }
    public TimeSpan Duration { get; set; }
    public Uri Website { get; set; }
    public Version Version { get; set; }
    public object DynamicValue { get; set; }
    public Status Status { get; set; }
}

public enum Status
{
    Pending,
    InProgress,
    Completed
}

var model = new MyModel
{
    Id = 123,
    Name = "My Model",
    CreatedAt = DateTime.UtcNow,
    IsActive = true,
    Price = 99.99m,
    Rating = 4.5,
    UniqueId = Guid.NewGuid(),
    Data = new byte[] { 1, 2, 3 },
    Tags = new List<string> { "tag1", "tag2" },
    Counts = new Dictionary<string, int> { { "a", 1 }, { "b", 2 } },
    BigNumber = 123456789012345,
    SmallNumber = 123,
    Initial = 'A',
    Duration = TimeSpan.FromMinutes(15),
    Website = new Uri("https://www.example.com"),
    Version = new Version(1, 0, 0),
    DynamicValue = new { Value = "dynamic" },
    Status = Status.InProgress
};
```

Then, we use two methods to serialize this object. First with BSON:

```csharp
using Newtonsoft.Json;
using Newtonsoft.Json.Bson;

using (var fs = File.OpenWrite(@"bson.dat"))
using (var writer = new BsonWriter(fs))
{
    var serializer = new JsonSerializer();
    serializer.Serialize(writer, model);
}
```

Then with MessagePack. To serialize the above model, we need to add some attributes to the model class, including:

- Add `[MessagePackObject]` to the class
- Add `[Key(0)]`, `[Key(1)]`, `[Key(2)]`, etc. to each property in sequence

Otherwise, the serialization code will throw an error. Here's the modified model class (partial) and serialization code:

```csharp
[MessagePackObject]
public class MyModel
{
    [Key(0)]
    public int Id { get; set; }
    [Key(1)]
    public string Name { get; set; }
    [Key(2)]
    public DateTime CreatedAt { get; set; }
    [Key(3)]
    public bool IsActive { get; set; }
    // ...
}

using (var fs = File.OpenWrite(@"msgpack.dat"))
{
    var data = MessagePackSerializer.Serialize(model);
    fs.Write(data);
}
```

Then, we observe the file sizes:

- BSON: 381 bytes
- MessagePack: 166 bytes

It's clear that BSON has no advantage in terms of space efficiency.

> **Note**
>
> In fact, even when compared to JSON, BSON may not necessarily have significant advantages in space efficiency. BSON's advantages mainly come from its handling of dates, binary data, numbers, etc., while JSON can only store everything as strings. For a large number like 100,000,000, JSON needs 9 characters, while BSON only needs 4 bytes; but for a small decimal like 1.0, JSON needs 3 characters, while BSON needs 8 bytes. Moreover, BSON also includes additional metadata, such as field names and field lengths, which also means that BSON's space efficiency does not show much improvement compared to JSON.

## Summary

In conclusion, both BSON and MessagePack are excellent and modern binary serialization formats. BSON's advantages lie in its compatibility with JSON and support for rich data types, making it excel in database applications like MongoDB; MessagePack's advantages lie in its high performance and high space efficiency, making it more advantageous in scenarios such as network communication and big data processing that require fast transmission and processing of large amounts of data. In practical applications, we should choose the appropriate format based on specific requirements.
