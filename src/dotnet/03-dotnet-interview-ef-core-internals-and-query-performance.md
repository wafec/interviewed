---
layout: default
title: ".NET Interview — EF Core Internals & Query Performance"
---

# .NET Interview — EF Core Internals & Query Performance

This set covers Entity Framework Core's change tracker, query translation, eager/lazy loading, and the performance pitfalls (N+1, cartesian explosion, unnecessary tracking) that separate developers who can use EF Core from those who understand what it's actually doing.

### Q1. What does EF Core's change tracker actually do, and what states can a tracked entity be in? {#q1}

**Question:**
What does EF Core's change tracker actually do, and what states can a tracked entity be in?

**Good answer:**
Every `DbContext` instance owns a change tracker that keeps a record of every entity instance it knows about, so it can figure out what needs to happen to the database when `SaveChanges` is called. Entities become tracked when they're returned from a query, or explicitly attached via `Add`/`Attach`/`Update`, or discovered connected to an already-tracked entity. Each tracked entity has one of five `EntityState` values: `Detached` (not tracked at all), `Added` (new, not yet in the database — will INSERT), `Unchanged` (matches what's in the database — all query results start here), `Modified` (at least one property changed since it was queried — will UPDATE), and `Deleted` (marked for removal — will DELETE). Tracking happens at the property level, so if you change one field on an entity, only that field goes into the generated UPDATE statement.

**Code example:**
```csharp
var blog = await context.Blogs.SingleAsync(b => b.Id == 1); // state: Unchanged
blog.Name = "New name";                                     // state: Modified
context.ChangeTracker.DetectChanges();
Console.WriteLine(context.ChangeTracker.DebugView.LongView); // shows Name Modified, others untouched
```

**Follow-up question:**
A `DbContext` is described as representing a "short-lived unit of work." What does that mean in practice, and why does it matter?

**Follow-up good answer:**
It means a `DbContext` instance is meant to live for exactly one business operation: create it, track some entities (via query or `Add`/`Attach`), make changes, call `SaveChanges`, then dispose it — not keep one instance alive for the whole application or a whole user session. This is literally Martin Fowler's Unit of Work pattern: "a Unit of Work keeps track of everything you do during a business transaction that can affect the database... when you're done, it figures out everything that needs to be done to alter the database." It matters because a long-lived context accumulates tracked entities indefinitely (memory bloat and slower `DetectChanges` calls since more entities must be scanned), can go into an unrecoverable state after certain exceptions, and — since `DbContext` isn't thread-safe — a long-lived instance is far more likely to get used concurrently by mistake. This is exactly why ASP.NET Core's `AddDbContext` registers it as a *scoped* service by default: one instance per HTTP request, matching the natural unit-of-work boundary.

**Glossary:**
- **Change tracker** — the `DbContext` subsystem that records entity states and detects modifications so `SaveChanges` knows what SQL to generate.
- **EntityState** — the enum (`Detached`/`Added`/`Unchanged`/`Modified`/`Deleted`) describing a tracked entity's relationship to the database.
- **Unit of Work** — a pattern where one object tracks all changes for a business transaction and commits them together.

**Mental model:**
This tests whether the candidate sees `DbContext` as a stateful, short-lived transaction boundary rather than just "the thing you query through" — a very common source of production bugs is treating it like a long-lived service.

**TL;DR:**
The change tracker records every entity's `EntityState` (Detached/Added/Unchanged/Modified/Deleted) so `SaveChanges` knows exactly what to INSERT/UPDATE/DELETE, and it's designed to live for one short unit of work, not the app's lifetime.

**References:**
- [Change Tracking - EF Core](https://learn.microsoft.com/en-us/ef/core/change-tracking/)
- [DbContext Lifetime, Configuration, and Initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [Martin Fowler — Unit of Work](https://www.martinfowler.com/eaaCatalog/unitOfWork.html)

---

### Q2. What's the difference between a tracking query and a no-tracking query, and when should you use each? {#q2}

**Question:**
What's the difference between a tracking query and a no-tracking query, and when should you use each?

**Good answer:**
By default, any LINQ query that returns entity types is a *tracking* query: EF Core sets up change-tracking information for every returned entity so that later modifications can be detected and persisted by `SaveChanges`. A *no-tracking* query, requested via `.AsNoTracking()`, skips all of that setup — it's read-only, generally faster, and returns fresh instances every time regardless of what's already in the context. You should reach for `AsNoTracking()` whenever the results are read-only (an API GET endpoint, a report, a dropdown list) and you have no intention of calling `SaveChanges` against them. Interestingly, tracking queries aren't always slower: if the entity is already in the change tracker, EF Core just returns that same instance instead of re-materializing it, which can use less memory and be faster than a no-tracking query for that case.

**Code example:**
```csharp
var readOnlyBlogs = await context.Blogs.AsNoTracking().ToListAsync();
```

**Follow-up question:**
What is "identity resolution," and how does `AsNoTrackingWithIdentityResolution()` differ from plain `AsNoTracking()`?

**Follow-up good answer:**
Identity resolution means that if the same logical entity (same key) appears more than once in a result set, EF Core returns the *same* .NET instance for every occurrence rather than a fresh object each time — this is a natural side effect of tracking queries, since they consult the change tracker for existing instances. Plain no-tracking queries don't do this: each occurrence of the same entity in the results gets its own separate instance, which can be a problem for object graphs (e.g. the same `Author` appearing on ten different `Post` results should arguably be one object). `AsNoTrackingWithIdentityResolution()` gives you the best of both: still no persistent tracking by the `DbContext` (so no `SaveChanges` support, no memory buildup across the context's lifetime), but a temporary, stand-alone change tracker used only during materialization of that one query, so duplicate entities in the same result set collapse into a single instance. That stand-alone tracker is discarded once the query is done.

**Glossary:**
- **AsNoTracking()** — a query operator that disables change tracking for that query's results.
- **Identity resolution** — returning the same object instance for repeated occurrences of the same entity (by key) within a result.

**Mental model:**
Probes whether the candidate has actually measured/reasoned about tracking overhead rather than reflexively slapping `AsNoTracking()` on everything — the "tracking can be faster on cache hits" nuance separates real experience from cargo-culting a performance tip.

**TL;DR:**
Tracking queries set up change-detection and enable `SaveChanges`; no-tracking queries skip that for faster, read-only access — use `AsNoTrackingWithIdentityResolution()` when you need read-only *and* de-duplicated object identity across a result set.

**References:**
- [Tracking vs. No-Tracking Queries - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/tracking)

---

### Q3. How does EF Core's change tracker actually detect that a property changed, given that entity classes are usually plain C# properties with no notification events? {#q3}

**Question:**
How does EF Core's change tracker actually detect that a property changed, given that entity classes are usually plain C# properties with no notification events?

**Good answer:**
By default EF Core uses *snapshot-based* change tracking: when an entity is first tracked (typically when a query materializes it), EF Core takes a snapshot of every property's current value. Later, when something needs to inspect the entity's state — most commonly `SaveChanges`, which internally calls `ChangeTracker.DetectChanges()` — EF Core walks every tracked entity and compares its current property values against the stored snapshot. Any property whose current value differs from the snapshot gets marked as modified, which is also what flips the entity's overall state to `Modified`. This is why you can just set `blog.Name = "x"` on a plain POCO and have EF Core notice it later, with no `INotifyPropertyChanged` boilerplate required — the cost is that `DetectChanges` has to scan every tracked entity's properties, which is O(number of tracked entities × properties) and can matter with a very large change tracker.

**Follow-up question:**
EF Core also supports change-tracking proxies as an alternative. What are they, and why aren't they the default?

**Follow-up good answer:**
Change-tracking proxies (`UseChangeTrackingProxies`, from the `Microsoft.EntityFrameworkCore.Proxies` package) generate a runtime subclass of your entity that implements `INotifyPropertyChanged`/`INotifyPropertyChanging`, so the change tracker is notified the instant a property is set rather than having to scan for differences later — similar in spirit to how lazy-loading proxies intercept navigation property access. They aren't the default because they come with real constraints: every property that should be tracked this way must be `virtual` (so the proxy can override it), the entity class can't be `sealed`, and you take on a dependency on an extra package plus proxy-generation overhead when the context is first used. Snapshot-based tracking works with plain, unmodified POCOs and no extra package, which is why it's the sensible default and proxies are an opt-in for scenarios with very large change trackers where avoiding `DetectChanges` scans is worth the trade-off.

**Glossary:**
- **Snapshot-based change tracking** — comparing an entity's current property values against a stored snapshot to find modifications.
- **DetectChanges()** — the change tracker method that performs this snapshot comparison across all tracked entities.
- **Change-tracking proxies** — runtime-generated subclasses that notify the change tracker immediately via `INotifyPropertyChanged`, avoiding the need for `DetectChanges` scans.

**Mental model:**
Tests whether the candidate understands change tracking as a real algorithm with a real cost, not "magic" — this is the foundation for reasoning about why a context with thousands of tracked entities gets slow.

**TL;DR:**
EF Core snapshots each tracked entity's properties at query time and later diffs the live object against that snapshot (during `DetectChanges`, called internally by `SaveChanges`) to figure out what changed — no notification interfaces required, at the cost of an O(tracked entities) scan.

**References:**
- [Change Tracking - EF Core](https://learn.microsoft.com/en-us/ef/core/change-tracking/)
- [DbContext Lifetime, Configuration, and Initialization — UseChangeTrackingProxies](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)

---

### Q4. How does `Include()` actually load related data, and what SQL does it generate for a chain of `Include`/`ThenInclude` calls? {#q4}

**Question:**
How does `Include()` actually load related data, and what SQL does it generate for a chain of `Include`/`ThenInclude` calls?

**Good answer:**
`Include(blog => blog.Posts)` tells EF Core to eagerly load the `Posts` navigation as part of the same query, which it does by translating the whole thing into a single SQL query with `JOIN`s — a `LEFT JOIN` from `Blogs` to `Posts` in this case, so blogs with no posts are still returned. `ThenInclude` lets you drill down further through a relationship (e.g. `.Include(b => b.Posts).ThenInclude(p => p.Author)` also loads each post's author), and EF Core combines multiple include paths that share a common prefix into the same joined query rather than generating redundant joins. You can also chain multiple independent `Include` calls (e.g. `Posts` and `Owner`) in one query, and EF Core will automatically fix up navigation properties to any other entities already loaded into the context, even ones you didn't explicitly include.

**Code example:**
```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ThenInclude(p => p.Author)
    .Include(b => b.Owner)
    .ToListAsync();
```

**Follow-up question:**
What happens to the generated SQL if you `Include` two separate collection navigations on the same entity — say, both `Posts` and `Contributors` on `Blog` — and why is that a performance concern?

**Follow-up good answer:**
Because both `Posts` and `Contributors` are collection navigations at the same level, joining both into a single query produces a cross product: every row from `Posts` gets paired with every row from `Contributors` for that blog. If a blog has 10 posts and 10 contributors, the database returns 100 rows for that one blog instead of 20 — this is the "cartesian explosion" problem, and it scales multiplicatively as you add more sibling collection includes, potentially transferring huge amounts of duplicated data. It does *not* happen when the includes are nested rather than siblings (e.g. `Posts.ThenInclude(Comments)`), because then each comment naturally corresponds to exactly one row, with no cross-multiplication. The fix is `AsSplitQuery()`, which tells EF Core to issue one SQL query per collection navigation instead of joining them all together, avoiding the cross product at the cost of extra round trips.

**Glossary:**
- **Eager loading** — loading related entities as part of the original query, via `Include`/`ThenInclude`.
- **Cartesian explosion** — the multiplicative row blow-up from joining multiple sibling collection navigations in one query.

**Mental model:**
Checks whether the candidate can predict the SQL shape a LINQ query produces — a core skill for anyone debugging an EF Core performance problem, since you can't diagnose bad SQL you can't picture.

**TL;DR:**
`Include`/`ThenInclude` translate to SQL JOINs in a single query by default; joining two *sibling* collection navigations causes a cartesian explosion (cross-product row blow-up), while nested includes don't.

**References:**
- [Eager Loading of Related Data - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/related-data/eager)
- [Single vs. Split Queries - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)

---

### Q5. What is `AsSplitQuery()`, and what are its trade-offs compared to the default single-query behavior? {#q5}

**Question:**
What is `AsSplitQuery()`, and what are its trade-offs compared to the default single-query behavior?

**Good answer:**
`AsSplitQuery()` tells EF Core to translate a query with `Include`d collection navigations into multiple separate SQL queries — one per collection navigation — instead of one big query with JOINs. For `context.Blogs.Include(b => b.Posts).AsSplitQuery()`, EF Core issues one query for the blogs and a second query (joined back to blogs by key) for the posts, avoiding both the cartesian explosion from sibling collections and the data duplication of principal-table columns being repeated once per joined row. The trade-off is real, though: single queries guarantee data consistency (most databases execute a single query atomically against one point-in-time snapshot), whereas split queries make multiple round trips, so if data changes between them the results can be inconsistent unless you wrap them in an explicit transaction. Split queries also mean an extra network round trip per collection (costly under high latency), and results from earlier queries must be buffered in memory since most databases can't interleave results from multiple concurrently open queries.

**Code example:**
```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .AsSplitQuery()
    .ToListAsync();
```

**Follow-up question:**
If you configure split queries as the global default for a context, can you still force a specific query to use the single-query behavior — and why might you want to?

**Follow-up good answer:**
Yes — `AsSingleQuery()` overrides the global default for one specific query, the same way `AsSplitQuery()` overrides a single-query default. You'd want this for a query where consistency matters more than avoiding the JOIN cost (e.g. you need the loaded graph to be a true point-in-time snapshot, and the collection sizes are small enough that cartesian explosion isn't a real concern), or simply where the extra round trips of a split query would hurt more than the JOIN overhead — for instance, a low-latency local database versus a high-latency cloud connection. EF Core actually emits a warning by default whenever it detects a query loading multiple collections without either global configuration or an explicit `AsSingleQuery`/`AsSplitQuery` call, precisely because this decision has real performance implications and shouldn't be left to an implicit default.

**Glossary:**
- **AsSplitQuery() / AsSingleQuery()** — operators that force split or single-query translation for a given LINQ query, overriding any global default.

**Mental model:**
Tests whether the candidate treats "should this be one query or several" as a real engineering trade-off (consistency vs. round trips vs. row bloat) rather than a checkbox to flip whenever N+1-adjacent symptoms appear.

**TL;DR:**
`AsSplitQuery()` trades the single-query JOIN's consistency and one-round-trip guarantee for avoiding cartesian explosion and duplicated columns — pick per-query based on which cost hurts more for that specific data shape and latency profile.

**References:**
- [Single vs. Split Queries - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)

---

### Q6. How does lazy loading work in EF Core, and what has to be true of your entity classes for it to work via proxies? {#q6}

**Question:**
How does lazy loading work in EF Core, and what has to be true of your entity classes for it to work via proxies?

**Good answer:**
Lazy loading defers fetching a related entity or collection until the navigation property is actually accessed, rather than loading it eagerly or requiring an explicit `Load()` call. The simplest way to enable it is installing the `Microsoft.EntityFrameworkCore.Proxies` package and calling `UseLazyLoadingProxies()`; EF Core then generates a runtime proxy subclass of each entity, and any navigation property that's `virtual` (and on a non-sealed class) gets overridden so that reading it triggers a database query behind the scenes if the data isn't already loaded. There's also a proxy-free approach where you inject an `ILazyLoader` (or a lazy-loading delegate) into the entity's constructor and call `Load()` explicitly inside the property getter — more boilerplate, but it works with plain classes that don't need to be inheritable and doesn't require `virtual` navigation properties.

**Code example:**
```csharp
public class Blog
{
    public int Id { get; set; }
    public virtual ICollection<Post> Posts { get; set; } // virtual required for proxy lazy-loading
}
```

**Follow-up question:**
Why is lazy loading specifically called out as a common cause of the N+1 query problem, and what would you actually see happen if you looped over blogs and accessed `blog.Posts` for each one?

**Follow-up good answer:**
Because each access to a not-yet-loaded navigation property fires its own database round trip, completely invisibly from the calling code's perspective — there's no visual cue in `foreach (var blog in blogs) { var count = blog.Posts.Count; }` that each `blog.Posts` access is a separate query. If `blogs` came from one query returning 100 rows, that loop silently issues 100 additional queries (one per blog) on top of the original — the classic 1+N pattern. The fix is to not rely on lazy loading for anything in a hot path: use eager loading (`Include`) when you know upfront which related data you need, or explicit loading with a batched query, so the number of round trips stays constant regardless of how many rows the outer query returns.

**Glossary:**
- **Lazy loading** — deferring the load of a related entity/collection until its navigation property is accessed.
- **N+1 problem** — one query for the initial result set plus one additional query per row for related data, instead of a constant number of queries.

**Mental model:**
This is the single most common EF Core performance bug in real applications — testing whether the candidate can spot it from code that looks completely innocent.

**TL;DR:**
Lazy loading (proxies over `virtual` navigation properties, or an injected `ILazyLoader`) fetches related data on first access — convenient, but looping over a collection and touching a lazy navigation per item silently fires one query per item, the classic N+1.

**References:**
- [Lazy Loading of Related Data - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/related-data/lazy)

---

### Q7. EF Core caches compiled queries by "query tree shape." What does that mean, and why does parameterizing a query matter for this? {#q7}

**Question:**
EF Core caches compiled queries by "query tree shape." What does that mean, and why does parameterizing a query matter for this?

**Good answer:**
Turning a LINQ expression tree into SQL is expensive, so EF Core caches the result of that translation keyed by the *shape* of the expression tree — two queries with structurally identical trees reuse the same cached translation even if the actual values differ. The catch is that a captured local variable becomes a SQL parameter (so `p => p.Title == someVariable` produces the same tree shape no matter what `someVariable` holds), but an inline constant becomes part of the tree itself: `p => p.Title == "post1"` and `p => p.Title == "post2"` are *different* tree shapes because the literal is baked into each expression, so each gets compiled and cached separately, and the database itself may need a distinct query plan for each. Parameterizing (using a variable instead of a literal) keeps the tree shape stable across calls, so EF Core only compiles once, and the generated SQL uses a `@parameter` the database can plan once and reuse.

**Code example:**
```csharp
// Two distinct cached translations, one per literal:
await context.Posts.FirstOrDefaultAsync(p => p.Title == "post1");
await context.Posts.FirstOrDefaultAsync(p => p.Title == "post2");

// One cached translation, reused for any title:
var postTitle = "post1";
await context.Posts.FirstOrDefaultAsync(p => p.Title == postTitle);
```

**Follow-up question:**
What are explicitly compiled queries (`EF.CompileQuery`/`EF.CompileAsyncQuery`), and when is the extra step of using them actually worth it?

**Follow-up good answer:**
`EF.CompileAsyncQuery` (or the sync `EF.CompileQuery`) lets you explicitly compile a LINQ query into a reusable delegate ahead of time, so calling it later skips even the cache-lookup step — comparing the incoming expression tree against cached shapes to find a match — and goes straight to execution. Microsoft's own benchmarks show a modest but real difference (roughly 550-650μs vs. 650-700μs per call in one comparison), so it mainly pays off for very hot, frequently-executed queries in high-throughput or latency-sensitive services, not for the average CRUD query where network and database I/O dominate. It comes with real constraints too: it only works against a single EF Core model, and parameters must be simple scalars — no complex member/method-access expressions — which is why it's positioned as an advanced, targeted optimization rather than a default habit.

**Glossary:**
- **Query tree shape** — the structural form of a LINQ expression tree, used as the cache key for EF Core's compiled-query cache.
- **Compiled query** — a LINQ query explicitly pre-compiled into a delegate via `EF.CompileQuery`/`EF.CompileAsyncQuery`, bypassing the cache-lookup step entirely.

**Mental model:**
Tests understanding of a subtle but real perf trap: writing "the same" query with inline literals instead of variables silently defeats EF Core's caching and can pollute the database's own plan cache too.

**TL;DR:**
EF Core caches query translation by expression-tree shape; use variables (not inline literals) so repeated queries share one cached shape and one parameterized SQL plan, and reach for explicit `EF.CompileQuery` only for genuinely hot queries where skipping the cache lookup itself is worth the added rigidity.

**References:**
- [Advanced Performance Topics - EF Core](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)

---

### Q8. How would you actually detect that an endpoint has an N+1 query problem, and what's your process for confirming a fix worked? {#q8}

**Question:**
How would you actually detect that an endpoint has an N+1 query problem, and what's your process for confirming a fix worked?

**Good answer:**
The most direct approach is to turn on EF Core's SQL logging — `optionsBuilder.LogTo(Console.WriteLine)`, or a structured logger in production — and look at how many `SELECT` statements a single request actually generates; seeing one query per item in a loop (instead of one query total) is the signature of N+1. `EnableSensitiveDataLogging()` during local debugging shows the actual parameter values inline, which makes it easier to confirm the queries are indeed one-per-row rather than a coincidence. Beyond logging, `ToQueryString()` on an `IQueryable` lets you inspect the generated SQL for a specific query without executing it, and EF Core's metrics (query cache hit rate) can hint at problems too. Once you suspect N+1, the fix is almost always eager loading (`Include`) or a projection that pulls related data into the original query; to confirm the fix, re-run with logging on and verify the query count for that endpoint dropped to a constant number regardless of how many rows the outer result has — not just "fewer queries," but a count independent of N.

**Follow-up question:**
Why is "fewer queries" not always the right goal — can eager loading itself introduce a *worse* performance problem than the N+1 it fixes?

**Follow-up good answer:**
Yes — if you naively `Include` several sibling collection navigations to avoid N+1, you can trade a moderate number of small queries for a single query that returns a cartesian-exploded, massively duplicated result set, which can transfer more total data and be slower overall than the N+1 it replaced. The right fix depends on the actual shape: a single collection include is usually a clear win; multiple sibling collections often call for `AsSplitQuery()` instead of jamming everything into one joined query; and if you don't need full entities at all, a `Select` projection into a DTO avoids loading (and duplicating) columns you don't need in the first place. "Fewer round trips" and "less data transferred" are two different axes, and a good fix optimizes for whichever one is actually the bottleneck for that specific query.

**Glossary:**
- **LogTo / EnableSensitiveDataLogging** — `DbContextOptionsBuilder` methods for observing generated SQL and parameter values.
- **ToQueryString()** — an `IQueryable` extension that returns the SQL a query would generate, without executing it.

**Mental model:**
This is exactly the "how do you detect a performance problem, which tools, and how do you validate the fix" pattern interviewers are trained to probe for — a candidate who only knows "use Include" without a detection/validation loop hasn't actually done this in production.

**TL;DR:**
Detect N+1 by counting generated SQL statements per request via `LogTo`/`ToQueryString()`, looking for one-query-per-row instead of a constant count; fix with `Include`/projection, but validate the fix didn't trade N+1 for a cartesian-exploded single query.

**References:**
- [DbContext Lifetime, Configuration, and Initialization — LogTo/EnableSensitiveDataLogging](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [Single vs. Split Queries - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)

---

### Q9. What problem does `DbContext` pooling solve, and what did Microsoft's own benchmark show about its impact? {#q9}

**Question:**
What problem does `DbContext` pooling solve, and what did Microsoft's own benchmark show about its impact?

**Good answer:**
Creating a `DbContext` is cheap in the sense that it doesn't hit the database, but it does set up various internal services and objects on every instantiation, and that setup cost can add up in high-throughput scenarios where a fresh context is created per request. `AddDbContextPool` (in place of `AddDbContext`) has EF Core reset and reuse context instances instead of discarding and recreating them: when a pooled context is disposed, its state is reset and it's returned to an internal pool rather than garbage-collected, so the next request gets a pre-initialized instance instead of paying setup cost again. Microsoft's published single-row-fetch benchmark showed roughly 701.6μs and 50.38 KB allocated without pooling versus 350.1μs and 4.63 KB with pooling — about a 2x latency improvement and over 10x less allocation for that scenario, though the docs caution results vary with row count, database latency, and other factors.

**Follow-up question:**
Context pooling means the same context instance gets reused across unrelated requests. What extra care does that require for something like multi-tenant state (e.g. a tenant ID needed by a global query filter)?

**Follow-up good answer:**
Because pooling effectively makes the context behave like a singleton internally — the same instance is handed out repeatedly — `OnConfiguring` only ever runs once, at first creation, so you can't rely on it to set per-request state like a tenant ID. The documented pattern is to register the pooled context via `AddPooledDbContextFactory` as a singleton factory, then write a small custom factory (registered as scoped) that calls the pooled factory to get a context instance and then explicitly sets the per-request state — e.g. `context.TenantId = tenant.TenantId` — before handing it out, with that value sourced from a scoped `ITenant` service resolved per request. Skipping this and assuming per-request state is naturally reset is a real production bug source with pooling, since EF Core only resets its *own* internal tracking state, not arbitrary custom properties you've added to your context subclass.

**Glossary:**
- **DbContext pooling** — reusing reset `DbContext` instances from an internal pool instead of creating a fresh instance per unit of work.
- **AddDbContextPool / AddPooledDbContextFactory** — the DI registration methods that enable pooling, with or without an explicit factory.

**Mental model:**
Checks whether the candidate can cite a concrete measured number (not just "it's faster") and understands the one real gotcha pooling introduces — state that doesn't naturally reset between reuses.

**TL;DR:**
`DbContext` pooling reuses reset context instances to avoid per-request setup cost (Microsoft measured roughly 2x lower latency and 10x less allocation in one benchmark), but any custom per-request state on your context subclass must be explicitly reset via a custom scoped factory, since pooling only resets EF Core's own internals.

**References:**
- [Advanced Performance Topics - EF Core — DbContext pooling](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)

---

### Q10. What are `ExecuteUpdate` and `ExecuteDelete`, and why are they more efficient than the traditional query-then-`SaveChanges` approach for bulk operations? {#q10}

**Question:**
What are `ExecuteUpdate` and `ExecuteDelete`, and why are they more efficient than the traditional query-then-`SaveChanges` approach for bulk operations?

**Good answer:**
`ExecuteDelete`/`ExecuteUpdate` let you express a bulk data change directly as LINQ and have EF Core translate it straight into a single SQL `UPDATE`/`DELETE` statement, executed immediately against the database. Compare this to the traditional approach for, say, deleting all low-rated blogs: you'd query for every matching `Blog`, materialize and track every instance, call `Remove` on each one, and then `SaveChanges` generates one `DELETE` statement per row. `ExecuteDeleteAsync()` on the same filtered query instead produces one `DELETE FROM Blogs WHERE Rating < 3` statement — no round trip to fetch the rows, no materialization, no change-tracking overhead, and no N separate DELETE statements. For any bulk operation touching a meaningful number of rows, this is a large, direct efficiency win precisely because it skips the query-then-track-then-individually-persist pipeline entirely.

**Code example:**
```csharp
await context.Blogs.Where(b => b.Rating < 3).ExecuteDeleteAsync();

await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(setters => setters.SetProperty(b => b.IsVisible, false));
```

**Follow-up question:**
What's the danger in mixing `ExecuteUpdate`/`ExecuteDelete` with regular tracked `SaveChanges` modifications in the same unit of work?

**Follow-up good answer:**
`ExecuteUpdate`/`ExecuteDelete` are completely unaware of the change tracker — they take effect immediately in the database and never touch tracked entity state. If you query a blog (now tracked, with its original rating snapshotted), then call `ExecuteUpdateAsync` to bump every blog's rating in the database, and then separately modify that same tracked blog's `Rating` property in memory and call `SaveChangesAsync`, EF Core computes the UPDATE from the *tracked entity's original snapshot* — which has no idea `ExecuteUpdate` already changed the database value — so the `ExecuteUpdate` change gets silently overwritten. On top of that, neither method starts an implicit transaction, so if you call them multiple times in sequence expecting one atomic operation, a failure partway through leaves earlier calls already committed; wrapping multiple such calls in an explicit `context.Database.BeginTransaction()` is required for atomicity. The safe rule is to avoid interleaving tracked `SaveChanges` work and bulk `ExecuteUpdate`/`ExecuteDelete` work against overlapping rows in the same unit of work.

**Glossary:**
- **ExecuteUpdate / ExecuteDelete** — LINQ operators that translate directly to a single SQL UPDATE/DELETE, bypassing the change tracker.

**Mental model:**
Tests whether the candidate knows there are two genuinely different data-modification models in EF Core and understands the sharp edge where mixing them silently loses data — a subtle but real production bug.

**TL;DR:**
`ExecuteUpdate`/`ExecuteDelete` skip querying, materializing, and change-tracking entirely to issue one direct SQL statement — much faster for bulk changes, but since they bypass the change tracker and don't auto-wrap in a transaction, mixing them with tracked `SaveChanges` work on the same rows can silently overwrite one or the other.

**References:**
- [ExecuteUpdate and ExecuteDelete - EF Core](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete)

---

### Q11. What are global query filters, and what are the two classic use cases for them? {#q11}

**Question:**
What are global query filters, and what are the two classic use cases for them?

**Good answer:**
A global query filter, configured via `modelBuilder.Entity<T>().HasQueryFilter(...)`, attaches a LINQ predicate to an entity type that EF Core automatically applies to every query against that type — functionally equivalent to adding the same `Where` clause to every query yourself, but centralized in one place. The two textbook use cases are soft deletion (filter out rows where `IsDeleted == true` by default, so "deleted" rows stay in the database and queryable via an explicit bypass, rather than being physically removed) and multi-tenancy (filter every query by `TenantId == currentTenantId`, so tenant isolation is enforced automatically instead of relying on every query author remembering to add the condition). For multi-tenancy specifically, the filter needs access to the *current* tenant, which usually means referencing a field on the `DbContext` instance itself (e.g. a constructor parameter) from within the filter expression.

**Code example:**
```csharp
modelBuilder.Entity<Blog>().HasQueryFilter(b => !b.IsDeleted);

var allBlogs = await context.Blogs.IgnoreQueryFilters().ToListAsync(); // bypass explicitly
```

**Follow-up question:**
There's a documented gotcha where combining a global query filter with a *required* navigation can silently return fewer results than expected. What causes that, and how do you fix it?

**Follow-up good answer:**
If `Post.Blog` is a required navigation and `Blog` has a query filter, `Include(p => p.Blog)` generates an `INNER JOIN` between `Posts` and the filtered `Blogs` subquery — required navigations imply the related row always exists, so EF Core reasonably uses an inner join. But if a post's blog got filtered out by the query filter, that post now has no matching row on the other side of the inner join, so the post itself silently disappears from the result too, even though `Posts` alone (without the include) would have returned it. `db.Posts.ToListAsync()` might return all 6 posts, while `db.Posts.Include(p => p.Blog).ToListAsync()` returns only 3 — a real, easy-to-miss correctness bug, not just a performance one. The documented fixes are either making the navigation optional (`IsRequired(false)`, so EF Core uses a `LEFT JOIN` instead) or applying a matching filter on `Post` itself so any post whose blog would be filtered out is filtered out consistently on both sides.

**Glossary:**
- **HasQueryFilter** — the model-building API that attaches an automatic filter predicate to an entity type.
- **IgnoreQueryFilters()** — the query operator that bypasses all (or, since EF 10, specifically named) global query filters for one query.

**Mental model:**
This tests whether the candidate has hit a real, subtle EF Core correctness trap rather than just knowing the feature's happy-path use case — global query filters interacting with joins is exactly the kind of thing that passes code review and breaks in production.

**TL;DR:**
Global query filters (`HasQueryFilter`) auto-apply a `Where` to every query against an entity — perfect for soft-delete and multi-tenancy — but combined with a required navigation they can silently drop rows via an unexpected INNER JOIN; fix by making the navigation optional or filtering both sides consistently.

**References:**
- [Global Query Filters - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/filters)

---

### Q12. What's the difference between "client evaluation" and "server evaluation" in an EF Core LINQ query, and why does EF Core throw an exception for some client evaluation but not other cases? {#q12}

**Question:**
What's the difference between "client evaluation" and "server evaluation" in an EF Core LINQ query, and why does EF Core throw an exception for some client evaluation but not other cases?

**Good answer:**
Server evaluation means a part of your LINQ query gets translated into SQL and runs in the database; client evaluation means EF Core instead pulls data back and evaluates that part of the expression in your application's memory — necessary when a method or operator has no SQL equivalent the provider can translate. Since EF Core 3.0, client evaluation is only permitted in the *top-level projection* (essentially the final `Select`): if you call a non-translatable helper method inside `Select`, EF Core evaluates it after fetching the needed columns, and everything else in the query still executes server-side. But if that same non-translatable call appears anywhere else — most commonly inside a `Where` — EF Core throws a runtime exception instead of silently pulling the entire table into memory to filter client-side, because doing so quietly could mean scanning millions of rows in application memory for what looks like a normal, safe-looking filter.

**Code example:**
```csharp
// OK: client evaluation only in the final projection
var blogs = await context.Blogs
    .OrderByDescending(b => b.Rating)
    .Select(b => new { Id = b.BlogId, Url = StandardizeUrl(b.Url) })
    .ToListAsync();

// Throws: StandardizeUrl can't run in a WHERE clause
var bad = await context.Blogs
    .Where(b => StandardizeUrl(b.Url).Contains("dotnet"))
    .ToListAsync();
```

**Follow-up question:**
If you deliberately want to filter using a non-translatable method because the table is known to be small, how do you opt into client-side filtering explicitly, and what's the memory trade-off between the two ways of doing it?

**Follow-up good answer:**
You explicitly materialize first — via `AsEnumerable()`/`ToList()` (or their async equivalents) — and then continue composing LINQ afterward, which switches you from `IQueryable` (translated to SQL) to plain in-memory `IEnumerable`/`List` LINQ. `AsEnumerable()`/`AsAsyncEnumerable()` streams results and applies the client-side filter lazily as they arrive, which uses less peak memory if you don't need to iterate multiple times; `ToList()`/`ToListAsync()` buffers everything into a list first, which costs more memory upfront but is worth it if you're going to enumerate the results more than once, since it only executes the underlying query one time either way. The key point either way: you're opting into potentially loading the *whole* unfiltered result set into memory, so this is only appropriate when you already know that data volume is small — the entire reason EF Core normally blocks this is to stop it from happening by accident.

**Glossary:**
- **Client evaluation** — evaluating part of a query in application memory rather than translating it to SQL.
- **Top-level projection** — the final `Select` in a query, the only place EF Core permits implicit client evaluation.

**Mental model:**
Probes whether the candidate understands *why* the exception exists (a safety rail against accidental full-table scans) rather than treating it as an arbitrary EF Core limitation to work around without thinking about the data volume implications.

**TL;DR:**
EF Core silently allows client evaluation only in a query's final `Select`; anywhere else (like `Where`) it throws rather than risk quietly pulling an entire table into memory — opt in explicitly via `AsEnumerable`/`ToList` only when you know the data volume is safe to do so.

**References:**
- [Client vs. Server Evaluation - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/client-eval)

---

### Q13. `DbContext` is documented as "not thread-safe." What exactly happens if you use one instance concurrently from multiple threads, and what's the most common way this happens by accident? {#q13}

**Question:**
`DbContext` is documented as "not thread-safe." What exactly happens if you use one instance concurrently from multiple threads, and what's the most common way this happens by accident?

**Good answer:**
EF Core doesn't support running parallel operations — including two async queries running concurrently, or explicit use from multiple threads — against the same `DbContext` instance. When EF Core detects this, it throws an `InvalidOperationException` with a message to the effect of "a second operation started on this context before a previous operation completed," and if that detection *doesn't* catch it, the result is undefined behavior, potential crashes, or data corruption. The single most common way this happens by accident is forgetting to `await` an async EF Core call before doing something else with the same context — e.g. firing off `context.SaveChangesAsync()` without awaiting it and then immediately running another query on the same context. In ASP.NET Core, this is usually *not* an issue for the ordinary request-response case, because `AddDbContext` registers the context as scoped and each request runs on essentially one logical thread with its own DI scope (and therefore its own context instance) — the danger appears specifically when code explicitly spins up multiple threads/tasks that each try to touch the one injected context instance concurrently.

**Follow-up question:**
If you genuinely need to do multiple database operations in parallel — say, fetching data for several independent sections of a dashboard concurrently — how do you do that safely with EF Core and dependency injection?

**Follow-up good answer:**
You need a separate `DbContext` instance per concurrent operation, not concurrent access to one shared instance. With DI, the documented approach is either registering the context as scoped and explicitly creating a new DI scope per thread/task via `IServiceScopeFactory` (each scope resolves its own context instance), or registering an `IDbContextFactory<T>` via `AddDbContextFactory` and calling `CreateDbContext()` once per parallel operation to get an independent instance — the factory pattern exists specifically for cases (like Blazor Server, or exactly this kind of manual fan-out) where the ambient DI scope doesn't naturally align with the unit-of-work boundary you need. Each instance can then safely run its own queries in parallel without any risk of the concurrent-access exception, since they don't share any tracked state.

**Glossary:**
- **IDbContextFactory** — a DI-registered factory for creating independent `DbContext` instances on demand, rather than relying on the ambient scoped instance.

**Mental model:**
Tests real-world debugging instinct — "second operation started on this context" is a message every EF Core developer eventually hits, and understanding *why* (missing await, or genuine multi-threaded misuse) versus just retrying is the difference between fixing it and masking it.

**TL;DR:**
A single `DbContext` instance can't be used by two operations at once (including two un-awaited async calls) — EF Core throws when it detects this, and the fix for genuine parallel work is a separate context instance per concurrent operation, via a new DI scope or `IDbContextFactory`.

**References:**
- [DbContext Lifetime, Configuration, and Initialization — Avoiding DbContext threading issues](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)

---

### Q14. Why is `AddDbContext` registered as a *scoped* service by default in ASP.NET Core, rather than singleton or transient? {#q14}

**Question:**
Why is `AddDbContext` registered as a *scoped* service by default in ASP.NET Core, rather than singleton or transient?

**Good answer:**
Scoped lifetime means one instance per DI scope, and in a typical ASP.NET Core app a DI scope is created per HTTP request — which lines up naturally with the fact that a `DbContext` is meant to represent one short-lived unit of work, and a single HTTP request is usually exactly one unit of work. A singleton `DbContext` would be shared across every concurrent request, which is unsafe given `DbContext` isn't thread-safe and would also mean the change tracker accumulates every entity ever touched by any request, for the lifetime of the app. A transient registration (a new instance every time it's *resolved*, potentially multiple times within one request if resolved from multiple places) would risk creating several disconnected contexts within what should be one logical unit of work, unable to see each other's tracked changes. Scoped is the sweet spot: one instance is shared consistently across everything in that request's dependency graph, and it's disposed cleanly when the request ends.

**Follow-up question:**
Blazor Server is called out as a case where the default scoped-per-request alignment breaks down. Why, and what's the recommended fix?

**Follow-up good answer:**
In Blazor Server, one DI scope is created per user's persistent circuit (the SignalR-backed connection that lasts for the whole time the user has the page open), not per individual UI interaction — so a scoped `DbContext` in Blazor Server would effectively live for the user's entire session, accumulating tracked entities across many unrelated operations, which is exactly the long-lived-context problem the scoped-per-request pattern was designed to avoid in the first place. The documented fix is `AddDbContextFactory`, registered once, which apps like Blazor Server use to explicitly create (and dispose) a fresh, short-lived `DbContext` instance for each individual unit of work — e.g. each button click's data operation — rather than relying on the ambient scope to hand out an appropriately-scoped instance automatically.

**Glossary:**
- **Scoped lifetime** — a DI lifetime where one instance is created per logical scope (typically one HTTP request in ASP.NET Core).

**Mental model:**
Tests whether the candidate understands *why* a default was chosen, not just that it exists — and whether they know the default's assumption (one scope ≈ one request ≈ one unit of work) can break down in non-traditional hosting models.

**TL;DR:**
Scoped lifetime ties one `DbContext` instance to one DI scope, which in ASP.NET Core normally means one HTTP request — matching the "one unit of work" design intent; hosting models like Blazor Server where a scope spans much more than one unit of work need `AddDbContextFactory` instead.

**References:**
- [DbContext Lifetime, Configuration, and Initialization — DbContext in dependency injection for ASP.NET Core](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)

---

### Q15. How does EF Core translate LINQ into SQL, and why can a query throw at runtime for something that compiles fine as C#? {#q15}

**Question:**
How does EF Core translate LINQ into SQL, and why can a query throw at runtime for something that compiles fine as C#?

**Good answer:**
An EF Core LINQ query is really just building up a .NET expression tree — a data structure describing the query's operations — rather than executing anything immediately; that's what makes `IQueryable` composable and deferred. When you finally enumerate the query (via `ToListAsync`, `FirstOrDefaultAsync`, etc.), the EF Core query pipeline walks that expression tree and attempts to translate each operation into an equivalent SQL construct for your specific database provider. Because this translation happens at runtime against the actual tree you built, and depends entirely on what the provider knows how to translate, code that compiles perfectly fine as C# — like calling an arbitrary instance method inside a `Where` clause — can still fail at runtime with a translation exception, because the compiler has no way to know ahead of time whether SQL Server's or PostgreSQL's LINQ provider can express that specific method as SQL. This is fundamentally different from, say, calling a method on an in-memory `List<T>`, where the compiler and runtime never need to ask "can this be expressed as SQL?"

**Follow-up question:**
Given that translation failures are a runtime-only concern, what practices actually catch these problems before they reach production?

**Follow-up good answer:**
Integration tests that run queries against a real (or realistic, e.g. a Testcontainers-backed) instance of your actual database provider are the most reliable check, since translation behavior is provider-specific and the EF Core in-memory database provider explicitly does not validate translatability the same way — a query can pass against the in-memory provider and still fail against SQL Server. Beyond tests, exercising new or unusual LINQ patterns manually against a real database during development, and keeping an eye on `LogTo`-based SQL logs to confirm what's actually being generated (versus what you assumed), catches most translation surprises early. There's no static analysis step that fully replaces this, precisely because translatability is a runtime property of the specific provider and EF Core version in use.

**Glossary:**
- **Expression tree** — the data structure representing a LINQ query's operations before it's translated or executed.
- **Query translation** — the runtime process of converting an expression tree into provider-specific SQL.

**Mental model:**
Tests whether the candidate has an accurate mental model of *when* EF Core actually does work (translation happens lazily, at enumeration time, against a specific provider) — a common source of confusion for developers coming from purely in-memory LINQ.

**TL;DR:**
LINQ queries build a deferred expression tree that's translated to SQL only when enumerated, against a specific provider — so a query can compile fine yet throw a runtime translation exception, which only real-database integration tests reliably catch (the in-memory provider doesn't validate this).

**References:**
- [Client vs. Server Evaluation - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/client-eval)

---

### Q16. What's the trade-off between using EF Core and using a lighter tool like Dapper (or raw ADO.NET) for a performance-critical read path? {#q16}

**Question:**
What's the trade-off between using EF Core and using a lighter tool like Dapper (or raw ADO.NET) for a performance-critical read path?

**Good answer:**
EF Core buys you a lot: LINQ query composition, automatic change tracking and `SaveChanges` for writes, migrations, navigation-property-based relationship loading, and provider abstraction — but all of that machinery has some inherent overhead compared to writing SQL by hand and mapping rows directly to objects, which is Dapper's whole model. For a genuinely hot, simple read path — a query that runs extremely frequently and where every microsecond matters — hand-written SQL executed via Dapper (or raw ADO.NET) skips expression-tree translation, change-tracker setup, and EF Core's abstraction layers entirely, and can be meaningfully faster. In practice, EF Core's own performance guidance is consistent with this: for reads, use `AsNoTracking()` (skip the tracking overhead you don't need); use `DbContext` pooling and compiled queries for the hottest paths; and reach for `ExecuteUpdate`/`ExecuteDelete` for bulk writes — largely because these features exist to close the gap for scenarios where EF Core's defaults (which favor developer productivity and correctness) cost more than a specific use case can afford.

**Follow-up question:**
Given that trade-off, is "always use Dapper for reads, EF Core for writes" a good blanket rule? What would make you say no?

**Follow-up good answer:**
No — for the vast majority of application code, EF Core's overhead versus Dapper is dwarfed by network latency and actual database I/O time, so the productivity, type-safety, and maintainability cost of hand-writing and hand-mapping SQL for every read (losing LINQ composability, losing automatic navigation-property loading, having to maintain your own mapping code) isn't worth it just because Dapper is theoretically faster in a microbenchmark. The right call is workload-specific: profile first, and reach for Dapper/raw SQL only for the specific hot paths where measurement shows EF Core's overhead is an actual bottleneck relative to the rest of the request's cost — not as a default architectural stance applied uniformly across an entire codebase before you've measured anything.

**Glossary:**
- **Dapper** — a lightweight micro-ORM that maps query results to objects without change tracking, LINQ translation, or migrations.

**Mental model:**
This is a classic trade-off question testing whether the candidate defaults to "the fancier/faster tool always wins" or actually reasons about where a specific technology's overhead matters relative to the rest of a system's cost profile.

**TL;DR:**
EF Core trades some raw performance for productivity (LINQ, change tracking, migrations); Dapper/raw SQL trades that productivity back for speed on hot paths — the right answer is to default to EF Core and drop to Dapper only for specific, measured bottlenecks, not as a blanket rule.

**References:**
- [Advanced Performance Topics - EF Core — Reducing runtime overhead](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)

---

### Q17. How does the Repository pattern relate to `DbContext`, and why do many teams consider wrapping `DbContext` in a custom repository redundant? {#q17}

**Question:**
How does the Repository pattern relate to `DbContext`, and why do many teams consider wrapping `DbContext` in a custom repository redundant?

**Good answer:**
The Repository pattern's classic intent is to hide data-access details behind a collection-like interface, so business logic doesn't depend on how persistence actually works. `DbSet<T>` already behaves like this: it's an in-memory-collection-like abstraction (`Add`, `Remove`, LINQ querying) over the actual database access, and `DbContext` itself is Microsoft's own description of a Unit-of-Work implementation, tracking everything that needs to change and committing it atomically via `SaveChanges`. Because of that overlap, many teams find that adding a custom `IRepository<T>`/`IUnitOfWork` layer on top of `DbContext` mostly re-implements functionality `DbContext`/`DbSet` already provide, at the cost of extra abstraction layers, without necessarily buying additional testability (since `DbContext` can already be tested against an in-memory or SQLite provider) or genuine database-swappability (which rarely materializes as an actual requirement in practice, and EF Core's own provider model already exists for that).

**Follow-up question:**
Is there ever a legitimate reason to add a repository layer on top of EF Core despite this overlap?

**Follow-up good answer:**
Yes — when the goal isn't "abstract away EF Core" but "encapsulate a specific, reusable query or business rule behind a clean name," a thin repository (or, more precisely, specification/query-object pattern) can genuinely help: e.g. `GetActiveOrdersForCustomer(customerId)` as a named method is more readable and testable-in-isolation than the equivalent LINQ scattered across call sites, and centralizing it avoids duplicating tricky `Include`/filtering logic. It's also legitimate when you specifically want to decouple domain logic from any EF Core dependency at all — e.g. a Clean/Hexagonal Architecture design where the domain layer must have zero reference to EF Core types — though that's an explicit architectural choice with its own cost, not a default best practice to apply everywhere. The distinction that matters is: are you adding a layer to genuinely encapsulate query/business logic, or just wrapping `DbContext`'s existing `DbSet`/Unit-of-Work behavior in a same-shaped interface for its own sake?

**Glossary:**
- **Repository pattern** — an abstraction presenting persisted data as an in-memory-collection-like interface, hiding data-access details from business logic.

**Mental model:**
This tests architectural judgment, not just pattern recall — a senior candidate should be able to articulate *why* a popular pattern is sometimes redundant with the tool already in hand, not reflexively apply every pattern from a textbook.

**TL;DR:**
`DbSet`/`DbContext` already implement much of what a hand-rolled Repository/Unit-of-Work layer would provide, so wrapping them in a same-shaped custom abstraction is often redundant — a repository still earns its keep when it encapsulates specific reusable query logic or serves a deliberate architectural boundary (e.g. keeping EF Core out of a domain layer).

**References:**
- [Change Tracking - EF Core](https://learn.microsoft.com/en-us/ef/core/change-tracking/)
- [DbContext Lifetime, Configuration, and Initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [Martin Fowler — Repository](https://www.martinfowler.com/eaaCatalog/repository.html)

---

### Q18. What real-world problem does an ORM like EF Core actually solve — what was application code doing before something like this existed? {#q18}

**Question:**
What real-world problem does an ORM like EF Core actually solve — what was application code doing before something like this existed?

**Good answer:**
Before ORMs, mapping between an object-oriented domain model and a relational schema was manual and repetitive: hand-writing SQL for every query, manually reading `DataReader` columns into object properties, manually tracking which loaded objects had been modified so you knew what to `UPDATE`, and manually writing INSERT/UPDATE/DELETE statements with the right parameters in the right order to respect foreign-key constraints — all of this is exactly what's sometimes called the "object-relational impedance mismatch": objects have identity, inheritance, and graphs of references; relational tables have rows, foreign keys, and joins, and translating between the two by hand for every feature is enormous, error-prone, repetitive work. EF Core's change tracker, LINQ-to-SQL translation, and `SaveChanges` mechanism automate exactly that translation: query results become tracked objects, in-memory changes are automatically diffed and turned into the right SQL statements, and relationships are navigable as object references instead of manual joins you re-write every time.

**Follow-up question:**
Given all that automation, what's the actual cost of choosing an ORM — where does the abstraction leak or add friction compared to hand-written data access?

**Follow-up good answer:**
The abstraction leaks exactly where this question set has spent most of its time: you still need to understand the generated SQL to reason about performance (N+1, cartesian explosion, unnecessary tracking overhead), the change tracker itself has real memory and CPU cost for large working sets, and some queries that look reasonable in LINQ simply can't be translated and either throw or force you into raw SQL/`FromSqlRaw` anyway. There's also a learning-curve cost — a developer who doesn't understand tracking states, query translation, or the difference between `ExecuteUpdate` and `SaveChanges` can write code that "works" in a demo and falls over under real load. The honest framing is that EF Core removes almost all of the *repetitive, boilerplate* mapping work, but it doesn't remove the need to understand what's happening underneath — it just changes what you need to understand from "manual SQL and mapping code" to "how EF Core's translation and tracking actually behave."

**Glossary:**
- **Object-relational impedance mismatch** — the structural mismatch between object-oriented models (identity, inheritance, references) and relational schemas (rows, foreign keys, joins) that ORMs exist to bridge.

**Mental model:**
This is the "why does this technology exist, what pain predates it" category — testing whether the candidate can articulate the actual motivating problem instead of just describing EF Core's feature list.

**TL;DR:**
ORMs like EF Core automate the tedious, error-prone manual work of mapping objects to relational rows and back (querying, dirty-checking, generating INSERT/UPDATE/DELETE) — the object-relational impedance mismatch — but they don't remove the need to understand what's happening underneath; they just relocate that understanding from "SQL and mapping code" to "translation and tracking behavior."

**References:**
- [Change Tracking - EF Core](https://learn.microsoft.com/en-us/ef/core/change-tracking/)

---

### Q19. You're designing a read-heavy, mostly-reporting API on top of EF Core. Would you flip the context's default tracking behavior globally to no-tracking, and what would you weigh in that decision? {#q19}

**Question:**
You're designing a read-heavy, mostly-reporting API on top of EF Core. Would you flip the context's default tracking behavior globally to no-tracking, and what would you weigh in that decision?

**Good answer:**
`UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)` in `OnConfiguring` (or the equivalent `ChangeTracker.QueryTrackingBehavior` setter) makes every query no-tracking by default, with `AsTracking()` available to opt specific queries back in — this is a genuinely reasonable choice for a context that's overwhelmingly used for reads, since it avoids paying tracking setup cost on every query when the vast majority of them will never call `SaveChanges`. The trade-off to weigh is that any code path in that same context that *does* need to modify and save an entity now has to explicitly opt into tracking with `AsTracking()`, or it'll silently do nothing on `SaveChanges` (a no-tracking query's returned entities aren't tracked, so mutating them and calling `SaveChanges` has no effect) — which is a footgun if the codebase mixes read and write paths against the same context type without very clear conventions. For a context type that's used *exclusively* for reporting/read endpoints (as opposed to a shared context type also used for writes elsewhere), flipping the global default is a clean, low-risk win; for a general-purpose context used for both, it's often safer to default to tracking and apply `AsNoTracking()` explicitly on the specific read-heavy queries.

**Follow-up question:**
Beyond the tracking default, what else from this question set would you specifically apply to a reporting-style workload, and why those specifically?

**Follow-up good answer:**
Projection-first queries (`Select` into DTOs instead of returning full entities) to avoid loading columns the report doesn't need and to sidestep tracking questions entirely, since projected non-entity results generally aren't tracked regardless of the context default; `AsSplitQuery()` on any report that joins multiple sibling collections, since reporting queries are exactly the kind that tend to pull in several related collections at once and are prone to cartesian explosion; and `DbContext` pooling plus compiled queries for whichever specific reports run frequently enough (e.g. a dashboard hit on every page load) that the setup/translation overhead is measurable relative to the query's own execution time. The common thread is that reporting workloads are read-heavy and often high-frequency, which is exactly the profile where EF Core's optional performance features (no-tracking, split queries, pooling, compiled queries) pay for themselves, versus a low-frequency admin CRUD screen where the defaults are perfectly fine.

**Glossary:**
- **UseQueryTrackingBehavior** — the `DbContextOptionsBuilder` method that sets the context-wide default tracking behavior.

**Mental model:**
This is a synthesis question — it checks whether the candidate can combine everything covered (tracking, split queries, pooling, compiled queries) into a coherent recommendation for a concrete scenario, rather than reciting each feature in isolation.

**TL;DR:**
Flipping a context's default to no-tracking is a good fit for a genuinely read-only/reporting context, but risks silently-ineffective saves if that same context type is also used for writes — combine it with projections, `AsSplitQuery()` for multi-collection reports, and pooling/compiled queries for the highest-frequency queries specifically.

**References:**
- [Tracking vs. No-Tracking Queries - EF Core — Configuring the default tracking behavior](https://learn.microsoft.com/en-us/ef/core/querying/tracking)
- [Single vs. Split Queries - EF Core](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)

---

### Q20. What's the fundamental difference in how EF Core executes `SaveChanges()` versus `ExecuteUpdate()`/`ExecuteDelete()`, in terms of when SQL actually runs against the database? {#q20}

**Question:**
What's the fundamental difference in how EF Core executes `SaveChanges()` versus `ExecuteUpdate()`/`ExecuteDelete()`, in terms of when SQL actually runs against the database?

**Good answer:**
`SaveChanges()` is a batch-commit model: you accumulate changes across potentially many entities — some added, some modified, some marked for deletion — entirely in memory via the change tracker, and only when you call `SaveChanges()` does EF Core walk all of that tracked state and generate the necessary INSERT/UPDATE/DELETE statements, sent to the database as part of that one call. `ExecuteUpdate()`/`ExecuteDelete()`, by contrast, execute immediately at the point they're called — there's no accumulation phase, no change tracker involvement, and each invocation is its own independent round trip with its own implicit transaction (unless you explicitly wrap several in one transaction yourself). This is why they can't be mixed and matched as if they were the same kind of operation: `SaveChanges()` naturally batches a whole unit of work's worth of changes into one commit point, while `ExecuteUpdate`/`ExecuteDelete` are closer in spirit to firing raw SQL statements one at a time, each taking effect the instant it's invoked.

**Code example:**
```csharp
// SaveChanges: changes accumulate, then commit together
blog.Rating = 5;
context.Posts.Remove(oldPost);
await context.SaveChangesAsync(); // one commit point for both changes

// ExecuteDelete: takes effect immediately, independently
await context.Posts.Where(p => p.IsSpam).ExecuteDeleteAsync(); // already done
```

**Follow-up question:**
Given this difference, which one would you reach for to implement a soft-delete-on-bulk-cleanup job that needs to mark thousands of stale records as deleted every night, and why?

**Follow-up good answer:**
`ExecuteUpdate` is the clear right tool here: a nightly cleanup job marking thousands of records is exactly the bulk-update scenario `ExecuteUpdate` exists for — one `UPDATE ... SET IsDeleted = 1 WHERE ...` statement handles all matching rows in the database directly, versus `SaveChanges`'s approach of querying and tracking every one of those thousands of rows just to flip one flag on each, which wastes memory and time proportional to the row count for no benefit (you're not doing anything else with those entities). The one thing to watch for, given this file's earlier point about `ExecuteUpdate` bypassing the change tracker: if any of those same rows happen to already be tracked elsewhere in that same `DbContext` instance (unlikely for a dedicated batch job, but worth checking), their in-memory `IsDeleted` value won't reflect the bulk change, so a subsequent `SaveChanges` on that context could stomp on it — for a standalone nightly job using its own fresh context, this isn't a practical concern.

**Glossary:**
- **Batch-commit model** — accumulating multiple changes and committing them together at one point (as `SaveChanges` does).

**Mental model:**
A closing synthesis question that forces the candidate to apply the SaveChanges-vs-ExecuteUpdate distinction to a concrete scenario and justify the choice, rather than just reciting the difference in the abstract.

**TL;DR:**
`SaveChanges()` accumulates tracked changes and commits them together at one point; `ExecuteUpdate`/`ExecuteDelete` take effect immediately per call with no accumulation — for a bulk nightly cleanup touching thousands of rows, `ExecuteUpdate` is the right tool precisely because there's nothing to accumulate or track.

**References:**
- [ExecuteUpdate and ExecuteDelete - EF Core](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete)
- [Change Tracking - EF Core](https://learn.microsoft.com/en-us/ef/core/change-tracking/)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=dotnet&tags=ef-core-internals-and-query-performance&autostart=1" | relative_url }})
