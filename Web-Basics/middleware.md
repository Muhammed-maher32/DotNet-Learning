# Middleware & The Request Pipeline

> Middleware is a chain of components that every request passes through on the way in, and every response passes back through on the way out. Each one can do something, then pass control to the next, or stop the chain.

The fundamentals note mentioned this. Here's the deeper version.

---

## The mental model: an onion

A request goes **in** through each layer, hits the endpoint, and the response comes **back out** through the same layers in reverse.

```
Request  ->  MW1  ->  MW2  ->  MW3  ->  Endpoint
Response <-  MW1  <-  MW2  <-  MW3  <-
```

So each middleware sees the request on the way in AND the response on the way out. That's why the structure is "before next(), after next()".

---

## Writing custom middleware (inline)

The simplest form is `app.Use` with a lambda. `next` is the rest of the pipeline.

```csharp
app.Use(async (context, next) =>
{
    // ---- before: runs on the way IN ----
    var sw = Stopwatch.StartNew();

    await next(); // hand off to the next middleware / the endpoint

    // ---- after: runs on the way OUT ----
    sw.Stop();
    Console.WriteLine($"{context.Request.Path} took {sw.ElapsedMilliseconds}ms");
});
```

If you don't call `await next()`, the pipeline **short circuits**: nothing after you runs, and the response starts heading back out. That's how auth middleware can reject a request early.

```csharp
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("X-Api-Key"))
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync("missing api key");
        return; // short circuit: don't call next()
    }
    await next();
});
```

---

## Use vs Map vs Run

Three verbs, different jobs:

| Method | What it does |
| ------ | ------------ |
| `app.Use(...)` | a middleware that can call `next()` to continue the chain |
| `app.Run(...)` | a terminal middleware, ends the pipeline (never calls next) |
| `app.Map(...)` | branches the pipeline based on the request path |

```csharp
app.Map("/admin", admin =>
{
    admin.Run(async ctx => await ctx.Response.WriteAsync("admin area"));
});

app.Run(async ctx => await ctx.Response.WriteAsync("everything else"));
```

---

## Order is everything

Middleware runs in the exact order you register it. Get the order wrong and things break silently.

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();      // serve files before hitting routing/auth
app.UseRouting();
app.UseAuthentication();   // must be before authorization
app.UseAuthorization();
app.MapControllers();      // endpoints last
```

Classic bugs:

* `UseAuthorization` before `UseAuthentication` -> auth doesn't work (user isn't identified yet).
* `UseStaticFiles` after auth -> your css/js needlessly go through the auth checks.
* Exception handling middleware registered late -> it won't catch errors from middleware registered before it. Put it near the **top**.

---

## The cleaner pattern: a middleware class

Inline lambdas are fine for small things. For real middleware, make a class. The convention: a constructor taking `RequestDelegate next`, and an `InvokeAsync(HttpContext)` method.

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestTimingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        Console.WriteLine($"{context.Request.Path} -> {sw.ElapsedMilliseconds}ms");
    }
}
```

Register it:

```csharp
app.UseMiddleware<RequestTimingMiddleware>();
```

---

## Middleware vs Filters (the recurring question)

* **Middleware** runs for *every* request, sits at the framework level, doesn't know about controllers/actions/models.
* **Filters** run only inside MVC, around actions, and *do* know the action and model.

Use middleware for low level, app wide concerns (logging, CORS, exception handling, static files). Use a filter when you need to know which action ran or inspect the model. (See the filters note.)

---

## Summary

| Concept | Point |
| ------- | ----- |
| Pipeline | request in through each middleware, response back out (onion) |
| `next()` | pass control onward; skip it to short circuit |
| `Use` / `Run` / `Map` | continue / terminate / branch |
| Order | runs top to bottom, auth before authorization, exceptions near the top |
| Class form | `RequestDelegate next` + `InvokeAsync`, register with `UseMiddleware<T>` |
