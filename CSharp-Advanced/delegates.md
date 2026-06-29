# Delegates in C#

---

## What is a Delegate?

A delegate is a type that represents a reference to a method.
It allows you to pass methods as parameters.

> a variable that holds a function

*Official_doc:* -> [Open](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/)


---

## Why use Delegates?

* Decouple logic from implementation
* More flexible code
* Reusable behavior
* Supports Open/Closed Principle (OCP)

---

## Built-in Delegates

### Func

```csharp
// Func<T, TResult>

Func<int, int> square = x => x * x;
```

* Takes input parameters
* Returns a value
* Supports up to 16 parameters

---

### Action

```csharp
// Action<T>

Action<string> print = msg => Console.WriteLine(msg);
```

* Takes input parameters
* Returns void
* Supports up to 16 parameters

---

### Predicate

```csharp
Predicate<int> isEven = x => x % 2 == 0;
```

* Takes one parameter
* Returns bool

---

## Example

```csharp
List<int> numbers = new() {1,2,3,4,5};

List<int> evenNumbers = numbers.FindAll(x => x % 2 == 0);
// FindAll uses Predicate<T>
//public System.Collections.Generic.List<T> FindAll(Predicate<T> match);


foreach (var num in evenNumbers)
{
    Console.WriteLine(num);
}
```

---

## Passing Behavior 

```csharp
void Process(List<int> nums, Func<int, bool> condition)
{
    foreach (var n in nums)
    {
        if (condition(n))
            Console.WriteLine(n);
    }
}
```

```csharp
//Now you can change behavior without changing the method:

Process(numbers, x => x > 3);
Process(numbers, x => x % 2 == 0);
```

---

## Real World Usage

* Filtering collections
* Event handling
* Callbacks and extensibility

---

## Common Mistakes

* Overusing delegates when simple methods are enough
* Confusing Func and Action
* Not understanding when to pass behavior

