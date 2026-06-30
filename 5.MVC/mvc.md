# ASP.NET Core MVC

> MVC is a way to organize a web app into three roles: Model (the data), View (the HTML the user sees), Controller (the code that handles the request and ties the two together). The browser asks the controller, the controller gets the data, picks a view, and the view renders HTML back.

Most of the examples here are pulled from my GymSystem project (link at the bottom), so they're real code, not toy snippets.

---

## The flow in one picture

```
Browser request
   -> Controller action       (handle the request, run logic)
       -> Model / Service      (get or save data)
   <- View (.cshtml)           (render HTML with the data)
HTML response
```

The controller never builds HTML, the view never talks to the database. Each one stays in its lane.

---

## The full flow, step by step (a real request)

Forget abstractions for a second. Let's follow one real click from start to finish: a user fills the "Add Member" form in GymSystem and presses Save. Here is literally everything that happens, in order.

```
[1] User submits the form
        |
        v  HTTP POST  /Members/CreateMember   (form data in the body)
[2] Routing reads the URL
        "Members" -> MembersController,  "CreateMember" -> the action
        |
        v
[3] Model binding
        ASP.NET reads the form fields (Name, Email, City, ...)
        and fills a CreateMemberViewModel object automatically
        |
        v
[4] Validation
        the [Required], [StringLength] rules on that ViewModel run
        results go into ModelState
        |
        +--- invalid? --> return the SAME view with the model
        |                 (user sees red error messages, nothing saved)
        |
        v  valid
[5] The action calls a service (BLL)
        await _memberService.CreateMemberAsync(model)
        |
        v
[6] The service does the real work
        checks rules (is the email already used?),
        maps the ViewModel to a Member entity,
        tells the repository to save it
        |
        v
[7] The repository / EF Core (DAL)
        turns that into an INSERT and hits SQL Server
        |
        v  returns a Result (success or failure) back up
[8] Back in the controller
        success? -> put a message in TempData
                    -> RedirectToAction(Index)   (PRG, step 9)
        failure? -> add the error to ModelState, redisplay the form
        |
        v
[9] The redirect triggers a NEW request: GET /Members/Index
        controller asks the service for all members,
        passes them to the Index view
        |
        v
[10] The Index view (Razor) renders an HTML table of members,
     plus the green "Member Created" banner from TempData
        |
        v
[11] Browser shows the page. Done.
```

### The same thing in plain words

Think of it like a restaurant:

* **You (the browser)** place an order.
* **The waiter (the controller)** takes it. The waiter doesn't cook. He writes down what you want and passes it to the kitchen.
* **The kitchen (the service + database)** actually makes the food (checks the rules, saves the data).
* The waiter brings back either your dish (**a view / HTML page**) or a "sorry, we're out of that" message (**a validation error**).

The whole point of MVC is keeping these roles separate. The waiter never cooks, the kitchen never talks to the customer. If every part does only its own job, the app stays easy to change.

The rest of this note zooms into each step above.

---

## 1. Routing: how a URL finds an action

In `Program.cs` the default convention maps a URL to `Controller/Action/id`:

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

So `/Members/MemberDetails/5` runs `MembersController.MemberDetails(5)`. No controller in the URL means `Home`, no action means `Index`. The `?` makes `id` optional.

---

## 2. Controllers & Actions

A controller is a class that groups related pages. An action is a method that handles one request and returns an `IActionResult` (usually a View or a redirect).

From GymSystem's `MembersController`:

```csharp
public class MembersController : Controller
{
    private readonly IMemberService _memberService;

    // the service is injected (see the DI note)
    public MembersController(IMemberService memberService)
    {
        _memberService = memberService;
    }

    public async Task<IActionResult> Index(CancellationToken ct)
    {
        var members = await _memberService.GetAllMembersAsync(ct);
        return View(members.Value);   // pass the data to the view
    }
}
```

Notice the controller is thin. It asks the service for data and hands it to a view. The real logic (rules, DB) lives in the BLL/DAL layers, not here.

---

## 3. The Model: ViewModels, not entities

In MVC the "Model" passed to a view is usually a **ViewModel**, a class shaped for that screen, not the raw database entity. GymSystem keeps these in the BLL layer (`CreateMemberViewModel`, `MemberDetailsViewModel`, etc.).

Why not the entity? Same reasons as DTOs: hide internal fields, avoid over-posting, and reshape data (for example flattening the address into one string for the details page). See the DTOs & AutoMapper note.

---

## 4. Views & Razor

A view is a `.cshtml` file: HTML with C# mixed in using `@`. The first line declares what model it expects.

```cshtml
@model IEnumerable<MemberViewModel>

<table class="table">
    @foreach (var member in Model)
    {
        <tr>
            <td>@member.Name</td>
            <td>@member.Email</td>
        </tr>
    }
</table>
```

`@model` sets the type, `@Model` is the actual object, `@foreach` / `@if` are normal C# driving the HTML.

### Shared view plumbing

* `_Layout.cshtml`: the master template (nav bar, footer, CSS/JS includes). Every page renders inside it via `@RenderBody()`.
* `_ViewStart.cshtml`: sets the default layout for all views, so you don't repeat it.
* `_ViewImports.cshtml`: shared `@using` namespaces and tag helpers, so views don't re-import them. GymSystem uses this to bring in the ViewModel namespaces.

---

## 5. Tag Helpers

Tag helpers are HTML-looking attributes that generate the right markup and links for you. Cleaner than hardcoding URLs.

```cshtml
<!-- builds the URL to MembersController.MemberDetails(id) -->
<a asp-controller="Members" asp-action="MemberDetails" asp-route-id="@member.Id">
    Details
</a>

<!-- binds an input to a model property (name, id, validation attrs) -->
<input asp-for="Email" class="form-control" />
<span asp-validation-for="Email" class="text-danger"></span>
```

`asp-for` is the important one: it wires the input to a property so model binding and validation work automatically.

---

## 6. Forms: GET to show, POST to submit

The standard pattern is two actions with the same name: a `GET` that shows the form, a `POST` that processes it. From GymSystem's create flow:

```csharp
// GET: show the empty form
public IActionResult Create() => View();

// POST: handle the submitted form
[HttpPost]
public async Task<IActionResult> CreateMember(CreateMemberViewModel model, CancellationToken ct)
{
    if (!ModelState.IsValid)
        return View(nameof(Create), model);   // redisplay with errors

    var result = await _memberService.CreateMemberAsync(model, ct);

    if (!result.Success)
    {
        ModelState.AddModelError(string.Empty, result.Error!);
        return View(nameof(Create), model);
    }

    TempData["SuccessMessage"] = "Member Created Successfully";
    return RedirectToAction(nameof(Index));     // PRG, see below
}
```

### Model binding

ASP.NET Core automatically fills `CreateMemberViewModel model` from the submitted form fields by matching names. You don't parse the request yourself.

### ModelState & validation

The data annotations on the ViewModel (`[Required]`, `[StringLength]`, etc.) are checked automatically and the result lands in `ModelState`. If invalid, you redisplay the form so the user sees the error messages (rendered by the `asp-validation-for` spans). See the validation note for the details.

---

## 7. Post-Redirect-Get (PRG)

Notice the successful POST ends with `RedirectToAction(nameof(Index))` instead of returning a view directly. That's the PRG pattern: after a successful form submit, redirect to a GET page.

Why: if you returned a view straight from the POST, hitting refresh would resubmit the form (double-create). Redirecting means refresh just reloads the list page. Standard practice for any create/update/delete.

---

## 8. TempData for one-time messages

`TempData` survives exactly one redirect, which is perfect for "success/error" banners after a PRG redirect. GymSystem sets it in the controller:

```csharp
TempData["SuccessMessage"] = "Member Created Successfully";
TempData["ErrorMessage"]   = result.Error;
```

and a shared partial (`_AlertBoxScript`) reads it and shows the alert on the next page. `ViewBag` / `ViewData` are similar but only live for the current request, not across a redirect.

---

## 9. Passing dropdown data (ViewBag + SelectList)

For dropdowns you often pass a `SelectList` through `ViewBag`. From GymSystem's session create:

```csharp
private async Task PopulateDropDownListsAsync(CancellationToken ct)
{
    ViewBag.Trainers   = new SelectList(await _sessionService.GetTrainersForDropDownAsync(ct), "Id", "Name");
    ViewBag.Categories = new SelectList(await _sessionService.GetCategoriesForDropDownAsync(ct), "Id", "Name");
}
```

```cshtml
<select asp-for="TrainerId" asp-items="ViewBag.Trainers" class="form-select">
    <option value="">-- choose trainer --</option>
</select>
```

`SelectList(data, "Id", "Name")` means "use `Id` as the value and `Name` as the text".

---

## 10. How it all sits in a layered app

GymSystem splits into three projects, and MVC only lives in the top one:

```
GymSystem.PL    -> Controllers + Views        (this note)
GymSystem.BLL   -> Services + ViewModels       (the logic)
GymSystem.DAL   -> EF Core + Repositories      (the data)
```

The controller's whole job is: take the request, call a BLL service, hand the result to a view. Keep controllers thin, push logic down. (See the repository/unit-of-work and result-pattern notes for what the lower layers look like.)

---

## Summary

| Piece | Job |
| ----- | --- |
| Routing | maps a URL to a controller action |
| Controller | handles the request, calls services, returns a view or redirect |
| ViewModel | data shaped for one screen (not the raw entity) |
| View (Razor) | HTML + C# that renders the model |
| `_Layout` / `_ViewStart` / `_ViewImports` | shared template, default layout, shared usings |
| Tag helpers | `asp-for`, `asp-action`, generate inputs/links/validation |
| Model binding | auto-fills action parameters from the form |
| ModelState | holds validation results, check before acting |
| PRG | redirect after a successful POST so refresh doesn't resubmit |
| TempData | one-time messages across a redirect (success/error banners) |

> Worked example: my **GymSystem** project is a full ASP.NET Core MVC app built exactly this way.
> Repo: https://github.com/Muhammed-maher32/GymManagementSystem
