# Views & Razor Deep Dive

> The view side of MVC beyond the basics: layouts, partials, tag helpers, view components, and where each one fits. The main mvc.md note covers the request flow; this one is about not repeating yourself in .cshtml files.

---

## Layouts and sections

`_Layout.cshtml` is the page skeleton (navbar, footer, script includes). Every view renders *into* it at `@RenderBody()`:

```html
<!-- Views/Shared/_Layout.cshtml -->
<body>
    <nav>...</nav>
    <main>
        @RenderBody()
    </main>
    @RenderSection("Scripts", required: false)
</body>
```

Sections let a specific view inject content into a specific slot of the layout, the classic case being page-specific scripts:

```html
<!-- Views/Members/Create.cshtml -->
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

`_ViewStart.cshtml` sets the layout for everything so views don't repeat it, and `_ViewImports.cshtml` holds the shared `@using` and `@addTagHelper` lines. Both apply to their folder and everything below it.

---

## Partials: reusable chunks of markup

A partial is a .cshtml fragment rendered inside another view. Pure markup reuse, no logic of its own.

```html
<!-- Views/Shared/_MemberCard.cshtml -->
@model MemberDto

<div class="card">
    <h5>@Model.Name</h5>
    <span class="badge">@Model.PlanName</span>
</div>
```

```html
<!-- used in a loop -->
@foreach (var member in Model.Members)
{
    <partial name="_MemberCard" model="member" />
}
```

The underscore prefix is just a naming convention for "not a full page". Partials get their model from the parent; they can't fetch their own data. The moment a partial needs to load something itself, you want a view component instead.

---

## Tag helpers: server code that looks like HTML

Tag helpers are the modern replacement for `@Html.TextBoxFor(...)` helpers. They attach server behavior to normal-looking HTML attributes:

```html
<form asp-action="Create" asp-controller="Members" method="post">
    <label asp-for="Name"></label>
    <input asp-for="Name" class="form-control" />
    <span asp-validation-for="Name" class="text-danger"></span>

    <select asp-for="PlanId" asp-items="Model.Plans"></select>

    <button type="submit">Save</button>
</form>

<a asp-action="Details" asp-route-id="@member.Id">Details</a>
```

What they buy you:

* `asp-for` generates the right `name`, `id`, value binding, and validation attributes from the model property. Rename the property and the compiler catches the view (with runtime compilation off, at publish).
* `asp-action`/`asp-controller` generate URLs from routing, so changing a route doesn't break links scattered across views.
* `asp-validation-for` wires up the client-side validation messages that pair with data annotations.

The form + label + input + validation-span block above is basically muscle memory for every Create/Edit page.

You can write custom ones (inherit `TagHelper`, override `Process`), useful for things like rendering a badge from an enum consistently. Register in `_ViewImports.cshtml` with `@addTagHelper`.

---

## View components: partials that fetch their own data

The problem: the cart icon in the navbar needs the item count. The navbar is in `_Layout`, which every page uses, so *every* controller action would have to load the count into ViewBag. Horrible.

A view component is a mini controller + view pair that runs independently of the parent view's model:

```csharp
public class ActiveMembersCountViewComponent : ViewComponent
{
    private readonly IMemberService _memberService;

    public ActiveMembersCountViewComponent(IMemberService memberService)
        => _memberService = memberService;

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var count = await _memberService.CountActiveAsync();
        return View(count);   // Views/Shared/Components/ActiveMembersCount/Default.cshtml
    }
}
```

```html
<!-- Default.cshtml -->
@model int
<span class="badge bg-success">@Model active</span>
```

```html
<!-- anywhere, e.g. _Layout.cshtml -->
<vc:active-members-count />
```

It gets DI like a controller, loads its own data, renders its own view. Rule of thumb:

* Markup only, data comes from parent -> **partial**
* Needs its own data/services -> **view component**

---

## Small things that keep views clean

* **Views should be dumb.** If a view has real logic (calculations, formatting decisions), push it into the ViewModel. `@if` for showing/hiding is fine; business rules are not.
* **ViewModels over ViewBag.** ViewBag is dynamic: typos compile fine and fail at runtime as empty output. I use it for one-off tiny things like Title, nothing else.
* **`asp-append-version="true"`** on script/css tags busts browser cache when the file changes.
* **Razor encodes by default.** `@Model.Name` is HTML-encoded, which is your XSS protection. `Html.Raw` turns that off; treat it as a code smell unless you sanitized the content yourself.

---

## Common Mistakes

* **Loading navbar/sidebar data in every controller action** instead of a view component.
* **Copy-pasting form markup between Create and Edit** instead of a shared partial bound to the same ViewModel.
* **Logic creep in views.** Two months later the .cshtml is doing pricing calculations and nobody can test it.
* **Forgetting `_ValidationScriptsPartial`** and concluding client-side validation "doesn't work".
* **Hardcoded URLs** (`href="/members/details/5"`) instead of `asp-` helpers, then a route change silently breaks half the links.

---

## Summary

| Idea | Point |
| ---- | ----- |
| Layout + sections | page skeleton; views inject scripts via sections |
| Partial | reusable markup, model comes from the parent |
| Tag helpers | asp-for/asp-action generate binding, URLs, validation wiring |
| View component | self-contained widget with DI and its own data |
| ViewModels | typed data for views; ViewBag only for trivia |

> Continues [mvc.md](./mvc.md). Validation attributes that asp-validation-for surfaces are covered in [../4.Web-Basics/validation.md](../4.Web-Basics/validation.md).
