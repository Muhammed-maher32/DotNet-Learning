# Validation

> Validation is checking that incoming data is sane before you act on it. In ASP.NET Core the framework does a lot of it for you and dumps the results into `ModelState`. Your job is to declare the rules and react when they fail.

Rule one: never trust client input. The browser/JS validation is just UX. The server must validate too, always.

---

## Data annotations (the built-in way)

You put attributes on your DTO/ViewModel properties:

```csharp
public class CreateProductDto
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; } = default!;

    [Range(0.01, 10000)]
    public decimal Price { get; set; }

    [EmailAddress]
    public string ContactEmail { get; set; } = default!;

    [RegularExpression(@"^(010|011|012|015)\d{8}$", ErrorMessage = "Invalid Egyptian phone")]
    public string Phone { get; set; } = default!;
}
```

Common ones: `[Required]`, `[StringLength]` / `[MaxLength]`, `[Range]`, `[EmailAddress]`, `[Phone]`, `[Url]`, `[RegularExpression]`, `[Compare]` (e.g. confirm password).

---

## How ModelState works

When a request comes in, the framework binds the JSON/form to your DTO and runs the annotations. The results land in `ModelState`.

### In an MVC controller (you check it yourself)

```csharp
[HttpPost]
public IActionResult Create(CreateProductDto dto)
{
    if (!ModelState.IsValid)
        return View(dto);   // redisplay form with the errors

    // safe to proceed
}
```

### In a Web API controller with `[ApiController]` (automatic)

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateProductDto dto)
    {
        // if invalid, you NEVER get here.
        // [ApiController] auto-returns 400 with the validation errors.
        return Ok();
    }
}
```

That automatic 400 is a big convenience. Without `[ApiController]` you do the manual `if (!ModelState.IsValid)` check.

---

## When annotations aren't enough

Annotations are great for single-field rules. They struggle with rules that span multiple fields or need a service/DB.

### Option A: IValidatableObject (cross-field, no dependencies)

```csharp
public class DateRangeDto : IValidatableObject
{
    public DateTime Start { get; set; }
    public DateTime End { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (End <= Start)
            yield return new ValidationResult(
                "End must be after Start.",
                new[] { nameof(End) });
    }
}
```

### Option B: FluentValidation (the real-project favorite)

A library where rules live in a separate class, not on the model. Cleaner for anything complex, and easy to unit test.

```csharp
public class CreateProductValidator : AbstractValidator<CreateProductDto>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty().Length(3, 100);
        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.ContactEmail).EmailAddress();

        // cross-field
        RuleFor(x => x.End).GreaterThan(x => x.Start);

        // can even call a service/DB (e.g. uniqueness)
        RuleFor(x => x.Email).MustAsync(BeUniqueEmail);
    }
}
```

Why people reach for it: the rules are separated from the model, composable, reusable, and readable for things annotations make ugly.

---

## Where validation fits in the bigger picture

There are usually two layers:

1. **Input validation** (this note): is the data well-formed? Required fields present, formats correct. Belongs at the edge (DTO + ModelState / FluentValidation).
2. **Business rules**: is this allowed *given the system state*? "Email already taken", "can't delete a member with active bookings". That's not really validation, it's logic. It belongs in your service and is better returned as a Result than thrown. (See the result-pattern note.)

Don't cram "email already exists" into a data annotation. It needs the database. Keep those in the service layer.

---

## Summary

| Tool | Use for |
| ---- | ------- |
| Data annotations | simple single-field rules (`[Required]`, `[Range]`, regex) |
| `ModelState.IsValid` | check result; `[ApiController]` auto-returns 400 |
| `IValidatableObject` | cross-field rules with no external dependencies |
| FluentValidation | complex/reusable rules, or anything needing a service or DB |
| service + Result | business rules that depend on system state (not "validation") |

> Always validate on the server. Client-side checks are only for nice UX, never for trust.
