# LINQ

*Official_doc* -> [Open](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)

- LINQ stands for (Language Integrated Query)

> It allows you to query data in a declarative way.

---

## Before and After LINQ

### Before
```csharp
List<int> myList = new();

foreach (var x in myList)
{
    if (x > 5)
    {
        // you would usually add to another list
    }
}
```

### After
```csharp
var result = myList.Where(x => x > 5);
```

> Many LINQ methods that return `IEnumerable<T>` use deferred execution.

---

## Why LINQ?

1. Cleaner code  
2. Method chaining  
3. Unified syntax  

---

## Query Syntax vs Method Syntax

### Query
```csharp
var answer =
    from s in students
    where s.Grade > 90
    select s.Name;
```

### Method (most used)
```csharp
var answer = students
    .Where(s => s.Grade > 90)
    .Select(s => s.Name);
```

---

## The Three Parts of a LINQ Query

> Data Source, Query Definition, Query Execution

```csharp
// DATA SOURCE
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };

// QUERY DEFINITION (NOT executed yet)
var query = numbers.Where(x => x > 5);

// QUERY EXECUTION (runs now)
foreach (var item in query)
{
    Console.WriteLine(item); // 6, 7, 8
}
```

---

## Deferred vs Immediate Execution

> Deferred execution means the query is NOT executed when defined, it runs only when iterated.

```csharp
var numbers = new List<int> { 1, 2, 3 };

var query = numbers.Where(x => x > 1);

numbers.Add(4);
numbers.Add(5);

foreach (var n in query)
    Console.WriteLine(n);

// Output: 2, 3, 4, 5
```

---

> Immediate execution forces the query to run  
> Methods: `ToList()`, `ToArray()`, `Count()`

```csharp
var numbers = new List<int> { 1, 2, 3 };

var result = numbers.Where(x => x > 1).ToList();

numbers.Add(10);

foreach (var n in result)
    Console.WriteLine(n);

// Output: 2, 3
```

---

## Projection

> Transform each element using `Select()`

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

var doubled = numbers.Select(n => n * 2);
// 2,4,6,8,10
```

### Anonymous object
```csharp
var result = users.Select(u => new
{
    u.Name,
    u.Email
});
```

### Different class
```csharp
var dtos = users.Select(u => new UserDto
{
    DisplayName = u.FirstName + " " + u.LastName,
    Contact = u.Email
});
```

- `Select()` has index overload like `Where()`

---

## SelectMany()

> Used when you have nested collections (List of Lists)

```csharp
var students = new[]
{
    new { Name = "Ahmed", Courses = new[] { "Math", "CS" } },
    new { Name = "Sara", Courses = new[] { "Physics", "Math" } }
};

// Nested
var nested = students.Select(s => s.Courses);
// IEnumerable<string[]>
```

```csharp
// Flatten
var allCourses = students.SelectMany(s => s.Courses);
// IEnumerable<string>
```

---

## Grouping

> Group data based on a key using `GroupBy()`

```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6 };

var groups = numbers.GroupBy(x => x % 2 == 0);

foreach (var group in groups)
{
    Console.WriteLine(group.Key ? "Even" : "Odd");

    foreach (var item in group)
        Console.WriteLine(item);
}
```

```csharp
var groupedUsers = users.GroupBy(u => u.Country);
```

---

## Aggregations

> Reduce data to a single value

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

var count = numbers.Count();
var sum   = numbers.Sum();
var max   = numbers.Max();
var min   = numbers.Min();
var avg   = numbers.Average();
```

```csharp
var count = numbers.Count(x => x > 2);
```

```csharp
var result = users
    .GroupBy(u => u.Country)
    .Select(g => new
    {
        Country = g.Key,
        Count = g.Count()
    });
```

---

## Joining

> Combine two collections

```csharp
var students = new[]
{
    new { Id = 1, Name = "Ahmed" },
    new { Id = 2, Name = "Sara" }
};

var grades = new[]
{
    new { StudentId = 1, Grade = 90 },
    new { StudentId = 2, Grade = 85 }
};
```

### Join()
```csharp
var result = students.Join(
    grades,               // second collection
    s => s.Id,            // key from first
    g => g.StudentId,     // key from second
    // these two keys represent the ON clause
    (s, g) => new
    {
        s.Name,
        g.Grade
    });

    //Query syntax
    var result =
    from s in students
    join g in grades
    on s.Id equals g.StudentId // NOT ==
    select new
    {
        s.Name,
        g.Grade
    };
```