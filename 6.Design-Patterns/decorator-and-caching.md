# Decorator & Caching

> A decorator wraps an existing implementation of an interface, adds behavior before/after, and passes the call through. The caller has no idea. The killer use case in backend work: adding caching to a service without touching the service.

---

## The idea

You have a working service:

```csharp
public interface IPlanService
{
    Task<IReadOnlyList<PlanDto>> GetPlansAsync(CancellationToken ct);
}

public class PlanService : IPlanService
{
    private readonly AppDbContext _context;

    public async Task<IReadOnlyList<PlanDto>> GetPlansAsync(CancellationToken ct)
        => await _context.Plans
            .Select(p => new PlanDto(p.Id, p.Name, p.Price))
            .AsNoTracking()
            .ToListAsync(ct);
}
```

Plans change maybe once a month, and every page hits this. Instead of stuffing cache code *into* PlanService (mixing data access with caching, making both harder to test), wrap it:

```csharp
public class CachedPlanService : IPlanService
{
    private readonly IPlanService _inner;      // the real one
    private readonly IMemoryCache _cache;

    public CachedPlanService(IPlanService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<IReadOnlyList<PlanDto>> GetPlansAsync(CancellationToken ct)
    {
        return (await _cache.GetOrCreateAsync("plans:all", entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
            return _inner.GetPlansAsync(ct);
        }))!;
    }
}
```

Same interface in, same interface out. `PlanService` still does exactly one thing (talk to the DB), `CachedPlanService` does exactly one thing (cache), and controllers keep injecting `IPlanService` with no idea a wrapper exists.

---

## Wiring it up in DI

The trick is registering so the decorator gets the real implementation:

```csharp
builder.Services.AddMemoryCache();
builder.Services.AddScoped<PlanService>();                 // concrete
builder.Services.AddScoped<IPlanService>(sp =>
    new CachedPlanService(
        sp.GetRequiredService<PlanService>(),
        sp.GetRequiredService<IMemoryCache>()));
```

Everyone asking for `IPlanService` now receives the cached wrapper. With the Scrutor NuGet package this collapses to one readable line:

```csharp
builder.Services.AddScoped<IPlanService, PlanService>();
builder.Services.Decorate<IPlanService, CachedPlanService>();
```

Decorators stack, order matters (last registered = outermost):

```csharp
builder.Services.Decorate<IPlanService, CachedPlanService>();
builder.Services.Decorate<IPlanService, LoggingPlanService>();
// call flow: Logging -> Cached -> real PlanService
```

Logging, retry (wrap calls in Polly policies), authorization checks, metrics: all the same shape.

---

## The caching side of things

Quick tour of the actual cache options, since the decorator is only half the story.

**IMemoryCache**: in-process dictionary with expiration. Fast, dead simple, gone on restart, and each server in a farm has its own copy.

```csharp
var plans = await _cache.GetOrCreateAsync("plans:all", entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
    entry.SlidingExpiration = TimeSpan.FromMinutes(10);
    return LoadPlansAsync();
});
```

Absolute expiration = hard deadline. Sliding = resets on every access (careful: a popular key with only sliding expiration never dies, pair it with an absolute cap).

**IDistributedCache / Redis**: shared across servers, survives app restarts, but values are byte[]/strings so you serialize, and every hit is a network hop. The move when you scale past one instance.

**HybridCache (.NET 9)**: two levels (memory + distributed) plus built-in stampede protection, with a nicer API. If it's available to you, prefer it over hand-rolling.

**Invalidation**, the famous hard problem. Expiration handles "eventually correct". When a write must be visible immediately, evict on write:

```csharp
public async Task UpdatePlanAsync(UpdatePlanDto dto, CancellationToken ct)
{
    await _inner.UpdatePlanAsync(dto, ct);
    _cache.Remove("plans:all");
}
```

which the decorator also handles nicely, since writes pass through it too.

---

## When NOT to use it

* **Data that changes constantly or must always be fresh** (balances, stock counts). Caching it creates bugs, not speed.
* **Queries that are already fast and rare.** Cache the top 3 hottest reads, not everything.
* **One more layer on a two-service app.** Decorators shine when the codebase is big enough that "don't touch working code" has value.

---

## Common Mistakes

* **Registering decorator and inner class against the same interface naively**, causing infinite recursion (decorator injecting itself). Register the concrete inner explicitly, or use Scrutor.
* **Caching entities instead of DTOs.** Tracked EF entities in a cache outlive their DbContext and go stale in weird ways. Cache plain DTOs.
* **No expiration** "because we evict on write", then a code path writes without evicting and the cache serves stale data forever. Always set a ceiling.
* **Cache stampede**: 200 requests miss simultaneously and all hit the DB. `GetOrCreateAsync` on IMemoryCache doesn't fully protect; HybridCache does, or a semaphore around the load.
* **Fat decorators.** The wrapper starts doing validation and mapping too, and now it's just a worse service. One concern per decorator.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Decorator | same interface, wraps inner, adds one behavior |
| Payoff | caching/logging/retry without touching working code |
| Wiring | factory registration or Scrutor's `.Decorate` |
| IMemoryCache | per-process, fast, fine for one server |
| Redis / HybridCache | shared cache for scaled-out apps |
| Invalidation | expiration ceiling always, evict-on-write when freshness matters |

> Related: [repository-and-unit-of-work.md](./repository-and-unit-of-work.md) (a natural decoration target) and [factory-and-strategy.md](./factory-and-strategy.md).
