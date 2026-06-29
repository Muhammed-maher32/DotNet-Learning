# EF Core: Loading Related Data & Tracking

> Two things that quietly decide whether your queries are fast or a disaster: how you load navigation properties, and whether EF is tracking the entities it hands you.

---

## Part 1: Loading related data

Say you have this:

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public ICollection<Post> Posts { get; set; }
}
```

When you load a `Blog`, do you also get its `Posts`? Depends which strategy you use.

### 1. Eager loading (`Include`)

You ask for the related data up front, in the same query. This is the one you'll use most.

```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)          // join Posts
        .ThenInclude(p => p.Comments) // go deeper
    .ToList();
```

EF turns this into a SQL `JOIN` (or a couple of queries) and fills in the navigation properties. Predictable, one trip planned.

### 2. Explicit loading

You load the main entity first, then load a navigation later, on demand.

```csharp
var blog = context.Blogs.First();

// later, when you actually need them:
context.Entry(blog)
    .Collection(b => b.Posts)
    .Load();
```

Useful when you only sometimes need the related data.

### 3. Lazy loading

The related data loads automatically the moment you touch the property. Looks magical, behaves like a trap.

```csharp
var blog = context.Blogs.First();
foreach (var post in blog.Posts) // hits the database HERE, on access
{
}
```

It needs setup (the `Microsoft.EntityFrameworkCore.Proxies` package, `virtual` navigation properties, `UseLazyLoadingProxies()`). The danger is you can't see the queries firing. It's the classic source of the next section.

---

## Part 2: The N+1 problem

This is the bug everyone hits. You load a list, then loop and touch a navigation property per item.

```csharp
var blogs = context.Blogs.ToList();          // 1 query: get all blogs

foreach (var blog in blogs)
{
    // with lazy loading, each access = its own query
    Console.WriteLine(blog.Posts.Count);     // N queries, one per blog
}
```

If you have 100 blogs that's **1 + 100 = 101** round trips to the database for data you could have fetched in one. It looks fine on your machine with 5 rows and dies in production with 50k.

Fix: load what you need in one go.

```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)   // one query, joins included
    .ToList();
```

Or project only the count if that's all you want:

```csharp
var data = context.Blogs
    .Select(b => new { b.Title, PostCount = b.Posts.Count })
    .ToList();
```

> If you ever see your logs spamming near-identical SQL in a loop, it's N+1. 99% of the time.

---

## Part 3: Tracking vs AsNoTracking

When EF runs a query, by default it **tracks** every entity it returns. Tracking means the change tracker keeps a snapshot so that `SaveChanges()` knows what changed.

```csharp
var user = context.Users.First();
user.Name = "new name";
context.SaveChanges(); // works because EF tracked `user` and saw the change
```

That snapshot costs memory and a bit of time. For **read only** queries (you're just displaying data, never saving it back) you don't need it.

```csharp
var users = context.Users
    .AsNoTracking()      // skip the snapshot, faster + less memory
    .ToList();
```

### Rule of thumb

| Query is... | Use |
| ----------- | --- |
| read only (list pages, reports, dropdowns) | `AsNoTracking()` |
| you'll modify and `SaveChanges()` | tracked (the default) |

In a typical web app the majority of your `GET` queries should be `AsNoTracking()`. It's basically free performance.

### Gotcha: identity resolution

With tracking, if the same row shows up twice in a query, you get the **same object instance** back both times. With `AsNoTracking()` you can get two separate instances of the same row. Almost never matters for read only display, but worth knowing.

---

## Summary

| Concept | One liner |
| ------- | --------- |
| Eager (`Include`) | load related data up front, planned joins |
| Explicit (`Load`) | load related data later, on demand |
| Lazy | loads on property access, hidden queries, easy to misuse |
| N+1 | 1 query + N queries in a loop, fix with `Include` or projection |
| Tracking | default, needed for updates, costs memory |
| `AsNoTracking()` | read only queries, faster, use it for most GETs |
