# Architecture Styles

> A map of the common ways to structure a backend: layered/n-tier, onion/clean, vertical slice, and microservices. None of them is "the right one"; each trades something for something. This note is the overview, the onion and clean notes go deep.

---

## Layered / N-Tier

The classic, and what my GymSystem project uses (PL / BLL / DAL):

```text
Presentation (controllers, views)
        |
Business Logic (services, rules)
        |
Data Access (repositories, DbContext)
        |
     Database
```

Each layer talks only to the one below. Simple to explain, everyone knows it, tooling and tutorials assume it.

Its weakness shows with time: **everything ultimately depends on the data layer.** Business logic references DAL, DAL references EF and the database schema. The domain (your actual business rules) sits at the mercy of infrastructure details. Change the persistence story and the ripples go up through everything. Also, layers invite anemic services: 40-method `MemberService` classes where every feature touches the same files.

Still my honest recommendation for small apps and for learning. The problems are real but they're *scale* problems.

---

## Onion / Clean (dependency-inverted)

Same ingredients, one crucial flip: **dependencies point inward, toward the domain, and the database becomes a detail on the outside.**

```text
   [ Infrastructure  (EF, email, files) ]
   [ Application     (use cases)        ]
   [ Domain          (entities, rules)  ]   <- depends on NOTHING
```

The domain defines interfaces (`IMemberRepository`); infrastructure implements them. Business rules stop caring what database exists. That's the whole trick, and it's important enough that [onion-architecture.md](./onion-architecture.md) and [clean-architecture.md](./clean-architecture.md) cover it properly.

Cost: more projects, more interfaces, more mapping. Pays off on long-lived apps with real domain logic.

---

## Vertical Slice

A rotation of the axis. Instead of grouping code by *technical* layer (all controllers together, all services together), group by *feature*:

```text
Features/
├── Members/
│   ├── CreateMember/       # endpoint + handler + validator + everything
│   ├── GetMemberDetails/
│   └── CancelMembership/
└── Subscriptions/
    ├── RenewSubscription/
    └── GetExpiring/
```

Each slice is self-contained top to bottom. The argument: when you work on "cancel membership", every file you need is in one folder, and slices don't share code by default, so touching one feature can't break another. A slice is allowed to be simple (a handler with 10 lines of EF) or complex (full domain model) *independently*, instead of every feature paying the same layer tax.

This pairs naturally with MediatR handlers ([../6.Design-Patterns/cqrs-and-mediatr.md](../6.Design-Patterns/cqrs-and-mediatr.md)) and minimal API groups. The risk is code duplication across slices; the answer is extracting shared bits *when they prove shared*, not preemptively.

Slices vs layers is a genuine philosophical split: layers optimize for consistency, slices optimize for locality of change. I lean slices for feature-heavy APIs, layers for CRUD-heavy admin systems.

---

## Monolith vs Microservices

Everything above describes the inside of one deployable app (a monolith). Microservices split the system into separately deployed services, each owning its own data:

```text
[Members Service]   [Billing Service]   [Notifications Service]
      DB A               DB B                 DB C
        \_______________ HTTP / messages ______________/
```

What you gain: independent deployment and scaling, team autonomy, tech freedom per service, one service crashing isn't the whole system down.

What it costs (and this list is why the default answer is "don't"):

* Network calls where method calls used to be: latency, timeouts, retries, partial failure *everywhere*.
* No cross-service transactions. Consistency becomes eventual, sagas replace `SaveChanges`.
* No joins across services. Reporting gets hard.
* You need real DevOps: containers, orchestration, centralized logging, tracing, service discovery. The operational floor is high.

The sane path for almost everyone: build a **well-structured monolith** (clean layers or slices with clear module boundaries), and if one module someday truly needs independent scaling or a separate team, extract *that module* into a service. A modular monolith gives you most of the boundaries with none of the network. "Microservices first" on a small team is how projects drown.

---

## Choosing, practically

| Situation | Reasonable default |
| --------- | ------------------ |
| Learning, small CRUD app, admin tool | layered (PL/BLL/DAL) |
| Long-lived product with real business rules | onion/clean |
| Feature-heavy API, CQRS-style team | vertical slices |
| Many teams, proven scaling pain, strong DevOps | extract microservices gradually |

Two rules that outrank any style:

1. **Consistency beats purity.** A codebase that follows one convention everywhere is worth more than the theoretically best architecture applied sometimes.
2. **Architecture is about the cost of change.** Ask "what happens here when requirements change" rather than "what does the diagram look like".

---

## Common Mistakes

* **Microservices because big companies do it.** They have hundreds of engineers; the pattern solves *their* problem.
* **Distributed monolith**: services that must deploy together and share a database. All the costs of microservices, none of the benefits.
* **Rewriting the architecture instead of the messy module.** Usually the pain is one tangled area, not the style.
* **Zero structure "because we're small".** A monolith with no module boundaries is what makes the later extraction impossible.

---

## Summary

| Style | One-liner |
| ----- | --------- |
| Layered | simple, universal, everything leans on the data layer |
| Onion/Clean | dependencies point at the domain, DB is a detail |
| Vertical slice | group by feature, optimize for locality of change |
| Modular monolith | one deploy, real internal boundaries, the sweet spot |
| Microservices | independent deploys at the price of a distributed system |

> Deep dives: [onion-architecture.md](./onion-architecture.md) and [clean-architecture.md](./clean-architecture.md).
