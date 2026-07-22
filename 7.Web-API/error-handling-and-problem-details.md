# Error Handling & ProblemDetails

> Every error your API returns should have the same shape, and unexpected exceptions should never leak stack traces to clients. ProblemDetails is the standard shape (RFC 7807/9457), and IExceptionHandler is where unhandled exceptions get converted to it.

---

## ProblemDetails: the standard error body

Instead of every API inventing its own `{ "error": ... }` vs `{ "message": ... }` vs `{ "errors": [...] }`, there's a standard:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "Member 5 does not exist.",
  "instance": "/api/members/5",
  "traceId": "00-4bf9..."
}
```

ASP.NET Core has first-class support. With `[ApiController]`, validation failures already return the extended `ValidationProblemDetails`:

```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": ["The Email field is not a valid e-mail address."],
    "Name": ["The Name field is required."]
  }
}
```

And you can return it deliberately from actions:

```csharp
return Problem(
    title: "Subscription conflict",
    detail: "Member already has an active subscription.",
    statusCode: StatusCodes.Status409Conflict);
```

The win is boring consistency: one error shape for the frontend to parse, always.

---

## The two kinds of failure

Worth keeping the distinction sharp, because they're handled in different places:

1. **Expected business failures** (email taken, not found, forbidden). These are return values: service returns a Result, controller maps it to 404/409/403 with a ProblemDetails body. No exceptions involved. That's [../6.Design-Patterns/result-pattern.md](../6.Design-Patterns/result-pattern.md).
2. **Unexpected exceptions** (DB down, null bug, timeout). These *should* throw, and one central place turns them into a 500 without leaking internals.

The rest of this note is about the second kind.

---

## IExceptionHandler: the central net

Since .NET 8 this is the clean mechanism (before it was custom middleware, which still works):

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken ct)
    {
        _logger.LogError(exception, "Unhandled exception on {Path}", httpContext.Request.Path);

        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Something went wrong.",
            Detail = "An unexpected error occurred. Try again or contact support.",
            Instance = httpContext.Request.Path
        };
        problem.Extensions["traceId"] = httpContext.TraceIdentifier;

        httpContext.Response.StatusCode = problem.Status.Value;
        await httpContext.Response.WriteAsJsonAsync(problem, ct);

        return true;   // handled, stop here
    }
}
```

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
```

Key points:

* **Log everything, return nothing sensitive.** The stack trace goes to logs with the traceId; the client gets the generic body + the same traceId. Support ticket says "traceId 00-4bf9...", you find the full story in the logs. That's the loop.
* Handlers chain: return `false` to pass to the next one, so you can have a specific handler before the catch-all:

```csharp
public class ConcurrencyExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not DbUpdateConcurrencyException) return false;

        ctx.Response.StatusCode = StatusCodes.Status409Conflict;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 409,
            Title = "Conflict",
            Detail = "The record was modified by someone else. Reload and retry."
        }, ct);
        return true;
    }
}

// order matters: specific first
builder.Services.AddExceptionHandler<ConcurrencyExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

---

## Don't build an exception-driven API

A tempting anti-pattern once the global handler exists: throw `NotFoundException`, `ConflictException`, `ValidationException` from services everywhere and let the handler map them all. It works, plenty of codebases do it. I avoid it as the *primary* flow because expected outcomes stop being visible in method signatures and exceptions become goto. My line:

* Expected outcome -> Result, mapped in the controller.
* Exceptional situation -> throw, caught centrally, 500 (or a specific handler like the concurrency one above).

Pick a lane per project and be consistent; half-Result half-exception codebases are the worst of both.

---

## Common Mistakes

* **`app.UseDeveloperExceptionPage()` behavior leaking to prod** (stack traces in responses). The default template guards it with `IsDevelopment()`; keep it that way.
* **try/catch in every action.** That's what the central handler removes. An action-level catch is only justified when that action can genuinely recover.
* **Catch-and-continue in services** (`catch (Exception) { return null; }`). Now it's not an error, it's a mystery null two layers up. Let it throw.
* **Different error shapes per endpoint.** One team member returns strings, another anonymous objects. ProblemDetails everywhere, no exceptions (pun intended).
* **Forgetting `AddProblemDetails()`**, so non-exception error paths (404 from routing, 405) return empty bodies instead of the standard shape.

---

## Summary

| Idea | Point |
| ---- | ----- |
| ProblemDetails | the standard error body; one shape for all failures |
| Business failures | Result -> controller maps to 4xx, no exceptions |
| Unexpected exceptions | IExceptionHandler: log full, return generic + traceId |
| Chaining | specific handlers first, catch-all last |
| Golden rule | client never sees internals; logs see everything |

> Builds on [rest-and-api-design.md](./rest-and-api-design.md) status codes and [../6.Design-Patterns/result-pattern.md](../6.Design-Patterns/result-pattern.md). The logging side is in [../8.Architecture/logging-and-serilog.md](../8.Architecture/logging-and-serilog.md).
