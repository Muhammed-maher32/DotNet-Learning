# Minimal APIs

> Endpoints as lambdas mapped straight onto the app, no controller classes. Less ceremony, same pipeline underneath. Since .NET 7-8 they've caught up on filters, validation hooks, and grouping, so the choice against controllers is now mostly taste and team convention.

---

## The shape

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IMemberService, MemberService>();

var app = builder.Build();

app.MapGet("/api/members/{id:int}", async (int id, IMemberService service, CancellationToken ct) =>
{
    var member = await service.GetByIdAsync(id, ct);
    return member is null ? Results.NotFound() : Results.Ok(member);
});

app.Run();
```

Things to notice:

* **Parameter binding is inferred.** `id` from the route, `IMemberService` from DI, `CancellationToken` from the request. Complex types come from the JSON body for POST/PUT. `[FromQuery]`/`[FromBody]` attributes exist for the ambiguous cases.
* **`Results.*`** is the return helper family (`Results.Ok`, `Results.NotFound`, `Results.Created`...). The typed variant `TypedResults.*` is better when you can use it: it feeds OpenAPI metadata without extra attributes.

---

## Route groups

Without structure, Program.cs becomes a 900-line dump. Groups fix the prefix repetition and shared config:

```csharp
var members = app.MapGroup("/api/members")
    .WithTags("Members")
    .RequireAuthorization();

members.MapGet("/", GetAllMembers);
members.MapGet("/{id:int}", GetMemberById);
members.MapPost("/", CreateMember);
members.MapDelete("/{id:int}", DeleteMember).RequireAuthorization("AdminOnly");
```

And the real structural fix: endpoints as static methods in feature files, not inline lambdas.

```csharp
// Features/Members/MemberEndpoints.cs
public static class MemberEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/members").WithTags("Members");
        group.MapGet("/{id:int}", GetById);
        group.MapPost("/", Create);
    }

    private static async Task<IResult> GetById(int id, IMemberService service, CancellationToken ct)
        => await service.GetByIdAsync(id, ct) is { } member
            ? TypedResults.Ok(member)
            : TypedResults.NotFound();

    private static async Task<IResult> Create(CreateMemberDto dto, IMemberService service, CancellationToken ct)
    {
        var result = await service.CreateMemberAsync(dto, ct);
        return result.Success
            ? TypedResults.Created($"/api/members/{result.Value}", result.Value)
            : TypedResults.Conflict(result.Error);
    }
}

// Program.cs stays tiny
MemberEndpoints.Map(app);
```

At this point the code looks suspiciously like a controller, which is sort of the point: organized minimal APIs and controllers converge.

---

## Endpoint filters

The minimal-API answer to action filters, for per-endpoint/group cross-cutting logic:

```csharp
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var dto = context.Arguments.OfType<T>().FirstOrDefault();
        if (dto is null)
            return TypedResults.BadRequest("Missing body.");

        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is not null)
        {
            var result = await validator.ValidateAsync(dto);
            if (!result.IsValid)
                return TypedResults.ValidationProblem(result.ToDictionary());
        }

        return await next(context);
    }
}
```

```csharp
group.MapPost("/", Create).AddEndpointFilter<ValidationFilter<CreateMemberDto>>();
```

One difference from MVC: there's no automatic `[ApiController]`-style model validation, you opt in with filters (FluentValidation + a filter like the above is the common combo).

---

## Minimal APIs vs controllers, honestly

| | Minimal APIs | Controllers |
| --- | --- | --- |
| Ceremony | low, great for small services | class + attributes per resource |
| Organization | your discipline (groups + feature files) | convention forces it |
| Model validation | opt-in via filters | automatic with [ApiController] |
| Performance | slightly better (less machinery) | fine |
| Team familiarity | newer | everyone knows it |

My take: small focused services, gateways, webhooks, anything with a dozen endpoints -> minimal APIs feel great. Big API with many resources and a team used to MVC -> controllers' forced structure is a feature. Mixing in one app works too (`MapControllers()` and `MapGet` coexist happily). The request pipeline, middleware, DI, and auth are identical either way; this is a surface-level choice, not an architecture choice.

---

## Common Mistakes

* **Everything inline in Program.cs.** Fine at 5 endpoints, unmaintainable at 40. Feature files + groups from early on.
* **Expecting automatic validation** like [ApiController]. Nothing validates until you add a filter.
* **Business logic in the lambda.** Same sin as fat controllers. Lambda binds and translates; services do the work.
* **`Results` when `TypedResults` would do**, losing free OpenAPI metadata and testability (TypedResults return concrete types you can assert on).

---

## Summary

| Idea | Point |
| ---- | ----- |
| MapGet/MapPost | endpoints as functions, binding inferred |
| Route groups | shared prefix, tags, auth per resource |
| Endpoint filters | validation and cross-cutting per endpoint/group |
| TypedResults | typed returns, better OpenAPI, testable |
| Choice | taste + team; the pipeline underneath is identical |

> Same design rules as [rest-and-api-design.md](./rest-and-api-design.md) apply. Error handling for both styles is in [error-handling-and-problem-details.md](./error-handling-and-problem-details.md).
