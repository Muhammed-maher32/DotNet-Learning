# Reflection & Attributes

> Reflection is code inspecting code at runtime: "what properties does this type have? does it carry this attribute? call this method by name". It's slow-ish and stringly, so you rarely write it yourself, but every framework you use (EF Core, ASP.NET Core, AutoMapper, serializers) is built on it.

---

## Type: the entry point

Every object knows its own `Type`, and every type has a metadata object describing it:

```csharp
var member = new Member { Name = "Ali" };

Type t1 = member.GetType();     // runtime type of an instance
Type t2 = typeof(Member);       // compile-time, no instance needed

Console.WriteLine(t1.Name);          // Member
Console.WriteLine(t1.FullName);      // GymSystem.DAL.Entities.Member
Console.WriteLine(t1.IsClass);       // True
```

From a `Type` you can walk everything:

```csharp
foreach (var prop in typeof(Member).GetProperties())
    Console.WriteLine($"{prop.PropertyType.Name} {prop.Name}");

// int Id
// string Name
// string Email
// ...
```

---

## Reading and writing values by name

```csharp
var prop = typeof(Member).GetProperty("Name")!;

object? value = prop.GetValue(member);   // "Ali"
prop.SetValue(member, "Omar");           // member.Name is now "Omar"
```

This is how AutoMapper copies `dto.Name` to `entity.Name` without knowing your types, and how a JSON serializer fills your object from `{"name": "Ali"}`. Property names as strings, matched at runtime.

You can also create instances and invoke methods dynamically:

```csharp
object obj = Activator.CreateInstance(typeof(Member))!;

var method = typeof(Member).GetMethod("Deactivate")!;
method.Invoke(obj, null);
```

`Activator.CreateInstance` is basically what the DI container does when it builds your services: look at the constructor via reflection, resolve each parameter, invoke it.

---

## Attributes: metadata you attach

An attribute is just a class inheriting `Attribute` that you stick on code. It does nothing by itself. Something has to *read* it with reflection for it to matter.

You already use them constantly:

```csharp
public class Member
{
    [Key]
    public int Id { get; set; }

    [Required, MaxLength(100)]
    public string Name { get; set; } = null!;
}

[HttpGet("{id}")]
public IActionResult Get(int id) { ... }
```

`[Required]` doesn't validate anything on its own. ASP.NET Core's validation system scans your model's properties with reflection, finds the attribute, and runs its logic. Same for `[Key]` (EF Core reads it building the model) and `[HttpGet]` (routing reads it at startup).

---

## Writing your own attribute

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class AuditAttribute : Attribute
{
    public string Reason { get; }
    public AuditAttribute(string reason) => Reason = reason;
}

public class Member
{
    [Audit("PII, log all changes")]
    public string Email { get; set; } = null!;
}
```

And the part everyone forgets, the reader:

```csharp
foreach (var prop in typeof(Member).GetProperties())
{
    var audit = prop.GetCustomAttribute<AuditAttribute>();
    if (audit != null)
        Console.WriteLine($"{prop.Name} is audited: {audit.Reason}");
}
```

Attribute + reflection reader is the pattern. Declare intent on the model, act on it somewhere central. That's the whole trick behind validation, routing, serialization options, test frameworks finding `[Fact]` methods, all of it.

---

## The cost

Reflection is much slower than a direct call (metadata lookups, no inlining, boxing). Frameworks get away with it by doing the reflection **once at startup** and caching the result, often compiling it into fast delegates. Follow the same rule:

* Reflect once, cache the `PropertyInfo`/`MethodInfo`, reuse it.
* Never reflect inside a hot loop per item.
* If you find yourself reflecting a lot, there's usually a library (or source generators these days) that did it properly.

---

## Common Mistakes

* **Typo'd member names.** `GetProperty("Nmae")` returns null, you get a `NullReferenceException` two lines later. Reflection trades compile-time safety for flexibility, so guard the nulls.
* **Forgetting binding flags.** `GetProperties()` returns public instance members only. Private or static needs `BindingFlags.NonPublic | BindingFlags.Instance` etc.
* **Using reflection where generics would do.** If you know the types at compile time, you don't need reflection. It's the last resort, not the first tool.
* **Attribute with no reader.** Sticking a custom attribute on things and expecting magic. Nothing happens until code scans for it.

---

## Summary

| Idea | Point |
| ---- | ----- |
| `Type` | runtime metadata object; `typeof(X)` or `obj.GetType()` |
| `PropertyInfo` etc. | read/write/invoke members by name at runtime |
| Attributes | passive metadata; only meaningful when something reads them |
| The pattern | declare with attribute, read with reflection, act centrally |
| Performance | reflect once and cache; never per-item in hot paths |

> Related: [generics.md](./generics.md) for the compile-time alternative, and [../4.Web-Basics/validation.md](../4.Web-Basics/validation.md) which is attribute + reflection in action.
