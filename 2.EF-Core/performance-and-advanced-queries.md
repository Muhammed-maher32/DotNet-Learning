# Performance & Advanced Queries

> A grab bag of the EF Core features you reach for when the basics are correct but slow, or when LINQ can't express what you need: projections, split queries, bulk operations, raw SQL, global query filters, and interceptors.

---

## Rule zero: select less

Before any fancy feature, the biggest win is almost always the same: stop loading whole entities when you need three columns.

```csharp
// loads every column, tracks every entity
var members = await _context.Members.Include(m => m.Subscriptions).ToListAsync();

// loads exactly what the page shows
var members = await _context.Members
    .Where(m => m.IsActive)
    .Select(m => new MemberListDto
    {
        Id = m.Id,
        Name = m.Name,
        ActivePlan = m.Subscriptions
            .Where(s => s.EndDate > DateTime.UtcNow)
            .Select(s => s.Plan.Name)
            .FirstOrDefault()
    })
    .AsNoTracking()
    .ToListAsync();
```

Projection with `Select` means smaller SQL, no tracking overhead, and no `Include` needed at all (EF builds the join from the projection). This plus `AsNoTracking` solves most "EF is slow" complaints.

---

## Split queries

One `Include` with big collections can explode into a huge joined resultset (cartesian explosion): 100 members x 50 subscriptions each = 5000 rows with member data repeated 50 times.

```csharp
var members = await _context.Members
    .Include(m => m.Subscriptions)
    .Include(m => m.Payments)
    .AsSplitQuery()
    .ToListAsync();
```

`AsSplitQuery` runs one SQL query per include instead of one giant join. Trade-off: multiple round trips, and no single consistent snapshot unless you wrap it in a transaction. Use it when includes multiply the row count badly.

---

## ExecuteUpdate / ExecuteDelete

The old way to deactivate 10,000 members: load all of them, loop, set a flag, SaveChanges. 10,000 rows pulled into memory to flip a bit.

```csharp
// one UPDATE statement, nothing loaded
await _context.Members
    .Where(m => m.LastVisit < cutoff)
    .ExecuteUpdateAsync(s => s.SetProperty(m => m.IsActive, false));

await _context.AuditLogs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();
```

Massive win for bulk operations. Two things to remember: they bypass the change tracker (already-loaded entities won't know they changed), and they run immediately, not at SaveChanges time.

---

## Raw SQL when LINQ gives up

```csharp
var results = await _context.Members
    .FromSqlInterpolated($"SELECT * FROM Members WHERE Name LIKE {pattern}")
    .Where(m => m.IsActive)          // you can still compose LINQ on top
    .ToListAsync();
```

Notes:

* `FromSqlInterpolated` parameterizes the interpolated values for you. Safe from SQL injection. Never build the string with `+`.
* The query must return all columns of the entity.
* For non-entity results there's `SqlQuery<T>`, and for statements `ExecuteSqlInterpolatedAsync`.

Use raw SQL for the 2% of queries that are genuinely awkward in LINQ (window functions, vendor-specific hints), not as an escape hatch from learning LINQ.

---

## Compiled queries

EF translates LINQ to SQL and caches the translation, so normally you don't care. For extremely hot queries (called thousands of times/sec) you can skip even the cache lookup:

```csharp
private static readonly Func<AppDbContext, int, Task<Member?>> GetMemberById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Members.FirstOrDefault(m => m.Id == id));

var member = await GetMemberById(_context, 5);
```

Honestly a micro-optimization. Measure first; most apps never need this.

---

## Global query filters

A predicate applied to *every* query of an entity, defined once in `OnModelCreating`:

```csharp
modelBuilder.Entity<Member>()
    .HasQueryFilter(m => !m.IsDeleted);
```

Now `_context.Members.ToList()` silently adds `WHERE IsDeleted = 0` everywhere. This is how you implement soft delete without sprinkling the condition into every query (and inevitably forgetting one). Also the standard tool for multi-tenancy (`WHERE TenantId = current`).

Opting out for the admin screen:

```csharp
var everything = await _context.Members.IgnoreQueryFilters().ToListAsync();
```

Gotcha: a required navigation to a filtered entity can make rows vanish unexpectedly. If Subscriptions requires a Member and the member is filtered out, the subscription disappears from query results too.

---

## Interceptors

Hooks into EF's pipeline. The classic use: setting audit fields in one place instead of remembering in every service.

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        foreach (var entry in eventData.Context!.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = DateTime.UtcNow;
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// registration
options.UseSqlServer(conn).AddInterceptors(new AuditInterceptor());
```

There are also command interceptors (see/modify SQL before it runs) and connection interceptors. Good for cross-cutting stuff: auditing, soft-delete conversion (turn deletes into updates), logging slow queries.

---

## Seeing what EF actually does

You can't fix what you can't see. Cheapest tool:

```csharp
options.UseSqlServer(conn)
       .LogTo(Console.WriteLine, LogLevel.Information)   // dumps the SQL
       .EnableSensitiveDataLogging();                     // + parameter values (dev only!)
```

Read the SQL for your slowest pages once in a while. N+1 patterns and missing indexes jump out immediately.

---

## Common Mistakes

* **Optimizing before looking at the SQL.** Half the time the "EF problem" is a missing index or an accidental N+1.
* **`ToList()` too early.** Everything after it runs in memory. Filter and project *before* materializing.
* **Mixing ExecuteUpdate with tracked entities** and being surprised the in-memory objects are stale.
* **String-concatenated raw SQL.** Injection. Always interpolated/parameterized.
* **Forgetting query filters exist** and spending an hour wondering why a row "isn't in the database".

---

## Summary

| Idea | Point |
| ---- | ----- |
| Projection + AsNoTracking | the fix for most read performance issues |
| AsSplitQuery | avoid cartesian explosion from multiple includes |
| ExecuteUpdate/Delete | bulk changes without loading entities |
| FromSqlInterpolated | raw SQL, parameterized, composable with LINQ |
| Query filters | soft delete / multi-tenancy declared once |
| Interceptors | audit fields and cross-cutting concerns in one place |

> Builds on [loading-and-tracking.md](./loading-and-tracking.md). The IQueryable mechanics behind all of this are in [../3.LINQ/iqueryable-vs-ienumerable.md](../3.LINQ/iqueryable-vs-ienumerable.md).
