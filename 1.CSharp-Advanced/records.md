# Records & Value vs Reference Semantics

> A record is a class (or struct) where the compiler writes equality, hashing, and a nice ToString for you, based on the values inside instead of the reference.

---

## First, the thing records fix: reference equality

Normal classes compare by **reference**. Two objects with identical data are still "not equal" because they live at different memory addresses.

```csharp
public class PointClass
{
    public int X { get; set; }
    public int Y { get; set; }
}

var a = new PointClass { X = 1, Y = 2 };
var b = new PointClass { X = 1, Y = 2 };

Console.WriteLine(a == b);        // False  (different references)
Console.WriteLine(a.Equals(b));   // False
```

A `struct` compares by **value**, but you write structs less often and they have copy semantics that can surprise you.

---

## What a record does

```csharp
public record PointRecord(int X, int Y);

var a = new PointRecord(1, 2);
var b = new PointRecord(1, 2);

Console.WriteLine(a == b);        // True  (same values)
Console.WriteLine(a.Equals(b));   // True
Console.WriteLine(a);             // PointRecord { X = 1, Y = 2 }
```

That `(int X, int Y)` shorthand is a **positional record**. The compiler generates:

* a constructor
* public init-only properties `X` and `Y`
* value based `Equals` and `GetHashCode`
* a readable `ToString`
* deconstruction (`var (x, y) = point;`)

You can also write it long form if you want more control:

```csharp
public record PointRecord
{
    public int X { get; init; }
    public int Y { get; init; }
}
```

---

## Immutability and `with`

Record properties are usually `init` only, so you can't mutate them after construction. To "change" one, you make a copy with the change applied. That's the `with` expression:

```csharp
var a = new PointRecord(1, 2);
var c = a with { Y = 99 };   // new record: X=1, Y=99, `a` is untouched

Console.WriteLine(a);  // X = 1, Y = 2
Console.WriteLine(c);  // X = 1, Y = 99
```

This is non-destructive mutation. Great for DTOs and anything you want to treat as read-only data.

---

## record vs record struct

```csharp
public record         Foo(int X);  // reference type, value equality
public record struct  Bar(int X);  // value type, value equality
```

* `record` -> still a reference type (lives on the heap, copied by reference), but equality is by value.
* `record struct` -> a value type (copied by value), also with value equality.

Most of the time you just want `record`.

---

## When to use what

| Use | When |
| --- | ---- |
| `class` | objects with identity and behavior (services, entities you mutate) |
| `record` | data you compare by value and mostly don't mutate (DTOs, ViewModels, messages, value objects) |
| `struct` | small, short lived value types where you care about allocations |
| `record struct` | same as struct but you want free value equality |

---

## Reference vs value, the one-liner

> Reference type: the variable holds an **address**. Copying the variable copies the address, both point to the same object.
> Value type: the variable holds the **data itself**. Copying it copies the whole thing.

```csharp
// reference type
var x = new PointClass { X = 1 };
var y = x;
y.X = 50;
Console.WriteLine(x.X); // 50  (same object)

// value type (struct)
PointStruct p = new() { X = 1 };
PointStruct q = p;
q.X = 50;
Console.WriteLine(p.X); // 1   (independent copy)
```

Records sit on top of this: a `record` is a reference type, but it borrows value type **equality** without borrowing value type **copying**.
