# 📘 .NET Learning Notes

> My personal notebook for the .NET backend journey. Concise, practical notes on C#, LINQ, EF Core, and ASP.NET Core, written to be re-read and revised.

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

### 2. EF Core
Talking to a database through objects with Entity Framework Core.

| Topic | Notes |
| ----- | ----- |
| Introduction (DbContext, DbSet, Migrations, CRUD) | [Open](./2.EF-Core/introduction.md) |
| Configurations & Relationships | [Open](./2.EF-Core/configurations-and-relationships.md) |
| Loading Related Data & Tracking (N+1, AsNoTracking) | [Open](./2.EF-Core/loading-and-tracking.md) |

### 3. LINQ
Querying and transforming data declaratively.

| Topic | Notes |
| ----- | ----- |
| LINQ Fundamentals | [Open](./3.LINQ/linq.md) |

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

**Worked example:**

| Project | What it is | Link |
| ------- | ---------- | ---- |
| GymSystem | A full ASP.NET Core **MVC** project (3-layer: PL / BLL / DAL) that the MVC note references throughout | [Repo](https://github.com/Muhammed-maher32/GymManagementSystem) |

### 6. Design Patterns
Patterns I picked up building real projects.

| Topic | Notes |
| ----- | ----- |
| Repository & Unit of Work | [Open](./6.design-patterns/repository-and-unit-of-work.md) |
| Result Pattern | [Open](./6.design-patterns/result-pattern.md) |
| DTOs & AutoMapper | [Open](./6.design-patterns/dtos-and-automapper.md) |

---

## 📝 To-Do (planned notes)

- [ ] EF Core: Transactions & Concurrency (explicit transactions, optimistic concurrency with rowversion, handling `DbUpdateConcurrencyException`)
- [ ] Web API & REST (REST conventions, status codes, versioning, Swagger/OpenAPI, content negotiation)

---

## 📂 Repository Structure

```
DotNet-Learning/
├── 1.CSharp-Advanced/   # collections, generics, delegates, events, records,
│                        #   exceptions, pattern matching, nullable refs, async
├── 2.EF-Core/           # introduction, configurations & relationships, loading & tracking
├── 3.LINQ/              # LINQ fundamentals
├── 4.Web-Basics/        # web fundamentals, ASP.NET Core, DI, middleware,
│                        #   filters, validation, auth
├── 5.MVC/               # MVC pattern + full request flow (references GymSystem)
├── 6.design-patterns/   # repository & unit of work, result pattern, DTOs & AutoMapper
└── README.md
```

---

## 🎯 Why This Repo

To document my backend learning in my own words. The act of writing a topic down is how I make sure I actually understand it. If you stumbled in here and it helps you too, even better. 🙌

---

## ✍️ Author

**Mohamed Maher** ([GitHub](https://github.com/Muhammed-maher32))
