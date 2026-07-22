# CQRS & MediatR

> CQRS: stop forcing reads and writes through the same service methods and models, they want different things. MediatR: the library most .NET projects use to implement it, turning every use case into a small request + handler pair.

---

## The idea behind CQRS

Command Query Responsibility Segregation. Sounds enterprise-y, the core is simple:

* **Commands** change state and return little or nothing: `CreateMember`, `CancelSubscription`.
* **Queries** return data and change nothing: `GetMemberById`, `GetExpiringSubscriptions`.

Why split them? Because they naturally diverge:

* Writes want validation, business rules, transactions, the full entity.
* Reads want speed: projections straight to DTOs, `AsNoTracking`, sometimes raw SQL or Dapper, no business rules at all.

A classic `IMemberService` mixes both, so it accumulates 30 methods where half fight for the entity model and half fight for query performance. Splitting per use case keeps each one small and honest.

To be clear: this is CQRS-lite, one database, separate code paths. Full CQRS with separate read/write databases and event sourcing is a different beast most apps never need.

---

## MediatR basics

MediatR is an in-process dispatcher. You send a request object, it finds the one handler for that type and invokes it.

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

A command:

```csharp
public record CreateMemberCommand(string Name, string Email, int PlanId)
    : IRequest<Result<int>>;

public class CreateMemberHandler : IRequestHandler<CreateMemberCommand, Result<int>>
{
    private readonly AppDbContext _context;

    public CreateMemberHandler(AppDbContext context) => _context = context;

    public async Task<Result<int>> Handle(CreateMemberCommand request, CancellationToken ct)
    {
        if (await _context.Members.AnyAsync(m => m.Email == request.Email, ct))
            return Result<int>.Fail("Email already in use.");

        var member = new Member { Name = request.Name, Email = request.Email, PlanId = request.PlanId };
        _context.Members.Add(member);
        await _context.SaveChangesAsync(ct);

        return Result<int>.Ok(member.Id);
    }
}
```

A query, free to take the fastest read path:

```csharp
public record GetMemberByIdQuery(int Id) : IRequest<MemberDetailsDto?>;

public class GetMemberByIdHandler : IRequestHandler<GetMemberByIdQuery, MemberDetailsDto?>
{
    private readonly AppDbContext _context;

    public GetMemberByIdHandler(AppDbContext context) => _context = context;

    public async Task<MemberDetailsDto?> Handle(GetMemberByIdQuery request, CancellationToken ct)
        => await _context.Members
            .Where(m => m.Id == request.Id)
            .Select(m => new MemberDetailsDto(m.Id, m.Name, m.Email, m.Plan.Name))
            .AsNoTracking()
            .FirstOrDefaultAsync(ct);
}
```

Controllers shrink to translation between HTTP and requests:

```csharp
[ApiController]
[Route("api/members")]
public class MembersController : ControllerBase
{
    private readonly ISender _sender;

    public MembersController(ISender sender) => _sender = sender;

    [HttpPost]
    public async Task<IActionResult> Create(CreateMemberCommand command, CancellationToken ct)
    {
        var result = await _sender.Send(command, ct);
        return result.Success
            ? CreatedAtAction(nameof(Get), new { id = result.Value }, null)
            : Conflict(result.Error);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id, CancellationToken ct)
    {
        var dto = await _sender.Send(new GetMemberByIdQuery(id), ct);
        return dto is null ? NotFound() : Ok(dto);
    }
}
```

Folder-wise this organizes by feature, which I like a lot:

```text
Features/
└── Members/
    ├── CreateMember/     # command + handler + validator together
    ├── GetMemberById/
    └── CancelMembership/
```

Everything about one use case in one folder, instead of spread across Services/, DTOs/, Validators/.

---

## Pipeline behaviors: the actual killer feature

A behavior wraps *every* request like middleware. This is where cross-cutting concerns go once instead of into every handler:

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        foreach (var validator in _validators)
        {
            var result = await validator.ValidateAsync(request, ct);
            if (!result.IsValid)
                throw new ValidationException(result.Errors);
        }
        return await next();    // continue to the handler
    }
}
```

```csharp
cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
```

Typical stack: logging behavior, validation behavior (usually with FluentValidation), then maybe a transaction behavior around commands. Every current and future handler gets all of it for free. This is the part that justifies the library; without behaviors MediatR is mostly indirection.

Licensing note: MediatR went commercial (free below a revenue threshold). Alternatives with the same shape exist (Wolverine, or a hand-rolled `ISender` which is honestly a small class), the pattern matters more than the package.

---

## When NOT to use it

* **Small CRUD apps.** Controller -> service -> repository is fewer moving parts and completely fine. CQRS+MediatR pays off with many use cases and cross-cutting concerns, not five endpoints.
* **When the team treats it as mandatory ceremony.** A `GetById` that spawns three files and a behavior pipeline to run one SELECT is cargo culting.
* **Don't confuse it with the mediator *pattern* solving coupling.** In-process MediatR is mostly a convention + pipeline tool. That's fine, but know what you're buying.

---

## Common Mistakes

* **Handlers calling other handlers.** Chains of `_sender.Send` inside handlers make flow impossible to follow. Extract shared logic into a normal service both handlers use.
* **Queries that modify state.** The moment a "query" writes, the mental model breaks. Keep the split honest.
* **One giant validator/behavior doing business rules.** Behaviors are for cross-cutting concerns; business rules live in the handler.
* **Returning entities from queries.** Queries should project to DTOs, that's half the point of separating reads.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Commands | change state, full rules and transactions |
| Queries | read-only, project to DTOs, fastest path allowed |
| MediatR | request in, one handler out, organized per use case |
| Behaviors | middleware around every handler: logging, validation, transactions |
| Cost | more files and indirection; skip for small CRUD apps |

> Builds on [result-pattern.md](./result-pattern.md) (handlers return Result) and pairs with [../8.Architecture/clean-architecture.md](../8.Architecture/clean-architecture.md), where handlers make up the application layer.
