# Swagger & OpenAPI

> OpenAPI is a standard JSON description of your API (endpoints, parameters, schemas, responses). Swagger UI is the interactive page generated from it. You get living documentation and a test client for free, and consumers can even generate typed clients from the spec.

---

## Names, because they confuse everyone

* **OpenAPI**: the specification format itself (`swagger.json` / `openapi.json` describing your API).
* **Swagger UI**: the web page that renders the spec as browsable, clickable docs.
* **Swashbuckle**: the classic NuGet package generating both in ASP.NET Core. (.NET 9 added built-in `Microsoft.AspNetCore.OpenApi` for spec generation, typically paired with a UI like Scalar since it ships no UI. Concepts identical.)

---

## Setup with Swashbuckle

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(o =>
{
    o.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Gym API",
        Version = "v1",
        Description = "Members, plans and subscriptions."
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();       // serves /swagger/v1/swagger.json
    app.UseSwaggerUI();     // serves the UI at /swagger
}
```

Run the app, open `/swagger`, and every endpoint is there with a "Try it out" button. During API development this replaces half your Postman usage.

---

## Making the docs actually good

Out of the box you get endpoints and schemas. The gap between "generated docs" and "good docs" is metadata.

**Declare what each endpoint returns.** The generator can't guess your status codes from `IActionResult`:

```csharp
[HttpPost]
[ProducesResponseType(typeof(MemberDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create(CreateMemberDto dto, CancellationToken ct) { ... }
```

Now the docs show exactly what a 201 body and a 409 body look like. Consumers stop guessing (and stop messaging you).

**XML comments** flow into the UI if you turn them on:

```xml
<!-- csproj -->
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<NoWarn>$(NoWarn);1591</NoWarn>
```

```csharp
o.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory,
    $"{Assembly.GetExecutingAssembly().GetName().Name}.xml"));
```

```csharp
/// <summary>Creates a new gym member.</summary>
/// <param name="dto">Member details. Email must be unique.</param>
[HttpPost]
public async Task<IActionResult> Create(CreateMemberDto dto, CancellationToken ct) { ... }
```

---

## JWT auth in Swagger UI

The first wall everyone hits: endpoints return 401 from "Try it out" because there's no token. Teach the UI about your auth:

```csharp
builder.Services.AddSwaggerGen(o =>
{
    o.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Paste the JWT only, no 'Bearer ' prefix."
    });

    o.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

An "Authorize" button appears in the UI; log in via your login endpoint, paste the token once, and every request carries it.

---

## The spec is a contract, not just docs

The `openapi.json` file is machine-readable, which unlocks:

* **Client generation.** Frontend runs NSwag/openapi-generator/Kiota against your spec and gets a typed TypeScript or C# client. Rename a field and their build breaks instead of their runtime.
* **Diff checks in CI** to catch accidental breaking changes between versions.
* **Importing into Postman** or other tools directly from the spec URL.

This changes how you think about it: sloppy DTOs and vague response types don't just make ugly docs, they generate bad clients.

---

## Common Mistakes

* **Swagger UI enabled in production unintentionally.** Docs of the whole attack surface, public. Keep it behind `IsDevelopment()` or auth, deliberately.
* **No ProducesResponseType**, so every endpoint documents only a 200 and consumers discover the 409 the hard way.
* **Action method name collisions or ambiguous routes** breaking spec generation with cryptic errors. Usually a missing `[HttpGet]`/route attribute on some action.
* **XML comments configured in csproj but never fed to SwaggerGen** (or the other way around). Needs both halves.
* **Treating the generated docs as done.** Generated structure + zero human descriptions is still bad documentation.

---

## Summary

| Idea | Point |
| ---- | ----- |
| OpenAPI | machine-readable spec of the whole API |
| Swagger UI | browsable docs + built-in test client |
| ProducesResponseType | documents real status codes and body shapes |
| Security definition | makes "Try it out" work with JWT |
| Client generation | the spec is a contract; typed clients come free |

> Pairs with [rest-and-api-design.md](./rest-and-api-design.md); the ProblemDetails bodies referenced above are explained in [error-handling-and-problem-details.md](./error-handling-and-problem-details.md).
