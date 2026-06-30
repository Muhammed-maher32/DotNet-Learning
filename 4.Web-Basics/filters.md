# ASP.NET Core Filters

> Filters let you run code at specific points in the MVC request pipeline, around your actions, without copy pasting the same logic into every controller. Think cross cutting stuff: logging, validation, auth checks, exception handling, caching.

---

## Where filters sit

Middleware runs for every request. Filters run **inside** MVC, only around controller actions, and they know about the action, the model, the result. So they're more specific than middleware.

```
Request
  -> Middleware
    -> Routing
      -> [ Authorization filter ]
        -> [ Resource filter ]
          -> Model binding
            -> [ Action filter ]  -> ACTION -> [ Action filter ]
              -> [ Result filter ] -> result executes -> [ Result filter ]
        [ Exception filter wraps the whole thing ]
```

---

## The filter types

| Filter | Runs | Used for |
| ------ | ---- | -------- |
| Authorization | first | is the user allowed? short circuit if not |
| Resource | after auth, around everything else | caching, short circuiting before model binding |
| Action | right before/after the action method | validation, logging args, modifying inputs/outputs |
| Exception | when an action throws | turn exceptions into clean responses |
| Result | before/after the result executes | tweak the response, add headers |

You'll mostly write **Action** and **Exception** filters.

---

## Action filter example

```csharp
public class LogActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // before the action runs
        Console.WriteLine($"calling {context.ActionDescriptor.DisplayName}");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        // after the action runs
        Console.WriteLine("done");
    }
}
```

A super common real use: auto-validate the model so you don't write `if (!ModelState.IsValid) return BadRequest()` in every action.

```csharp
public class ValidateModelFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
            context.Result = new BadRequestObjectResult(context.ModelState); // short circuit
    }

    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

Setting `context.Result` short circuits: the action never runs.

---

## Exception filter example

Catch exceptions from actions and return a consistent error shape:

```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        context.Result = new ObjectResult(new { error = context.Exception.Message })
        {
            StatusCode = 500
        };
        context.ExceptionHandled = true; // mark it handled so it doesn't bubble further
    }
}
```

---

## How to apply a filter

Three scopes, smallest to biggest:

```csharp
// 1. on a single action
[ServiceFilter(typeof(LogActionFilter))]
public IActionResult Get() { ... }

// 2. on a whole controller
[ServiceFilter(typeof(LogActionFilter))]
public class ProductsController : ControllerBase { }

// 3. globally, every action in the app (in Program.cs)
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidateModelFilter>();
});
```

If your filter has dependencies (it needs DI), register it and use `[ServiceFilter]` or add it by type globally. A plain `[TypeFilter]` or attribute works for filters without DI.

---

## Filter vs Middleware (the usual question)

| | Middleware | Filter |
| --- | --- | --- |
| Scope | every request | only MVC actions |
| Knows about action/model | no | yes |
| Good for | auth, CORS, static files, low level stuff | validation, action logging, action level exception handling |

Rule of thumb: if it needs to know which action ran or see the model, use a filter. If it's lower level and applies to all traffic, use middleware.

---

## Summary

* Filters = run code around actions, for cross cutting concerns.
* Main ones you'll write: Action (validation/logging) and Exception (clean error responses).
* Set `context.Result` to short circuit.
* Apply per action, per controller, or globally.
* Use a filter (not middleware) when you need action/model context.
