# Pattern Matching in C#

> A set of features that let you test "what shape is this value" and pull data out of it in one move. Modern C# leans on it heavily, so it's worth knowing all the forms.

---

## `is` with a type pattern

The old way: check type, then cast.

```csharp
if (obj is string)
{
    var s = (string)obj;   // redundant cast
    Console.WriteLine(s.Length);
}
```

The pattern way: check and capture in one shot.

```csharp
if (obj is string s)   // if it's a string, bind it to `s`
{
    Console.WriteLine(s.Length);
}
```

---

## switch expression

Not the old `switch` statement with `case`/`break`. This is an expression that returns a value.

```csharp
string Describe(int n) => n switch
{
    < 0      => "negative",
    0        => "zero",
    > 0 and < 10 => "small",
    _        => "big"        // _ is the default, required if not exhaustive
};
```

Way less noise than a chain of `if/else if`. The `_` is the discard, meaning "anything else".

---

## relational and logical patterns

You can combine comparisons with `and`, `or`, `not`:

```csharp
bool IsLetter(char c) => c is (>= 'a' and <= 'z') or (>= 'A' and <= 'Z');

string Grade(int score) => score switch
{
    >= 90 => "A",
    >= 80 => "B",
    >= 70 => "C",
    _     => "F"
};
```

---

## type patterns in a switch

Switch on the actual type of an object:

```csharp
decimal Area(Shape shape) => shape switch
{
    Circle c    => Math.PI * c.Radius * c.Radius,
    Rectangle r => r.Width * r.Height,
    Triangle t  => 0.5m * t.Base * t.Height,
    null        => throw new ArgumentNullException(nameof(shape)),
    _           => throw new ArgumentException("unknown shape")
};
```

This is the clean replacement for a big `if (shape is Circle) ... else if (shape is Rectangle) ...` ladder.

---

## property patterns

Match on the values of an object's properties:

```csharp
string Ship(Order o) => o switch
{
    { Total: > 100 }                  => "free shipping",
    { Country: "EG", Total: > 50 }    => "discounted shipping",
    { IsExpress: true }               => "express",
    _                                 => "standard"
};
```

You can nest them and even capture:

```csharp
if (person is { Address: { City: "Cairo" } })
{
    // person has an address and the city is Cairo
}
```

---

## positional patterns (with deconstruction / records)

If a type can deconstruct (records do this for free), you can match by position:

```csharp
public record Point(int X, int Y);

string Quadrant(Point p) => p switch
{
    (0, 0)           => "origin",
    (> 0, > 0)       => "Q1",
    (< 0, > 0)       => "Q2",
    var (x, y)       => $"({x},{y})"   // capture both
};
```

---

## when to reach for it

* Replacing long `if/else if` chains that branch on type or value.
* Mapping one thing to another (status -> label, enum -> behavior).
* Pulling fields out while you check a condition.

It makes code shorter and, once you're used to reading it, clearer. Don't force it where a plain `if` is obviously simpler.

---

## cheat sheet

| Pattern | Example |
| ------- | ------- |
| type | `obj is string s` |
| constant | `n switch { 0 => ... }` |
| relational | `>= 90` |
| logical | `> 0 and < 10`, `"a" or "b"`, `not null` |
| property | `{ Total: > 100 }` |
| positional | `(0, 0)`, `var (x, y)` |
| discard | `_` |
