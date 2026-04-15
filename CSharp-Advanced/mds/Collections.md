# Collections in C#

---

## What are Collections?

Collections are data structures used to store and manage groups of objects.

---

## Why Collections matter

* Organizing data efficiently
* Improving performance
* Core part of backend systems

---

## Common Types

### List<T>

```csharp
List<int> numbers = new();
```

* Dynamic array
* Fast indexed access
* Allows duplicates

**Complexity:**

* Access (by index): O(1)
* Add (end): O(1) average / O(n) worst (resize)
* Insert / Remove (middle): O(n)

---

### Dictionary<TKey, TValue>

```csharp
Dictionary<int, string> users = new();
```

* Key-value pairs
* Very fast lookup
* Keys must be unique

**Complexity:**

* Lookup: O(1) average / O(n) worst (collision)
* Add: O(1) average
* Remove: O(1) average

---

### HashSet<T>

```csharp
HashSet<string> emails = new();
```

* Stores unique values only
* No duplicates
* Fast lookup

**Complexity:**

* Add: O(1) average
* Remove: O(1) average
* Contains: O(1) average

---

### Queue<T>

```csharp
Queue<string> queue = new();
```

* FIFO (First In First Out)
* Used in task processing

**Complexity:**

* Enqueue: O(1)
* Dequeue: O(1)
* Peek: O(1)

---

### Stack<T>

```csharp
Stack<string> stack = new();
```

* LIFO (Last In First Out)
* Used in undo operations

**Complexity:**

* Push: O(1)
* Pop: O(1)
* Peek: O(1)

---

## Real World Usage

* Caching systems
* Task queues
* Data processing pipelines

---

## Common Mistakes

* Using List instead of HashSet for uniqueness
* Ignoring performance differences
* Not choosing the right data structure
