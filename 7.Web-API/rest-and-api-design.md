# REST & API Design

> REST in practice: resources as nouns, HTTP methods as verbs, status codes that tell the truth, and the conventions that make an API feel predictable to whoever consumes it.

---

## Resources and verbs

The mental shift from MVC pages: an API exposes *resources* (things), and HTTP methods already are the verbs. You don't invent action names.

```text
GET    /api/members          list members
GET    /api/members/5        one member
POST   /api/members          create a member
PUT    /api/members/5        replace member 5
PATCH  /api/members/5        partially update member 5
DELETE /api/members/5        delete member 5

GET    /api/members/5/subscriptions     nested resource: this member's subscriptions
```

Signs you're doing it wrong: verbs in URLs (`/api/getMembers`, `/api/members/delete/5`). The method is the verb; the URL is only ever a noun.

Semantics worth knowing:

* **GET** is safe (no side effects) and cacheable.
* **PUT** is *idempotent*: sending it twice leaves the same result as once. So is DELETE. POST is not (twice = two members). Idempotency matters because clients and proxies retry.
* **PATCH** sends only the fields to change; PUT conceptually sends the whole resource.

For actions that genuinely aren't CRUD (cancel, activate, transfer), the pragmatic convention is a sub-resource verb: `POST /api/subscriptions/5/cancel`. Purists argue; everybody does it anyway.

---

## Status codes that tell the truth

The consumer should be able to react from the status code alone, before parsing anything.

| Code | When |
| ---- | ---- |
| 200 OK | successful GET/PUT/PATCH with a body |
| 201 Created | successful POST, plus a `Location` header to the new resource |
| 204 No Content | success, nothing to return (DELETE, sometimes PUT) |
| 400 Bad Request | malformed/invalid input |
| 401 Unauthorized | not authenticated (who are you?) |
| 403 Forbidden | authenticated but not allowed (I know you, no) |
| 404 Not Found | resource doesn't exist |
| 409 Conflict | valid request, current state refuses it (email taken) |
| 422 Unprocessable | some teams use it for semantic validation errors instead of 400 |
| 500 | your bug. Never return 500 for a business failure |

The worst API habit: returning 200 with `{ "success": false, "error": "..." }` in the body. Clients, monitoring, caches, and retry logic all rely on status codes; lying with 200 breaks all of it.

In a controller this mapping looks like:

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateMemberDto dto, CancellationToken ct)
{
    var result = await _memberService.CreateMemberAsync(dto, ct);

    return result.Kind switch
    {
        ResultKind.Ok        => CreatedAtAction(nameof(Get), new { id = result.Value }, result.Value),
        ResultKind.Conflict  => Conflict(new { error = result.Error }),
        ResultKind.NotFound  => NotFound(),
        _                    => BadRequest(new { error = result.Error })
    };
}
```

---

## [ApiController] and what it gives you

```csharp
[ApiController]
[Route("api/[controller]")]
public class MembersController : ControllerBase   // not Controller, no views
```

The attribute switches on API behaviors: automatic 400 with validation details when ModelState fails (no manual `if (!ModelState.IsValid)`), inference that complex parameters come from the body, and ProblemDetails-shaped error responses.

---

## Query parameters: filtering, sorting, paging

Collections grow. Bake this in from day one, retrofitting paging onto a public API is painful:

```text
GET /api/members?search=ali&planId=2&sort=-joinDate&page=2&pageSize=20
```

```csharp
public record MemberQueryParams(
    string? Search, int? PlanId, string? Sort,
    int Page = 1, int PageSize = 20);

[HttpGet]
public async Task<IActionResult> List([FromQuery] MemberQueryParams query, CancellationToken ct)
{
    var result = await _memberService.GetPagedAsync(query, ct);
    return Ok(result);   // { items: [...], page: 2, pageSize: 20, totalCount: 143 }
}
```

Return the paging metadata with the items (wrapper object or headers, pick one and stay consistent). Cap `pageSize` server-side; someone will send `pageSize=1000000`.

---

## Versioning

The day you must change a response shape, existing clients break. Version from the start, even if v1 is the only version for years. URL versioning is the most common because it's visible and easy to route:

```text
/api/v1/members
/api/v2/members
```

With `Asp.Versioning.Mvc`:

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
});
```

```csharp
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/members")]
public class MembersController : ControllerBase { ... }
```

Header and query-string versioning exist too; URL versioning wins on simplicity. The deeper rule: additive changes (new optional field) don't need a version bump, breaking changes (rename, remove, retype) do.

---

## Content negotiation, briefly

The `Accept` request header says what format the client wants, `Content-Type` says what the body is. ASP.NET Core answers JSON by default and that's what everyone uses. Know the headers exist, add other formatters only when a real client demands XML.

---

## Common Mistakes

* **200 for everything** with success flags in the body.
* **Returning entities.** Leaks your schema, creates EF serialization cycles, and couples clients to your database. DTOs always ([../6.Design-Patterns/dtos-and-automapper.md](../6.Design-Patterns/dtos-and-automapper.md)).
* **No paging on collections** until the table is big and the endpoint times out.
* **Inconsistency.** camelCase here, PascalCase there, wrapper object on one endpoint, bare array on another. Consumers script against patterns; pick conventions and never deviate.
* **500 for business failures.** "Email taken" is a 409, not an exception bubbling into a server error.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Resources | nouns in URLs, HTTP methods as verbs |
| Idempotency | PUT/DELETE safe to retry, POST is not |
| Status codes | react-able without parsing the body; never lie with 200 |
| Paging/filtering | query params + metadata, capped page size, from day one |
| Versioning | /api/v1/ from the start; version on breaking changes only |

> Continues into [error-handling-and-problem-details.md](./error-handling-and-problem-details.md) for the error body format, and [swagger-and-openapi.md](./swagger-and-openapi.md) for documenting all of this.
