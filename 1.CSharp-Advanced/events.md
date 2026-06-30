# Events in C#

> An event is just a delegate with restrictions slapped on it so outside code can subscribe/unsubscribe but can't invoke it or wipe the list.

If you already read the delegates note, events are the next step. A delegate is "a variable that holds methods". An event is "a delegate I expose safely to other classes".

---

## Why not just use a public delegate?

You technically can, but a public delegate field is dangerous:

```csharp
public Action OnSomething; // public delegate field

// any other class can do this:
obj.OnSomething = null;        // wipes everyone else's subscriptions
obj.OnSomething();             // raises it from outside, which is not their job
```

The `event` keyword blocks both of those. Subscribers can only do `+=` and `-=`. Only the declaring class can raise it.

```csharp
public event Action OnSomething; // now it's protected

obj.OnSomething += Handler;  // allowed
obj.OnSomething -= Handler;  // allowed
obj.OnSomething = null;      // compile error
obj.OnSomething();           // compile error (outside the class)
```

---

## The classic pattern (EventHandler)

The convention in .NET is `EventHandler<TEventArgs>`. The handler signature is always `(object sender, TArgs e)`.

```csharp
public class Button
{
    // 1. declare the event
    public event EventHandler<EventArgs> Clicked;

    // 2. a method to raise it safely
    public void Press()
    {
        // null check matters: if nobody subscribed, the event is null
        Clicked?.Invoke(this, EventArgs.Empty);
    }
}
```

```csharp
var button = new Button();

// 3. subscribe
button.Clicked += (sender, e) => Console.WriteLine("clicked!");

button.Press(); // prints "clicked!"
```

The `?.Invoke` is important. If no one subscribed, the event is `null`, and calling it would throw a `NullReferenceException`.

---

## Custom event data

Make a class that carries the info you want to send:

```csharp
public class OrderPlacedEventArgs : EventArgs
{
    public int OrderId { get; }
    public decimal Total { get; }

    public OrderPlacedEventArgs(int orderId, decimal total)
    {
        OrderId = orderId;
        Total = total;
    }
}

public class OrderService
{
    public event EventHandler<OrderPlacedEventArgs> OrderPlaced;

    public void Place(int id, decimal total)
    {
        // ... save the order ...
        OrderPlaced?.Invoke(this, new OrderPlacedEventArgs(id, total));
    }
}
```

```csharp
service.OrderPlaced += (s, e) =>
    Console.WriteLine($"Order {e.OrderId} placed, total {e.Total}");
```

---

## Publisher / Subscriber

That's the whole mental model:

* **Publisher**: the class that owns and raises the event (`OrderService`).
* **Subscriber**: whoever attaches a handler with `+=`.

The publisher doesn't know or care who is listening. Could be zero handlers, could be ten. That's the decoupling you get.

---

## The memory leak trap

If a long lived object subscribes to an event on another object, it keeps that object alive. The garbage collector can't free it because the event still holds a reference to the handler.

```csharp
longLivedThing.Updated += this.OnUpdate;
// if you never unsubscribe, `this` can't be collected
```

Rule of thumb: if you `+=`, remember to `-=` when you're done (in `Dispose`, when a page closes, etc). For short lived objects it doesn't matter, but for things living the whole app lifetime it bites you.

---

## Quick summary

| Thing | Meaning |
| ----- | ------- |
| `event` | a delegate exposed safely (only `+=` / `-=` from outside) |
| Publisher | owns and raises the event |
| Subscriber | attaches a handler |
| `EventHandler<T>` | standard signature `(object sender, T e)` |
| `?.Invoke` | always null check before raising |
| leak | unsubscribe long lived handlers or they pin objects in memory |
