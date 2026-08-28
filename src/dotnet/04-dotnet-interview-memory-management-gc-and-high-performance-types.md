---
layout: default
title: ".NET Interview — Memory Management, GC & High-Performance Types"
---

# .NET Interview — Memory Management, GC & High-Performance Types

This set goes deeper into the CLR's memory model than file 01's brief GC mention: the generational garbage collector's internals (Gen0/1/2, LOH, POH), Workstation vs Server GC, how `Span<T>`/`Memory<T>`/`ArrayPool<T>`/`stackalloc` let you write allocation-light, high-throughput code, and the diagnostic tools (`dotnet-counters`, `dotnet-gcdump`, BenchmarkDotNet's memory diagnoser) you'd actually reach for to prove any of it.

### Q1. Where does a value type actually live in memory — always the stack? {#q1}

**Question:**
Where does a value type actually live in memory — always the stack?

**Good answer:**
No — "value types go on the stack" is a common oversimplification. A value type lives wherever the variable holding it lives. A local variable of a struct type in a method typically lives on the stack. But a struct that's a field of a class instance lives inline inside that object on the *heap*, because the containing object is a reference type. A struct that's an element of an array lives inline inside the array's heap-allocated storage. And a struct captured by a lambda or used inside an `async` method's state machine gets hoisted into the compiler-generated class/struct backing that closure or state machine, which itself often ends up on the heap. The only universally true statement is: reference types are always heap-allocated (the reference itself may live on the stack), and value types are allocated wherever their containing storage is allocated — the stack is just the common case for simple locals.

**Code example:**
```csharp
struct Point { public int X, Y; }

class Container
{
    public Point P; // lives inline inside this Container instance, on the heap
}

void Method()
{
    Point local = new Point(); // lives on the stack
    var points = new Point[10]; // the Point structs live inline inside the array's heap storage
}
```

**Follow-up question:**
If structs can end up on the heap anyway, what's the actual benefit of using a struct over a class?

**Follow-up good answer:**
The benefit isn't "avoids the heap" — it's avoiding a *separate* heap allocation and the indirection/pointer-chasing that comes with it. When a struct is a local variable, a field, or an array element, its data is stored inline in whatever already-allocated block of memory contains it, with no extra object header and no extra GC-tracked allocation. Contrast with a class field or array of a reference type: each instance is a separate heap object with its own object header (method table pointer + sync block), and accessing it means dereferencing a pointer to a potentially non-contiguous location. For an array of a million small values, `structs[]` is one contiguous allocation; `classes[]` is one array allocation of pointers plus a million separate object allocations — worse cache locality and much more GC tracking work. This is exactly why `Span<T>`/`ReadOnlySpan<T>` and value-type-heavy APIs matter for high-throughput code: fewer, denser allocations.

**Glossary:**
- **Value type** — a type whose variable directly contains its data (structs, enums, primitives like `int`).
- **Reference type** — a type whose variable holds a reference (pointer) to data allocated separately on the managed heap.
- **Boxing** — wrapping a value type in a heap-allocated object so it can be treated as `object`/an interface.

**Mental model:**
Tests whether the candidate actually understands allocation semantics or just recites "value type = stack, reference type = heap" without knowing why it's an oversimplification — which matters directly for reasoning about `Span<T>`, arrays of structs, and boxing.

**TL;DR:**
A value type lives wherever its containing storage lives (stack for a simple local, inline in the heap if it's a field/array element/closure capture) — the real benefit is avoiding a separate heap allocation and pointer indirection, not "staying off the heap" categorically.

**References:**
- [Value types - C# reference (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)
- [Boxing and Unboxing (C# Programming Guide) (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)

---

### Q2. What makes the .NET garbage collector "generational," and why does that assumption pay off? {#q2}

**Question:**
What makes the .NET garbage collector "generational," and why does that assumption pay off?

**Good answer:**
The GC has three generations — generation 0, 1, and 2. Newly allocated objects start in generation 0. If an object survives a collection, it's promoted to generation 1, and if it survives another, to generation 2. The generational design rests on the empirical observation that most objects die young: a huge fraction of allocations (temporary strings, short-lived DTOs, LINQ intermediates) become garbage almost immediately, while objects that survive a while tend to keep surviving. So the GC collects generation 0 very frequently and cheaply — it only has to scan a small, young pool — and reserves the expensive full-heap collections (which include generation 2 plus the large and pinned object heaps) for far rarer occasions. This means a well-behaved app spends the vast majority of its GC time on fast Gen0 collections rather than expensive full collections.

**Code example:**
```csharp
// A short-lived object: allocated, used, immediately eligible for Gen0 collection.
string temp = ComputeGreeting(); // likely collected in the next Gen0 GC

// A long-lived object: survives, gets promoted toward Gen2.
static readonly Dictionary<string, User> _cache = new(); // lives for the app's lifetime
```

**Follow-up question:**
Where do large objects fit into this generational scheme, and why are they treated specially?

**Follow-up good answer:**
Objects at or above 85,000 bytes are allocated on the Large Object Heap (LOH) instead of the normal small-object heap, and the LOH is sometimes informally called "generation 3" — but it's a physical generation that's logically collected as part of a generation 2 collection, not its own independent tier. The reason for special treatment is cost: compacting (moving) a very large object during a collection is expensive, so historically the LOH wasn't compacted by default (only swept, leaving free gaps that can fragment), though you can opt into LOH compaction. There's also the Pinned Object Heap (POH), introduced for objects allocated via `GC.AllocateArray` with `pinned: true` — both the LOH and POH are only collected during generation 2 GCs, which is exactly why large or pinned allocations are far more expensive to reclaim than a short-lived Gen0 object.

**Glossary:**
- **Generation 0/1/2** — the GC's age-based tiers; 0 is youngest/cheapest to collect, 2 is oldest/most expensive.
- **LOH (Large Object Heap)** — heap segment for objects ≥ 85,000 bytes, collected only during Gen2 GCs.
- **POH (Pinned Object Heap)** — heap segment for objects pinned via `GC.AllocateArray(pinned: true)`, also Gen2-only.

**Mental model:**
Tests whether the candidate understands *why* the GC is structured the way it is (the generational hypothesis) rather than just memorizing "there are 3 generations" as trivia.

**TL;DR:**
The GC is generational because most objects die young — collecting Gen0 frequently and cheaply while reserving expensive full (Gen2 + LOH/POH) collections for the rare long-lived survivors is what makes .NET's GC fast in practice.

**References:**
- [Fundamentals of garbage collection - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
- [Large object heap on Windows - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap)

---

### Q3. Why is `Span<T>` declared as a `ref struct`, and what does that cost you? {#q3}

**Question:**
Why is `Span<T>` declared as a `ref struct`, and what does that cost you?

**Good answer:**
`Span<T>` needs to be able to point into memory that isn't necessarily on the GC heap at all — it can wrap a `stackalloc` buffer, a slice of an array, or unmanaged memory, using an internal `ref T` field plus a length. A `ref` field can point at stack memory, but the CLR can't safely let something holding a stack pointer escape onto the heap (the stack frame it points into could be gone by the time the heap object is inspected). Making `Span<T>` a `ref struct` is how the C# compiler *enforces* that it can never do that: `ref struct` instances are compiler-restricted to stack-only lifetime. Concretely, that means you can't use a `Span<T>` as an array element type, as a field of an ordinary class or non-`ref struct`, you can't box it, you can't capture it in a lambda or local function, and — before C# 13 — you couldn't use it in an `async` method at all (C# 13 relaxed this to just forbidding it across `await`/`yield` boundaries specifically). The cost is real: no `List<Span<T>>`, no storing one on a class field for later, and no naive "pass it into an async pipeline."

**Code example:**
```csharp
public ref struct CustomRef
{
    public Span<int> Data; // legal: a ref struct can contain another ref struct
}

class Widget
{
    // public Span<byte> Buffer; // ILLEGAL: ref struct can't be a field of a non-ref-struct class
}
```

**Follow-up question:**
Given that restriction, why does `Memory<T>` exist, and when would you reach for it instead?

**Follow-up good answer:**
`Memory<T>` is the heap-safe counterpart: instead of holding a raw `ref T`, it holds a reference to the underlying array/owner plus an offset and length, which is perfectly fine to store on a heap-allocated object, pass across `await` boundaries, or hold as a field. You reach for `Memory<T>` (and `ReadOnlyMemory<T>`) specifically in async APIs, or anywhere you need to store a "slice" for later rather than use it immediately — for example, a method signature like `Task WriteAsync(ReadOnlyMemory<byte> buffer)`. When you actually need to touch the data synchronously, you call `.Span` on the `Memory<T>` to get a (temporary, stack-only) `Span<T>` view over it, do your work, and let it go out of scope before the next `await`.

**Glossary:**
- **`ref struct`** — a struct type the compiler restricts to stack-only usage so it (or a `ref` field it holds) can never escape to the heap.
- **`Memory<T>`** — a heap-safe, non-`ref struct` alternative to `Span<T>` usable across `await` boundaries and as a field.

**Mental model:**
Probes whether the candidate understands *why* the restriction exists (a safety guarantee tied to stack lifetime) rather than treating it as an arbitrary compiler rule to work around.

**TL;DR:**
`Span<T>` is a `ref struct` so the compiler can guarantee it (and any stack pointer it wraps) never escapes to the heap — which is exactly what makes it unusable across `await`/in fields/in lambdas, and exactly why `Memory<T>` exists as the heap-safe alternative for those cases.

**References:**
- [ref struct types - C# reference (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct)
- [System.Span<T> struct - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-span%7BT%7D)

---

### Q4. What does `stackalloc` actually do, and what's the risk if you use it carelessly? {#q4}

**Question:**
What does `stackalloc` actually do, and what's the risk if you use it carelessly?

**Good answer:**
`stackalloc` allocates a block of memory directly on the current thread's stack rather than the GC heap. That memory is automatically reclaimed when the method returns — there's no explicit free, and it's never subject to garbage collection at all, so it can't create GC pressure. In modern C# you assign the result to a `Span<T>`/`ReadOnlySpan<T>` without needing an `unsafe` block; only assigning it to a raw pointer requires `unsafe`. The risk is that stack space is small and fixed per thread — allocate too much (especially inside a loop, or with a size that scales with unbounded input) and you get a `StackOverflowException`, which the runtime can't gracefully catch; it terminates the process. The standard defensive pattern is to only `stackalloc` below a small, conservative constant threshold and fall back to a heap array above it.

**Code example:**
```csharp
const int MaxStackLimit = 256;
Span<byte> buffer = inputLength <= MaxStackLimit
    ? stackalloc byte[inputLength]
    : new byte[inputLength];
```

**Follow-up question:**
Why is `stackalloc` memory left uninitialized by default, unlike memory from `new`?

**Follow-up good answer:**
It's a deliberate performance trade-off: zeroing memory costs time proportional to its size, and `stackalloc` is specifically meant for tight, hot-path buffer use where that cost may matter, so the language doesn't pay for zero-initialization you might not need (e.g. if you're about to fully overwrite the buffer anyway). This is explicitly called out as a difference from `new`, which always zero-initializes. If you do need a clean buffer, you initialize it yourself with an initializer syntax (`stackalloc byte[3] { 1, 2, 3 }`) or call `Span<T>.Clear()` before use — the language leaves that choice to you rather than forcing the cost on every allocation.

**Glossary:**
- **`stackalloc`** — a C# operator allocating a block on the current stack frame, freed automatically on method return.
- **`StackOverflowException`** — an uncatchable runtime failure when a thread's stack space is exhausted.

**Mental model:**
Checks whether the candidate knows `stackalloc` trades GC-freedom for a hard, easy-to-blow capacity limit, and whether they'd actually guard that limit in real code rather than just knowing the keyword exists.

**TL;DR:**
`stackalloc` gives you GC-free, auto-freed stack memory, but its capacity is small and fixed, so unguarded or loop-based use risks an unrecoverable `StackOverflowException` — always cap it and fall back to a heap array above the threshold.

**References:**
- [stackalloc expression - C# reference (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc)

---

### Q5. How does `ArrayPool<T>` reduce GC pressure, and what's the contract you must honor to use it safely? {#q5}

**Question:**
How does `ArrayPool<T>` reduce GC pressure, and what's the contract you must honor to use it safely?

**Good answer:**
`ArrayPool<T>` maintains a pool of reusable arrays instead of letting you allocate-and-discard a new array on every call — you `Rent(minimumLength)` a buffer, use it, and `Return` it when done, so the same backing array gets reused across many operations instead of generating garbage on every one. This matters most in situations where arrays are created and destroyed frequently (parsers, serializers, network buffer handling), where naive allocation would flood Gen0 and force constant collections. The contract: `Rent` guarantees an array *at least* as long as you asked for — it may hand you a larger one from the pool, so you must track and use only the logical length you requested, not `array.Length`. And `Return` is your responsibility: if you never return a rented array, it isn't an outright leak (the GC can still eventually collect it once unreferenced), but you lose all the pooling benefit and the pool has to keep allocating fresh arrays for other callers.

**Code example:**
```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096); // may return a buffer longer than 4096
try
{
    int bytesRead = stream.Read(buffer, 0, 4096);
    Process(buffer.AsSpan(0, bytesRead)); // use only the logical length
}
finally
{
    pool.Return(buffer); // mandatory to get pooling's benefit
}
```

**Follow-up question:**
Why does the `Return` method have a `clearArray` parameter, and when should you actually set it to `true`?

**Follow-up good answer:**
By default, `Return` doesn't clear the array's contents for performance — zeroing it costs time you often don't need to pay, since the next renter is expected to overwrite the region it uses anyway. You set `clearArray: true` when the array may contain sensitive data (cryptographic key material, PII, auth tokens) that shouldn't linger in memory and potentially be readable by a future renter that doesn't overwrite the full buffer, or when your code relies on default-zeroed contents rather than explicitly initializing what it reads. It's a security/correctness trade-off you opt into per call, not a default, matching the same "don't pay for zeroing you don't need" philosophy as `stackalloc`.

**Glossary:**
- **`ArrayPool<T>`** — a `System.Buffers` pool of reusable arrays, reducing allocation/GC churn for short-lived buffers.
- **`ArrayPool<T>.Shared`** — the process-wide default pool instance most code should use.

**Mental model:**
Tests real-world usage discipline — knowing the API exists is easy, but knowing you must respect the "at least this long" contract and actually call `Return` (ideally in a `finally`) is what separates using it correctly from introducing a subtle bug.

**TL;DR:**
`ArrayPool<T>` cuts GC pressure by reusing buffers instead of allocating fresh ones each time — but `Rent` only guarantees a length "at least" what you asked for, and you must `Return` every rented array (in a `finally`) to actually get the benefit.

**References:**
- [ArrayPool<T> Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)

---

### Q6. What's the difference between Workstation GC and Server GC, and which does ASP.NET Core use by default? {#q6}

**Question:**
What's the difference between Workstation GC and Server GC, and which does ASP.NET Core use by default?

**Good answer:**
Workstation GC is the default flavor for standalone client apps: collection happens on the same user thread that triggered it, at normal thread priority, competing with the rest of the app's threads for CPU time. Server GC is designed for high-throughput server workloads: it creates a dedicated GC heap and a dedicated collector thread *per logical CPU*, all running at the highest thread priority, and collects all those heaps in parallel — which makes it faster than Workstation GC on the same heap size, at the cost of being more memory- and CPU-resource-intensive (multiple processes on the same box all running Server GC can end up contending for those same cores). ASP.NET Core apps default to Server GC, because the host determines the default GC flavor for hosted apps, and a web server is exactly the high-throughput scenario Server GC targets. It's also worth knowing Server GC isn't available at all on a single-logical-CPU machine — Workstation GC is used regardless of configuration in that case.

**Code example:**
```xml
<PropertyGroup>
  <ServerGarbageCollection>false</ServerGarbageCollection>
</PropertyGroup>
```

**Follow-up question:**
You're running many instances of the same ASP.NET Core app on one host (e.g. many containers packed onto one node). Why might you deliberately switch away from Server GC?

**Follow-up good answer:**
Server GC's design assumes it can claim a heap and a high-priority collector thread per core — great for one instance owning the whole machine, but actively harmful when a dozen instances of the same app are all doing that simultaneously on a handful of shared cores: if their collections happen to overlap, you get many high-priority GC threads fighting over the same logical CPUs, causing exactly the kind of contention and context-switching that hurts throughput across all instances at once. In that scenario, switching to Workstation GC (optionally with concurrent/background GC disabled) reduces the number of competing high-priority threads and the resulting context-switch overhead, which can improve aggregate throughput even though each individual instance's GC is technically "slower" in isolation.

**Glossary:**
- **Workstation GC** — collects on the triggering thread at normal priority; default for standalone client apps.
- **Server GC** — dedicated heap + thread per logical CPU at highest priority; default for hosted apps like ASP.NET Core.

**Mental model:**
Checks whether the candidate can reason about GC mode choice as a function of deployment topology (one big instance vs. many small instances on shared cores) rather than treating "Server GC is faster" as universally true.

**TL;DR:**
Server GC (ASP.NET Core's default) parallelizes collection across one heap+thread per core for higher throughput, but that same design causes thread contention when many instances share a host — which is when switching to Workstation GC (with concurrent GC off) can win instead.

**References:**
- [Workstation vs. server garbage collection (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)
- [Memory management and patterns in ASP.NET Core (Microsoft Learn)](https://learn.microsoft.com/en-us/aspnet/core/performance/memory)

---

### Q7. What does "background" or "concurrent" GC actually let the rest of your app do during a collection? {#q7}

**Question:**
What does "background" or "concurrent" GC actually let the rest of your app do during a collection?

**Good answer:**
Without concurrency, a garbage collection is a fully "stop-the-world" event: every managed thread is suspended while the collector walks the heap. Concurrent (in .NET Framework 4+, called *background*) garbage collection relaxes this for generation 2 collections specifically — it lets most application threads keep running and even keep *allocating* new generation 0/1 objects while a background thread walks the older generation 2 objects looking for garbage. Both Workstation GC and Server GC in .NET Core support running in this background mode (as well as fully non-concurrent). The trade-off is that background collection takes longer wall-clock time to complete a Gen2 sweep than a stop-the-world one would, in exchange for the app staying responsive (much shorter individual pause times) throughout.

**Follow-up question:**
If background GC lets threads keep allocating during a Gen2 collection, what stops Gen0/Gen1 collections from starving it entirely?

**Follow-up good answer:**
Background GC doesn't disable ordinary Gen0/Gen1 collections while it runs — those can still happen and are handled as brief, cooperative pauses interleaved with the background Gen2 work (sometimes described as the background GC "yielding" briefly for a foreground Gen0/1 collection to complete), rather than either blocking the other indefinitely. This is exactly why background GC is a *latency* optimization, not a *throughput* one: the app keeps making forward progress and allocating, but the overall collector is doing strictly more coordination work than a simple stop-the-world collector would, so total CPU time spent on GC can be higher even as individual pauses are shorter.

**Glossary:**
- **Stop-the-world collection** — a GC pause during which all managed application threads are suspended.
- **Background/concurrent GC** — a mode allowing most app threads to keep running (and allocating) during a Gen2 collection.

**Mental model:**
Tests understanding that "concurrent GC" is a latency/pause-time trade, not a free lunch — it costs more total collector work in exchange for shorter individual stalls.

**TL;DR:**
Background/concurrent GC lets the app keep running and allocating through most of a Gen2 collection instead of stopping the world, trading some total collector overhead for much shorter individual pause times.

**References:**
- [Workstation vs. server garbage collection (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)
- [Fundamentals of garbage collection - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)

---

### Q8. How do finalizers interact with the garbage collector, and why do finalizable objects take longer to reclaim? {#q8}

**Question:**
How do finalizers interact with the garbage collector, and why do finalizable objects take longer to reclaim?

**Good answer:**
The GC only tracks finalization for types that actually override `Finalize` (a C# destructor `~Type()`) — for those, it adds an entry per instance to an internal *finalization queue*, which lists every heap object whose finalizer must run before its memory can be reclaimed. When a collection finds that a finalizable object is otherwise unreachable, it does *not* collect it immediately: instead the object is moved off the finalization queue onto a queue of objects ready to be finalized, kept alive so a dedicated finalizer thread can run its `Finalize` method, and only becomes eligible for actual memory reclamation on a *subsequent* collection once finalization has completed. That's why Microsoft's own docs state plainly that reclaiming memory takes much longer when a finalizer is involved — it requires at least two garbage collections instead of one. This is also exactly the mechanism `IDisposable`'s `Dispose()` pattern is designed to sidestep: calling `GC.SuppressFinalize(this)` from `Dispose()` removes the object from the finalization queue entirely, so a well-behaved caller that disposes deterministically avoids paying that extra-collection tax altogether.

**Code example:**
```csharp
public class Resource : IDisposable
{
    public void Dispose()
    {
        // release resources...
        GC.SuppressFinalize(this); // skip finalization entirely if Dispose() was called
    }

    ~Resource()
    {
        // fallback cleanup only if Dispose() was never called
    }
}
```

**Follow-up question:**
If a class only holds managed resources (no unmanaged handles), should it implement a finalizer "just to be safe"?

**Follow-up good answer:**
No — Microsoft's guidance is explicit that you should only override `Finalize`/write a destructor for a class that directly owns unmanaged resources (file handles, unmanaged memory, native handles), and that you shouldn't implement one for purely managed objects, because the garbage collector already reclaims managed memory automatically without any help. Adding an unnecessary finalizer only adds unfinalized-object overhead (the extra-collection cost above) with no corresponding benefit, and it also creates a maintenance hazard: finalizers run on an unspecified thread in an unspecified order relative to other objects' finalizers, so a finalizer that touches other managed objects can crash intermittently if those objects were already finalized. The recommended pattern for unmanaged resources specifically is to wrap them in a `SafeHandle`-derived type, which provides its own correctly-implemented finalizer, rather than writing one by hand.

**Glossary:**
- **Finalization queue** — internal GC structure listing heap objects whose `Finalize` must run before reclamation.
- **`GC.SuppressFinalize`** — call from `Dispose()` to remove an object from the finalization queue, avoiding the extra-collection cost.

**Mental model:**
Probes whether the candidate understands finalizers as a *deliberate, costly escape hatch* for unmanaged resources specifically, not a general-purpose cleanup mechanism — a very common junior-vs-senior distinction in .NET interviews.

**TL;DR:**
A finalizable object survives its first "dead" collection to run its finalizer and is only actually reclaimed on the next one — reclaiming it takes at least two GCs — which is exactly the extra cost `Dispose()` + `GC.SuppressFinalize` is designed to let you skip.

**References:**
- [Object.Finalize Method (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.object.finalize)
- [Implement a Dispose method - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)

---

### Q9. You suspect a service has excessive GC pressure. Which `dotnet-counters` metrics would you watch, and what pattern would confirm it? {#q9}

**Question:**
You suspect a service has excessive GC pressure. Which `dotnet-counters` metrics would you watch, and what pattern would confirm it?

**Good answer:**
`dotnet-counters monitor` (or `collect`) against the running process's `System.Runtime` provider exposes exactly the counters needed: `Gen 0/1/2 GC Count` (or the newer `dotnet.gc.collections` broken out per generation) tells you collection *frequency* per generation; `Allocation Rate` (`dotnet.gc.heap.total_allocated`) tells you how fast the app is generating garbage; `% Time in GC` is the single clearest smoking gun — the percentage of wall-clock time the process is spending inside the collector rather than doing application work; and `GC Heap Size` / `GC Fragmentation` show overall memory footprint and how much of it is unusable holes. The confirming pattern for real GC pressure is: high allocation rate, correspondingly high Gen0 (and worse, Gen1/Gen2) collection counts, and a `% Time in GC` that's meaningfully elevated (a healthy app is typically low single digits; double digits or more under normal load is a red flag) — as opposed to, say, a CPU-bound problem with no GC involvement at all, where `% Time in GC` would stay low even with the process pegged.

**Code example:**
```bash
dotnet-counters monitor --process-id 1234 \
  --counters System.Runtime[dotnet.gc.collections,dotnet.gc.heap.total_allocated]
```

**Follow-up question:**
`% Time in GC` is elevated. How do you tell whether the fix is "reduce allocations" versus "tune the GC mode/heap settings"?

**Follow-up good answer:**
Start with allocation rate: if it's very high, the root cause is almost always allocation volume in application code (boxing, unnecessary LINQ intermediate collections, string concatenation in hot paths, missing pooling) — no amount of GC tuning fixes a fundamentally too-high allocation rate, it can only shuffle where the cost lands. You confirm this with a memory profiler (`dotnet-gcdump` for a point-in-time snapshot, or BenchmarkDotNet's memory diagnoser on the specific hot method) to see *what* is allocating and how much, then fix that code. GC-mode tuning (Server vs. Workstation, concurrent on/off, `GCHeapHardLimit`) is the right lever instead when allocation rate is already reasonable but collection *behavior* is wrong for the workload — e.g. many small Gen2 promotions causing background-GC thrash, or Server GC's per-core heaps being wasteful in a densely-packed container. In short: high allocation rate → fix the code; reasonable allocation rate but bad pause/throughput pattern → tune the GC.

**Glossary:**
- **`% Time in GC`** — the fraction of wall-clock time the process spends collecting rather than running application code.
- **Allocation rate** — bytes allocated per unit time, the leading indicator of GC pressure.

**Mental model:**
Tests whether the candidate has an actual diagnostic *methodology* (which counters, in what combination, and what pattern confirms the hypothesis) rather than just knowing the tool's name.

**TL;DR:**
Watch allocation rate, per-generation collection counts, and `% Time in GC` together via `dotnet-counters` — high allocation rate plus elevated `% Time in GC` confirms real GC pressure and points you toward fixing allocations in code rather than just tuning GC settings.

**References:**
- [dotnet-counters diagnostic tool (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters)

---

### Q10. `dotnet-counters` tells you GC pressure is high. How do you find *which* objects/types are actually driving the allocations? {#q10}

**Question:**
`dotnet-counters` tells you GC pressure is high. How do you find *which* objects/types are actually driving the allocations?

**Good answer:**
`dotnet-counters` gives you the "is there a problem" signal, but it doesn't attribute allocations to specific types or call stacks — for that you move to `dotnet-gcdump`, which captures a point-in-time snapshot of the managed heap (essentially triggering a GC and dumping the surviving object graph), viewable in Visual Studio, PerfView, or `dotnet-gcdump report` to see a breakdown by type: counts, retained size, and what's holding references to what. That's ideal for finding "why is this object graph still alive" (a leak or unexpected retention) rather than "what's allocating so much." For the allocation-*rate* question specifically — which call stacks are producing the most bytes/sec, not just what's currently alive — `dotnet-trace` collecting the GC/allocation-sampling event provider gives you an actual allocation profile with call stacks, which you'd typically view in PerfView or Visual Studio's profiler. The practical workflow: `dotnet-counters` tells you there's a fire; `dotnet-trace` (allocation events) tells you which code is lighting it; `dotnet-gcdump` tells you what's stuck around afterward.

**Follow-up question:**
You take a `dotnet-gcdump` and see a large retained size under a type you don't recognize as intentionally long-lived. What's your next step?

**Follow-up good answer:**
Look at the dump's "paths to root" / reference-graph view for that type's instances — that tells you exactly what's holding a reference to them (a static field, an event subscription, a cached collection that's never trimmed, a closure captured by a long-lived delegate). The single most common real-world cause is an event handler subscription that was never unsubscribed: the publisher (often long-lived, e.g. a singleton service) holds a reference to the subscriber via its event's invocation list, which keeps the subscriber (and its whole retained graph) alive indefinitely even though nothing else in the app still needs it. Once you find the retaining root, the fix is almost always either unsubscribing/disposing appropriately, switching to a weak reference/weak event pattern, or scoping the cache with eviction instead of unbounded growth.

**Glossary:**
- **`dotnet-gcdump`** — captures a snapshot of the managed heap for offline analysis of live object types/retention.
- **`dotnet-trace`** — collects runtime event traces (including allocation sampling) for call-stack-level profiling.

**Mental model:**
Checks whether the candidate has the full diagnostic toolchain (detect → attribute-by-rate → attribute-by-retention) rather than only knowing one tool and trying to force every question through it.

**TL;DR:**
Use `dotnet-trace`'s allocation events to find which call stacks are allocating fastest, and `dotnet-gcdump`'s heap snapshot plus paths-to-root to find what's unexpectedly retaining memory — they answer different questions ("what's allocating" vs. "what's still alive and why").

**References:**
- [dotnet-counters diagnostic tool (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters)
- [.NET runtime metrics (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-runtime)

---

### Q11. How does BenchmarkDotNet's `MemoryDiagnoser` actually measure allocations, and what do the Gen0/Gen1/Gen2 columns mean? {#q11}

**Question:**
How does BenchmarkDotNet's `MemoryDiagnoser` actually measure allocations, and what do the Gen0/Gen1/Gen2 columns mean?

**Good answer:**
`MemoryDiagnoser` is an opt-in (not default) diagnoser that reports GC and allocation statistics per benchmark. It measures total bytes allocated using `GC.GetAllocatedBytesForCurrentThread`, a cross-platform API, giving an "Allocated" column that BenchmarkDotNet's own documentation describes as ~99.5% accurate under default or longer job settings. The Gen0/Gen1/Gen2 columns don't show a raw per-run collection count; they show the number of GC collections *per 1,000 operations* of that generation — so a Gen0 value like 0.12 means roughly one Gen0 collection happens for every ~8,000 invocations of the benchmarked method, giving you a normalized sense of collection frequency regardless of how many iterations the benchmark actually ran. In practice you enable it with `[MemoryDiagnoser]` on the benchmark class, and a healthy micro-optimization shows both a lower "Allocated" number and correspondingly lower Gen0 frequency after your change.

**Code example:**
```csharp
[MemoryDiagnoser]
public class StringBenchmarks
{
    [Benchmark]
    public string Concat() => "a" + "b" + "c";

    [Benchmark]
    public string Interpolate() => $"{"a"}{"b"}{"c"}";
}
```

**Follow-up question:**
Two implementations show the same "Allocated" bytes, but one shows a noticeably higher Gen0 collection frequency. What might that indicate, and does it matter?

**Follow-up good answer:**
Since "Allocated" is total bytes and the Gen-count columns are collections per 1,000 ops, an equal total-bytes figure with a higher Gen0 frequency for the same op count would typically mean the allocations are happening in a pattern that fills Gen0's budget faster per operation — e.g. many small, short-lived allocations per call versus fewer, larger ones, or allocations that happen to line up unfavorably with the Gen0 budget/segment size at benchmark time. It matters because collection frequency, not just total bytes, is what shows up as real-world latency: more frequent Gen0 pauses (even individually cheap ones) means more total interruptions to your hot path, which can matter for tail latency even when aggregate throughput/memory looks identical on paper. When you see this, it's worth also checking object *size and count* distribution (e.g. via `dotnet-trace` allocation profiling) rather than trusting total-bytes alone as the complete picture.

**Glossary:**
- **`MemoryDiagnoser`** — a BenchmarkDotNet diagnoser reporting allocated bytes and per-generation GC frequency per benchmark.
- **`GC.GetAllocatedBytesForCurrentThread`** — the cross-platform API BenchmarkDotNet uses to measure allocation.

**Mental model:**
Tests whether the candidate can read benchmark output correctly (per-1000-ops normalization) rather than misinterpreting the Gen columns as raw counts — a real footgun when comparing benchmarks with different iteration counts.

**TL;DR:**
`MemoryDiagnoser` reports total allocated bytes (via `GC.GetAllocatedBytesForCurrentThread`) plus per-generation GC frequency normalized per 1,000 operations — both matter, since equal total bytes with higher collection frequency can still mean worse real-world pause behavior.

**References:**
- [BenchmarkDotNet Diagnosers — MemoryDiagnoser](https://benchmarkdotnet.org/articles/configs/diagnosers.html)

---

### Q12. Your service has occasional large latency spikes correlated with big allocations. How do you diagnose Large Object Heap fragmentation specifically? {#q12}

**Question:**
Your service has occasional large latency spikes correlated with big allocations. How do you diagnose Large Object Heap fragmentation specifically?

**Good answer:**
Objects ≥ 85,000 bytes land on the LOH, which — unlike the small object heap — historically isn't compacted by default: dead large objects are swept (freed in place) rather than the survivors being moved together, which over time can leave the LOH full of unusable gaps between live objects (fragmentation), forcing the heap to grow even though the *live* data size hasn't grown. `dotnet-counters`' `dotnet.gc.last_collection.heap.fragmentation.size` (or the older "GC Fragmentation %" counter), broken out per generation including `loh`, is the direct signal: rising LOH fragmentation alongside a growing overall process working set despite stable live-object counts points squarely at this. A `dotnet-gcdump` heap snapshot confirms it by showing LOH segment size versus actual live large-object bytes. The fix options are: reduce the number/frequency of large allocations (pool large buffers instead of allocating fresh ones, or keep objects under 85,000 bytes where feasible), or explicitly opt into LOH compaction via `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce` before a forced `GC.Collect()`, which is an occasional, deliberate maintenance operation rather than something to do routinely (compacting the LOH is expensive precisely because the objects are large).

**Follow-up question:**
Why would pooling large buffers with `ArrayPool<T>` specifically help avoid this problem in the first place?

**Follow-up good answer:**
Every time you allocate-and-discard a buffer ≥ 85,000 bytes, you create a fresh LOH allocation that will eventually die and leave a gap when swept — do that repeatedly with varying sizes and you get exactly the fragmentation pattern described above. `ArrayPool<T>.Rent`/`Return` sidesteps this by reusing the *same* set of large arrays across many operations instead of continuously allocating and abandoning new ones on the LOH, so the LOH's set of live large objects stays comparatively stable rather than constantly cycling — dramatically reducing both the allocation *rate* (fewer Gen2/LOH collections triggered) and the fragmentation that repeated LOH churn causes. This is precisely why high-throughput scenarios involving large buffers (network I/O, big serialization payloads) are one of `ArrayPool<T>`'s primary intended use cases.

**Glossary:**
- **LOH fragmentation** — unusable gaps left in the Large Object Heap after dead large objects are swept without compaction.
- **`GCSettings.LargeObjectHeapCompactionMode`** — opt-in flag to compact the LOH on the next full collection.

**Mental model:**
Probes whether the candidate connects a specific diagnostic counter pattern (fragmentation + stable-but-growing footprint) to a specific, non-obvious root cause (LOH's non-default compaction) rather than reaching for "must be a leak" reflexively.

**TL;DR:**
LOH fragmentation shows up as a growing process footprint despite stable live large-object counts, confirmed via `dotnet-counters`' fragmentation metric or a `dotnet-gcdump`; pooling large buffers with `ArrayPool<T>` (or, rarely, an explicit one-off LOH compaction) is the fix.

**References:**
- [Large object heap on Windows - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap)
- [ArrayPool<T> Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)

---

### Q13. Why does reducing allocation *count* often matter more for throughput than reducing GC pause *time* directly? {#q13}

**Question:**
Why does reducing allocation *count* often matter more for throughput than reducing GC pause *time* directly? Walk through the underlying cost model.

**Good answer:**
Every allocation has an amortized cost even before any collection happens: bumping the allocation pointer, occasionally triggering a Gen0 collection when the budget is exhausted, and — critically — the *rate* of allocation is what determines *how often* any collection (even a cheap Gen0 one) has to run at all. A service allocating aggressively might trigger thousands of Gen0 collections per second even if each one is individually fast; the cumulative interruption and cache-pollution cost (every collection walks live objects, which evicts your working set from CPU cache) adds up in aggregate throughput terms even when no single pause is user-visible. Trying to fix this by tuning GC *pause time* settings (e.g. `SustainedLowLatency`) doesn't address the root cause — it just reshapes when and how the same underlying collection work happens, and can even make total throughput worse by deferring more work into larger, rarer collections. Reducing the allocation rate itself (pooling, `Span<T>`/`stackalloc` for transient buffers, avoiding boxing, avoiding unnecessary LINQ allocation) reduces how often the GC has to run in the first place, which is the throughput lever that actually compounds.

**Follow-up question:**
Does this mean pause-time-focused settings like `SustainedLowLatency` are never useful for a throughput-sensitive service?

**Follow-up good answer:**
They're useful, but for a different axis of the problem: pause-time settings target *latency consistency* (avoiding a long, disruptive collection at an inopportune moment — e.g. mid-request in a low-latency API), not raw throughput. A service can have a perfectly reasonable allocation rate and still occasionally suffer an unlucky full blocking collection that spikes p99 latency; `SustainedLowLatency` specifically biases the collector to avoid full blocking collections over an extended period, trading some memory growth for smoother latency. The two levers aren't in competition — a well-tuned service typically does both: minimizes allocation rate first (because it's the bigger lever and helps throughput and latency together), then applies a latency mode if there's still an unacceptable tail-latency pattern from the collections that remain.

**Glossary:**
- **Amortized allocation cost** — the average per-allocation overhead once bump-pointer allocation and periodic collection costs are spread across many allocations.
- **Throughput vs. latency (GC)** — allocation-rate reduction primarily helps throughput/overall collector load; latency-mode settings primarily smooth pause-time consistency.

**Mental model:**
Tests whether the candidate has internalized the *cost model* behind "allocations matter" rather than repeating it as received wisdom — specifically, distinguishing throughput-oriented fixes (reduce allocation rate) from latency-oriented fixes (tune GC mode).

**TL;DR:**
Allocation rate drives how *often* the collector runs at all, which compounds across the whole app's throughput — that's a bigger lever than tuning individual pause behavior, which mainly smooths latency consistency rather than reducing total collector work.

**References:**
- [Fundamentals of garbage collection - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
- [GCLatencyMode Enum (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.gclatencymode)

---

### Q14. Give a concrete real-world scenario where `Span<T>` eliminates allocations that would otherwise happen on every request. {#q14}

**Question:**
Give a concrete real-world scenario where `Span<T>` eliminates allocations that would otherwise happen on every request.

**Good answer:**
A classic case is parsing: given a request line like `"GET /users/42 HTTP/1.1"`, a naive implementation calls `.Split(' ')`, which allocates a new `string[]` plus a new `string` for every substring, on every single request. Using `ReadOnlySpan<char>` (via `line.AsSpan()`) and `IndexOf`/slicing operations instead lets you carve out the method, path, and version as zero-copy *views* into the original string's memory — no new string allocations at all for the intermediate pieces, and you only allocate a final string (if you even need one) for the specific substring you actually keep. The same pattern applies to numeric parsing (`int.Parse(ReadOnlySpan<char>)` overloads avoid an intermediate substring just to parse a number out of a larger buffer) and to network/serialization code walking a byte buffer without slicing it into copies at every step. Multiply "avoid 3-4 allocations" by a high-throughput service handling thousands of requests per second, and this is a direct, measurable reduction in Gen0 collection frequency — exactly the kind of change that shows up as a lower Gen0 count in a BenchmarkDotNet `MemoryDiagnoser` run.

**Code example:**
```csharp
ReadOnlySpan<char> line = "GET /users/42 HTTP/1.1".AsSpan();
int firstSpace = line.IndexOf(' ');
ReadOnlySpan<char> method = line[..firstSpace];               // no allocation
ReadOnlySpan<char> rest = line[(firstSpace + 1)..];
int secondSpace = rest.IndexOf(' ');
ReadOnlySpan<char> path = rest[..secondSpace];                 // no allocation
```

**Follow-up question:**
If `Span<T>`-based parsing avoids allocations for the intermediate pieces, why would you still ever allocate a `string` at the end?

**Follow-up good answer:**
Because a `Span<T>`/`ReadOnlySpan<T>` is a stack-only view with a lifetime tied to its source — you can't store it as a field, return it from an API that outlives the current call in most shapes, put it in a collection, or hand it to code that needs an actual `string`/heap object for identity or longer-lived storage (e.g. a dictionary key, a value cached beyond the current request, or an API contract that takes `string`). You allocate at the boundary where the data genuinely needs to outlive the zero-copy view or cross an API surface that requires a real object — the win isn't "never allocate a string," it's "don't allocate N intermediate throwaway strings when you only actually need to materialize the one piece you're keeping."

**Glossary:**
- **Zero-copy parsing** — extracting substructure from a buffer via slices/views instead of copying into new allocations.
- **`ReadOnlySpan<char>.Slice`/range indexers** — carve a view into existing memory without allocating.

**Mental model:**
Tests whether the candidate can connect `Span<T>`'s abstract "avoids allocation" property to a concrete, request-path scenario an interviewer can picture actually mattering in production.

**TL;DR:**
`Span<T>`-based slicing turns per-request parsing that used to allocate several intermediate strings/arrays into zero-copy views over existing memory, allocating only the one piece you actually need to keep — a direct, measurable Gen0-reduction win at scale.

**References:**
- [System.Span<T> struct - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-span%7BT%7D)

---

### Q15. Why can't you pass a `Span<T>` as a parameter to an `async` method, and how does this actually bite you in practice? {#q15}

**Question:**
Why can't you pass a `Span<T>` as a parameter to an `async` method, and how does this actually bite you in practice?

**Good answer:**
An `async` method's local state (including its parameters, once it needs to suspend) gets captured into a compiler-generated state machine object so execution can resume later — and that state machine, to survive across an `await`, generally ends up allocated on the heap. Since a `ref struct` like `Span<T>` is specifically forbidden from being stored anywhere that could put it on the heap (it might be holding a stack pointer that won't be valid later), the compiler disallows `ref struct` parameters on `async` methods outright (before C# 13), and even with C# 13's relaxation, you still can't have a `ref struct` variable alive across an actual `await` or `yield` point within the method. In practice this bites you when you're writing a method that both needs to do something `Span<T>`-based *and* something asynchronous — you can't just sprinkle `async`/`await` into a method that takes a `Span<T>` parameter and expect it to compile.

**Code example:**
```csharp
// Won't compile: Span<T> can't be a parameter of an async method
// async Task ProcessAsync(Span<byte> data) { await Task.Delay(1); }

// Works: do the sync Span<T> work first, await afterward with no ref struct alive across it
void ProcessSync(Span<byte> data) { /* touch the span here */ }

async Task ProcessAsync(Memory<byte> data)
{
    ProcessSync(data.Span); // Span<T> obtained and used, out of scope before any await
    await Task.Delay(1);
}
```

**Follow-up question:**
Given that restriction, how would you design an API that needs to both process a buffer synchronously and eventually do async I/O with it?

**Follow-up good answer:**
Take `Memory<T>`/`ReadOnlyMemory<T>` as the method parameter or field type — since it's heap-safe, it can be stored, passed across `await` boundaries, and captured by the async state machine without restriction. Then, within any *synchronous* section of the method (before or after an `await`, never spanning one), call `.Span` to get a temporary `Span<T>` view for the actual data manipulation, and let that `Span<T>` go out of scope before the next `await`. This is precisely the design pattern behind APIs like `Stream.WriteAsync(ReadOnlyMemory<byte> buffer, ...)` — the public surface uses the async-safe `Memory<T>`, and internally, wherever the implementation needs synchronous, allocation-free access to the bytes, it drops down to `.Span` for that scoped portion of the work.

**Glossary:**
- **Async state machine** — the compiler-generated object holding an `async` method's suspended state across `await` points.
- **`Memory<T>`/`.Span`** — the async-safe wrapper, and the temporary stack-only view you get from it for synchronous work.

**Mental model:**
Checks whether the candidate can predict and explain a specific, commonly-hit compiler error rather than only knowing the abstract "ref struct is stack-only" rule in isolation.

**TL;DR:**
`Span<T>` can't survive in an async state machine because that state machine may live on the heap, so async APIs take `Memory<T>` instead and drop down to `.Span` only for strictly synchronous portions of the work, never across an `await`.

**References:**
- [ref struct types - C# reference (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct)

---

### Q16. What is boxing, and why can it silently degrade the performance of otherwise "efficient" value-type code? {#q16}

**Question:**
What is boxing, and why can it silently degrade the performance of otherwise "efficient" value-type code?

**Good answer:**
Boxing is the CLR wrapping a value type in a heap-allocated object so it can be used somewhere that expects a reference type — most commonly `object`, or a non-generic interface. It's an allocation, full stop: every boxed value is a new heap object with its own header, subject to GC tracking exactly like any reference type. The danger is that boxing often happens *implicitly*, without an obvious syntactic cue: passing a struct to a method parameter typed `object` (including old-style non-generic collections or `string.Format`/`Console.WriteLine` overloads before params/generic overloads existed), storing a struct in a non-generic `ArrayList`, or calling a virtual method on a value type through an interface it implements (calling an interface method on a struct via the interface type, rather than the concrete struct type, typically boxes it) can all box silently. Code that looks like it's using cheap, allocation-free structs can, in reality, be boxing on every call — completely defeating the point of choosing a struct in the first place.

**Code example:**
```csharp
struct Point { public int X, Y; }

object boxed = new Point(); // boxing: heap allocation
int Sum(object o) => ((Point)o).X; // unboxing on cast

// silent boxing via non-generic collection:
System.Collections.ArrayList list = new();
list.Add(new Point()); // boxes the struct
```

**Follow-up question:**
Generic collections like `List<T>` were introduced specifically to fix this for collections — how, mechanically, do they avoid boxing where `ArrayList` didn't?

**Follow-up good answer:**
`ArrayList` stores `object` internally, so every value type added to it must be boxed to fit that storage type — there's no way around it given its signature. `List<T>` (and generics generally) are compiled so that, for a value-type type argument, the JIT generates a specialized implementation with the value type stored directly and contiguously (an internal `T[]` array of actual `Point` structs, not boxed `object` references to scattered `Point` instances), because `T` is resolved to the concrete value type rather than erased to `object`. That's the core reason .NET generics exist as a runtime feature rather than a compile-time-only trick like Java's type erasure: it lets `List<int>` store actual `int`s inline with zero boxing, versus `ArrayList` where every `int` added was a separate boxed heap object.

**Glossary:**
- **Boxing** — wrapping a value type in a heap-allocated object to use it as `object`/an interface type.
- **Unboxing** — extracting the value back out of a boxed object (via a cast), copying it back to a value-type variable.

**Mental model:**
Tests whether the candidate can spot the *implicit*, easy-to-miss triggers for boxing (interfaces, `object` parameters, legacy non-generic APIs) rather than only recognizing the single obvious textbook example.

**TL;DR:**
Boxing silently allocates whenever a value type is used as `object`/a non-generic interface — including through easy-to-miss paths like interface method calls on structs or legacy non-generic collections — which is exactly the hidden-allocation problem generics were introduced to eliminate.

**References:**
- [Boxing and Unboxing (C# Programming Guide) (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)

---

### Q17. What does `GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency` actually change, and what's the trade-off? {#q17}

**Question:**
What does `GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency` actually change, and what's the trade-off?

**Good answer:**
`SustainedLowLatency` is one of five `GCLatencyMode` values (`Batch`, `Interactive` — the default, `LowLatency`, `SustainedLowLatency`, `NoGCRegion`). Per the official enum documentation, it "enables garbage collection that tries to minimize latency over an extended period" by having the collector try to perform only generation 0, generation 1, and *concurrent* generation 2 collections — explicitly trying to avoid full, blocking collections for as long as possible. It's meant for scenarios that need consistently low pause times over a sustained window, like an interactive UI app or a latency-sensitive service handling live traffic, as opposed to `LowLatency` mode (which is more aggressive about avoiding full collections but isn't available under Server GC and is meant for shorter critical sections). The trade-off, stated directly in the docs: "full blocking collections may still occur if the system is under memory pressure" — so it's not a hard guarantee, and by deferring full Gen2 collections, memory usage can grow larger than it would under the default mode before the system is eventually forced to do a full collection anyway.

**Code example:**
```csharp
System.Runtime.GCSettings.LatencyMode = System.Runtime.GCLatencyMode.SustainedLowLatency;
// ... perform the latency-sensitive operation ...
System.Runtime.GCSettings.LatencyMode = System.Runtime.GCLatencyMode.Interactive; // restore default
```

**Follow-up question:**
Why is `SustainedLowLatency` a better fit than `LowLatency` for a long-running ASP.NET Core service, rather than the other way around?

**Follow-up good answer:**
`LowLatency` mode is documented as "not available for the server garbage collector" — since ASP.NET Core defaults to Server GC, `LowLatency` simply isn't usable in the typical hosting scenario at all. Beyond that availability constraint, `LowLatency`'s design intent is a short, bounded critical section where you want to aggressively suppress collections temporarily (e.g. around a specific latency-critical operation) and then return to normal — not something you'd leave engaged for a service's entire runtime, since deferring collections that aggressively for an extended period risks unbounded memory growth. `SustainedLowLatency`, by contrast, is explicitly designed to be sustained over an extended period while still allowing background/concurrent Gen2 collections to happen (just trying to avoid *full blocking* ones), making it the mode that actually fits "run this way for the service's whole lifetime" rather than "wrap one operation."

**Glossary:**
- **`GCLatencyMode`** — an enum controlling how aggressively the GC avoids intrusive (especially full blocking) collections.
- **`SustainedLowLatency`** — mode minimizing latency over an extended period by avoiding full blocking collections where possible.

**Mental model:**
Tests whether the candidate knows the specific trade-offs and constraints (Server GC incompatibility for `LowLatency`, "not a guarantee" caveat) rather than treating latency modes as an unconditional "make GC faster" switch.

**TL;DR:**
`SustainedLowLatency` biases the collector to avoid full blocking Gen2 collections over an extended period (unlike `LowLatency`, which isn't even available under Server GC) — at the cost of no hard guarantee and potentially higher sustained memory use.

**References:**
- [GCLatencyMode Enum (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.gclatencymode)

---

### Q18. Does Native AOT compilation change which garbage collector your app uses? {#q18}

**Question:**
Does Native AOT compilation change which garbage collector your app uses?

**Good answer:**
No — Native AOT changes the *compilation and execution model*, not the memory manager. Native AOT ahead-of-time compiles your IL straight to native machine code at publish time; the resulting app doesn't use a just-in-time compiler at all when it runs, and it's self-contained, including — per Microsoft's own description — "a stripped-down version of the coreclr runtime" bundled into the published output. That stripped-down CoreCLR is still the same runtime family that provides the GC, so a Native AOT app still allocates on a generational managed heap and is still collected by (a build of) the same garbage collector used by the normal JIT-based CLR — Native AOT's actual limitations are things like no `Assembly.LoadFile`, no `System.Reflection.Emit`, and generic instantiations over value types being fully pre-generated at publish time rather than created on demand, none of which changes the GC's fundamental behavior.

**Follow-up question:**
Given that the GC is unchanged, why does Native AOT still get pitched primarily as a *performance* improvement?

**Follow-up good answer:**
Because its performance win comes from a different part of the runtime cost than GC: eliminating the JIT means no time spent compiling methods at startup (a major contributor to cold-start latency for JIT-based apps, especially short-lived ones like CLI tools or scale-to-zero cloud functions), and it produces a smaller, self-contained deployment without needing the full JIT/dynamic-loading machinery present at runtime. Microsoft's docs are explicit that the benefit is "most significant for workloads with a high number of deployed instances, such as cloud infrastructure and hyper-scale services" — i.e. scenarios dominated by *startup time and footprint* across many short-lived or frequently-restarted instances, not scenarios dominated by steady-state GC throughput, which Native AOT doesn't specifically improve since it's running the same collector doing the same generational collection work once the app is up and running.

**Glossary:**
- **Native AOT** — ahead-of-time compilation of IL to native code at publish time, removing the JIT from the runtime app.
- **CoreCLR** — the .NET runtime implementation providing the GC, type system, and (normally) the JIT; Native AOT ships a stripped-down build of it without the JIT.

**Mental model:**
Tests whether the candidate can correctly scope what a well-known feature (Native AOT) does and doesn't change — distinguishing "compilation/startup model" from "memory management model," which are easy to conflate.

**TL;DR:**
Native AOT removes the JIT and bundles a stripped-down CoreCLR, but that stripped-down runtime still provides the same generational GC — Native AOT's performance win is in startup time/footprint, not in changing how garbage collection itself works.

**References:**
- [Native AOT deployment overview - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

---

### Q19. Design question: you're building a high-throughput binary protocol parser that must sustain very low p99 latency. Walk through the memory-management decisions you'd make. {#q19}

**Question:**
Design question: you're building a high-throughput binary protocol parser that must sustain very low p99 latency. Walk through the memory-management decisions you'd make.

**Good answer:**
Start from the allocation-rate-first principle: the biggest lever for both throughput and latency consistency is minimizing how often the GC has to run at all. For incoming byte buffers, use `ArrayPool<byte>.Shared.Rent`/`Return` instead of allocating a fresh array per message — this is exactly the "arrays created and destroyed frequently" scenario the API is built for. For parsing the buffer's contents (headers, length-prefixed fields, numeric values), use `ReadOnlySpan<byte>` slicing rather than copying substrings/subarrays out — zero-copy views over the rented buffer, with numeric parsing done directly against spans where the BCL supports it. Where a fixed small scratch buffer is needed for transient work within a single synchronous call, `stackalloc` under a conservative size guard avoids even the pooling overhead. For the hosting environment, since this sounds like a server workload, Server GC is the sensible default — but if this parser runs as one of many densely-packed instances per host (containers), that default needs revisiting per Q6. Finally, for sustained low p99 specifically, `GCLatencyMode.SustainedLowLatency` is worth setting for the process's lifetime, since it directly targets avoiding the disruptive full blocking collections that would otherwise show up as latency spikes.

**Code example:**
```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(expectedSize);
try
{
    ReadOnlySpan<byte> data = buffer.AsSpan(0, bytesReceived);
    int length = BinaryPrimitives.ReadInt32BigEndian(data[..4]); // zero-copy
    ReadOnlySpan<byte> payload = data.Slice(4, length);          // zero-copy
    // ... process payload ...
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

**Follow-up question:**
How would you actually validate that these choices worked, rather than assuming they did because the APIs are "known to be fast"?

**Follow-up good answer:**
Measure, don't assume — the same discipline the References sections throughout this set point back to. Before and after each change, run a BenchmarkDotNet benchmark with `[MemoryDiagnoser]` on the parsing hot path to confirm "Allocated" bytes and Gen0 frequency actually dropped (a `Span<T>` rewrite that still boxes somewhere, or an `ArrayPool` usage that forgot to `Return`, wouldn't show the expected improvement, and the benchmark would catch that immediately). Under realistic load, watch `dotnet-counters`' allocation rate and `% Time in GC` to confirm the improvement holds up outside the microbenchmark, and take a `dotnet-gcdump` periodically to confirm nothing is being unintentionally retained (e.g. a rented buffer whose reference leaked somewhere and never returned, quietly reducing the pool's effectiveness). And for the latency-mode change specifically, track actual p99/p999 request latency under load before and after — `SustainedLowLatency` is a hypothesis about pause behavior, not a guarantee, and the only way to know it helped *this* workload is to measure the tail latency directly.

**Glossary:**
- **p99 latency** — the 99th-percentile request latency; a common tail-latency SLO metric sensitive to occasional long GC pauses.
- **`BinaryPrimitives`** — BCL helpers for reading/writing primitive values directly from/to byte spans without intermediate allocation.

**Mental model:**
A synthesis question checking whether the candidate can combine everything in this set (generational GC, `Span<T>`/`Memory<T>`, `ArrayPool<T>`, `stackalloc`, GC mode choice, and diagnostic tooling) into one coherent design, and whether they instinctively reach for measurement rather than assumption.

**TL;DR:**
A low-latency high-throughput parser combines `ArrayPool<T>` for buffers, `Span<T>`-based zero-copy parsing, `stackalloc` for small scratch space, a deliberately-chosen GC mode (Server GC plus possibly `SustainedLowLatency`), and — critically — validation via `MemoryDiagnoser`, `dotnet-counters`, and real p99 measurement rather than assuming the "known-fast" APIs alone guarantee the result.

**References:**
- [ArrayPool<T> Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
- [BenchmarkDotNet Diagnosers — MemoryDiagnoser](https://benchmarkdotnet.org/articles/configs/diagnosers.html)
- [dotnet-counters diagnostic tool (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters)

---

### Q20. Struct vs. class: beyond "value type vs. reference type," what's the concrete decision criteria for a high-frequency allocated type? {#q20}

**Question:**
Struct vs. class: beyond "value type vs. reference type," what's the concrete decision criteria for a high-frequency allocated type?

**Good answer:**
For a type that will be created in large volumes on a hot path, a struct avoids a per-instance heap allocation and its GC tracking overhead when used as a local, a field, or an array element — for a million-element collection, an array of structs is one contiguous allocation versus an array of class references being one array allocation *plus* a million separate object allocations. But structs have real costs of their own that make them the wrong choice past a certain point: they're copied by value on assignment/parameter-passing, so a large struct (rule of thumb: much more than ~16 bytes, though "it depends on the field layout and usage pattern" is the honest answer) can make copying more expensive than the allocation you avoided; mutable structs are a notorious footgun (mutating a copy instead of the original, especially through properties or LINQ, silently does nothing useful); and structs can't participate in inheritance-based polymorphism the way classes can (though they can implement interfaces). The practical decision: small, immutable, frequently-created-in-bulk data (coordinates, money amounts, small value objects, array elements in a hot loop) favors structs; anything large, mutable-by-design, or needing reference semantics/polymorphism favors a class — and when in doubt, the `Span<T>`/`ArrayPool<T>`/pooling techniques from this set are usually a bigger lever than the struct-vs-class choice alone.

**Follow-up question:**
Why does that "prefer immutable" guidance matter specifically for structs, more so than for classes?

**Follow-up good answer:**
Because struct copy-by-value semantics make mutable structs actively dangerous in ways that don't have a class equivalent: if a struct is copied (passed to a method by value, read from a property getter, enumerated from a `List<T>` via `foreach`'s value-copy semantics in older patterns, boxed and unboxed), mutating that copy has no effect on the original — a class reference, by contrast, always points at the same single object, so mutating through any reference to it is visible everywhere. This produces a specific, well-known bug class: code that looks like it's mutating a struct in place (`myList[i].SomeField = x` on a struct-typed property, or mutating a struct returned from a property getter) either fails to compile or silently mutates a throwaway copy, doing nothing observable. Making structs immutable (`readonly struct`, no exposed mutating methods) sidesteps the entire bug class by making "did I mutate the original or a copy" a non-question — there's nothing to mutate, you always construct a new value instead.

**Glossary:**
- **Copy-by-value semantics** — a struct's contents are copied whenever the struct is assigned, passed, or returned, unlike a class reference.
- **`readonly struct`** — a struct the compiler verifies has no mutable state, eliminating the copy-mutation footgun.

**Mental model:**
A closing trade-offs question tying the whole file together — checking that the candidate has moved past "struct = fast, class = slow" to a real cost-model understanding of when each choice actually wins.

**TL;DR:**
Structs win for small, immutable, high-volume data by avoiding per-instance heap allocations, but copy-by-value semantics make mutable structs a serious correctness footgun — favor `readonly struct` for anything small enough to justify the choice at all.

**References:**
- [Value types - C# reference (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=dotnet&tags=memory-management-gc-and-high-performance-types&autostart=1" | relative_url }})
