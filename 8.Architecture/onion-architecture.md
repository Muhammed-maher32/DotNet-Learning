# Onion Architecture

> Layered architecture with the dependency arrow flipped: instead of business logic depending on data access, data access depends on the business core. The domain sits in the middle knowing nothing about databases, and everything else orbits it.

---

## The flip, concretely

Traditional 3-layer:

```text
PL  ->  BLL  ->  DAL  ->  database
```

BLL references the DAL project. Business code *knows about* repositories, EF, connection details. The most important code in the system depends on the least important details.

Onion:

```text
        +---------------------------+
        |  Infrastructure (EF, SMTP)|
        |   +-------------------+   |
        |   |  Application      |   |
        |   |  (use cases)      |   |
        |   |   +-----------+   |   |
        |   |   |  Domain   |   |   |
        |   |   +-----------+   |   |
        |   +-------------------+   |
        +---------------------------+

references point INWARD only
```

The domain references nothing. Application references domain. Infrastructure references application and domain. The web project references everything only to wire DI.

How can the application layer save to a database it doesn't reference? **It defines the interface; infrastructure implements it.** That's dependency inversion, and it's the entire pattern:

```csharp
// Application project. No EF anywhere.
public interface IMemberRepository
{
    Task<Member?> GetByIdAsync(int id, CancellationToken ct);
    Task<bool> EmailExistsAsync(string email, CancellationToken ct);
    void Add(Member member);
}
```

```csharp
// Infrastructure project. References Application, implements its needs.
public class MemberRepository : IMemberRepository
{
    private readonly AppDbContext _context;
    // ... EF Core lives here and only here
}
```

At runtime DI connects them. At *compile time*, the business core has no idea EF exists.

---

## The layers in a solution

```text
GymSystem.Domain/           # entities, value objects, domain rules, domain interfaces
GymSystem.Application/      # use cases (services or handlers), DTOs, repo interfaces
GymSystem.Infrastructure/   # EF Core, repositories, email sender, file storage, clock
GymSystem.Web/              # controllers/endpoints, DI wiring, Program.cs
```

**Domain**: entities with actual behavior, not just property bags:

```csharp
public class Subscription
{
    public DateTime EndDate { get; private set; }
    public SubscriptionStatus Status { get; private set; }

    public void Renew(int months)
    {
        if (Status == SubscriptionStatus.Cancelled)
            throw new DomainException("Cancelled subscriptions cannot be renewed.");

        var from = EndDate > DateTime.UtcNow ? EndDate : DateTime.UtcNow;
        EndDate = from.AddMonths(months);
        Status = SubscriptionStatus.Active;
    }
}
```

The renewal rule lives *on the entity*, testable with zero infrastructure. That's the payoff of protecting the core: rules like this don't get smeared across services and controllers.

**Application**: orchestrates use cases. Loads entities via interfaces, calls domain behavior, saves via interfaces. Contains no HTTP and no SQL.

**Infrastructure**: all the I/O. EF configurations, migrations, repository implementations, the SMTP client, `ISystemClock` implementations, external API clients.

**Web**: thin. Binds requests, calls application, maps results to HTTP.

---

## What this buys you, really

* **Testable core.** Domain and application test with fakes of a few small interfaces. No in-memory database gymnastics.
* **Deferred decisions.** Which database, which email provider, cache or not: infrastructure concerns, swappable without touching rules.
* **A compiler-enforced rule.** This one is underrated: since Domain physically doesn't reference Infrastructure, nobody *can* sneak a SQL query into a business rule. Project references are the architecture, not a wiki page nobody reads.

And what it costs: four projects instead of one, interfaces for everything I/O-shaped, mapping between layers, and a longer "where do I put this file" conversation for juniors. On a small CRUD app this is pure overhead, see [architecture-styles.md](./architecture-styles.md) for when it's worth it.

---

## Onion vs Clean Architecture

Practically the same idea from different authors (Palermo's onion, Uncle Bob's clean). Both: domain at the center, dependencies inward, infrastructure at the edge. Clean architecture formalizes the ring names (entities / use cases / interface adapters / frameworks) and talks more about boundaries and crossing them with DTOs. If you understand one, you understand the other; teams say "onion" or "clean" and mean roughly this same picture. Details in [clean-architecture.md](./clean-architecture.md).

---

## Common Mistakes

* **Domain entities as bare property bags** with all logic in application services. That's 3-layer wearing an onion costume; the center is empty.
* **Leaking IQueryable or EF entities out of infrastructure** through the interfaces. The interface contract should be expressible without knowing EF exists.
* **Domain referencing anything.** The moment `GymSystem.Domain.csproj` gets a NuGet reference to EF Core "just for an attribute", the compile-time guarantee is gone. Use fluent configuration in Infrastructure instead.
* **Interfaces for things that aren't I/O.** Not every class needs an interface; the inversion is for infrastructure boundaries, not for `PriceCalculator`.

---

## Summary

| Idea | Point |
| ---- | ----- |
| The flip | infrastructure depends on the core, never the reverse |
| Mechanism | core defines interfaces, outer rings implement them |
| Domain | entities with behavior, zero references |
| Enforcement | project references make violations compile errors |
| Cost | more projects and mapping; overkill for small CRUD |

> Continues in [clean-architecture.md](./clean-architecture.md). The repository interfaces at the boundary are the ones from [../6.Design-Patterns/repository-and-unit-of-work.md](../6.Design-Patterns/repository-and-unit-of-work.md).
