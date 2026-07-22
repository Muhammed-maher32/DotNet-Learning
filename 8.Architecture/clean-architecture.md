# Clean Architecture

> The same core idea as onion (dependencies point inward, domain at the center) organized into a concrete, opinionated .NET solution layout. This note is the practical version: the projects, what goes where, and the wiring.

---

## The dependency rule

One rule governs everything: **source code dependencies only point inward.** Inner circles know nothing about outer circles. The domain doesn't know use cases exist; use cases don't know HTTP or SQL exists.

```text
Frameworks & Drivers   (ASP.NET, EF Core, external APIs)
    Interface Adapters (controllers, presenters, gateways)
        Use Cases      (application business rules)
            Entities   (enterprise business rules)
```

In .NET practice those four rings collapse into the familiar four projects (same as onion, this is the part where the two patterns are basically one):

```text
src/
├── GymSystem.Domain/         # Entities ring
├── GymSystem.Application/    # Use cases ring
├── GymSystem.Infrastructure/ # EF, external services
└── GymSystem.WebApi/         # ASP.NET entry point + DI composition
```

Reference graph: Domain -> nothing. Application -> Domain. Infrastructure -> Application. WebApi -> Application + Infrastructure (for DI only).

---

## What goes in each project

**Domain** (no NuGet packages at all, ideally):

```text
Domain/
├── Entities/           Member.cs, Subscription.cs (with behavior)
├── ValueObjects/       Money.cs, DateRange.cs
├── Enums/              SubscriptionStatus.cs
├── Exceptions/         DomainException.cs
└── Common/             BaseEntity.cs, IAuditable.cs
```

**Application** (references Domain; typical packages: MediatR/FluentValidation abstractions):

```text
Application/
├── Abstractions/       IMemberRepository.cs, IUnitOfWork.cs, IEmailSender.cs, IAppDbContext.cs
├── Features/
│   └── Members/
│       ├── CreateMember/      command, handler, validator
│       └── GetMemberById/     query, handler, dto
└── Common/             Result.cs, PagedList.cs, behaviors
```

Notice both flavors are possible here: repository interfaces, or an `IAppDbContext` interface exposing DbSets so handlers write LINQ directly. Purists prefer repositories (Application stays persistence-ignorant); pragmatists expose the context interface and accept the EF coupling for much less boilerplate. Pick one per project, both count as clean-ish. I covered that trade-off in [../6.Design-Patterns/repository-and-unit-of-work.md](../6.Design-Patterns/repository-and-unit-of-work.md).

**Infrastructure** (references Application; all the packages live here):

```text
Infrastructure/
├── Persistence/
│   ├── AppDbContext.cs
│   ├── Configurations/       IEntityTypeConfiguration per entity
│   ├── Repositories/
│   └── Migrations/
├── Services/                 SmtpEmailSender.cs, SystemClock.cs
└── DependencyInjection.cs    <- see below
```

**WebApi**: controllers or minimal endpoints, exception handling, auth setup, Program.cs.

---

## The composition root trick

Each outer project exposes one extension method that registers its own services, so Program.cs stays a table of contents:

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(o =>
            o.UseSqlServer(config.GetConnectionString("Default")));

        services.AddScoped<IMemberRepository, MemberRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IEmailSender, SmtpEmailSender>();
        return services;
    }
}
```

```csharp
// Program.cs
builder.Services
    .AddApplication()                          // MediatR, validators, behaviors
    .AddInfrastructure(builder.Configuration); // EF, repositories, email
```

WebApi is the *composition root*: the only place that sees all projects, exists to glue them, and contains no logic worth testing.

---

## Following one request through

`POST /api/members`:

1. **WebApi**: controller binds `CreateMemberCommand` from JSON, `_sender.Send(command)`.
2. **Application**: validation behavior runs the FluentValidation rules. Handler asks `IMemberRepository` whether the email exists, creates the `Member` (domain constructor/factory enforces invariants), adds it, `IUnitOfWork.SaveChangesAsync()`.
3. **Domain**: the Member entity itself guarantees it can't be constructed in an invalid state.
4. **Infrastructure**: the repository implementation translates all of that to EF Core; SaveChanges hits SQL Server.
5. Back in **WebApi**: Result maps to `201 Created` or `409 Conflict`.

Every arrow in that flow points inward or is an interface implemented outward. HTTP never touched Application; SQL never touched Domain.

---

## Honest assessment

When it shines: long-lived products, real business rules, teams larger than two, anything where "we might change infra" is a genuine sentence, and codebases where tests of business logic must not need a database.

When it's cosplay: a 6-table CRUD API with four projects, AutoMapper, MediatR, and zero domain behavior, where every feature means touching 7 files to move a string from HTTP to SQL. If the Domain project is all property bags, the architecture is decoration. Grow into it: start layered or sliced ([architecture-styles.md](./architecture-styles.md)), extract a domain when rules actually accumulate.

---

## Common Mistakes

* **Anemic domain + clean folders** = ceremony without benefit. The center must contain actual rules.
* **Infrastructure types leaking through Application interfaces** (returning `DbSet`, EF entities in DTOs, IQueryable across the boundary).
* **WebApi referencing Infrastructure for logic**, not just DI. If a controller news up a repository, the boundary is dead.
* **Skipping the Application layer**, controllers calling repositories directly. Fine! But that's n-tier, name it honestly.
* **One giant "Core" project** mixing domain and application until neither boundary means anything.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Dependency rule | references point inward, enforced by csproj references |
| Domain | entities with invariants, zero packages |
| Application | use cases + interfaces it needs the outside to fulfill |
| Infrastructure | implements those interfaces, owns all I/O packages |
| WebApi | thin composition root |
| Reality check | needs real domain logic to pay for its ceremony |

> The theory is in [onion-architecture.md](./onion-architecture.md); the Feature-folder style inside Application comes from [../6.Design-Patterns/cqrs-and-mediatr.md](../6.Design-Patterns/cqrs-and-mediatr.md).
