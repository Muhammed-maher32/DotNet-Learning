# Advanced LINQ

> The operators past Where/Select/OrderBy: GroupBy, the join family, SelectMany for flattening, set operators, and Aggregate. These are the ones that turn "loop with a dictionary" code into one readable query.

---

## GroupBy

Splits a sequence into buckets by a key. Each bucket is an `IGrouping<TKey, TElement>`: it has a `.Key` and it *is* the sequence of items.

```csharp
var byPlan = members.GroupBy(m => m.PlanId);

foreach (var group in byPlan)
{
    Console.WriteLine($"Plan {group.Key}: {group.Count()} members");
    foreach (var m in group)
        Console.WriteLine($"  {m.Name}");
}
```

In practice you almost always project the group immediately:

```csharp
var report = members
    .GroupBy(m => m.PlanId)
    .Select(g => new
    {
        PlanId = g.Key,
        Count = g.Count(),
        Newest = g.Max(m => m.JoinDate)
    })
    .ToList();
```

That shape (`GroupBy` then `Select` with aggregates) translates to `GROUP BY` in SQL when used through EF Core. Grouping by multiple keys uses an anonymous type: `GroupBy(m => new { m.PlanId, m.Gender })`.

EF gotcha: SQL GROUP BY can only return keys and aggregates. If you group in EF and then try to iterate the items *inside* each group, EF either translates it differently or complains. Aggregate in the query; if you truly need the items per group, group in memory after fetching.

---

## Join

Matches elements from two sequences on equal keys, like an INNER JOIN:

```csharp
var query = members.Join(
    plans,
    m => m.PlanId,      // key from members
    p => p.Id,          // key from plans
    (m, p) => new { m.Name, Plan = p.Name });
```

Real talk though: with EF Core you rarely write `Join` by hand. If your entities have navigation properties, just use them:

```csharp
var query = _context.Members
    .Select(m => new { m.Name, Plan = m.Plan.Name });   // EF makes the join
```

Manual `Join` is for when there's no navigation (unrelated tables, in-memory lists from different sources).

`GroupJoin` is the "left join, grouped" version: each left element paired with a *collection* of matches. The classic left-join-with-nulls pattern is `GroupJoin` + `SelectMany` + `DefaultIfEmpty`, which is ugly enough that I look it up every time. Navigations spare you this too.

---

## SelectMany

`Select` maps one item to one result. `SelectMany` maps one item to a *sequence* and flattens all the sequences into one.

```csharp
// List<Member>, each member has List<Subscription>

var allSubs = members.Select(m => m.Subscriptions);      // IEnumerable<List<Subscription>>, nested
var allSubs = members.SelectMany(m => m.Subscriptions);  // IEnumerable<Subscription>, flat
```

With the second overload you keep the parent too, which is the useful form:

```csharp
var pairs = members.SelectMany(
    m => m.Subscriptions,
    (member, sub) => new { member.Name, sub.StartDate });
```

Whenever you catch yourself with nested foreach loops building a flat list, that's SelectMany.

---

## Set operators

```csharp
var a = new[] { 1, 2, 3, 4 };
var b = new[] { 3, 4, 5 };

a.Distinct();      // 1 2 3 4 (removes dups)
a.Union(b);        // 1 2 3 4 5
a.Intersect(b);    // 3 4
a.Except(b);       // 1 2
```

For objects they compare by reference unless the type overrides equality, which is why these pair so nicely with records. Since .NET 6 there are `DistinctBy`, `UnionBy`, `ExceptBy`, `IntersectBy` that take a key selector, plus `MaxBy`/`MinBy` which return the *element*, not the value:

```csharp
var newest = members.MaxBy(m => m.JoinDate);      // the Member, not the date
var uniqueByEmail = members.DistinctBy(m => m.Email);
```

`MaxBy` replaces the old `OrderByDescending(...).First()` idiom.

---

## Aggregate

The general fold: carry an accumulator through the sequence.

```csharp
var total = numbers.Aggregate((acc, n) => acc + n);            // same as Sum
var csv = names.Aggregate((acc, n) => acc + ", " + n);         // string.Join is better
var product = numbers.Aggregate(1, (acc, n) => acc * n);       // with a seed
```

Truthfully I almost never use it. `Sum`, `Count`, `Min`, `Max`, `Average`, and `string.Join` cover the real cases, and a plain foreach is often clearer than a clever Aggregate. Know it exists, reach for it last.

---

## Chunk, Zip, and friends

Small ones worth knowing:

```csharp
foreach (var batch in ids.Chunk(100))      // arrays of up to 100, great for batching DB calls
    await ProcessBatch(batch);

var paired = names.Zip(scores);            // tuples (name, score) until the shorter runs out

var page = members
    .OrderBy(m => m.Id)
    .Skip((pageNumber - 1) * pageSize)     // paging, always AFTER OrderBy
    .Take(pageSize);
```

---

## Common Mistakes

* **GroupBy then iterating group contents in an EF query.** SQL can't do it; aggregate in the query or group in memory.
* **Manual Join where a navigation property exists.** More code, same SQL, easier to get wrong.
* **Skip/Take without OrderBy** against a database. Order is not guaranteed, pages will overlap or skip rows.
* **Distinct on classes without value equality** and wondering why duplicates remain.
* **Building strings with Aggregate.** O(n^2) allocations. `string.Join` exists.

---

## Summary

| Idea | Point |
| ---- | ----- |
| GroupBy | buckets with `.Key`; project to aggregates for SQL translation |
| Join / GroupJoin | manual joins; prefer navigation properties in EF |
| SelectMany | flatten nested sequences, replaces nested foreach |
| Set ops + *By variants | Distinct, Union, Except; DistinctBy/MaxBy since .NET 6 |
| Aggregate | general fold, rarely the clearest option |
| Chunk / Skip / Take | batching and paging |

> Continues [linq.md](./linq.md). How these translate (or fail to translate) to SQL is the subject of [iqueryable-vs-ienumerable.md](./iqueryable-vs-ienumerable.md).
