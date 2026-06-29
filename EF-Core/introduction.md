# EF Core - TheBase

> A practical introduction to Entity Framework Core for .NET developers.

---

## 1. What is EF Core?

EF Core is an **ORM (Object-Relational Mapper)** for .NET.

Simple idea:

- Your code uses **objects** → Your database uses **tables** → EF Core connects both.

Instead of writing raw SQL for everything, you work with C# objects and let EF Core handle the database communication.

### Without EF Core

```csharp
var command = new SqlCommand("SELECT Id, Name FROM Students", connection);
// Raw SQL: no compile-time error checking — bugs appear only at runtime
var reader = command.ExecuteReader();

while (reader.Read())
{
    var student = new Student
    {
        Id = (int)reader["Id"],
        Name = reader["Name"].ToString()
    };
}
```

### With EF Core

```csharp
var students = context.Students.ToList();
```

---

## 2. What Problem Does EF Core Solve?

### Problem 1: Object vs Table Mismatch

C#:

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

Database:

```
Students Table
--------------
Id  | Name
```

EF Core maps between them automatically.

---

### Problem 2: Repetitive Data Access Code

EF Core eliminates:

- Manual column-to-property mapping
- Repetitive SQL strings
- Manual connection open/close handling

---

### Problem 3: Schema Changes

When your model changes, EF Core updates the database schema using **migrations** — no manual `ALTER TABLE` scripts needed.

---

## 3. How EF Core Works

### Pipeline

```
C# Objects
    ↓
DbContext
    ↓
Change Tracking
    ↓
SQL Generation
    ↓
Database
    ↓
Results → C# Objects
```

---

### Query Flow

```csharp
var students = context.Students.ToList();
```

Steps:

1. Build LINQ query
2. Translate to SQL
3. Execute against the database
4. Map result rows → C# objects

---

### Insert Flow

```csharp
var student = new Student { Name = "Ali" };
context.Students.Add(student);
context.SaveChanges();
```

Steps:

1. EF Core starts tracking the new object
2. `SaveChanges()` generates an `INSERT` statement
3. SQL is executed against the database

---

### Key Rule

> **Nothing is saved to the database until `SaveChanges()` is called.**

---

## 4. DbContext

### What is it?

`DbContext` is your **session with the database**.

Responsibilities:

- Manage the database connection
- Track changes to objects
- Run queries
- Save changes

---

### Example

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Student> Students { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer("Server=.;Database=SchoolDb;Trusted_Connection=True;");
    }
}
```

---

### ❗ Common Mistake: Forgetting SaveChanges()

```csharp
using var context = new AppDbContext();

context.Students.Add(new Student { Name = "Ali" });
context.SaveChanges(); // Without this, nothing is inserted
```

---

### ❗ Avoid Long-Lived DbContext

```csharp
// Bad — static/shared context
public static AppDbContext context = new AppDbContext();
```

**Why this is a problem:**

- The change tracker accumulates every object it ever touched → memory grows
- The database connection stays open longer than needed
- Cached data goes stale; you read old values without knowing it
- Hard to debug because the context state is unpredictable

```csharp
//short-lived context with using
using var context = new AppDbContext();

var student = new Student
{
    Name = "Ali",
    Email = "ali@example.com"
};

context.Students.Add(student);
context.SaveChanges();
// context.Dispose() is called automatically when the block ends
```

---

## 5. DbSet

### What is DbSet?

A `DbSet<T>` represents a **database table** in your context:

```csharp
public DbSet<Student> Students { get; set; }
```

---

### Common Operations

```csharp
context.Students.Add(student);       // INSERT
context.Students.Remove(student);    // DELETE
context.Students.ToList();           // SELECT *
context.Students.Find(id);           // SELECT WHERE Id = @id
context.Students.Where(s => s.Name == "Ali").ToList(); // filtered SELECT
```

---

### ⚠️ DbSet is NOT a List

||`DbSet<T>`|`List<T>`|
|---|---|---|
|Lives in|Database|Memory|
|Operations|Translated to SQL|In-memory|
|Execution|Deferred (lazy)|Immediate|

```csharp
var query = context.Students.Where(s => s.Name == "Ali");
// No SQL executed yet — query is just built

var result = query.ToList();
// SQL executes HERE
```

---
## 6. Migrations

### What are migrations?

Migrations track your **schema changes over time**, similar to version control for your database.

---
### Example: Adding a Property

Initial model:

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

You add `Email` later:

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; } // new
}
```

---
### Apply the Migration

**Package Manager Console (Visual Studio):**

```powershell
Add-Migration AddEmailToStudent
Update-Database
```

**dotnet CLI:**

```bash
dotnet ef migrations add AddEmailToStudent
dotnet ef database update
```

EF Core generates and applies the SQL:

```sql
ALTER TABLE Students ADD Email NVARCHAR(MAX) NULL;
```

---
### Remove a Migration (before applying)

```powershell
Remove-Migration
```

```bash
dotnet ef migrations remove
```

> Only remove migrations that have **not** been applied to the database yet.

---

## 7. Full CRUD Scenario

### Step 1: Define the Entity

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}
```

---

### Step 2: Configure the Context

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Student> Students { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer("Server=.;Database=SchoolDb;Trusted_Connection=True;");
    }
}
```

---

### Step 3: Create and Apply Migration

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

---

### Step 4: Insert

```csharp
using var context = new AppDbContext();

context.Students.Add(new Student { Name = "Ali", Email = "ali@example.com" });
context.SaveChanges();
```

---

### Step 5: Read

```csharp
using var context = new AppDbContext();

var students = context.Students.ToList();

foreach (var s in students)
    Console.WriteLine($"{s.Id} - {s.Name}");
```

---

### Step 6: Update

```csharp
using var context = new AppDbContext();

var student = context.Students.First();
student.Name = "Ali Updated";
context.SaveChanges(); // EF Core detects the change and generates UPDATE
```

---

### Step 7: Delete

```csharp
using var context = new AppDbContext();

var student = context.Students.First();
context.Students.Remove(student);
context.SaveChanges();
```

##  Summary

|Concept|Role|
|---|---|
|**Entity**|C# class that maps to a table|
|**DbContext**|Your session/manager with the database|
|**DbSet<T>**|Represents a single table|
|**Migrations**|Version control for your schema|
|**SaveChanges()**|Commits all tracked changes to the database|
