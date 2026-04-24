# EF CORE - Relationships & Configuration

**Important => ** (Introduction)[https://github.com/Muhammed-maher32/DotNet-Learning/blob/main/Ef-Core/mds/Important_Introduction.md]
## Revision on Relationships

> **1 - 1:** Take foreign key from one table to another
> **1 - M:** The foreign key goes to the "Many" side; take the primary key from the One side.
> **M - M:** Create a junction table that contains 2 foreign keys from the two tables' primary keys.

---

## Configurations

**What is configuration?**
It is the set of rules that tells the EF Core mapping engine how to translate your POCOs (C# classes) to your database schema.

**Configuration types in priority (lowest to highest):**

1. Naming Convention
2. Data Annotation
3. Fluent API ← highest priority

---

### Naming Convention

EF Core infers configuration from property names and types automatically. Examples:
- A property named `Id` or `<TypeName>Id` is inferred as the primary key.
- A property of type `<TypeName>` paired with a `<TypeName>Id` property is inferred as a navigation property with its foreign key.

---

### Data Annotation

Attributes applied directly on model classes/properties. Examples:

```csharp
[Key]
public int Id { get; set; }

[Required]
[MaxLength(100)]
public string Name { get; set; }

[ForeignKey("CategoryId")]
public Category Category { get; set; }
```

> **Limitation:** Cannot configure composite keys — use Fluent API for that.

---

### Fluent API

You can write it in two places:

1. `OnModelCreating()` — works but not recommended for large projects.
2. `IEntityTypeConfiguration<T>` classes — preferred, gives you more control and keeps configuration organized.

```csharp
// IEntityTypeConfiguration<T> example (recommended)
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).IsRequired().HasMaxLength(100);
        builder.HasOne(p => p.Category)
               .WithMany(c => c.Products)
               .HasForeignKey(p => p.CategoryId);
    }
}

// Register in DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

---

#### Fluent API Examples

**1. Primary Keys**

```csharp
// Basic primary key (Id is already detected by convention — just for illustration)
builder.HasKey(p => p.Id);
builder.Property(p => p.Id).ValueGeneratedOnAdd(); // Auto-increment
```

**GUID as Primary Key**

> Adds a layer of security — GUIDs are 16 bytes and not guessable.
> However, random GUIDs cause index fragmentation in SQL Server.
> `NEWSEQUENTIALID()` solves this by generating sequential GUIDs, which are safe for clustered indexes.

```csharp
public class MyClass
{
    public Guid Id { get; set; }
}

// Let the database generate a random GUID (causes index fragmentation)
builder.Property(p => p.Id)
       .HasDefaultValueSql("NEWID()");

// Better: sequential GUID — avoids index fragmentation in SQL Server
builder.Property(p => p.Id)
       .HasDefaultValueSql("NEWSEQUENTIALID()");

// Generate GUID manually in C# (set it yourself, EF won't generate it)
builder.Property(p => p.Id).ValueGeneratedNever();
```

**2. Composite Key**

```csharp
public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
}

// Fluent API — only way to define composite keys
builder.HasKey(ck => new { ck.OrderId, ck.ProductId });

// [Key] annotation cannot define composite keys at all.
```

---

## Relationships

### 1. One-to-One

```csharp
// Naming Convention
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }

    public UserProfile Profile { get; set; } // Navigation property
}

public class UserProfile
{
    public int Id { get; set; }
    public string Bio { get; set; }

    public int UserId { get; set; }          // Foreign key
    public User User { get; set; }           // Navigation property
}

// Fluent API (mandatory to resolve ambiguity — specify which side holds the FK)
builder.HasOne(u => u.Profile)
       .WithOne(p => p.User)
       .HasForeignKey<UserProfile>(p => p.UserId);
```

---

### 2. One-to-Many

One category has many products, but each product belongs to one category.

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Product> Products { get; set; } // Navigation property
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }

    public int CategoryId { get; set; }       // FK
    public Category Category { get; set; }    // Navigation property
}

// Fluent API (optional — convention handles this, but explicit is clearer)
builder.HasOne(p => p.Category)
       .WithMany(c => c.Products)
       .HasForeignKey(p => p.CategoryId);
```

---

### 3. Many-to-Many (Simple — No Payload)

EF Core automatically creates the join table. No extra data on the relationship.

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Course> Courses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }

    public ICollection<Student> Students { get; set; }
}
```

---

### 3.1 Many-to-Many with Payload

When you need extra data on the relationship, it becomes an independent entity.

```
Student (1) ──────── (*) Enrollment (*) ──────── (1) Course
                             │
                       Extra data here
                    (EnrollmentDate, Grade)
```

```csharp
public class Enrollment
{
    public int StudentId { get; set; }          // FK
    public int CourseId { get; set; }           // FK

    // Extra data on the relationship
    public DateTime EnrollmentDate { get; set; }
    public string Grade { get; set; }

    public Student Student { get; set; }
    public Course Course { get; set; }
}

public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Enrollment> Enrollments { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }

    public ICollection<Enrollment> Enrollments { get; set; }
}

// Fluent API — mandatory
builder.HasKey(e => new { e.StudentId, e.CourseId }); // Composite key for join entity
```

---

### 4. Self-Referencing

Navigate to both parent and children within the same entity.

```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }

    public int? ManagerId { get; set; }                   // Nullable — top-level managers have no manager

    public Employee? Manager { get; set; }                // Navigation to parent
    public ICollection<Employee> Subordinates { get; set; } // Navigation to children
}

// Fluent API — mandatory due to ambiguity
builder.HasOne(e => e.Manager)
       .WithMany(e => e.Subordinates)
       .HasForeignKey(e => e.ManagerId)
       .OnDelete(DeleteBehavior.Restrict); // Prevent deleting a manager who has subordinates
```

---

## Relationships Summary

### One-to-One

```
                ┌───────────────┐
                │  ONE-TO-ONE   │
                └───────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
   Convention                      FK on dependent side
 (navigation on both sides)               │
                                          │
                               ┌──────────▼──────────┐
                               │      Ambiguity       │
                               └──────────┬──────────┘
                                          │
                               Fluent API → MANDATORY
                               (specify FK side explicitly)
```

### One-to-Many

```
                ┌───────────────┐
                │  ONE-TO-MANY  │
                └───────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
      ONE SIDE                        MANY SIDE
        │                                 │
 Navigation property               FK + Navigation property
 ICollection<Many>                        │
        │                                 │
        └──────────── Convention works ✔️ ┘
                                │
                        Fluent API → OPTIONAL
```

### Many-to-Many

```
                ┌────────────────┐
                │ MANY-TO-MANY   │
                └────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
     SIMPLE                          WITH PAYLOAD
        │                                 │
 ICollection<> both sides          ┌──────────────┐
        │                          │  JOIN ENTITY │
 EF creates join table             └──────────────┘
 automatically                            │
        │                      ┌──────────┼──────────┐
   No extra data               │                     │
                           FK A + FK B           Extra Data
                               │              (Date, Grade)
                               └──────────┬──────────┘
                                          │
                               Becomes REAL ENTITY
                                          │
                              Fluent API → MANDATORY
                                          │
                                Composite Key (AId, BId)
```

### Self-Referencing

```
                ┌──────────────────┐
                │ SELF-REFERENCING │
                └──────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
     Parent                           Children
  (Manager)                       (Subordinates)
        │                                 │
   FK → ManagerId (nullable)      ICollection<Employee>
        │
        ▼
   Same entity linked to itself
                │
        ┌───────▼────────┐
        │   Ambiguity    │
        └───────┬────────┘
                │
      Fluent API → MANDATORY
                │
      OnDelete → Restrict ✔️
      (prevent cascade delete disaster ❌)
```