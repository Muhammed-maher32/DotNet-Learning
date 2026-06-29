# Repository & Unit of Work

> Repository = a class that wraps data access for one entity type so the rest of your code says `repo.GetById(5)` instead of writing EF queries everywhere. Unit of Work = the thing that owns the shared DbContext and the single `SaveChanges()`, so multiple repositories commit together.

These two almost always show up as a pair.

---

## Why bother (EF already does this?)

Fair point, `DbContext` + `DbSet` is already a Unit of Work + Repository under the hood. People still wrap it for a few reasons:

* You don't want EF specifics (`Include`, `AsNoTracking`, LINQ) leaking into every service.
* You want one obvious place for the common CRUD methods instead of repeating them.
* Easier to fake in unit tests (mock `IRepository<T>` instead of `DbContext`).

It's a tradeoff. On small projects it can be overkill. On a layered app it keeps the data layer tidy. Decide per project, don't cargo cult it.

---

## Generic repository

One repository that works for any entity, using generics:

```csharp
public interface IGenericRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
}
```

```csharp
public class GenericRepository<T> : IGenericRepository<T> where T : BaseEntity
{
    private readonly AppDbContext _context;
    private readonly DbSet<T> _set;

    public GenericRepository(AppDbContext context)
    {
        _context = context;
        _set = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id, CancellationToken ct)
        => await _set.FindAsync(new object[] { id }, ct);

    public async Task<IEnumerable<T>> GetAllAsync(CancellationToken ct)
        => await _set.ToListAsync(ct);

    public void Add(T entity)    => _set.Add(entity);
    public void Update(T entity) => _set.Update(entity);
    public void Delete(T entity) => _set.Remove(entity);

    public async Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken ct)
        => await _set.AnyAsync(predicate, ct);
}
```

Notice `Add/Update/Delete` don't touch the database. They just stage changes in memory. Nothing is saved until someone calls `SaveChanges`. That's the Unit of Work's job.

---

## When the generic one isn't enough

Some entities need special queries (joins, counts, custom filters). Make a specific repository that inherits the generic one and adds methods:

```csharp
public interface ISessionRepository : IGenericRepository<Session>
{
    Task<IEnumerable<Session>> GetAllWithTrainerAndCategoryAsync(CancellationToken ct);
}

public class SessionRepository : GenericRepository<Session>, ISessionRepository
{
    private readonly AppDbContext _context;
    public SessionRepository(AppDbContext context) : base(context) => _context = context;

    public async Task<IEnumerable<Session>> GetAllWithTrainerAndCategoryAsync(CancellationToken ct)
        => await _context.Sessions
            .Include(s => s.Trainer)
            .Include(s => s.Category)
            .AsNoTracking()
            .ToListAsync(ct);
}
```

---

## Unit of Work

The UoW holds the shared `DbContext`, hands out repositories, and owns the single `SaveChanges`. Because every repo shares the same context, their staged changes all commit in one transaction.

```csharp
public interface IUnitOfWork
{
    IGenericRepository<T> GetRepository<T>() where T : BaseEntity;
    ISessionRepository SessionRepository { get; }
    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private readonly Dictionary<string, object> _repos = new();

    public UnitOfWork(AppDbContext context, ISessionRepository sessionRepository)
    {
        _context = context;
        SessionRepository = sessionRepository;
    }

    public ISessionRepository SessionRepository { get; }

    // cache one repo instance per entity type
    public IGenericRepository<T> GetRepository<T>() where T : BaseEntity
    {
        var key = typeof(T).Name;
        if (_repos.TryGetValue(key, out var existing))
            return (IGenericRepository<T>)existing;

        var repo = new GenericRepository<T>(_context);
        _repos[key] = repo;
        return repo;
    }

    public Task<int> SaveChangesAsync(CancellationToken ct)
        => _context.SaveChangesAsync(ct);
}
```

---

## How it reads in a service

```csharp
public async Task<Result> CreateMemberAsync(CreateMemberDto dto, CancellationToken ct)
{
    var members = _unitOfWork.GetRepository<Member>();

    if (await members.AnyAsync(m => m.Email == dto.Email, ct))
        return Result.Fail("Email already in use.");

    var member = _mapper.Map<Member>(dto);
    members.Add(member);                          // staged, not saved

    var rows = await _unitOfWork.SaveChangesAsync(ct); // committed here
    return rows > 0 ? Result.Ok() : Result.Fail("Try again.");
}
```

The point: the service reads like business steps, not SQL. (See the `Result` type in [result-pattern.md](./result-pattern.md).)

---

## Wiring it up (DI)

All scoped, so one `DbContext` is shared per request:

```csharp
builder.Services.AddScoped(typeof(IGenericRepository<>), typeof(GenericRepository<>));
builder.Services.AddScoped<ISessionRepository, SessionRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

---

## Summary

| Piece | Job |
| ----- | --- |
| Generic repository | reusable CRUD for any entity |
| Specific repository | custom queries for one entity (inherits generic) |
| Unit of Work | owns the shared context, hands out repos, owns `SaveChanges` |
| Lifetime | all Scoped, so they share one context per request = one transaction |

> Real example: I used exactly this setup in my GymSystem MVC project (generic repo + a SessionRepository + UnitOfWork).
