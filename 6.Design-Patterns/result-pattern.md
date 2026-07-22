# The Result Pattern

> Instead of throwing exceptions for expected failures (email taken, not found, validation failed), a method returns an object that says "did it work, and if not, why". The caller checks it and reacts.

---

## The problem with exceptions for everything

Exceptions are for **exceptional** stuff: the database is down, a null blew up, disk is full. Things you didn't expect.

But "the email is already in use" is not exceptional, it's a normal business outcome that happens all the time. Using exceptions for it has downsides:

```csharp
public void CreateMember(CreateMemberDto dto)
{
    if (EmailExists(dto.Email))
        throw new EmailInUseException();   // expensive, and now everyone needs try/catch
}
```

* Exceptions are relatively expensive (stack traces).
* Control flow gets ugly, every caller wraps things in try/catch.
* It's not obvious from the method signature that it can "fail" in a normal way.

---

## The Result type

You make a small type that carries success/failure plus a reason:

```csharp
public enum ResultKind
{
    Ok,
    NotFound,
    Conflict,
    ValidationFailed,
    Forbidden
}

public sealed record Result(bool Success, string? Error = null, ResultKind Kind = ResultKind.Ok)
{
    public static Result Ok()                              => new(true);
    public static Result Fail(string error)               => new(false, error, ResultKind.Conflict);
    public static Result NotFound(string error = "Not found") => new(false, error, ResultKind.NotFound);
    public static Result Validation(string error)         => new(false, error, ResultKind.ValidationFailed);
}
```

And a generic version for when you want to return a value on success:

```csharp
public sealed record Result<T>(bool Success, T? Value, string? Error = null, ResultKind Kind = ResultKind.Ok)
{
    public static Result<T> Ok(T value)                       => new(true, value);
    public static Result<T> Fail(string error)               => new(false, default, error, ResultKind.Conflict);
    public static Result<T> NotFound(string error = "Not found") => new(false, default, error, ResultKind.NotFound);
}
```

The `Kind` enum is the useful bit. It lets the caller map a failure to the right response without parsing the error string.

---

## In a service

```csharp
public async Task<Result> CreateMemberAsync(CreateMemberDto dto, CancellationToken ct)
{
    var repo = _unitOfWork.GetRepository<Member>();

    if (await repo.AnyAsync(m => m.Email == dto.Email, ct))
        return Result.Fail("Email already in use.");

    repo.Add(_mapper.Map<Member>(dto));
    var rows = await _unitOfWork.SaveChangesAsync(ct);

    return rows > 0 ? Result.Ok() : Result.Fail("Try again.");
}
```

No exceptions, no try/catch. The method clearly returns a `Result`, so the caller knows it might not succeed.

---

## In an MVC controller

```csharp
public async Task<IActionResult> Create(CreateMemberDto dto, CancellationToken ct)
{
    if (!ModelState.IsValid) return View(dto);

    var result = await _memberService.CreateMemberAsync(dto, ct);

    if (!result.Success)
    {
        ModelState.AddModelError(string.Empty, result.Error!);
        return View(dto);
    }

    TempData["SuccessMessage"] = "Member created.";
    return RedirectToAction(nameof(Index));
}
```

## In a Web API controller (mapping Kind to status codes)

This is where `ResultKind` pays off:

```csharp
public async Task<IActionResult> Get(int id, CancellationToken ct)
{
    var result = await _service.GetByIdAsync(id, ct);

    if (result.Success)
        return Ok(result.Value);

    return result.Kind switch
    {
        ResultKind.NotFound         => NotFound(result.Error),
        ResultKind.ValidationFailed => BadRequest(result.Error),
        ResultKind.Forbidden        => Forbid(),
        _                           => Conflict(result.Error)
    };
}
```

One clean place translates business outcomes into HTTP, instead of throwing custom exceptions and catching them in middleware.

---

## When NOT to use it

* For genuinely unexpected errors (DB connection lost, bug), let it throw. Don't wrap everything.
* On tiny apps it can be more ceremony than it's worth. Judge it.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Exceptions | for unexpected failures only |
| Result | for expected business outcomes (taken, not found, invalid) |
| `Kind` | lets callers map failures to responses (HTTP codes, UI messages) |
| Payoff | clean control flow, honest method signatures, cheaper than throwing |

> Real example: I used this `Result` / `Result<T>` setup in my GymSystem project to keep the services free of exception spaghetti. Pairs naturally with [repository-and-unit-of-work.md](./repository-and-unit-of-work.md).
