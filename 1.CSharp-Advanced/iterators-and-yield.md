# Iterators & yield

> `yield return` lets a method produce a sequence one item at a time, only when someone actually asks for the next item. That "only when asked" part is the whole story, and it's the same idea behind deferred execution in LINQ.

---

## The manual way first

`foreach` works on anything that implements `IEnumerable<T>`. Under the hood it asks for an `IEnumerator<T>` and keeps calling `MoveNext()`:

```csharp
var list = new List<int> { 1, 2, 3 };

using var enumerator = list.GetEnumerator();
while (enumerator.MoveNext())
{
    Console.WriteLine(enumerator.Current);
}
```

That's literally what the compiler turns a `foreach` into. `IEnumerable` = "you can iterate me", `IEnumerator` = "the cursor doing the iterating".

Writing an enumerator by hand means building a class with `MoveNext`, `Current`, and state fields to remember where you stopped. Painful. That's what `yield` saves you from.

---

## yield return

```csharp
public IEnumerable<int> GetNumbers()
{
    Console.WriteLine("starting");
    yield return 1;
    Console.WriteLine("after 1");
    yield return 2;
    Console.WriteLine("after 2");
    yield return 3;
}
```

The surprising part:

```csharp
var numbers = GetNumbers();   // prints NOTHING. The method body did not run.

foreach (var n in numbers)    // NOW it runs, piece by piece
    Console.WriteLine(n);
```

Output:

```text
starting
1
after 1
2
after 2
3
```

Calling the method just hands you an object. The body executes lazily, pausing at each `yield return` and resuming from that exact spot on the next `MoveNext()`. The compiler generates a state machine class for you (same trick it does for `async`).

`yield break` stops the sequence early:

```csharp
public IEnumerable<int> TakeWhilePositive(IEnumerable<int> source)
{
    foreach (var n in source)
    {
        if (n <= 0) yield break;
        yield return n;
    }
}
```

---

## Why this matters: deferred execution

This is exactly how LINQ operators like `Where` and `Select` are built. `Where` is roughly:

```csharp
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate)
{
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}
```

So when you write:

```csharp
var query = members.Where(m => m.IsActive);   // nothing happens yet
```

nothing has been filtered. The work happens during the `foreach` (or `ToList`, `Count`, etc.). Two consequences you will hit in real code:

1. **The query re-runs every time you enumerate it.** Enumerate twice, filter twice.
2. **The query sees the data as it is at enumeration time**, not at definition time. If the list changed in between, results change.

```csharp
var list = new List<int> { 1, 2, 3 };
var query = list.Where(n => n > 1);

list.Add(10);

Console.WriteLine(query.Count());   // 3, not 2. The 10 is included.
```

---

## Streaming instead of buffering

The practical win: you can process huge data without holding it all in memory.

```csharp
public IEnumerable<string> ReadLargeFile(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) != null)
        yield return line;
}

// only ever one line in memory at a time
foreach (var line in ReadLargeFile("huge-log.txt"))
    if (line.Contains("ERROR"))
        Console.WriteLine(line);
```

Compare that with `File.ReadAllLines` which loads the whole file into an array first.

---

## Common Mistakes

* **Enumerating twice by accident.** Passing an `IEnumerable<T>` around and calling `Count()` then `foreach` runs the whole pipeline twice. If the source is a DB query, that's two DB hits. Materialize with `ToList()` once if you need multiple passes.
* **Assuming the exception is thrown at call time.** Argument checks inside an iterator method don't run until first enumeration. If you want eager validation, split into two methods: a normal one that validates and returns a private iterator.
* **Modifying the collection while iterating it.** `List<T>` throws `InvalidOperationException` if it changes mid-`foreach`.
* **`yield` inside try/catch limits.** You can `yield return` inside `try` with `finally`, but not inside a `try` that has a `catch`. The compiler will tell you.

---

## Summary

| Idea | Point |
| ---- | ----- |
| `IEnumerable` / `IEnumerator` | the "can be iterated" contract vs the cursor doing it |
| `yield return` | compiler builds the state machine, body runs lazily |
| Deferred execution | nothing runs until enumeration; re-runs on every enumeration |
| Streaming | process one item at a time, no giant buffers |
| `ToList()` | the off switch for laziness when you need it |

> This connects directly to [../3.LINQ/linq.md](../3.LINQ/linq.md) (deferred execution) and to [async-programming.md](./async-programming.md), since `async`/`await` uses the same compiler state machine trick.
