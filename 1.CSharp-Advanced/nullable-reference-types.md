# Nullable Reference Types (NRT)

> A compiler feature that makes reference types "non-null by default" and warns you when you might be dereferencing a null. It doesn't change runtime behavior, it's all compile-time warnings to help you dodge `NullReferenceException`.

---

## The problem it targets

The billion dollar mistake: `NullReferenceException`. Before NRT, every reference could be null and the compiler said nothing.

```csharp
string name = null;       // totally fine before NRT
Console.WriteLine(name.Length); // boom at runtime
```

---

## Turning it on

In the `.csproj` (it's on by default in new projects):

```xml
<Nullable>enable</Nullable>
```

Now the compiler splits reference types into two flavors:

```csharp
string  name;   // non-nullable: "I promise this won't be null"
string? note;   // nullable: "this might be null, handle it"
```

The `?` is the difference. Same as nullable value types (`int?`), now extended to reference types.

---

## What you get

```csharp
string name = null;     // ⚠️ warning: assigning null to non-nullable
string? note = null;    // fine, you said it can be null

Console.WriteLine(note.Length);  // ⚠️ warning: note may be null here
```

The compiler now nags you at exactly the spots where a null could sneak in. You fix them at build time instead of finding them in production.

---

## Handling the nullable case

Once you check for null, the compiler is smart enough to know it's safe (flow analysis):

```csharp
string? note = GetNote();

if (note != null)
{
    Console.WriteLine(note.Length); // no warning, compiler knows it's not null here
}
```

Other tools:

```csharp
// null-conditional: only access if not null, otherwise the whole thing is null
int? len = note?.Length;

// null-coalescing: fallback value
string display = note ?? "no note";

// null-coalescing assignment
note ??= "default";
```

---

## The `!` null-forgiving operator

`!` tells the compiler "trust me, this isn't null here, stop warning". It's an override, not a fix.

```csharp
string value = note!;   // I swear note isn't null. (if you're wrong -> runtime crash)
```

Use it sparingly. Every `!` is you taking responsibility away from the compiler. If you find yourself spamming it, your nullability annotations are probably wrong somewhere.

---

## Common patterns you'll write

```csharp
// required-ish property that EF/serializers will fill, silence the warning honestly:
public string Name { get; set; } = default!;   // "it'll be set, trust me"

// or better when you can, give a real default:
public string Name { get; set; } = string.Empty;

// constructor guard for public APIs:
public Service(IRepo repo)
{
    _repo = repo ?? throw new ArgumentNullException(nameof(repo));
}
```

That `= default!;` shows up a lot on entity/ViewModel properties. It means "non-nullable, and I'm promising it gets assigned later (by EF, by the binder, etc), so don't warn me".

---

## Important: it's only compile-time

NRT does **not** add runtime null checks. A `string` can still be null at runtime if something bypasses the compiler (reflection, JSON deserialization, an old library, an explicit `!`). The annotations are guidance and warnings, not a runtime guarantee. So for public API boundaries you still validate input.

---

## Summary

| Thing | Meaning |
| ----- | ------- |
| `string` | non-nullable, warns if you assign/return null |
| `string?` | nullable, warns if you use it without a null check |
| `?.` | access only if not null |
| `??` | fallback value |
| `!` | "trust me it's not null", overrides the warning (use rarely) |
| `= default!` | non-nullable property assigned later, silence the warning |
| runtime | NRT is compile-time only, no runtime null checks added |
