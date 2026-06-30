# Dependency Injection & Service Lifetimes

> DI just means: a class doesn't `new` up the things it needs, it asks for them in the constructor and someone else hands them over. ASP.NET Core has a container built in that does the handing over.

---

## The problem it solves

Without DI, a class creates its own dependencies:

```csharp
public class OrderService
{
    private readonly EmailSender _email = new EmailSender(); // hard wired

    // now OrderService is glued to EmailSender forever.
    // can't swap it, can't fake it in a test.
}
```

With DI, you ask for it:

```csharp
public class OrderService
{
    private readonly IEmailSender _email;

    public OrderService(IEmailSender email) // someone gives it to me
    {
        _email = email;
    }
}
```

Now `OrderService` depends on the **interface**, not a concrete class. You can swap the real sender for a fake one in tests, or change implementations without touching this code. This is the D in SOLID (Dependency Inversion).

---

## Registering services

In `Program.cs` you tell the container "when someone asks for `IEmailSender`, give them `SmtpEmailSender`":

```csharp
builder.Services.AddScoped<IEmailSender, SmtpEmailSender>();
builder.Services.AddScoped<IOrderService, OrderService>();
```

Then any controller/service that takes `IEmailSender` in its constructor gets one automatically. You never call `new`.

---

## The three lifetimes (this is the part people mix up)

When you register a service you pick how long its instance lives.

### Transient

New instance **every single time** it's requested. Even twice in the same request.

```csharp
builder.Services.AddTransient<IThing, Thing>();
```

Use for: lightweight, stateless things. Cheap to create, hold no shared state.

### Scoped

One instance **per HTTP request**. Everyone in the same request shares it, the next request gets a fresh one.

```csharp
builder.Services.AddScoped<IThing, Thing>();
```

Use for: most of your stuff. Services, repositories, and especially anything touching the database. `DbContext` is registered as Scoped for exactly this reason, so all your repositories in one request share the same context (and the same transaction).

### Singleton

**One instance for the whole app**, created once and reused forever (every request, every user).

```csharp
builder.Services.AddSingleton<IThing, Thing>();
```

Use for: stateless, thread safe things you want to create once (config, caches, a logger factory). Be careful: a singleton is shared across all requests at the same time, so any mutable state inside it must be thread safe.

---

## Cheat sheet

| Lifetime | Lives for | Typical use |
| -------- | --------- | ----------- |
| Transient | one resolve (can be many per request) | small stateless helpers |
| Scoped | one HTTP request | services, repositories, `DbContext` |
| Singleton | whole app | config, caches, stateless shared services |

---

## The captive dependency trap

This one actually breaks things, so remember it:

> A service can only safely depend on something with an **equal or longer** lifetime.

If a **Singleton** injects a **Scoped** service, that scoped instance gets "captured" and lives forever inside the singleton. So your `DbContext` (scoped) gets stuck alive for the whole app, shared across every request at once. That's a bug factory (stale data, threading issues, disposed-object errors).

```
Singleton  ->  Scoped    ❌ captured, scoped thing never released
Scoped     ->  Transient ✔️ fine
Scoped     ->  Singleton ✔️ fine
Transient  ->  anything  ✔️ fine
```

ASP.NET Core actually has a scope validator that throws on this in Development, which is nice.

---

## One liner to remember

> Register against interfaces, inject through the constructor, and default to **Scoped** unless you have a reason not to.
