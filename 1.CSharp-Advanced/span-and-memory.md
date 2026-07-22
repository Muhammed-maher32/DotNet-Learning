# Span\<T\> & Memory\<T\>

> `Span<T>` is a window over existing memory (an array, a string, the stack) that lets you slice and read it without copying anything. It exists for one reason: killing allocations in hot paths.

---

## The problem: hidden copies everywhere

String and array APIs in .NET love to allocate:

```csharp
string input = "2026-07-22T14:30:00";

string datePart = input.Substring(0, 10);   // NEW string on the heap
string[] parts = input.Split('T');          // NEW array + NEW strings
```

Each of those creates fresh heap objects the GC has to clean up later. In a controller that runs once, who cares. In a parser processing millions of lines, this is where your performance goes.

---

## Span\<T\>: a view, not a copy

```csharp
string input = "2026-07-22T14:30:00";

ReadOnlySpan<char> span = input.AsSpan();
ReadOnlySpan<char> datePart = span.Slice(0, 10);   // NO allocation, just a view

// most parsing APIs accept spans now
DateOnly date = DateOnly.Parse(datePart);
```

`Slice` doesn't copy characters. It produces a struct holding a pointer + length into the *same* memory. Same idea with arrays:

```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6 };

Span<int> middle = numbers.AsSpan(2, 3);   // view over {3, 4, 5}
middle[0] = 99;                            // writes through!

Console.WriteLine(numbers[2]);             // 99
```

`Span<T>` is read/write, `ReadOnlySpan<T>` is read-only (strings only give you the read-only one, strings are immutable).

---

## stackalloc: skip the heap entirely

For small temporary buffers you can allocate on the stack, which is free to clean up:

```csharp
Span<byte> buffer = stackalloc byte[64];

if (BitConverter.TryWriteBytes(buffer, 123456L))
{
    // use buffer, no heap allocation happened at all
}
```

Keep stackalloc small (the stack is ~1 MB total). A common pattern is stack for small sizes, rent from a pool for big ones:

```csharp
Span<char> buffer = length <= 256
    ? stackalloc char[256]
    : new char[length];
```

---

## The big restriction

`Span<T>` is a `ref struct`. It may point into the stack, so the compiler forbids anything that could make it outlive the current method:

* can't be a field of a normal class
* can't be used in `async` methods or iterators (`yield`)
* can't be boxed or put in a `List<Span<T>>`

```csharp
public async Task DoWork()
{
    Span<char> s = stackalloc char[10];   // compiler error in async method
}
```

That's what `Memory<T>` is for. It's the heap-safe cousin: same "view over memory" idea, but a normal struct you can store in fields and pass through `async`:

```csharp
public async Task ProcessAsync(Memory<byte> data)
{
    await stream.ReadAsync(data);          // async APIs take Memory<T>
    Span<byte> span = data.Span;           // convert to Span for the sync work
    Parse(span);
}
```

Rule of thumb: `Memory<T>` at the async boundaries, `.Span` once you're in synchronous code.

---

## Where you actually meet this

Honestly, in day-to-day CRUD backend work you mostly *consume* span-based APIs rather than write them:

* `int.TryParse(ReadOnlySpan<char>, out ...)`, `DateOnly.Parse(span)` and friends
* `string.Create`, `MemoryExtensions` (`span.Trim()`, `span.IndexOf(...)`)
* `Stream.ReadAsync(Memory<byte>)`
* Anything in `System.Text.Json`, which is span-based top to bottom

Reach for spans yourself when profiling shows allocation pressure in parsing/formatting code. Not before.

---

## Common Mistakes

* **Optimizing cold code.** Rewriting a controller action to use spans saves nothing measurable. This is a hot-path tool.
* **Trying to store a Span in a class field or async method.** Compiler stops you. Use `Memory<T>` instead.
* **Huge stackalloc.** Stack overflow is not catchable. Keep it in the low hundreds of bytes/elements.
* **Forgetting spans are views.** Writing through a `Span<int>` mutates the original array. Sometimes that's the point, sometimes it's a bug.

---

## Summary

| Idea | Point |
| ---- | ----- |
| `Span<T>` | stack-only view over memory; slice without copying |
| `ReadOnlySpan<char>` | how you slice strings for free |
| `stackalloc` | small temp buffers with zero GC cost |
| `Memory<T>` | heap-safe version for fields and async |
| When | hot paths with proven allocation pressure, otherwise skip it |

> Related: [collections.md](./collections.md) for the normal-day data structures, and [async-programming.md](./async-programming.md) for why `Span<T>` can't cross an `await`.
