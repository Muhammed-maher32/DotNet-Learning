# ASP.NET Core Fundamentals

> Practical notes on how an ASP.NET Core app is structured and how it handles a request: configuration, middleware, routing, controllers, model binding, validation, and results.

---

### 1. ASP.NET Project Structure

When you create a new ASP.NET Core project, here's what you actually care about:

```
MyProject/
├── Program.cs           ← entry point, where everything is wired up
├── appsettings.json     ← config (connection strings, app settings, etc.)
├── Controllers/         ← your controllers live here
├── Models/              ← your data models
├── Views/               ← Razor views (MVC only)
├── wwwroot/             ← static files (css, js, images)
└── Properties/
    └── launchSettings.json  ← local dev settings (ports, env vars)
```

`Program.cs` is where it all starts. You register services into DI, configure the middleware pipeline, and run the app. Everything else is just folders you put code in.

---
### 2. Configuration

ASP.NET Core has a layered config system. Sources are read in this order (later ones override earlier):
1. `appsettings.json`
2. `appsettings.{Environment}.json` (e.g. `appsettings.Development.json`)
3. Environment variables
4. Command-line args

```csharp
// Reading config in a controller or service
public class MyService
{
    private readonly IConfiguration _config;

    public MyService(IConfiguration config)
    {
        _config = config;
    }

    public void DoSomething()
    {
        var connStr = _config["ConnectionStrings:Default"];
    }
}
```

Better pattern : bind to a typed class using `IOptions<T>`:

```json
// appsettings.json
{
  "JwtSettings": {
    "Secret": "supersecret",
    "ExpiresInMinutes": 60
  }
}
```

```csharp
// Register
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));

// Use
public class AuthService(IOptions<JwtSettings> opts)
{
    var secret = opts.Value.Secret;
}
```

---
### 3. Middlewares

Your request doesn't hit the controller directly. It passes through a **pipeline** of middlewares, each one can do something before passing it to the next, or short-circuit the whole thing.

```
Request → MW1 → MW2 → MW3 → Endpoint
Response ← MW1 ← MW2 ← MW3 ←
```

You register them in `Program.cs` using `app.Use...()`:

```csharp
app.UseHttpsRedirection();   // redirect http → https
app.UseAuthentication();     // who are you?
app.UseAuthorization();      // are you allowed?
app.MapControllers();        // route to controllers
```

**Order matters.** Authentication must come before Authorization. If you flip them, auth won't work.

Custom middleware:
```csharp
app.Use(async (context, next) =>
{
    // before
    Console.WriteLine($"Request: {context.Request.Path}");
    
    await next(); // pass to next middleware
    
    // after
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});
```

---

### 4. Static Files

By default ASP.NET Core doesn't serve static files. You have to tell it to:

```csharp
app.UseStaticFiles();
```

This serves anything in `wwwroot/`. So if you have `wwwroot/css/app.css`, it's accessible at `/css/app.css`. That's it. Don't overthink it.

---
### 5. Routing

Routing is how ASP.NET Core figures out which endpoint handles a given request.

Two styles:
**Conventional routing** (MVC style):

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

**Attribute routing** (Web API style, what you'll use 99% of the time):

```csharp
[Route("api/[controller]")]
[ApiController]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]          // GET /api/products/5
    public IActionResult Get(int id) { ... }

    [HttpPost]                 // POST /api/products
    public IActionResult Create([FromBody] Product p) { ... }
}
```

Route constraints exist too: `{id:int}`, `{name:minlength(3)}`, etc.

---
### 6. Controllers & Actions

A **Controller** is just a class that groups related endpoints. An **Action** is a method inside it that handles a specific request.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "order1", "order2" });

    [HttpGet("{id:int}")]
    public IActionResult GetById(int id) => Ok($"order {id}");

    [HttpPost]
    public IActionResult Create([FromBody] CreateOrderDto dto)
    {
        // create logic
        return CreatedAtAction(nameof(GetById), new { id = 1 }, dto);
    }
}
```

Actions receive input via parameters, process something, and return an `IActionResult`.

---
### 7. Controller vs ControllerBase

```csharp
// Web API
public class UsersController : ControllerBase { }

// MVC (returns Views)
public class HomeController : Controller { }
```

---
### 8. Model Binding

Model binding is how ASP.NET Core takes incoming request data and maps it to your action parameters automatically. You don't manually parse the request body or query string.

Sources it binds from (using explicit attributes):
```csharp
public IActionResult Example(
    [FromRoute]  int id,         // /api/items/5
    [FromQuery]  string search,  // ?search=phone
    [FromBody]   CreateDto dto,  // JSON body
    [FromHeader] string token,   // Authorization header
    [FromForm]   IFormFile file  // multipart form
)
```

When you use `[ApiController]`, `[FromBody]` is applied automatically to complex types, and `[FromRoute]`/`[FromQuery]` are inferred for simple types. So you often don't need to write them explicitly.

---
### 9. Validation & ModelState
You put data annotation attributes on your models:
```csharp
public class CreateProductDto
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }

    [Range(0.01, 10000)]
    public decimal Price { get; set; }

    [EmailAddress]
    public string ContactEmail { get; set; }
}
```
ASP.NET Core validates the incoming data and populates `ModelState`. Without `[ApiController]`:
```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateProductDto dto)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    // proceed
}
```

With `[ApiController]` you don't even need to check. It automatically returns `400 Bad Request` with validation errors if `ModelState` is invalid. The check is done for you.
For complex validation that doesn't fit in attributes, implement `IValidatableObject` on the model or use FluentValidation (much cleaner for real projects).

---
### 10. Action Results

Action methods return `IActionResult` (or the generic `ActionResult<T>`). There are helpers on `ControllerBase` for every HTTP status:

```csharp
return Ok(data);                          // 200
return Created("/api/items/1", data);     // 201
return CreatedAtAction(nameof(Get), ...); // 201 with Location header
return NoContent();                       // 204
return BadRequest(ModelState);            // 400
return Unauthorized();                    // 401
return Forbid();                          // 403
return NotFound();                        // 404
return StatusCode(500, "something broke");// custom
```

Prefer `ActionResult<T>` over `IActionResult` when you can, it gives you type safety and Swagger picks up the response type automatically:

```csharp
[HttpGet("{id}")]
public ActionResult<ProductDto> GetById(int id)
{
    var product = _service.GetById(id);
    if (product is null) return NotFound();
    return Ok(product);
}
```

---
### 11. MVC vs Web API
Both use controllers and routing. The difference is what they return.