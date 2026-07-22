# 📘 .NET Learning Notes

> My personal notebook for the .NET backend journey. Concise, practical notes on C#, LINQ, EF Core, ASP.NET Core, and architecture, written to be re-read and revised.

Each note is self-contained: a topic explained in plain language, with minimal runnable examples, common mistakes, and "why it matters" context.

---

## 🗺️ Contents

A rough learning order. Top to bottom is a sensible path, but each note stands on its own.

### 1. Advanced C#
Core language features every backend developer leans on.

| Topic | Notes |
| ----- | ----- |
| Collections | [Open](./1.CSharp-Advanced/collections.md) |
| Generics | [Open](./1.CSharp-Advanced/generics.md) |
| Delegates | [Open](./1.CSharp-Advanced/delegates.md) |
| Events | [Open](./1.CSharp-Advanced/events.md) |
| Records & Value vs Reference | [Open](./1.CSharp-Advanced/records.md) |
| Exception Handling | [Open](./1.CSharp-Advanced/exceptions.md) |
| Pattern Matching | [Open](./1.CSharp-Advanced/pattern-matching.md) |
| Nullable Reference Types | [Open](./1.CSharp-Advanced/nullable-reference-types.md) |
| Async Programming | [Open](./1.CSharp-Advanced/async-programming.md) |
| Iterators & yield | [Open](./1.CSharp-Advanced/iterators-and-yield.md) |
| Reflection & Attributes | [Open](./1.CSharp-Advanced/reflection-and-attributes.md) |
| Span\<T\> & Memory\<T\> | [Open](./1.CSharp-Advanced/span-and-memory.md) |

### 2. EF Core
Talking to a database through objects with Entity Framework Core.

| Topic | Notes |
| ----- | ----- |
| Introduction (DbContext, DbSet, Migrations, CRUD) | [Open](./2.EF-Core/introduction.md) |
| Configurations & Relationships | [Open](./2.EF-Core/configurations-and-relationships.md) |
| Loading Related Data & Tracking (N+1, AsNoTracking) | [Open](./2.EF-Core/loading-and-tracking.md) |
| Transactions & Concurrency | [Open](./2.EF-Core/transactions-and-concurrency.md) |
| Performance & Advanced Queries | [Open](./2.EF-Core/performance-and-advanced-queries.md) |

### 3. LINQ
Querying and transforming data declaratively.

| Topic | Notes |
| ----- | ----- |
| LINQ Fundamentals | [Open](./3.LINQ/linq.md) |
| Advanced LINQ (GroupBy, Join, SelectMany, Aggregate) | [Open](./3.LINQ/linq-advanced.md) |
| IQueryable vs IEnumerable | [Open](./3.LINQ/iqueryable-vs-ienumerable.md) |

### 4. Web & ASP.NET Core
How the web works and how ASP.NET Core handles a request.

| Topic | Notes |
| ----- | ----- |
| Web Fundamentals (DNS, TCP/IP, HTTP) | [Open](./4.Web-Basics/web-fundamentals.md) |
| ASP.NET Core Fundamentals | [Open](./4.Web-Basics/aspnet-core-fundamentals.md) |
| Dependency Injection & Service Lifetimes | [Open](./4.Web-Basics/dependency-injection.md) |
| Middleware & Request Pipeline | [Open](./4.Web-Basics/middleware.md) |
| Filters | [Open](./4.Web-Basics/filters.md) |
| Validation | [Open](./4.Web-Basics/validation.md) |
| Authentication & Authorization (JWT, Identity) | [Open](./4.Web-Basics/authentication-and-authorization.md) |

### 5. MVC
The MVC pattern in ASP.NET Core: controllers, views, Razor, and the full request flow.

| Topic | Notes |
| ----- | ----- |
| MVC (controllers, views, Razor, the full flow) | [Open](./5.MVC/mvc.md) |
| Views & Razor Deep Dive (tag helpers, partials, view components) | [Open](./5.MVC/views-and-razor-deep-dive.md) |

**Worked example:**

| Project | What it is | Link |
| ------- | ---------- | ---- |
| GymSystem | A full ASP.NET Core **MVC** project (3-layer: PL / BLL / DAL) that the MVC note references throughout | [Repo](https://github.com/Muhammed-maher32/GymManagementSystem) |

### 6. Design Patterns
Patterns I picked up building real projects.

| Topic | Notes |
| ----- | ----- |
| Repository & Unit of Work | [Open](./6.Design-Patterns/repository-and-unit-of-work.md) |
| Result Pattern | [Open](./6.Design-Patterns/result-pattern.md) |
| DTOs & AutoMapper | [Open](./6.Design-Patterns/dtos-and-automapper.md) |
| Options Pattern | [Open](./6.Design-Patterns/options-pattern.md) |
| Factory & Strategy | [Open](./6.Design-Patterns/factory-and-strategy.md) |
| Decorator & Caching | [Open](./6.Design-Patterns/decorator-and-caching.md) |
| CQRS & MediatR | [Open](./6.Design-Patterns/cqrs-and-mediatr.md) |

### 7. Web API
Building HTTP APIs the right way.

| Topic | Notes |
| ----- | ----- |
| REST & API Design (conventions, status codes, versioning) | [Open](./7.Web-API/rest-and-api-design.md) |
| Swagger & OpenAPI | [Open](./7.Web-API/swagger-and-openapi.md) |
| Minimal APIs | [Open](./7.Web-API/minimal-apis.md) |
| Error Handling & ProblemDetails | [Open](./7.Web-API/error-handling-and-problem-details.md) |

### 8. Architecture
How to structure a whole application, not just one class.

| Topic | Notes |
| ----- | ----- |
| Architecture Styles (monolith, n-tier, vertical slice, microservices) | [Open](./8.Architecture/architecture-styles.md) |
| Onion Architecture | [Open](./8.Architecture/onion-architecture.md) |
| Clean Architecture | [Open](./8.Architecture/clean-architecture.md) |
| Logging & Serilog | [Open](./8.Architecture/logging-and-serilog.md) |
| Background Jobs | [Open](./8.Architecture/background-jobs.md) |

---

## 📂 Repository Structure

```
DotNet-Learning/
├── 1.CSharp-Advanced/   # collections, generics, delegates, events, records,
│                        #   exceptions, pattern matching, nullable refs, async,
│                        #   iterators, reflection, Span<T>
├── 2.EF-Core/           # introduction, configurations & relationships, loading & tracking,
│                        #   transactions & concurrency, performance
├── 3.LINQ/              # fundamentals, advanced operators, IQueryable vs IEnumerable
├── 4.Web-Basics/        # web fundamentals, ASP.NET Core, DI, middleware,
│                        #   filters, validation, auth
├── 5.MVC/               # MVC pattern + full request flow, Razor deep dive
├── 6.Design-Patterns/   # repository & UoW, result, DTOs & AutoMapper, options,
│                        #   factory & strategy, decorator & caching, CQRS & MediatR
├── 7.Web-API/           # REST design, Swagger, minimal APIs, error handling
├── 8.Architecture/      # architecture styles, onion, clean, logging, background jobs
└── README.md
```

---

## 🎯 Why This Repo

To document my backend learning in my own words. The act of writing a topic down is how I make sure I actually understand it. If you stumbled in here and it helps you too, even better. 🙌

---

## ✍️ Author

**Mohamed Maher** ([GitHub](https://github.com/Muhammed-maher32))
