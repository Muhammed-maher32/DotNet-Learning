# Async Programming in .NET

> A `Task` represents an operation that will complete in the future (not the result itself).

---

```csharp
Task<string> task = httpClient.GetStringAsync(url);
// I don't have the string yet —
// I have a Task<string> (a promise of a future result)


// "Open the box" (I now have the value)
string result = await httpClient.GetStringAsync(url);
// Now I have the actual string.
// `await` unwraps the Task and gives me the result.
```

---

## The Core Idea

* `Task` / `Task<T>` is a type that represents an **asynchronous operation** — work that is in progress.
* It answers "what is running now?" When it finishes it either returns a **result** or throws an **exception**.

```csharp
Task<int> task = GetNumberAsync();

int x = task;        // ❌ invalid — a Task<int> is not an int.
                     // This means "give me the result without waiting for it to complete!"

int x = await task;  // ✅ wait for completion, then unwrap the int.
```

---

## Async and Await

### `async`

* The `async` keyword **enables** you to use `await` inside a method, and turns the method into a **state machine** (it can be paused and resumed).
* It does **not** mean parallel, faster, or "creates a thread."

### Async return types

```
- Task          → async method with no result
- Task<T>       → async method returning a value
- ValueTask     → like Task, but avoids allocation in hot paths
- ValueTask<T>
- void          → only for event handlers (avoid otherwise)
```

---

### `await`

* `await` **pauses** the method **without blocking the thread** (it frees the thread to do other work).

```csharp
public async Task MyMethod()
{
    Console.WriteLine("Before AWAIT");
    var data = await httpClient.GetStringAsync(url);
    Console.WriteLine("After AWAIT");
}
// The method pauses at the await and returns control to the caller.
// The thread is released.
// The async operation continues in the background.
// When it completes, the method resumes from this point —
// possibly on the same thread, possibly not.
```

---

## WhenAll & WhenAny

### `Task.WhenAll`

* Runs tasks **concurrently** and waits for **all** of them.
* Use it when the tasks are **independent**.

```csharp
var userTask  = GetUserAsync();
var orderTask = GetOrderAsync();

// Wait for both. For Task<T>, WhenAll returns an array of results.
var results = await Task.WhenAll(userTask, orderTask);

// Or read each result individually (the tasks are already complete):
var user  = await userTask;
var order = await orderTask;
```

* If more than one task throws, `await Task.WhenAll(...)` rethrows only the **first** exception.
* To inspect every exception, read the task's `Exception.InnerExceptions`.

### `Task.WhenAny`

* Returns the **first task to finish** (useful for timeouts or "fastest wins").

```csharp
var winner = await Task.WhenAny(task01, task02);
// `winner` is the Task that completed first.
```

---

## CancellationToken

* Used to **cooperatively cancel** an async operation and clean up.
* Pass the token down through the call chain.

```csharp
cancellationToken.ThrowIfCancellationRequested();
```

---

# Additional Backend Notes

## Task Lifecycle

A task moves through several states:

* Created / Waiting
* Running
* RanToCompletion (completed successfully)
* Faulted (an exception occurred)
* Canceled

---

## Exception Handling in Async

* Exceptions are **stored inside the Task**.
* `await` unwraps and rethrows the exception.
* `.Result` and `.Wait()` wrap exceptions in an `AggregateException` (and can deadlock — avoid them).
* With `Task.WhenAll`, multiple exceptions can be aggregated.

---

## Task vs Thread

* **Thread** = an execution resource (the actual worker).
* **Task** = a representation of work (the operation).

> A Task is **not** a Thread.

---

## I/O-bound vs CPU-bound

* **I/O-bound** (HTTP, database, file):
  * Use `async`/`await`.
  * Does not block a thread while waiting.

* **CPU-bound** (heavy calculations):
  * Needs a thread to do the work.
  * Offload with `Task.Run`.

```csharp
await Task.Run(() => HeavyCalculation());
```

---

## Async Does NOT Mean Multithreading

* Async = **non-blocking**.
* Not necessarily parallel.
* Not necessarily multiple threads.

---

## Async All the Way

* Once a call is async, keep it async through the whole chain.
* Avoid blocking on async code:

```csharp
.Result   // ❌ can deadlock
.Wait()   // ❌ can deadlock
```

---

## Fire-and-Forget Warning

```csharp
DoSomethingAsync(); // ⚠️ no await
```

Dangerous because:

* Exceptions are lost (unobserved).
* Execution is untracked.
* The work may not finish before the app moves on.

---

## ConfigureAwait(false)

* Avoids capturing the synchronization context when resuming.
* Useful in **library** code.
* Less important in ASP.NET Core (no special context to capture).

---

## ValueTask Note

* Reduces allocations for methods that often complete **synchronously**.
* Not always better than `Task` — it has usage rules (don't await it twice).
* Use only in performance-critical paths.
