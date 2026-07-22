# Transactions & Concurrency

> Two separate problems that live in the same neighborhood. Transactions: making several changes succeed or fail as one unit. Concurrency: two users editing the same row at the same time, and deciding who wins.

---

## SaveChanges is already a transaction

First thing to know: every `SaveChanges()` call is wrapped in a transaction by EF Core automatically.

```csharp
_context.Members.Add(member);
_context.Subscriptions.Add(subscription);
await _context.SaveChangesAsync();   // both INSERTs in ONE transaction
```

If the second insert fails, the first is rolled back. You get atomicity for free as long as everything happens in a single `SaveChanges`.

So the honest rule: **if you can batch your changes into one SaveChanges, you don't need an explicit transaction.** Most of the time you can.

---

## Explicit transactions

You need one when the unit of work spans multiple `SaveChanges` calls, for example when you need a generated Id in the middle:

```csharp
await using var transaction = await _context.Database.BeginTransactionAsync();

try
{
    _context.Members.Add(member);
    await _context.SaveChangesAsync();           // need member.Id now

    _context.AuditLogs.Add(new AuditLog
    {
        EntityId = member.Id,
        Action = "MemberCreated"
    });
    await _context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Without the transaction, a crash between the two saves would leave a member with no audit log. With it, either both land or neither.

A detail that bites people: if you use `EnableRetryOnFailure` (connection resiliency), you can't just call `BeginTransactionAsync`, you have to wrap the whole thing in the execution strategy:

```csharp
var strategy = _context.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    await using var tx = await _context.Database.BeginTransactionAsync();
    // ... saves ...
    await tx.CommitAsync();
});
```

---

## The concurrency problem

Classic lost update:

1. Admin A opens member #5 for editing. Phone = "0100".
2. Admin B opens the same member. Phone = "0100".
3. A saves phone "0111". Fine.
4. B saves email change... and B's stale phone "0100" goes with it, silently wiping A's update.

Nobody got an error. Data just quietly disappeared. That's the default behavior ("last writer wins") if you do nothing.

---

## Optimistic concurrency with a row version

The fix used almost everywhere: add a version column that the database changes on every update.

```csharp
public class Member
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
    public string Phone { get; set; } = null!;

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
}
```

Or fluent:

```csharp
builder.Property(m => m.RowVersion).IsRowVersion();
```

Now EF includes the version in every UPDATE's WHERE clause:

```sql
UPDATE Members SET Phone = @p0
WHERE Id = @p1 AND RowVersion = @p2
```

If someone updated the row since you read it, the version no longer matches, zero rows are affected, and EF throws `DbUpdateConcurrencyException`. Nothing gets silently overwritten.

It's called *optimistic* because you don't lock anything. You assume conflicts are rare and just detect them when they happen. (The pessimistic alternative, locking rows while someone edits, doesn't fit web apps: the user might close the tab and hold the lock forever.)

---

## Handling DbUpdateConcurrencyException

You have to decide the policy. Usually: tell the user and let them retry with fresh data.

```csharp
try
{
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync();

    if (dbValues == null)
        return Result.NotFound("Member was deleted by someone else.");

    // simplest sane policy: refresh and ask the user to re-apply
    entry.OriginalValues.SetValues(dbValues);
    return Result.Fail("This member was modified by someone else. Reload and try again.");
}
```

For web apps there's one more piece: the row version must round-trip through the form or the API (hidden field / DTO property), so that you're checking against the version *the user saw*, not the version you just re-fetched.

---

## ConcurrencyCheck on a normal column

Alternative to rowversion: mark specific properties with `[ConcurrencyCheck]` and EF puts their old values in the WHERE clause. Works, but rowversion is cheaper and catches changes to *any* column, so it's the default choice.

---

## Common Mistakes

* **Opening explicit transactions around a single SaveChanges.** Pure noise, it's already transactional.
* **Long transactions.** Don't call external APIs or do slow work inside an open transaction; you're holding locks the whole time.
* **Swallowing DbUpdateConcurrencyException.** Catching it and retrying `SaveChanges` in a loop without refreshing values just throws again forever.
* **Forgetting to round-trip RowVersion in the UI/DTO.** Then you always compare against the freshly loaded version and never detect a real conflict.

---

## Summary

| Idea | Point |
| ---- | ----- |
| `SaveChanges` | already atomic, one transaction per call |
| Explicit transaction | only when the unit spans multiple SaveChanges |
| Lost update | default behavior with no concurrency token, silent data loss |
| `[Timestamp]` / rowversion | version in WHERE clause, conflict = `DbUpdateConcurrencyException` |
| Handling | refresh, inform the user, let them retry with current data |

> Builds on [introduction.md](./introduction.md) and [loading-and-tracking.md](./loading-and-tracking.md). The Result returns in the examples are the pattern from [../6.Design-Patterns/result-pattern.md](../6.Design-Patterns/result-pattern.md).
