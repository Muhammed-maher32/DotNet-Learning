# Factory & Strategy

> Two patterns that kill the same enemy: the giant switch statement that grows a new case every sprint. Strategy says "put each behavior behind one interface". Factory says "put the choosing of which one in a single place".

---

## The smell

```csharp
public decimal CalculateFee(Member member)
{
    switch (member.PlanType)
    {
        case PlanType.Monthly:
            return 500;
        case PlanType.Yearly:
            return 500 * 12 * 0.8m;
        case PlanType.Student:
            return member.HasValidId ? 300 : 500;
        // new plan? edit this method. And the other switch in the report service.
        // And the one in the invoice generator...
    }
}
```

The logic for each plan type gets smeared across every switch in the codebase. Adding a plan means hunting them all down.

---

## Strategy: one interface, many behaviors

```csharp
public interface IFeeCalculator
{
    PlanType Handles { get; }
    decimal Calculate(Member member);
}

public class MonthlyFeeCalculator : IFeeCalculator
{
    public PlanType Handles => PlanType.Monthly;
    public decimal Calculate(Member member) => 500;
}

public class YearlyFeeCalculator : IFeeCalculator
{
    public PlanType Handles => PlanType.Yearly;
    public decimal Calculate(Member member) => 500 * 12 * 0.8m;
}

public class StudentFeeCalculator : IFeeCalculator
{
    public PlanType Handles => PlanType.Student;
    public decimal Calculate(Member member)
        => member.HasValidId ? 300 : 500;
}
```

Each plan's rules live in exactly one class. A new plan = a new class + registration, zero edits to existing code (the open/closed principle, in practice).

---

## Factory: one place that picks

Somebody still has to choose the right strategy. That decision goes in a factory, and with DI the clean trick is injecting *all* implementations:

```csharp
builder.Services.AddScoped<IFeeCalculator, MonthlyFeeCalculator>();
builder.Services.AddScoped<IFeeCalculator, YearlyFeeCalculator>();
builder.Services.AddScoped<IFeeCalculator, StudentFeeCalculator>();
builder.Services.AddScoped<FeeCalculatorFactory>();
```

```csharp
public class FeeCalculatorFactory
{
    private readonly Dictionary<PlanType, IFeeCalculator> _calculators;

    // DI hands you ALL registered IFeeCalculator implementations
    public FeeCalculatorFactory(IEnumerable<IFeeCalculator> calculators)
        => _calculators = calculators.ToDictionary(c => c.Handles);

    public IFeeCalculator GetFor(PlanType planType)
        => _calculators.TryGetValue(planType, out var calc)
            ? calc
            : throw new NotSupportedException($"No calculator for {planType}");
}
```

Consumer code never switches again:

```csharp
public class SubscriptionService
{
    private readonly FeeCalculatorFactory _factory;

    public decimal GetFee(Member member)
        => _factory.GetFor(member.PlanType).Calculate(member);
}
```

The `IEnumerable<IFeeCalculator>` injection is the part worth remembering: register multiple implementations of one interface and the container gives you all of them. The factory is then just a lookup, no `new`, no service locator.

Since .NET 8 there's also built-in keyed services, which is the same idea with less code for simple cases:

```csharp
builder.Services.AddKeyedScoped<IFeeCalculator, MonthlyFeeCalculator>(PlanType.Monthly);

public SubscriptionService([FromKeyedServices(PlanType.Monthly)] IFeeCalculator calc) { ... }
```

Useful when the key is known at injection time; the factory wins when it's decided at runtime per call.

---

## Where I'd actually use this

Real cases from typical backend work:

* **Payment providers.** `IPaymentGateway` with Stripe/PayPal/Fawry implementations, factory picks by the method the user chose.
* **Notifications.** `INotificationSender` for email/SMS/WhatsApp, pick per user preference.
* **File exports.** `IReportExporter` for PDF/Excel/CSV, pick by requested format.
* **Discount rules,** shipping cost calculations, anything where the business says "it depends on the type".

The tell is always the same: a switch on some "type" enum that keeps growing, duplicated in more than one place.

---

## When NOT to use it

* **Two cases that will stay two cases.** A switch with two arms is clearer than three classes, an interface, and a factory. Patterns have a ceremony cost.
* **The branches share 90% of their code.** Then the variation is a parameter, not a strategy.
* **You're guessing about future flexibility.** Wait for the third case to actually show up, then refactor. Takes ten minutes when the pattern is familiar.

---

## Common Mistakes

* **Factory with a switch inside that news up concrete classes.** You moved the switch, didn't remove it, and you lost DI (no constructor injection into the strategies). The dictionary-from-`IEnumerable` approach avoids both.
* **Strategies holding state between calls.** Keep them stateless so scoped/singleton lifetimes don't surprise you.
* **Interface named after the pattern, not the job.** `IFeeCalculator` beats `IFeeStrategy`. Callers care what it does.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Strategy | each variant behind one interface, one class per behavior |
| Factory | the choosing logic in exactly one place |
| DI trick | inject `IEnumerable<IFeeCalculator>`, build a dictionary |
| Keyed services | built-in shortcut since .NET 8 |
| The smell | the same growing switch duplicated across services |

> Related: [decorator-and-caching.md](./decorator-and-caching.md) (another composition-over-conditionals pattern) and [../4.Web-Basics/dependency-injection.md](../4.Web-Basics/dependency-injection.md).
