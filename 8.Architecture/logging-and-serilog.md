# Logging & Serilog

> Logs are how you debug production, where there's no breakpoint. The built-in ILogger gets the API right; Serilog plugs underneath it to make logs *structured*, meaning searchable data instead of text soup.

---

## ILogger, the abstraction you code against

ASP.NET Core ships logging in the box. Inject `ILogger<T>` (the `T` becomes the log's category, so you can filter by class):

```csharp
public class MemberService
{
    private readonly ILogger<MemberService> _logger;

    public MemberService(ILogger<MemberService> logger) => _logger = logger;

    public async Task<Result> CreateAsync(CreateMemberDto dto, CancellationToken ct)
    {
        _logger.LogInformation("Creating member with email {Email}", dto.Email);
        // ...
        _logger.LogWarning("Duplicate email attempt: {Email}", dto.Email);
    }
}
```

**The levels**, and how I actually use them:

| Level | Use for |
| ----- | ------- |
| Trace / Debug | dev-time noise, off in production |
| Information | normal business events: member created, payment received |
| Warning | odd but handled: duplicate attempt, retry succeeded, slow query |
| Error | operation failed, someone should look: unhandled exception, payment gateway down |
| Critical | the app itself is dying: can't reach DB at startup |

Levels matter because filtering is per level per category in config, so you can run `Information` globally but silence EF's chatter:

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "Microsoft.EntityFrameworkCore": "Warning"
  }
}
```

---

## The most important habit: message templates

This line is not string interpolation, and the difference is the whole point:

```csharp
// RIGHT: template + parameters
_logger.LogInformation("Member {MemberId} renewed plan {PlanName}", member.Id, plan.Name);

// WRONG: interpolated string
_logger.LogInformation($"Member {member.Id} renewed plan {plan.Name}");
```

Both print the same text. But with the template, structured sinks store `MemberId=42` and `PlanName="Gold"` as *fields*. Later you query "all events where MemberId = 42" across millions of logs in a second. With the interpolated version you're grepping strings and it's gone forever. (Bonus: templates skip the string formatting entirely when the level is filtered out.)

This habit costs nothing and is the difference between logs you can investigate and logs you can only scroll.

---

## Serilog

The de facto structured logging provider. It implements the ILogger abstractions, so services don't change at all; only Program.cs knows Serilog exists.

```csharp
// Serilog.AspNetCore package
builder.Host.UseSerilog((context, config) => config
    .ReadFrom.Configuration(context.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "GymSystem")
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 14));
```

Concepts:

* **Sinks** = destinations. Console, rolling files, Seq, Elasticsearch, Application Insights... one log call fans out to all configured sinks.
* **Enrichers** = properties stamped on every event (machine name, request id, user id) without passing them manually.
* **ReadFrom.Configuration** moves sink/level setup into appsettings.json, so prod and dev differ by config, not code.

Replace the noisy default request logging with Serilog's one-line-per-request version:

```csharp
app.UseSerilogRequestLogging();
// HTTP POST /api/members responded 201 in 43.2 ms
```

For local development, pair console logs with **Seq** (free single-user) or any structured viewer; querying `MemberId = 42` beats scrolling a console. In real deployments logs ship to a central store (Seq/ELK/App Insights), because grepping files across servers is not a plan.

---

## Correlation: following one request

A request touches controller, behavior, handler, repository. Five log lines, and in production they interleave with hundreds of others. Two tools stitch them together:

* ASP.NET Core already stamps a TraceIdentifier per request, and `UseSerilogRequestLogging` + `FromLogContext` attach it.
* For your own scopes:

```csharp
using (_logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = orderId }))
{
    // every log inside here, including from called services, carries OrderId
}
```

This is the traceId loop from [../7.Web-API/error-handling-and-problem-details.md](../7.Web-API/error-handling-and-problem-details.md): the client reports the id from the error response, you filter logs by it, the whole request story appears.

---

## What (not) to log

* **Do**: business events, external calls with durations, decisions ("skipped renewal, subscription cancelled"), all exceptions with `LogError(ex, "...")` (the exception as the *first argument*, so the stack trace is captured, not `ex.Message` glued into text).
* **Don't**: passwords, tokens, connection strings, full card numbers, personal data you don't need. Logs get copied into tickets and third-party services; treat them as semi-public. Mask what must appear (`ali***@gmail.com`).
* **Don't log-and-rethrow at every layer.** One exception should appear once, at the boundary that handles it, not five times up the stack.

---

## Common Mistakes

* **String interpolation in log calls.** Kills structure, the #1 habit to fix.
* **`LogError(ex.Message)`** instead of `LogError(ex, "template")`, throwing away the stack trace.
* **Information-level logs inside hot loops**, generating gigabytes about nothing.
* **No retention policy** on file sinks until the disk fills. `retainedFileCountLimit` is right there.
* **Logging as an afterthought.** The time to add the "payment failed because X" log is when writing the payment code, not during the incident.

---

## Summary

| Idea | Point |
| ---- | ----- |
| ILogger\<T\> | inject the abstraction; category = class |
| Levels | Information for events, Warning for handled-odd, Error for broken |
| Message templates | named placeholders, never $"" interpolation |
| Serilog | sinks + enrichers + config-driven setup under the same ILogger |
| Correlation | traceId/scopes tie a request's logs together |
| Hygiene | no secrets, exceptions logged once with the ex object |

> Pairs with [../7.Web-API/error-handling-and-problem-details.md](../7.Web-API/error-handling-and-problem-details.md) and matters double for [background-jobs.md](./background-jobs.md), where logs are the only visibility you have.
