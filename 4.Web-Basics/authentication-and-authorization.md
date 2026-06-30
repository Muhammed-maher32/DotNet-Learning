# Authentication & Authorization

> Authentication = who are you. Authorization = what are you allowed to do. Two different steps, in that order. You authenticate first, then authorize.

People say "auth" for both and it causes endless confusion. Keep them separate in your head.

---

## The two steps

```
Authentication  ->  "prove who you are"  (login, token, cookie)
Authorization   ->  "are you allowed to do THIS?"  (roles, policies)
```

A user can be authenticated (logged in) but not authorized (not an admin). Both have to pass.

---

## Claims, the common currency

After you authenticate, the user is represented as a set of **claims**. A claim is just a key/value fact about them.

```
sub: 42
name: "mohamed"
email: "m@example.com"
role: "Admin"
```

These claims live in a `ClaimsPrincipal` (`User` in your controller). Authorization decisions read from these claims.

---

## Two common approaches

### 1. Cookie based (classic MVC apps)

After login, the server sends a cookie. The browser sends it back on every request automatically. Server reads it, knows who you are. Stateful-ish, great for server rendered apps. ASP.NET Core Identity uses this.

### 2. Token based / JWT (APIs and SPAs)

After login the server hands back a **JWT** (a signed token). The client stores it and sends it on each request in a header:

```
Authorization: Bearer eyJhbGciOi...
```

The server verifies the signature and reads the claims out of the token. Stateless, no server side session needed. This is what you use for Web APIs and mobile/React frontends.

A JWT has 3 parts, dot separated: `header.payload.signature`. The payload holds the claims (it's just base64, not encrypted, so never put secrets in it). The signature is what proves it wasn't tampered with.

---

## ASP.NET Core Identity (the batteries-included option)

Identity is a full membership system: user store, password hashing, roles, login/logout, email confirmation, lockout, all of it. You plug it in instead of hand rolling user management.

```csharp
builder.Services
    .AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<AppDbContext>();
```

`ApplicationUser` extends `IdentityUser` and adds your own fields:

```csharp
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

You mostly work through two services it gives you:

```csharp
UserManager<ApplicationUser>    // create users, find them, manage roles/claims
SignInManager<ApplicationUser>  // sign in / out, check passwords
```

---

## Wiring the pipeline (order matters)

In `Program.cs`, authentication middleware must come **before** authorization:

```csharp
app.UseAuthentication();  // figure out who you are (read cookie/token)
app.UseAuthorization();   // decide if you're allowed
```

Flip them and auth silently breaks, because authorization runs before the user is even identified.

---

## Protecting endpoints

```csharp
[Authorize]                       // must be logged in
public IActionResult Dashboard() { ... }

[Authorize(Roles = "Admin")]      // must be logged in AND have the Admin role
public IActionResult AdminPanel() { ... }

[AllowAnonymous]                  // opt out, even on an [Authorize] controller
public IActionResult Login() { ... }
```

---

## Roles vs Policies

Roles are simple buckets ("Admin", "Manager"). Policies are more flexible rules you define once and reuse:

```csharp
// define
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Over18", policy =>
        policy.RequireClaim("Age", "18", "19", "20")); // simplified example
});

// use
[Authorize(Policy = "Over18")]
public IActionResult BuyTicket() { ... }
```

Use roles for coarse "what kind of user". Use policies when the rule is more than just a role name (claims, multiple conditions, custom logic).

---

## The login flow, end to end (JWT version)

1. User POSTs username + password.
2. Server checks credentials (via `SignInManager` / `UserManager`).
3. If valid, server builds claims and signs a JWT.
4. Server returns the token.
5. Client stores it, sends `Authorization: Bearer <token>` on later requests.
6. `UseAuthentication` validates the token and fills `User` with claims.
7. `[Authorize]` / `UseAuthorization` checks if the user passes.

---

## Summary

| Term | Meaning |
| ---- | ------- |
| Authentication | who you are (login, cookie, JWT) |
| Authorization | what you can do (roles, policies) |
| Claim | a key/value fact about the user |
| JWT | signed stateless token, `header.payload.signature`, for APIs/SPAs |
| Cookie auth | server cookie, for classic MVC apps |
| Identity | full built-in user management (users, passwords, roles) |
| `UseAuthentication` before `UseAuthorization` | order is mandatory |
