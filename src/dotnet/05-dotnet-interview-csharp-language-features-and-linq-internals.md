---
layout: default
title: ".NET Interview — C# Language Features & LINQ Internals"
---

# .NET Interview — C# Language Features & LINQ Internals

This set covers the modern C# features that show up constantly in senior interviews — records, pattern matching, nullable reference types, primary constructors, collection expressions, and `required`/`init` members — plus how LINQ actually works under the hood: deferred execution, the `IEnumerable<T>`/`IQueryable<T>` split, and the performance pitfalls that trip people up when a "simple" LINQ chain turns into a correctness or performance bug.

### Q1. What does the `record` modifier actually add on top of a `class` or `struct`, and how is a `record class`'s equality different from a plain class's? {#q1}

**Question:**
What does the `record` modifier actually add on top of a `class` or `struct`, and how is a `record class`'s equality different from a plain class's?

**Good answer:**
`record` is a modifier you apply to a `class` or `struct` declaration. It doesn't change whether the type is a reference type or a value type — a plain `record` (shorthand for `record class`) is still a reference type, and `record struct` is still a value type. What it adds is compiler-generated, property-by-property *value equality* (`Equals`, `GetHashCode`, and the `==`/`!=` operators), a formatted `ToString()`, and support for non-destructive mutation via `with` expressions. A plain `class` uses reference equality by default (`==` checks whether two variables point to the same object); a `record class` compares every property's value instead, so two distinct objects with identical property values are equal even though `ReferenceEquals` on them is `false`.

**Code example:**
```csharp
public record Person(string FirstName, string LastName);
public class PersonClass(string FirstName, string LastName);

var p1 = new Person("Grace", "Hopper");
var p2 = new Person("Grace", "Hopper");
Console.WriteLine(p1 == p2);               // True (value equality)
Console.WriteLine(ReferenceEquals(p1, p2)); // False
```

**Follow-up question:**
If a positional record has an array property, does value equality compare the array's contents?

**Follow-up good answer:**
No. Value equality is property-by-property, and for a reference-typed property like an array, "the property's value" is the reference itself — so two records with different array instances holding identical elements are *not* equal, even though the arrays look the same. If two records share the same array reference, mutating the array through one record is visible through the other, since neither record copies the array.

**Glossary:**
- **Record** — a `class`/`struct` modifier that adds value equality, `ToString`, and `with`-expression support.
- **Value equality** — equality based on comparing data/state rather than object identity.
- **Reference equality** — equality based on whether two variables point to the same object (`ReferenceEquals`).

**Mental model:**
This checks whether the candidate understands that `record` is additive syntax sugar over the existing class/struct semantics, not a third kind of type — and whether they know the sharp edge where "value equality" doesn't mean "deep equality" for reference-typed members.

**TL;DR:**
`record` adds compiler-generated value equality/`ToString`/`with`-support on top of a class or struct, but reference-typed properties (like arrays) are still compared by reference, not by contents.

**References:**
- [C# record types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

---

### Q2. How does a `with` expression achieve "non-destructive mutation," and what actually happens under the hood when you write `original with { FirstName = "X" }`? {#q2}

**Question:**
How does a `with` expression achieve "non-destructive mutation," and what actually happens under the hood when you write `original with { FirstName = "X" }`?

**Good answer:**
A `with` expression copies the existing record instance and then applies the specified property changes to that copy, leaving the original untouched. For `record class`, this is implemented via a compiler-generated copy constructor plus a hidden `<Clone>$` method; the `with` expression calls the clone method to get a shallow copy, then applies each specified property assignment (which requires the property to have an `init` or `set` accessor) to the copy. For `record struct`, since assignment already copies the whole value, `with` just copies the struct and mutates the copy's fields directly — no cloning method is needed. Either way, the original reference/value is never mutated, which is what makes it safe to treat records as immutable while still being able to derive variations cheaply.

**Code example:**
```csharp
var original = new Person("Grace", "Hopper");
var modified = original with { FirstName = "Margaret" };

Console.WriteLine(original);  // Person { FirstName = Grace, LastName = Hopper }
Console.WriteLine(modified);  // Person { FirstName = Margaret, LastName = Hopper }
Console.WriteLine(original == modified); // False
```

**Follow-up question:**
Does `original with { }` (no changes) produce a new object, or does it return the same reference?

**Follow-up good answer:**
For a `record class`, it always produces a new object — the clone method still runs, allocating a new instance, even though no properties change. It's equal to the original (`original == copy` is `true`, since value equality compares data) but `ReferenceEquals(original, copy)` is `false`. For a `record struct`, there's no separate allocation either way since it's a value type — `with { }` just produces another copy of the value, which is what happens on any struct assignment.

**Glossary:**
- **`with` expression** — copies a record and applies specified property changes to the copy.
- **Non-destructive mutation** — producing a modified copy instead of mutating the original in place.

**Mental model:**
Tests whether the candidate can explain the actual mechanism (copy-then-assign) rather than treating `with` as magic, and whether they understand it still allocates for `record class` even on a no-op copy.

**TL;DR:**
`with` copies the record (via a compiler-generated clone for `record class`, or plain value copy for `record struct`) and applies the specified property changes to the copy — the original is never mutated.

**References:**
- [C# record types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

---

### Q3. What does "deferred execution" mean for a LINQ query, and what mechanism in C# makes it possible? {#q3}

**Question:**
What does "deferred execution" mean for a LINQ query, and what mechanism in C# makes it possible?

**Good answer:**
Deferred execution means a LINQ query isn't run at the point where you write/assign it — it's just a stored description of the operation. The actual work happens later, when the result is enumerated (e.g., via `foreach`, or a method that forces enumeration like `ToList()`). This is supported directly by the C# `yield return` statement inside an iterator block, which the compiler turns into a state machine implementing `IEnumerator<T>`. Most LINQ-to-Objects operators that return `IEnumerable<T>` (`Where`, `Select`, `SkipWhile`, etc.) are implemented this way, so chaining several of them together doesn't do any per-element work until enumeration starts — at that point, each element flows through the whole chain one at a time rather than materializing an intermediate collection per step.

**Code example:**
```csharp
var query = numbers.Where(n => n % 2 == 0); // nothing executes yet
Console.WriteLine("Query created");
foreach (var n in query) { /* work happens here, one element at a time */ }
```

**Follow-up question:**
Are all LINQ operators deferred? Name one that isn't, and explain why it can't be.

**Follow-up good answer:**
No — operators that return a single scalar value execute immediately, because they have to consume the whole source (or enough of it) to produce that one value right away: `Count()`, `Sum()`, `Average()`, `First()`, `Max()`. There's no way to return "a description of the count" — by definition, computing a count means iterating. `OrderBy` is a more subtle case: it returns `IOrderedEnumerable<T>` (still deferred in the sense that it doesn't run until enumerated), but it's classified as "non-streaming" because once enumeration starts, it has to read the *entire* source before it can produce even the first sorted element.

**Glossary:**
- **Deferred execution** — evaluation of an expression delayed until its value is actually needed.
- **Iterator block** — a method/property using `yield return` that the compiler compiles into a state machine.
- **Streaming vs. non-streaming** — whether an operator can yield results as it reads the source, or must consume it all first.

**Mental model:**
Checks whether the candidate actually understands *why* LINQ is lazy (the `yield return` state-machine mechanism) rather than just knowing the word "deferred," and whether they can identify the boundary cases (immediate, non-streaming) instead of treating all LINQ as uniformly lazy.

**TL;DR:**
LINQ's deferred execution is built on `yield return` iterator blocks — the query is just a stored description until something enumerates it; scalar-returning operators like `Count()`/`Sum()` are the exception and run immediately.

**References:**
- [Deferred execution and lazy evaluation | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/linq/deferred-execution-lazy-evaluation)
- [Introduction to LINQ Queries | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries)

---

### Q4. What's the performance/correctness risk of enumerating the same deferred LINQ query variable twice, and how do you fix it? {#q4}

**Question:**
What's the performance/correctness risk of enumerating the same deferred LINQ query variable twice, and how do you fix it?

**Good answer:**
Because the query variable doesn't store results — just the recipe for producing them — enumerating it again re-runs the entire operation from scratch against the current state of the data source. This has two consequences: it's wasted work (an expensive filter/projection chain runs twice instead of once), and it can be a correctness bug if the underlying source can change between enumerations — the two enumerations can return *different* results, which is surprising if the code looks like it's just "reading" a value twice. The fix is to materialize the query once with `ToList()` or `ToArray()`, store that in a variable, and enumerate the materialized collection as many times as needed — that also pins down a stable snapshot instead of re-querying a possibly-changed source.

**Code example:**
```csharp
// Bad: re-evaluates the query (and the DB round-trip, if IQueryable) on every enumeration
var query = source.Where(x => x.IsActive);
var count = query.Count();      // enumerates once
var list  = query.ToList();     // enumerates AGAIN, from scratch

// Good: materialize once, reuse the result
var results = source.Where(x => x.IsActive).ToList();
var count = results.Count;
var list  = results;
```

**Follow-up question:**
Is `.Count()` on an `IEnumerable<T>` always an O(n) re-enumeration, or can it be cheaper?

**Follow-up good answer:**
`Enumerable.Count()` has a fast path: if the source at runtime actually implements `ICollection<T>` (or `ICollection`), it uses that interface's `Count` property directly instead of enumerating — so calling `.Count()` on a `List<T>` or an array is O(1), not O(n). But for a genuinely deferred LINQ query — one built from `Where`/`Select`/etc. that hasn't been materialized — the runtime type is an iterator, not a collection, so `Count()` falls back to enumerating every element, which is O(n) and (per the earlier point) re-runs the whole upstream chain.

**Glossary:**
- **Materialize** — force a deferred query to execute now and store its results in a concrete collection.
- **Re-enumeration** — running a deferred query's logic again from the start on a second `foreach`/terminal call.

**Mental model:**
This is a classic "looks free, isn't" LINQ pitfall — it tests whether the candidate has actually been burned by silently doubled work (or worse, a query that returns different results the second time) rather than just knowing LINQ is "lazy" as trivia.

**TL;DR:**
A deferred LINQ query re-runs its entire chain every time it's enumerated — call `.ToList()`/`.ToArray()` once and reuse that if you need the results more than once.

**References:**
- [Deferred execution and lazy evaluation | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/linq/deferred-execution-lazy-evaluation)
- [Enumerable.Count Method | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.count)

---

### Q5. What's the fundamental difference between `IEnumerable<T>` and `IQueryable<T>`, and why does that difference matter for something like Entity Framework? {#q5}

**Question:**
What's the fundamental difference between `IEnumerable<T>` and `IQueryable<T>`, and why does that difference matter for something like Entity Framework?

**Good answer:**
`IQueryable<T>` inherits from `IEnumerable<T>` but adds an `Expression` property (the query represented as an *expression tree* — data describing the query's structure) and a `Provider` (an `IQueryProvider` that knows how to interpret that tree). With `IEnumerable<T>`, a LINQ method like `Where` takes a `Func<T, bool>` — a compiled delegate — and runs it in-process, item by item, in .NET (this is "LINQ to Objects"). With `IQueryable<T>`, the same `Where` call takes an `Expression<Func<T, bool>>` instead: the lambda is captured as an inspectable expression tree rather than compiled to IL. A provider like EF Core's walks that tree and translates it into the target query language (SQL), so the filtering happens in the database, not by pulling every row into memory first. This is why swapping `IEnumerable<Customer>` for `IQueryable<Customer>` over the same-looking LINQ code can turn "download the whole table and filter in .NET" into "run a `WHERE` clause in SQL."

**Code example:**
```csharp
IQueryable<Customer> q = dbContext.Customers.Where(c => c.City == "London");
// Where's lambda becomes an Expression<Func<Customer,bool>> here,
// which EF Core translates into SQL — not run in-process.
```

**Follow-up question:**
If `IQueryable<T>` is so much more efficient for database queries, why does `IEnumerable<T>` still exist and get used for the same kinds of data?

**Follow-up good answer:**
`IEnumerable<T>`/LINQ to Objects is what you use once the data is already in memory (arrays, `List<T>`, or after you've materialized a query) — there's no query provider to translate an expression tree for, and no benefit to paying the overhead of building/walking expression trees when a compiled delegate can just run directly. `IQueryable<T>` only earns its keep when there's a provider on the other end (a database, a remote service) that can push work down to where the data lives; using it over an already-in-memory collection (via `AsQueryable()`) doesn't get you anything IEnumerable-based LINQ wouldn't.

**Glossary:**
- **Expression tree** — an in-memory data structure representing code as data (e.g., a lambda's structure) rather than compiled, executable IL.
- **Query provider** — an `IQueryProvider` implementation that translates an `IQueryable<T>`'s expression tree into a target query (e.g., SQL).

**Mental model:**
Tests whether the candidate actually understands the expression-tree-vs-delegate split, not just "IQueryable is for databases, IEnumerable is for lists" as a memorized rule with no mechanism behind it.

**TL;DR:**
`IQueryable<T>` captures LINQ operations as expression trees a provider can translate (e.g., to SQL); `IEnumerable<T>` runs compiled delegates in-process — that's why the same-looking `Where` call behaves completely differently depending on which one you're chained onto.

**References:**
- [IQueryable Interface | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.linq.iqueryable)
- [Introduction to LINQ Queries | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries)

---

### Q6. What is "client evaluation" in EF Core, why is it dangerous, and how would you detect it before it causes a production incident? {#q6}

**Question:**
What is "client evaluation" in EF Core, why is it dangerous, and how would you detect it before it causes a production incident?

**Good answer:**
EF Core tries to translate as much of an `IQueryable` LINQ expression as possible into SQL. If part of the query — say, a call to a .NET method the SQL provider can't translate — can't be pushed to the database, older/permissive configurations would silently fall back to pulling the untranslated data to the client and finishing the filter in memory ("client evaluation"). The danger is that this can pull an entire table over the network before filtering, which looks fine in development against a tiny seed dataset but becomes a severe performance problem (or an outage) once the table has millions of rows in production — the query still "works," it's just doing enormously more work than the developer assumed. You detect it by reviewing generated SQL/query logs (EF Core logs a warning when client evaluation happens) and, defensively, by configuring EF Core to throw on client evaluation in the `Where`/predicate parts of a query instead of silently allowing it, so an untranslatable expression fails fast in development/CI rather than degrading silently in production.

**Code example:**
```csharp
// Configure EF Core to throw instead of silently falling back to client evaluation
optionsBuilder.ConfigureWarnings(warnings =>
    warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
```

**Follow-up question:**
Since EF Core 3.0 changed the default behavior here, what's different now compared to earlier EF Core versions?

**Follow-up good answer:**
Starting with EF Core 3.x, client evaluation in the `Where`, `HAVING`, or join-key parts of a query is no longer silently allowed — if part of the filtering logic can't be translated to SQL, EF Core throws at runtime instead of quietly pulling extra data and finishing in memory. Client evaluation is still permitted, without a warning, in the final projection (the `Select` that shapes the result), since that only affects the shape of already-filtered rows rather than how much data gets pulled from the database.

**Glossary:**
- **Client evaluation** — completing part of a query in application memory because the database provider couldn't translate it.
- **Query translation** — converting a LINQ/`IQueryable` expression tree into the target query language (e.g., SQL).

**Mental model:**
Tests real production experience with an ORM, not just LINQ syntax knowledge — this is the kind of bug that passes code review and local testing and only shows up under real data volume.

**TL;DR:**
EF Core "client evaluation" silently pulls untranslatable query logic into memory to finish it there — since EF Core 3.x this throws by default for filtering (not projection), specifically to prevent this from silently shipping to production.

**References:**
- [Client vs. Server Evaluation | Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/querying/client-eval)

---

### Q7. What does C# pattern matching let you express that a plain `if`/`is`/`==` chain doesn't, and what's the difference between a type pattern and a property pattern? {#q7}

**Question:**
What does C# pattern matching let you express that a plain `if`/`is`/`==` chain doesn't, and what's the difference between a type pattern and a property pattern?

**Good answer:**
Pattern matching lets you test *and destructure* an expression's shape in one step, with the compiler tracking flow-sensitive information (like null-state) for you. A *declaration/type pattern* (`obj is int number`) tests the runtime type of an expression and, if it matches, safely casts and binds it to a new variable in that scope — this is safer than an explicit cast because there's no `InvalidCastException` risk, and it correctly rejects `null` regardless of the compile-time type. A *property pattern* (`order is { Items: > 10, Cost: > 1000m }`) instead inspects one or more properties of an object directly, without needing a type test first (or combined with one), and can nest arbitrarily. `switch` expressions combine many such patterns into a single exhaustive dispatch, with the compiler warning if not every case is handled.

**Code example:**
```csharp
public decimal CalculateDiscount(Order order) => order switch
{
    { Items: > 10, Cost: > 1000.00m } => 0.10m,
    { Items: > 5, Cost: > 500.00m }   => 0.05m,
    { Cost: > 250.00m }               => 0.02m,
    null => throw new ArgumentNullException(nameof(order)),
    _ => 0m,
};
```

**Follow-up question:**
What's a *list pattern*, and what kind of problem is it specifically good for?

**Follow-up good answer:**
A list pattern matches elements of a sequence (array/list) by position — e.g., `[_, "DEPOSIT", _, var amount]` matches a 4-element sequence where the second element is exactly `"DEPOSIT"`, discards the first and third, and captures the fourth as `amount`. A slice pattern (`..`) can match a variable number of elements in the middle. This is well suited to parsing loosely-structured, delimited data (like CSV rows with a varying column count depending on record type) where you want to match on both the *shape* (how many elements, which ones matter) and specific values, without manually indexing and length-checking the array yourself.

**Glossary:**
- **Declaration/type pattern** — tests an expression's runtime type and binds it to a new variable if it matches.
- **Property pattern** — matches based on the values of one or more properties of an object.
- **List pattern** — matches elements of a sequence by position, optionally using a slice pattern for a variable-length middle section.

**Mental model:**
Checks whether the candidate sees pattern matching as more than "syntax sugar for `is`" — specifically the exhaustiveness checking and the shape-testing that turns validation/dispatch logic into declarative, compiler-checked code.

**TL;DR:**
Type patterns safely test-and-cast; property patterns test an object's shape by its property values; `switch` expressions combine them with compiler-enforced exhaustiveness — all more than a chain of `if`/`is`/`==` can express as concisely.

**References:**
- [Pattern matching overview | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)

---

### Q8. Nullable reference types are described as a "compile-time feature." What does that mean in practice, and what's the actual runtime behavior difference between `string` and `string?`? {#q8}

**Question:**
Nullable reference types are described as a "compile-time feature." What does that mean in practice, and what's the actual runtime behavior difference between `string` and `string?`?

**Good answer:**
There is no runtime behavior difference at all — `string` and `string?` are both exactly `System.String` at the IL/runtime level; the `?` is purely an annotation the compiler uses for static analysis. With `<Nullable>enable</Nullable>` set, the compiler tracks each expression's *null-state* (not-null vs. maybe-null) based on assignments and null checks, and emits *warnings* — not errors, not runtime checks — when code's behavior doesn't match its declared intent (e.g., dereferencing a `maybe-null` value without a check, or assigning a possibly-null expression to a non-nullable variable). Because it's compile-time only, nothing stops a `null` from actually ending up in a non-nullable-typed variable at runtime (e.g., via a library compiled without nullable annotations, deserialization, or `default`/`new()` on a struct) — the feature reduces `NullReferenceException`s by catching a wide class of mistakes at compile time, it doesn't eliminate the possibility of them.

**Code example:**
```csharp
string? optional = null;      // allowed
string required = "always";   // compiler expects this to never be null
// required = optional;       // warning: possible null reference assignment
```

**Follow-up question:**
Give a concrete example of code where a non-nullable reference type variable ends up holding `null` at runtime with no compiler warning.

**Follow-up good answer:**
A `struct` with non-nullable reference-type fields, created via `default` or `new()`, leaves those fields at their default value — which for a reference type is `null` — with no warning, because the compiler's static analysis doesn't flag `default`/`new()` on the struct itself as a problem for its individual fields. Similarly, a newly allocated array of a non-nullable reference type (`new string[3]`) has every element `null` until explicitly assigned, and indexing into it before assignment also produces no compiler warning. Both are documented limitations of the static analysis, not runtime enforcement gaps introduced by the feature.

**Glossary:**
- **Null-state analysis** — the compiler's tracking of whether an expression is *not-null* or *maybe-null* at each point in the code.
- **Null-forgiving operator (`!`)** — overrides the compiler's null-state analysis, asserting an expression is not-null.

**Mental model:**
Distinguishes candidates who understand nullable reference types as a static-analysis/warnings system from those who believe (incorrectly) that it's a runtime null-safety guarantee like Kotlin's or Swift's type systems provide.

**TL;DR:**
Nullable reference types change zero runtime behavior — `string?` is still just `System.String` — the feature is purely compiler warnings from static null-state analysis, so `null` can still reach a "non-nullable" variable at runtime in cases the analysis can't see (structs, arrays, unannotated libraries).

**References:**
- [Nullable reference types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types)

---

### Q9. What does the null-forgiving operator (`!`) actually do, and why is overusing it considered a code smell? {#q9}

**Question:**
What does the null-forgiving operator (`!`) actually do, and why is overusing it considered a code smell?

**Good answer:**
`!`, appended to an expression, tells the compiler "treat this as not-null from here on," overriding whatever the null-state analysis concluded — it does nothing at runtime; it's purely a signal to the compiler to suppress a specific nullable warning at that point. It's appropriate when the developer genuinely has information the compiler can't infer (e.g., "I just checked this three lines up in a way the analysis doesn't trace, so I know it's safe"). The reason overusing it is a smell: every `!` is a place where the compiler's protection is turned off — if the assumption is wrong (the value actually is `null` there), you get a runtime `NullReferenceException` at that exact line with no warning to have caught it earlier, which defeats the entire point of enabling nullable reference types in the first place.

**Code example:**
```csharp
string? maybeName = LookUpName("ada"); // returns string?
int length = maybeName!.Length;         // "trust me, this isn't null"
```

**Follow-up question:**
What's a better alternative to `!` when the compiler's analysis genuinely can't follow the logic that proves a value is safe?

**Follow-up good answer:**
Prefer adding an explicit null check (`if (value is not null)`) so the analysis can track it naturally, restructuring the code so the non-null guarantee is local and visible, or — for library authors — annotating the API with nullable analysis attributes like `[NotNullWhen(true)]` or `[MemberNotNull]` so the compiler itself learns the contract (e.g., "if this method returns true, this parameter is not-null") and propagates that knowledge to every caller, rather than every caller individually reaching for `!`.

**Glossary:**
- **Null-forgiving operator (`!`)** — suppresses a nullable warning for an expression without changing runtime behavior.
- **Nullable analysis attributes** — attributes like `[NotNullWhen]`/`[MemberNotNull]` that describe an API's null contract to the compiler.

**Mental model:**
Tests whether the candidate treats `!` as a targeted escape hatch used sparingly and deliberately, versus a way to silence warnings without understanding what's actually being asserted.

**TL;DR:**
`!` only silences the compiler's nullable warning at that spot — it has zero runtime effect, so an incorrect `!` just converts a compile-time warning into a runtime `NullReferenceException` with no earlier warning to have prevented it.

**References:**
- [Nullable reference types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types)

---

### Q10. What real-world problem do nullable reference types solve, and why was this added as an opt-in, warnings-based feature instead of a breaking change to the type system? {#q10}

**Question:**
What real-world problem do nullable reference types solve, and why was this added as an opt-in, warnings-based feature instead of a breaking change to the type system?

**Good answer:**
Before nullable reference types, every reference-typed variable could always be `null`, with no way to express "this parameter/property is never allowed to be null" in the type itself — that intent lived only in comments, documentation, or defensive `ArgumentNullException` checks scattered through the codebase, and the compiler couldn't help catch violations. Nullable reference types let you encode that intent directly in the signature (`string` vs. `string?`), and get compiler warnings anywhere the code's behavior doesn't match — directly reducing the single most common cause of unhandled exceptions in C# codebases, `NullReferenceException`. It's opt-in and warnings-only (not a new runtime type or a breaking change) specifically for backward compatibility: virtually every existing C# codebase and NuGet package predates this feature, and every reference type before it was implicitly nullable — making it a hard error instead of a warning would break nearly all existing code and every unannotated dependency the moment nullable checking was turned on.

**Follow-up question:**
How does an application project handle consuming an older library that hasn't been annotated for nullable reference types?

**Follow-up good answer:**
An unannotated library's public API is treated as being in an "oblivious" nullable context from the consuming project's point of view — the compiler doesn't emit nullable warnings for types coming from or going into that API, since it has no annotation information to reason about. This means the safety nullable reference types provide only extends as far as the annotated boundary; calling into unannotated code is a place where a `null` can cross into "supposedly non-nullable" code with no warning, which is one of the known gaps in the feature's coverage, not a bug.

**Glossary:**
- **Nullable context** — the compiler mode (`enable`/`disable`/`warnings`/`annotations`) controlling whether nullable analysis and warnings are active.
- **Oblivious context** — the pre-nullable-reference-types state where a reference type carries no null annotation information.

**Mental model:**
Tests whether the candidate understands nullable reference types as a pragmatic, incrementally-adoptable static-analysis layer bolted onto an existing type system — including its honest limits at unannotated boundaries — rather than a language redesign.

**TL;DR:**
Nullable reference types let you express "this can never be null" directly in the type signature and get compile-time warnings on violations — it's warnings-only and opt-in specifically so it doesn't break the vast amount of pre-existing, unannotated C# code and libraries.

**References:**
- [Nullable reference types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types)

---

### Q11. `record class` equality compares reference-typed properties by reference, not contents. Walk through a concrete case where this surprises someone who's used to "records are like value objects." {#q11}

**Question:**
`record class` equality compares reference-typed properties by reference, not contents. Walk through a concrete case where this surprises someone who's used to "records are like value objects."

**Good answer:**
Consider `public record Person(string FirstName, string LastName, string[] PhoneNumbers)`. Two `Person` instances built from two *different* `string[]` arrays containing the exact same phone numbers are **not** equal — `person1 == person2` is `false` — because array equality is reference equality, and each `Person` was given its own array instance. This surprises people who expect records to behave like "deep value objects," when what's actually generated is *shallow*, property-by-property equality: each property is compared with the equality semantics *its own type* defines, and arrays (like most collection types) don't override `Equals` for content comparison. The practical fix is to use an immutable, content-comparable collection type (like `ImmutableArray<T>` with structural equality, or a `IReadOnlyList<T>` wrapped with a custom `Equals` override) if you actually need value semantics to extend into a collection-typed property.

**Code example:**
```csharp
public record Person(string FirstName, string LastName, string[] PhoneNumbers);

var a = new Person("Grace", "Hopper", new[] { "555-1234" });
var b = new Person("Grace", "Hopper", new[] { "555-1234" });
Console.WriteLine(a == b); // False — different array instances, same contents
```

**Follow-up question:**
If instead both `Person` instances were constructed by sharing the *same* array reference, and you mutate the array through one of them, what happens to the other?

**Follow-up good answer:**
The mutation is visible through both — since the array itself was never copied by the record (only the reference to it is stored as the property's value), both `Person` instances' `PhoneNumbers` property point at the exact same array object. This is a direct consequence of the same shallow-equality/shallow-copy behavior: records don't deep-copy or deep-compare reference-typed members, they just hold and compare the reference like any other property value.

**Glossary:**
- **Shallow equality** — comparing each property's value directly, without recursing into a reference-typed property's own contents.
- **Structural equality** — equality based on comparing the actual contents/structure of two values (what `ImmutableArray<T>` provides, unlike a plain array).

**Mental model:**
This is a real production gotcha, not a theoretical one — it tests whether the candidate has actually reasoned about what "value equality" means for a record with non-primitive properties, rather than assuming records give you free deep equality.

**TL;DR:**
Record equality is shallow — a reference-typed property like an array is compared by reference, not contents, so two records holding equal-looking-but-different array instances are not equal.

**References:**
- [C# record types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

---

### Q12. What's the practical difference in behavior between assigning a `record class` variable and assigning a `record struct` variable? {#q12}

**Question:**
What's the practical difference in behavior between assigning a `record class` variable and assigning a `record struct` variable?

**Good answer:**
`record class` is still a reference type, so `var p2 = p1;` copies the reference — both variables point at the same object, and mutating state through one is visible through the other (to whatever extent the record's properties are mutable). `record struct` is still a value type, so `var c2 = c1;` copies the entire value — the two variables are now independent, and changing a field/property on `c2` has no effect on `c1`. Both kinds get the same compiler-generated value equality, but `record struct`'s equality doesn't use reflection the way a plain struct's default `ValueType.Equals` does, so it's meaningfully faster than plain-struct equality while still being value-type-copy semantics.

**Code example:**
```csharp
var p1 = new Person("Grace", "Hopper");        // record class
var p2 = p1;
Console.WriteLine(ReferenceEquals(p1, p2));    // True — same object

var c1 = new Coordinate(47.6, -122.3);         // record struct
var c2 = c1;
c2 = c2 with { Longitude = 0.0 };
Console.WriteLine(c1.Longitude);               // Unchanged — c1 and c2 are independent
```

**Follow-up question:**
Given that trade-off, what's the concrete guidance for choosing between `record class` and `record struct`?

**Follow-up good answer:**
Use `record class` when you need inheritance (record structs can't inherit, since structs can't inherit from other types) or when instances are large enough that copying on every assignment/parameter-pass would be expensive. Use `record struct` for small, self-contained data where value-type copy semantics are actually what you want (e.g., a coordinate pair, a small immutable measurement) — you avoid a heap allocation per instance and get faster equality than a hand-written struct would have by default.

**Glossary:**
- **`record class`** — reference-type record; assignment copies the reference.
- **`record struct`** — value-type record; assignment copies the data.

**Mental model:**
Checks whether the candidate connects "record" back to the underlying class/struct semantics correctly, since it's a common misconception that all records behave like reference types.

**TL;DR:**
`record class` assignment copies a reference (shared mutable state, if any); `record struct` assignment copies the whole value (independent copies) — choose based on whether you need inheritance/large-object sharing or small independent value semantics.

**References:**
- [C# record types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

---

### Q13. What does the `required` modifier enforce, and at what point is that enforcement checked? {#q13}

**Question:**
What does the `required` modifier enforce, and at what point is that enforcement checked?

**Good answer:**
`required`, applied to a field or property, means any code that constructs a new instance of the type via an object initializer must set that member — the compiler issues an **error** (not just a warning) at the call site if a required member is left unset. This is enforced entirely at compile time, as part of overload/initializer resolution — there's no runtime check inserted. Required members must have a setter (`set` or `init`) that's at least as visible as the containing type, since the whole mechanism only works through object initializer syntax; you can still initialize a required member to `null` if its type is nullable (or get a nullable-reference-type warning if you initialize it to `null` on a non-nullable type), but you can't skip initializing it entirely — that's a compile error.

**Code example:**
```csharp
public class Person
{
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
}

var p = new Person { FirstName = "Ada" };
// Compile error: required member 'Person.LastName' must be set
```

**Follow-up question:**
How do positional records interact with `required`, since positional parameters already look mandatory via the constructor?

**Follow-up good answer:**
You can't apply `required` directly to a record's positional parameter list — the constructor already requires an argument for every positional parameter, so it would be redundant there. If a positional record also declares an explicit property that duplicates a positional parameter and marks *that* `required`, or if any properties are `required`, the compiler adds a `[SetsRequiredMembers]` attribute to the generated primary constructor, asserting to the compiler that the constructor does initialize all required members (so calling the positional constructor directly satisfies the required-member check, without needing an object initializer too).

**Glossary:**
- **`required` modifier** — forces callers using an object initializer to set the member; enforced at compile time.
- **`SetsRequiredMembers`** — an attribute asserting a constructor initializes all of a type's required members.

**Mental model:**
Tests whether the candidate knows this is purely a compile-time initializer-completeness check (much like a non-nullable constructor parameter, but for object-initializer syntax) rather than confusing it with runtime validation.

**TL;DR:**
`required` forces every object-initializer construction site to set that member, enforced as a compile-time error — it has no runtime component and doesn't apply directly to record positional parameters.

**References:**
- [required modifier | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/required)

---

### Q14. How do primary constructor parameters actually get stored, and why does that matter for a type used with dependency injection? {#q14}

**Question:**
How do primary constructor parameters actually get stored, and why does that matter for a type used with dependency injection?

**Good answer:**
Primary constructor parameters aren't automatically members of the type and aren't automatically turned into fields — they're genuinely parameters, in scope throughout the class/struct body. The compiler only generates hidden backing storage for a primary constructor parameter *if it's actually referenced somewhere in a member of the type* (a method body, a property initializer that needs it at construction time only doesn't need ongoing storage, etc.); if a parameter is only used to initialize a property/field and never referenced again, no extra storage is created for the parameter itself. This matters for a DI-style controller/service class: `public class ExampleController(IService service) : ControllerBase` — because `service` is referenced inside a method, the compiler creates a private field to hold it, giving you the same effect as manually writing a constructor and a `private readonly IService _service` field, but with far less boilerplate.

**Code example:**
```csharp
public class ExampleController(IService service) : ControllerBase
{
    [HttpGet]
    public ActionResult<Distance> Get() => service.GetDistance();
    // compiler generates a hidden field to hold `service`, since it's used here
}
```

**Follow-up question:**
What's the pitfall with primary constructor parameters in an inheritance hierarchy where a derived class also references the same parameter name after passing it to the base constructor?

**Follow-up good answer:**
If a derived class's primary constructor both forwards a parameter to the base constructor (`: BankAccount(accountID, owner)`) *and* references that same parameter later in its own members, the compiler creates a **separate** copy of storage for the derived class's parameter, distinct from the base class's property. If the base class's corresponding property can later change (e.g., via a setter), the derived class's copy of the primary constructor parameter won't reflect that change, because they're now two independent pieces of storage holding what was originally the same value. The compiler warns about this specific situation, and the fix is to reference the base class's property instead of the primary constructor parameter directly in the derived type's members.

**Glossary:**
- **Primary constructor** — a constructor declared via parameters directly on the class/struct/record declaration.
- **Captured parameter** — a primary constructor parameter the compiler gives hidden field storage to because it's used after construction.

**Mental model:**
Tests understanding that primary constructors are a "no storage unless needed" feature — a common interview trap is assuming every primary constructor parameter always becomes a field, when the compiler is actually more selective (and that selectivity has an inheritance-related sharp edge).

**TL;DR:**
The compiler only allocates hidden storage for a primary constructor parameter if it's used somewhere in the type's members after construction — and in a derived class, a parameter forwarded to and also referenced independently of the base class becomes a separate, potentially stale copy.

**References:**
- [Declare C# primary constructors | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/primary-constructors)

---

### Q15. What do collection expressions (`[1, 2, 3]`) actually compile to, and how can a custom type opt into supporting them? {#q15}

**Question:**
What do collection expressions (`[1, 2, 3]`) actually compile to, and how can a custom type opt into supporting them?

**Good answer:**
A collection expression isn't tied to one specific collection type — it's a target-typed construct that the compiler converts differently depending on what it's being assigned/passed to: an array, a `Span<T>`/`ReadOnlySpan<T>` (potentially stack-allocated), any type supporting a collection initializer (an accessible `Add` method plus `IEnumerable<T>`), or one of several standard interfaces (`IEnumerable<T>`, `IReadOnlyList<T>`, `ICollection<T>`, etc.), in which case the compiler picks a concrete backing type like `List<T>`. The compiler performs static analysis to pick the most efficient realization for the target — e.g., an empty `[]` targeting an immutable/never-modified collection can become `Array.Empty<T>()` with no allocation at all. For a custom type that doesn't fit any of those built-in conversions, you opt in by writing a `static Create(ReadOnlySpan<T> values)` method and applying `[CollectionBuilder(typeof(BuilderType), "Create")]` to the collection type — the compiler then calls that method with the collection expression's elements packed into a span.

**Code example:**
```csharp
string[] vowels = ["a", "e", "i", "o", "u"];
string[] consonants = ["b", "c", "d"];
string[] combined = [.. vowels, .. consonants, "y"]; // spread element inlines both arrays

[CollectionBuilder(typeof(LineBufferBuilder), "Create")]
public class LineBuffer : IEnumerable<char> { /* ... */ }
```

**Follow-up question:**
Collection expressions always fully materialize their elements, unlike LINQ. Why does that distinction matter, and what's a concrete case where it bites someone who assumes the two behave the same?

**Follow-up good answer:**
A collection expression always evaluates eagerly and produces a real, fully-populated collection containing every specified element, regardless of the target type — even if the target is `IEnumerable<T>`, which for a LINQ query would normally imply lazy, deferred enumeration. This means you can't use a collection expression to build an infinite or lazily-generated sequence the way you could with a hand-written `yield return` iterator or `Enumerable.Range` — someone assuming `IEnumerable<int> naturals = [1, 2, 3, ...]`-style laziness would be surprised that a collection expression targeting `IEnumerable<T>` still allocates and populates a concrete backing collection immediately.

**Glossary:**
- **Collection expression** — the `[...]` literal syntax, target-typed to many collection kinds.
- **Spread element (`..`)** — inlines the elements of an enumerable collection expression source into the surrounding collection expression.
- **Collection builder** — the `Create`-method + `[CollectionBuilder]` mechanism letting custom types support collection expressions.

**Mental model:**
Tests whether the candidate understands collection expressions as a compiler-driven, target-typed, eager-materialization feature distinct from LINQ's laziness — an easy but consequential mix-up given how similar `[...]` looks to LINQ query results in code.

**TL;DR:**
Collection expressions are target-typed and eagerly materialize a real collection (never lazily, unlike LINQ) — custom types opt in via a static `Create(ReadOnlySpan<T>)` method plus `[CollectionBuilder]`.

**References:**
- [Collection expressions | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions)

---

### Q16. What does an `init`-only property accessor enable that a `get`-only auto-property or a `private set` doesn't? {#q16}

**Question:**
What does an `init`-only property accessor enable that a `get`-only auto-property or a `private set` doesn't?

**Good answer:**
An `init` accessor allows the property to be assigned only during object construction — either from within the type's own constructors, or from the *caller's* object initializer (`new Person { FirstName = "Ada" }`). A `get`-only auto-property (no `set`/`init` at all) can only be assigned from inside the type's constructors — a caller can't use an object initializer for it at all, forcing every combination of properties to go through a constructor overload. A `private set` allows the type itself to mutate the property after construction (from any instance method), which `init` doesn't allow — once construction finishes, an `init`-only property is immutable from every angle, including the type's own code. So `init` sits specifically between those two: caller-friendly like a mutable property (object-initializer syntax works), but truly immutable after construction like a `get`-only property (no post-construction mutation from anywhere, including internally).

**Code example:**
```csharp
class PersonInit
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
}
var p = new PersonInit { FirstName = "Bill", LastName = "Gates" }; // works
// p.FirstName = "Other"; // compile error — init-only after construction
```

**Follow-up question:**
Does `init` by itself force a caller to actually set the property, the way `required` does?

**Follow-up good answer:**
No — `init` only controls *when* a property can be assigned (during construction) and *how* (object initializer or constructor), not *whether* it must be assigned. Without `required`, a caller can simply omit the property from the object initializer, and it's left at its type's default value (`null` for a reference type, absent any warning unless nullable reference types are enabled and the property is non-nullable). Combining `required` with `init` is what forces both "must be set" and "can never be changed after construction."

**Glossary:**
- **`init` accessor** — a property setter callable only during construction (constructor body or caller's object initializer).
- **Object initializer** — the `new T { Prop = value, ... }` syntax for setting properties right after construction.

**Mental model:**
Checks whether the candidate can precisely place `init` on the spectrum between "freely mutable," "construction-time only via constructor," and "immutable everywhere, even internally" — a distinction people often gloss over as "basically readonly."

**TL;DR:**
`init` allows assignment only during construction (including via the caller's object initializer) and never afterward, even from inside the type itself — it doesn't by itself require the caller to set the property; pair it with `required` for that.

**References:**
- [The init keyword | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/init)

---

### Q17. What are C# source generators, and how do they compare to reflection as a way to generate behavior based on a type's shape? {#q17}

**Question:**
What are C# source generators, and how do they compare to reflection as a way to generate behavior based on a type's shape?

**Good answer:**
A source generator is a component that plugs into the compiler (via the Roslyn compiler platform APIs) and runs *during compilation*, reading the actual syntax/semantic model of the code being compiled (plus any additional files) and emitting new C# source files that get compiled alongside the rest of the program — this is what the docs call "compile-time metaprogramming." Reflection, by contrast, inspects a type's metadata *at runtime*, discovering its members/attributes and acting on them dynamically (e.g., building a serializer's behavior by reflecting over properties every time it runs, or the first time and caching). The practical trade-off: source-generated code is ordinary, ahead-of-time-compiled C# — it can be inspected/debugged like any other generated file, has no runtime reflection cost, and is compatible with trimming/Native AOT scenarios where reflecting over arbitrary types is restricted or slow; reflection-based approaches pay a runtime cost (often mitigated with caching) but don't require a build-time step and can react to types not known until runtime.

**Follow-up question:**
Why would a library like a JSON serializer specifically want a source-generator-based mode in addition to (or instead of) its reflection-based mode?

**Follow-up good answer:**
A source-generator-based serializer emits the actual (de)serialization code for each type at compile time — direct property gets/sets, no runtime type inspection — which is both faster at runtime (no reflection overhead, no per-type metadata caching needed on first use) and compatible with ahead-of-time compilation/trimming, where reflecting over a type that the trimmer couldn't statically prove is "used reflectively" can throw at runtime because the metadata was stripped. The reflection-based mode remains valuable for scenarios needing full runtime dynamism (types not known until runtime, plugin-style extensibility) where paying the reflection cost is an acceptable trade for that flexibility.

**Glossary:**
- **Source generator** — a compiler component that emits additional C# source during compilation, based on the existing compilation's syntax/semantics.
- **Reflection** — runtime inspection of type metadata (members, attributes) to drive dynamic behavior.

**Mental model:**
Tests whether the candidate can articulate the compile-time-vs-runtime trade-off precisely (performance, AOT/trimming compatibility, debuggability of generated code) rather than vaguely saying "source generators are faster."

**TL;DR:**
Source generators emit real C# code at compile time based on inspecting the compilation, avoiding the runtime cost (and AOT/trimming friction) of reflecting over types dynamically at the price of needing to know the shape at build time.

**References:**
- [The .NET Compiler Platform SDK (Roslyn APIs) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/)

---

### Q18. Is there a performance difference between LINQ query syntax (`from x in ... where ... select ...`) and method syntax (`.Where(...).Select(...)`)? {#q18}

**Question:**
Is there a performance difference between LINQ query syntax (`from x in ... where ... select ...`) and method syntax (`.Where(...).Select(...)`)?

**Good answer:**
No — the C# compiler translates query syntax directly into the equivalent chain of method calls (`Where`, `Select`, `OrderBy`, etc., the same standard query operator extension methods method syntax calls explicitly). They're semantically identical and compile to the same IL; there is no runtime cost difference between writing `from n in numbers where n % 2 == 0 select n` and writing `numbers.Where(n => n % 2 == 0)`. The practical difference is purely about readability and expressiveness: query syntax is often clearer for multi-clause queries involving `join`/`group by`/`orderby`, but some operations — anything returning a single scalar, like `Count()`, `Sum()`, `Max()` — have no query-syntax clause at all and must be written as a method call, sometimes mixed with query syntax by wrapping the query expression in parentheses and calling the method on the result.

**Code example:**
```csharp
// Semantically and, after compilation, literally identical:
var q1 = from n in numbers where n % 2 == 0 orderby n select n;
var q2 = numbers.Where(n => n % 2 == 0).OrderBy(n => n);
```

**Follow-up question:**
Why might a hand-written `foreach` loop still outperform an equivalent LINQ chain, even though LINQ compiles down to method calls that are themselves implemented with iteration?

**Follow-up good answer:**
LINQ's generality has real overhead compared to a loop written for one specific case: each `Where`/`Select` in a chain introduces its own iterator object (and its own `MoveNext` state machine) that the runtime has to advance and check for every element, there's often a delegate invocation per element per operator (which can't always be inlined the way a direct loop body can), and value-type elements can incur extra overhead through generic iterator plumbing. None of this makes LINQ "slow" in absolute terms for most application code — it's a readability/expressiveness trade against a small, usually-irrelevant constant-factor overhead — but in a genuinely hot path (a tight loop executed millions of times), a hand-rolled loop that avoids the extra iterator/delegate indirection can measurably outperform an equivalent LINQ chain, which is why performance-critical inner loops are one of the few places LINQ is often deliberately avoided.

**Glossary:**
- **Query syntax** — the declarative `from`/`where`/`select` LINQ expression form.
- **Method syntax** — LINQ expressed as chained extension method calls with lambda arguments.

**Mental model:**
Checks whether the candidate knows this is a non-issue at the query-vs-method-syntax level (a common but wrong guess is "one is faster"), while still being able to reason about LINQ's real (different) performance trade-off against manual iteration.

**TL;DR:**
Query syntax and method syntax compile to the same method calls with zero performance difference between them — the real (separate) performance trade-off is LINQ's per-operator iterator/delegate overhead versus a hand-written loop, which only matters in genuinely hot paths.

**References:**
- [Write LINQ queries | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/write-linq-queries)

---

### Q19. You're designing a set of DTOs for an API layer. When would you reach for a `record`, when for a plain `class`, and when for a `struct`? {#q19}

**Question:**
You're designing a set of DTOs for an API layer. When would you reach for a `record`, when for a plain `class`, and when for a `struct`?

**Good answer:**
For DTOs — types whose whole purpose is carrying data, where two instances with the same values should be considered interchangeable, and immutability is desirable (you don't want a shared DTO instance quietly mutated by something deep in a call chain) — `record` (specifically `record class`, since most DTOs are reference-sized/potentially large or nested) is the natural default: it gives you value equality, a debuggable `ToString`, and `with`-based copying essentially for free, all things you'd otherwise hand-write. Reach for a plain `class` when identity matters more than data — e.g., an Entity Framework entity, where you specifically want reference-equality-based tracking (the records docs explicitly warn against using records for EF Core entities for this reason), or when the type has real behavior/mutable internal state beyond simple data-holding. Reach for `struct` (or `record struct`) only for small, short-lived, genuinely value-like data (a coordinate, a small money/quantity pair) where avoiding a heap allocation matters and copy-by-value semantics are actually desired — a struct-per-request-DTO across a large object graph can backfire by copying more data than a reference would have, so it's the exception, not the default, for DTOs of any real size.

**Follow-up question:**
Why does the official guidance specifically warn against using records for Entity Framework Core entity types?

**Follow-up good answer:**
EF Core's change tracking relies on reference equality/identity to know whether it's looking at the same tracked entity instance across operations — its internal bookkeeping is built around "is this the same object I'm already tracking," not "does this object have the same property values as one I'm tracking." A record's compiler-generated value equality overrides `Equals`/`GetHashCode` to compare by data instead of identity, which conflicts with how EF Core's tracking mechanisms (and things like dictionary/set-based lookups keyed on tracked entities) are designed to work, and can produce subtle tracking bugs — two different rows that happen to have identical column values could be treated as "the same entity" in code that relies on `Equals`.

**Glossary:**
- **DTO (Data Transfer Object)** — a type whose purpose is carrying data between layers/processes, with no meaningful behavior of its own.
- **Change tracking** — EF Core's mechanism for detecting which tracked entities have been modified since being loaded.

**Mental model:**
Tests whether the candidate can apply "when to use records" as a genuine design decision tied to identity-vs-value semantics, rather than reflexively using records everywhere just because they're new and concise.

**TL;DR:**
Default to `record` (class) for data-only DTOs needing value equality and immutability; use a plain `class` when identity matters (notably EF Core entities, which rely on reference-equality-based tracking); reserve `struct`/`record struct` for genuinely small, short-lived value-like data.

**References:**
- [C# record types | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

---

### Q20. In general software engineering terms, what problem do property/type patterns and exhaustive `switch` expressions solve that's traditionally associated with the "expression problem" in OOP design? {#q20}

**Question:**
In general software engineering terms, what problem do property/type patterns and exhaustive `switch` expressions solve that's traditionally associated with the "expression problem" in OOP design?

**Good answer:**
The expression problem describes a tension in typed OOP/functional design: with classic virtual-dispatch polymorphism, adding a new *type* (a new subclass implementing a virtual method) is easy and safe — the compiler doesn't need you to touch existing code — but adding a new *operation* over an existing closed set of types means either editing every existing class to add a new virtual method, or writing an external `if`/type-check chain that the compiler won't verify is exhaustive. Pattern matching with `switch` expressions is exactly that second style — operations live outside the type hierarchy as functions that pattern-match over the possible shapes — but the compiler's exhaustiveness checking (warning when a `switch` expression doesn't cover every case) gives back some of the safety that was previously virtual dispatch's exclusive advantage: if you add a new case to what you're matching on and forget to handle it somewhere, the compiler flags the gap instead of silently falling through to a default or throwing at runtime.

**Follow-up question:**
Given that trade-off, when would you actually prefer pattern-matching-based dispatch over adding a new virtual method to an existing class hierarchy?

**Follow-up good answer:**
Prefer pattern matching when you expect the *set of operations* to grow more often than the *set of types* — e.g., a fixed, stable domain model (a small `enum`-like hierarchy of order states, message kinds) that many different, independently-evolving parts of the codebase need to compute different things over; adding each new computation as an external `switch`-based function avoids having to modify the core type hierarchy (and its unrelated existing operations) every time. Prefer virtual dispatch/adding a subclass when the set of *types* grows more often (e.g., a plugin-style system where third parties add new implementations) — the type hierarchy approach lets each new type slot in without touching a shared, growing pattern-matching function that every existing case lives inside of.

**Glossary:**
- **Expression problem** — the difficulty of extending both the set of types and the set of operations over them without modifying existing code, in a typed language.
- **Exhaustiveness checking** — the compiler verifying that a `switch` (expression or statement) handles every possible case of what it matches on.

**Mental model:**
This is a "does the candidate think about design trade-offs, not just syntax" question — testing whether they can connect a concrete C# feature (exhaustive pattern matching) to the underlying, language-agnostic design tension it partially addresses.

**TL;DR:**
Pattern matching moves operations outside the type hierarchy (easy to add new operations, harder to add new types) — the opposite trade-off from virtual dispatch — and the compiler's exhaustiveness checking on `switch` expressions is what makes that safe against forgetting a case.

**References:**
- [Pattern matching overview | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=dotnet&tags=csharp-language-features-and-linq-internals&autostart=1" | relative_url }})
