# Background Jobs

> Work that shouldn't happen inside an HTTP request: sending emails, nightly cleanups, processing queues, recurring reports. The built-in tool is BackgroundService; Hangfire adds persistence, retries, and a dashboard when jobs must survive restarts.

---

## Why not just do it in the request?

```csharp
public async Task<Result> RegisterAsync(RegisterDto dto, CancellationToken ct)
{
    // ... create the member ...
    await _emailSender.SendWelcomeAsync(member.Email, ct);   // 3 seconds of SMTP
    return Result.Ok();
}
```

The user stares at a spinner for 3 seconds waiting for an email *they can't see anyway*, and if SMTP hiccups, registration fails even though the member was created fine. Anything slow, retryable, or non-essential to the response belongs outside the request.

(The tempting shortcut, `_ = Task.Run(...)` fire-and-forget inside the request, is a trap: unobserved exceptions vanish, the work dies silently on app shutdown/recycle, and scoped services like DbContext are disposed under it. Don't.)

---

## IHostedService / BackgroundService

The built-in mechanism: a singleton service the host starts with the app and stops gracefully with it.

```csharp
public class SubscriptionExpiryWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<SubscriptionExpiryWorker> _logger;

    public SubscriptionExpiryWorker(IServiceScopeFactory scopeFactory,
        ILogger<SubscriptionExpiryWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(6));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

                var expired = await context.Subscriptions
                    .Where(s => s.EndDate < DateTime.UtcNow && s.Status == SubscriptionStatus.Active)
                    .ExecuteUpdateAsync(s =>
                        s.SetProperty(x => x.Status, SubscriptionStatus.Expired), stoppingToken);

                _logger.LogInformation("Marked {Count} subscriptions expired", expired);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Subscription expiry sweep failed");
                // swallow and let the next tick retry; a throw here kills the worker for good
            }
        }
    }
}
```

```csharp
builder.Services.AddHostedService<SubscriptionExpiryWorker>();
```

Three details in that snippet are the ones that bite everyone:

1. **The scope dance.** The worker is a singleton, DbContext is scoped, so injecting it directly throws at startup. Create a scope per iteration via `IServiceScopeFactory`.
2. **The try/catch inside the loop.** An escaped exception ends `ExecuteAsync` permanently, and (by default since .NET 6) stops the whole host. Catch, log, continue.
3. **Respect the stoppingToken** so deploys shut down cleanly instead of hanging.

For "email after registration" style work, the standard in-process pattern is a `System.Threading.Channels` queue: the request writes a message to the channel and returns instantly; a BackgroundService reads the channel and does the sending.

---

## The limits, and where Hangfire comes in

BackgroundService jobs live in the process's memory. App restarts mid-job? Work lost, no retry, no history. Deploy at the wrong moment and the nightly report just doesn't happen. Fine for tolerable-loss work, not fine for "must eventually happen".

**Hangfire** persists jobs to your database, so they survive restarts, retry automatically with backoff, and show up in a dashboard.

```csharp
builder.Services.AddHangfire(cfg => cfg
    .UseSqlServerStorage(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddHangfireServer();

app.UseHangfireDashboard("/hangfire");   // put auth on this in production
```

The job types cover most needs:

```csharp
// fire-and-forget, but PERSISTED: retried until it succeeds
_backgroundJobs.Enqueue<IEmailSender>(s => s.SendWelcomeAsync(member.Email));

// delayed
_backgroundJobs.Schedule<IEmailSender>(
    s => s.SendRenewalReminderAsync(member.Id), TimeSpan.FromDays(25));

// recurring, cron syntax
RecurringJob.AddOrUpdate<IReportService>(
    "monthly-revenue", s => s.GenerateMonthlyAsync(), Cron.Monthly);
```

Notice you enqueue an *expression* (`s => s.Send(...)`), not a running task. Hangfire serializes the call (type + method + arguments) into the database and a worker executes it later, possibly on another server, with DI resolving the service fresh. That's why arguments must be simple/serializable: pass the member *Id*, never the EF entity.

Alternatives in the same space: **Quartz.NET** (heavier, strong cron/calendars, no dashboard by default) and cloud-native options (Azure Functions timers, queues). And when jobs must scale beyond one app or coordinate across services, you're drifting toward real message queues (RabbitMQ, Azure Service Bus), which is a bigger topic than this note.

**Choosing:**

| Need | Tool |
| ---- | ---- |
| periodic in-process sweep, loss tolerable | BackgroundService + PeriodicTimer |
| offload work from requests, in-process | Channel + BackgroundService |
| must survive restarts, retries, visibility | Hangfire |
| complex schedules/calendars | Quartz.NET |
| cross-service messaging at scale | a proper queue |

---

## Common Mistakes

* **Injecting scoped services into a BackgroundService constructor.** Startup crash. Scope per iteration.
* **Letting one exception kill the loop** (and the host). The catch-inside-while is not optional.
* **`Task.Run` fire-and-forget from requests.** Silent data loss with extra steps.
* **Passing entities or fat objects to Hangfire jobs.** Serialized snapshots go stale; pass ids and reload inside the job.
* **Non-idempotent jobs + automatic retries.** Hangfire *will* run your job again after a failure; "send invoice email" must cope with having half-run. Check state before acting.
* **Unsecured /hangfire dashboard in production.**

---

## Summary

| Idea | Point |
| ---- | ----- |
| BackgroundService | in-process worker, host-managed lifetime |
| Scope per iteration | the singleton/scoped bridge for DbContext |
| Channels | request enqueues, worker processes, response returns fast |
| Hangfire | DB-persisted jobs: retries, delays, cron, dashboard |
| Idempotency | retried jobs must be safe to run twice |

> The ExecuteUpdate call is from [../2.EF-Core/performance-and-advanced-queries.md](../2.EF-Core/performance-and-advanced-queries.md); lifetime rules are in [../4.Web-Basics/dependency-injection.md](../4.Web-Basics/dependency-injection.md). For visibility into workers you only have logs: [logging-and-serilog.md](./logging-and-serilog.md).
