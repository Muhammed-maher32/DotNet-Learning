# The Options Pattern

> Instead of injecting `IConfiguration` everywhere and pulling values out by magic strings, you bind a section of appsettings.json to a typed class once, and inject that class. Typos become compile errors, and settings become testable.

---

## The problem

```csharp
public class EmailService
{
    private readonly IConfiguration _config;

    public EmailService(IConfiguration config) => _config = config;

    public void Send(string to)
    {
        var host = _config["Email:SmtpHost"];          // magic string
        var port = int.Parse(_config["Email:Port"]!);  // manual parsing
        // ...
    }
}
```

Problems: strings everywhere (`"Email:SmptHost"` compiles fine and returns null), parsing by hand, no single place that says "these are the email settings", and testing needs a fake IConfiguration.

---

## Bind a class instead

appsettings.json:

```json
{
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "Port": 587,
    "FromAddress": "noreply@gym.com"
  }
}
```

The settings class (plain class, must have a parameterless ctor and settable props):

```csharp
public class EmailSettings
{
    public const string SectionName = "Email";

    public string SmtpHost { get; set; } = null!;
    public int Port { get; set; }
    public string FromAddress { get; set; } = null!;
}
```

Registration in Program.cs:

```csharp
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection(EmailSettings.SectionName));
```

Consumption:

```csharp
public class EmailService
{
    private readonly EmailSettings _settings;

    public EmailService(IOptions<EmailSettings> options)
        => _settings = options.Value;

    public void Send(string to)
    {
        var client = new SmtpClient(_settings.SmtpHost, _settings.Port);
        // ...
    }
}
```

Typed, parsed, autocompleted. Testing is trivial: `Options.Create(new EmailSettings { ... })`.

---

## IOptions vs IOptionsSnapshot vs IOptionsMonitor

Three interfaces, one question: what happens when appsettings.json changes while the app is running?

| Interface | Lifetime | Reads config | Use when |
| --------- | -------- | ------------ | -------- |
| `IOptions<T>` | Singleton | once at first use, frozen forever | settings never change at runtime (99% of cases) |
| `IOptionsSnapshot<T>` | Scoped | re-evaluated once per request | you edit config on a live server and want the next request to see it |
| `IOptionsMonitor<T>` | Singleton | always current + change events | singletons/background services that need live values |

```csharp
// snapshot: fresh per request
public MyService(IOptionsSnapshot<EmailSettings> options)
    => _settings = options.Value;

// monitor: current value any time, plus notifications
public MyWorker(IOptionsMonitor<EmailSettings> monitor)
{
    _monitor = monitor;
    monitor.OnChange(s => Console.WriteLine("Email settings changed"));
}
// use _monitor.CurrentValue when you need it
```

Note the trap: injecting `IOptionsSnapshot` into a singleton throws (scoped into singleton). For singletons that need reload, it's `IOptionsMonitor` or nothing.

My honest default: `IOptions<T>` everywhere until a real need for hot reload shows up.

---

## Validation

Settings arrive from a text file, so they can be garbage. Validate at startup, not at first email:

```csharp
public class EmailSettings
{
    [Required]
    public string SmtpHost { get; set; } = null!;

    [Range(1, 65535)]
    public int Port { get; set; }

    [Required, EmailAddress]
    public string FromAddress { get; set; } = null!;
}
```

```csharp
builder.Services.AddOptions<EmailSettings>()
    .BindConfiguration(EmailSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();     // app refuses to boot with bad config
```

`ValidateOnStart` is the important line. Without it, validation fires lazily on first use, which means the error surfaces at 2 AM when the first email of the day goes out, instead of at deploy time.

For rules annotations can't express: `.Validate(s => s.Port != 25 || s.AllowInsecure, "message")`.

---

## Common Mistakes

* **Injecting IConfiguration into services.** That's the thing this pattern removes. IConfiguration belongs in Program.cs, nowhere else.
* **Section name typo.** Binding a wrong name doesn't throw, you just silently get default values. `ValidateOnStart` + `[Required]` catches this.
* **IOptionsSnapshot in a singleton.** Runtime DI error.
* **Forgetting secrets don't go in appsettings.json.** Connection strings with passwords, API keys: user-secrets in dev, environment variables or a vault in prod. The options class doesn't care where the value came from, that's the nice part.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Configure\<T\> | bind a config section to a typed class |
| IOptions | frozen at startup, the default choice |
| IOptionsSnapshot | per-request reload, scoped only |
| IOptionsMonitor | live values + change events, works in singletons |
| ValidateOnStart | fail at boot, not at first use |

> Related: [../4.Web-Basics/dependency-injection.md](../4.Web-Basics/dependency-injection.md) for the lifetimes that explain the snapshot/monitor split.
