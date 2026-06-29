# Exception Handling in C#

> An exception is the runtime's way of saying "I can't continue normally". You either handle it or it bubbles up the call stack until something does, or the app crashes.

---

## The basics

```csharp
try
{
    var result = 10 / divisor;   // might throw DivideByZeroException
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"can't divide by zero: {ex.Message}");
}
finally
{
    // runs no matter what: success, caught exception, even uncaught
    Console.WriteLine("cleanup");
}
```

* `try` -> code that might blow up.
* `catch` -> handle a specific exception type.
* `finally` -> always runs, used for cleanup (close files, release locks).

---

## Catch specific, not everything

Order matters: most specific first, because the first matching catch wins.

```csharp
try { /* ... */ }
catch (FileNotFoundException ex) { /* handle missing file */ }
catch (IOException ex)           { /* other IO problems */ }
catch (Exception ex)             { /* last resort */ }
```

Avoid the lazy `catch (Exception)` that swallows everything and does nothing:

```csharp
catch (Exception) { }  // ❌ silent failure, you'll never find the bug
```

If you can't actually handle it, don't catch it. Let it bubble up.

---

## Throwing

```csharp
public void Withdraw(decimal amount)
{
    if (amount <= 0)
        throw new ArgumentException("Amount must be positive.", nameof(amount));

    if (amount > _balance)
        throw new InvalidOperationException("Insufficient funds.");
}
```

Pick the exception type that fits:

| Type | When |
| ---- | ---- |
| `ArgumentException` | bad argument value |
| `ArgumentNullException` | argument was null |
| `InvalidOperationException` | the object is in a state where this call makes no sense |
| `NotImplementedException` | placeholder, not done yet |
| custom exception | a domain specific error you want callers to catch by type |

---

## rethrow: `throw;` vs `throw ex;`

This trips people up and it actually matters for debugging.

```csharp
catch (Exception ex)
{
    Log(ex);
    throw;      // ✅ keeps the original stack trace
}
```

```csharp
catch (Exception ex)
{
    Log(ex);
    throw ex;   // ❌ resets the stack trace to THIS line, you lose where it really came from
}
```

Always use bare `throw;` when rethrowing.

---

## Custom exceptions

When you want callers to catch a specific business error:

```csharp
public class InsufficientFundsException : Exception
{
    public InsufficientFundsException(string message) : base(message) { }
}
```

But honestly, for expected business outcomes (not found, already exists, invalid input) you often don't want exceptions at all. A Result type is cleaner and cheaper. See the result-pattern note in the design-patterns folder.

---

## exception filters (`when`)

You can add a condition to a catch:

```csharp
try { /* ... */ }
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // only catches 404s, other HttpRequestExceptions keep bubbling
}
```

Cleaner than catching, checking with an `if`, and rethrowing.

---

## Rules of thumb

* Don't use exceptions for normal control flow (they're expensive and noisy).
* Catch only what you can actually handle.
* Never swallow silently. At minimum log it.
* Rethrow with `throw;` not `throw ex;`.
* `finally` (or a `using` block) for cleanup, so it runs even on failure.

```csharp
// `using` is the clean way to guarantee disposal, no finally needed
using var file = new StreamReader("data.txt");
var text = file.ReadToEnd();
// file.Dispose() runs automatically at the end of scope, even if ReadToEnd throws
```
