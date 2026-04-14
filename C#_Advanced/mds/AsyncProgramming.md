
Got it. Styled only — no content changes, just formatting for GitHub README.

---

# TASKInDotNet

> Task represents an operation that will complete in the future (not the result itself).

---

```csharp
Task<string> task = httpClient.GetStringAsync(url);
//I don’t have the string yet
//I have a Task<string> (a promise of a future result)


//Opened Box (I now have the value)
string result = await httpClient.GetStringAsync(url);
//Now I have the actual string
//`await` unwraps the Task and gives me the result
```

---

## What i understand?

* Task is a generic type that represents a asynchronous operation, when you create an object.
* It tells you "What is running now?" If succeed it return a result if failed return an exception.

```csharp
Task<int> task = GetNumberAsync();
int x = task //It's not valid,Why? it means give me the result without waiting the operation to be compeleted!

int x = await task;
```

---

## Async and Await

### Async

* Async is keyword enables you to write Await, converts the method to state machine(maybe paused or resumed in any time).
* Doesn't mean parallel or fast or create thread.

### Async_Retrun_Types

```
- Task
- Task<T>
- ValueTask
- ValueTask<T>
```

---

### Await

* Await pauses method, without blocking the thread(FREE the thread).

```csharp
public async task Mymethod()
{
Console.WriteLine("Befroe AWAIT");
var data = await httpClient.GetStringAsync(url);
Console.WriteLine("After AWAIT");
}
//The method pauses at the wait, exits the current call stack
//Thread is realsed,
//Async task is working in bg,
//When is compeletes, the method resumes from thhis point,
//possibly the same thread or not
```

---

## WhenAll - WhenAny

### WhenAll

* Parallel execution.
* If tasks are independent use it.

```csharp
var UserTask = GetUserAsync();
var OrderTask = GetOrderAsync();

array[] results = await Task.WhenAll(UserTask,OrderTask);//It returns an array of results

//results
var user = await UserTask;
var orded = await OrderTask;
```

* In more than one task throwed exceptions, WhenAll catches the first one.
* You can loop to get every single exception using `Exception.InnerExceptions`.

---

### WhenAny

* It returns the first response.

```csharp
var winner = await Task.WhenAny(Task01,Task02);

//code..
```

---

## CancellationToken

* Stop and cleanup.
