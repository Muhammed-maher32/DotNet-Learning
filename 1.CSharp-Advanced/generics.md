# Generics in C#

---

## What are Generics?

Generics allow you to define classes, methods, and data structures with a placeholder type.

> write once, use with any type (type-safe)

---

## Why use Generics?

* Type safety (no casting)
* Code reusability
* Better performance (no boxing/unboxing)

---

## Basic Example

```csharp
List<int> numbers = new();
List<string> names = new();
//List<T> Class
//Same structure → different types
```
*List:* [official_dotnet_repo](https://github.com/dotnet/dotnet/blob/b0f34d51fccc69fd334253924abd8d6853fad7aa/src/runtime/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/List.cs)


---

## The Real Idea

Generics are not just about types…

> they allow writing logic that works with any type while keeping type safety

---

## Invariance (Important)

```csharp
List<Cat> cats = new();
List<Animal> animals = cats; //invalid
```

Even though:

```text
Cat : Animal
```
**Still not allowed!!**
---

## Why?

Because List allows adding:
```csharp
animals.Add(new Dog()); //breaks type safety
```
---

## Covariance (out)

```csharp
IEnumerable<Animal> animals = cats; // valid

//Covariance allow read-only
```
Why?
* IEnumerable is **read-only**
* no adding which is safe

---

## Contravariance (in)

```csharp
Action<Animal> act = (Animal a) => Console.WriteLine(a);
Action<Cat> catAct = act; //valid
```

Why?
* Method that accepts Animal can accept Cat
---

## Constraints

```csharp
class Repository<T> where T : class, new()
{
}
```

Common constraints:

* `where T : class`
* `where T : struct`
* `where T : new()`
* `where T : BaseClass`
* `where T : interface`

---

## Real World Usage

* Collections (List, Dictionary)
* APIs

---

## Common Mistakes

* Assuming all generics are interchangeable
* Ignoring variance (in / out)

