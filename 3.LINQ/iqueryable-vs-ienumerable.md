# IQueryable vs IEnumerable

> They look identical in code, both give you Where/Select/OrderBy. The difference: `IEnumerable` runs your lambdas as C# code on objects in memory, `IQueryable` records them as expression trees and lets a provider (EF Core) translate them to SQL. Getting this wrong means downloading the table to filter it in RAM.

---

## The same line, two meanings

```csharp
IEnumerable<Member> membersE = _context.Members;
IQueryable<Member>  membersQ = _context.Members;

var activeE = membersE.Where(m => m.IsActive).Take(10).ToList();
var activeQ = membersQ.Where(m => m.IsActive).Take(10).ToList();
```

Both compile. Both return the same 10 members. But:

* **IQueryable version:** SQL sent is `SELECT TOP(10) ... WHERE IsActive = 1`. The database does the work, 10 rows travel over the wire.
* **IEnumerable version:** SQL sent is `SELECT * FROM Members`. The *entire table* is streamed into memory, then C# filters it and takes 10.

Same syntax, wildly different cost. On a 5-row dev database you'll never notice. On the 2-million-row production table you will.

---

## Why: Func vs Expression

Look at the actual signatures:

```csharp
// IEnumerable version (LINQ to Objects)
Where(Func<Member, bool> predicate)

// IQueryable version
Where(Expression<Func<Member, bool>> predicate)
```

A `Func` is compiled code. A black box. You can only *run* it.

An `Expression<Func<...>>` is a **data structure describing the lambda**: "property IsActive of parameter m, compared with true". The compiler builds this tree from the same lambda text. Because EF can *inspect* the tree instead of running it, it can translate it:

```text
m => m.IsActive && m.Age > 18

        AndAlso
       /       \
  m.IsActive   GreaterThan
               /         \
            m.Age         18
```

becomes `WHERE IsActive = 1 AND Age > 18`.

That's the entire magic of EF Core queries: your lambdas are never executed for DB queries, they're read like a spec and rewritten as SQL.

---

## Where the line gets crossed

The moment a query becomes `IEnumerable`, everything after it happens in memory:

```csharp
var result = _context.Members
    .Where(m => m.IsActive)      // IQueryable, goes to SQL
    .AsEnumerable()              // <- the border
    .Where(m => MyComplexCheck(m))  // C# code, runs in memory
    .ToList();
```

Things that cross the border: `AsEnumerable()`, `ToList()`, `ToArray()`, `foreach`. Declaring your variable or repository return type as `IEnumerable<T>` does it too, silently:

```csharp
// repository that ruins everything downstream
public IEnumerable<Member> GetMembers() => _context.Members;

// caller thinks this filters in SQL. It doesn't.
var page = repo.GetMembers().Where(m => m.IsActive).Take(20).ToList();
```

Sometimes crossing deliberately is correct: filter as much as possible in SQL, then `AsEnumerable()`, then apply the C# logic EF can't translate. Just make sure the SQL part already shrank the data.

---

## "Could not be translated"

EF can only translate what it understands. Call your own method inside the lambda and you get the famous runtime error:

```csharp
var adults = await _context.Members
    .Where(m => IsAdult(m))       // your C# method, EF has no idea
    .ToListAsync();               // InvalidOperationException: could not be translated
```

Fixes, in order of preference:

1. Inline the logic so it's translatable: `.Where(m => m.BirthDate <= DateTime.Today.AddYears(-18))`
2. Use EF helpers: `EF.Functions.Like(m.Name, "%ali%")`
3. Genuinely untranslatable? Filter what you can in SQL, then `AsEnumerable()` and finish in memory.

Note it's a *runtime* error, not compile time. The compiler happily builds an expression tree from almost anything; only the provider knows what it supports. This is why queries need integration-level testing against a real provider.

---

## Deferred execution applies to both

Neither interface does anything until enumerated:

```csharp
var query = _context.Members.Where(m => m.IsActive);   // no SQL yet

var list = await query.ToListAsync();    // SQL happens here
var count = await query.CountAsync();    // and AGAIN here, second DB hit
```

Every enumeration is a fresh database round trip. Materialize once with `ToListAsync()` if you need the data multiple times. (Deferred execution mechanics are in [../1.CSharp-Advanced/iterators-and-yield.md](../1.CSharp-Advanced/iterators-and-yield.md).)

---

## Practical rules

* Inside data access code, keep things `IQueryable` while composing, so filters/paging/ordering reach the SQL.
* From a repository or service, return **materialized** results (`List<T>`) or a DTO, not a live `IQueryable`, so callers can't accidentally extend queries or trigger them after the context is disposed. (Leaking IQueryable vs returning lists is a genuine design debate; my default is materialize at the service boundary.)
* Compose freely before materializing: EF builds ONE SQL query from the whole chain.

```csharp
IQueryable<Member> query = _context.Members;

if (!string.IsNullOrEmpty(search))
    query = query.Where(m => m.Name.Contains(search));

if (planId is not null)
    query = query.Where(m => m.PlanId == planId);

var page = await query
    .OrderBy(m => m.Name)
    .Skip((pageNo - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();     // one SQL statement with all conditions
```

Dynamic filtering like this is where IQueryable really shines.

---

## Common Mistakes

* **`IEnumerable<T>` return types in repositories** turning later Where/Take into in-memory operations on the full table.
* **`ToList()` early** "to be safe", then filtering. Same table-download problem, self-inflicted.
* **Counting like `GetAll().Count()`** where GetAll materializes. That's `SELECT *` to answer a `COUNT(*)` question.
* **Assuming compile = translatable.** Translation errors appear at runtime only.
* **Enumerating the same IQueryable multiple times** without realizing each one is a DB query.

---

## Summary

| Idea | Point |
| ---- | ----- |
| IEnumerable | lambdas are `Func`, executed in memory |
| IQueryable | lambdas are `Expression`, translated by the provider |
| The border | AsEnumerable/ToList/foreach; everything after runs in RAM |
| Could not be translated | runtime error; inline logic or use EF.Functions |
| Rule | filter/order/page while still IQueryable, materialize once |

> Pairs with [linq-advanced.md](./linq-advanced.md) and explains half the advice in [../2.EF-Core/performance-and-advanced-queries.md](../2.EF-Core/performance-and-advanced-queries.md).
