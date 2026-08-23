---
layout: default
title: ".NET Interview — CLR & Async Internals"
---

# .NET Interview — CLR & Async Internals

This set covers the CLR's execution and memory model — value vs. reference types, boxing, the garbage collector's generational design, JIT/tiered compilation and Native AOT — and the mechanics of `async`/`await`: the compiler-generated state machine, `SynchronizationContext`, `ThreadPool` starvation, and the deadlocks and leaks that trip up even experienced .NET developers. It also covers the diagnostic tools (`dotnet-trace`, `dotnet-counters`, `dotnet-gcdump`, BenchmarkDotNet) senior engineers reach for when something is slow.

### Q1. What's the difference between a value type and a reference type in C#, and where does each actually live in memory?

**Question:**
What's the difference between a value type and a reference type in C#, and where does each actually live in memory?

**Good answer:**
A variable of a value type (`struct`, `enum`, and built-ins like `int`, `bool`, `double`) directly contains its data; assigning it or passing it copies the whole value. A variable of a reference type (`class`, `interface`, `delegate`, `array`, `string`) holds a reference (pointer) to an object on the managed heap; copying the variable copies the reference, not the object. The common "value types live on the stack, reference types live on the heap" rule of thumb is a simplification: a value type is stored inline wherever it's declared — if it's a local variable it's typically stack-allocated, but if it's a field of a class instance, it's stored inline inside that object on the heap. Reference types are always heap-allocated (the object itself), with only the reference/pointer stored wherever the variable lives.

**Code example:**
```csharp
struct Point { public int X, Y; }   // value type
class Box { public int Value; }     // reference type

void Demo()
{
    Point p1 = new Point { X = 1 };
    Point p2 = p1;      // copies the struct — p2 is independent
    p2.X = 99;          // p1.X is still 1

    Box b1 = new Box { Value = 1 };
    Box b2 = b1;         // copies the reference — both point to the same object
    b2.Value = 99;        // b1.Value is now 99 too
}
```

**Follow-up question:**
If a `struct` is a field inside a `class`, where does it actually get stored?

**Follow-up good answer:**
It's stored inline as part of the containing object's memory layout on the managed heap — there's no separate boxed allocation for it. The struct's fields sit contiguously inside the class instance, so allocating the class object also allocates the struct's storage as part of that same block. It only gets a separate heap allocation if it's explicitly boxed (e.g., assigned to an `object` or interface-typed variable).

**Glossary:**
- **Value type** — a type whose variable directly holds its data (`struct`, `enum`, primitives).
- **Reference type** — a type whose variable holds a reference to heap-allocated data (`class`, `array`, `string`, `delegate`, `interface`).
- **Managed heap** — the memory region managed and garbage-collected by the CLR.

**Mental model:**
This question checks whether the candidate has the "stack vs. heap" myth or the accurate mental model — that storage location is determined by *where the variable is declared*, not by whether the type is a value or reference type in isolation.

**References:**
- [Value types - C# reference | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)
- [Reference types - C# reference | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/reference-types)

---

### Q2. What happens during boxing, and why is it expensive?

**Question:**
What happens during boxing, and why is it expensive?

**Good answer:**
Boxing converts a value type to `object` (or to an interface it implements) by allocating a new object on the managed heap and copying the value type's data into it — implicitly, whenever the compiler needs an `object`/interface reference from a value type. Unboxing is the reverse: it checks that the boxed object is actually the expected value type, then copies the value back out into a value-type variable; it's explicit (a cast) and throws `InvalidCastException`/`NullReferenceException` on a bad cast or a null. Boxing is expensive for two reasons: it's a heap allocation (adding GC pressure), and it defeats the whole point of using a value type (avoiding heap allocation and enabling stack-friendly, cache-local storage). It's a classic hidden-cost gotcha in code like non-generic collections (`ArrayList`) or `string.Format` overloads that take `object[]`, where every `int` you pass gets boxed.

**Code example:**
```csharp
int i = 123;
object o = i;       // boxing: heap allocation + copy
int j = (int)o;      // unboxing: type check + copy back
```

**Follow-up question:**
How do generics help avoid boxing, and is there any case where a generic method can still box a value type?

**Follow-up good answer:**
Generics avoid boxing because the JIT generates specialized native code per value-type instantiation (e.g., `List<int>` gets its own compiled implementation with `int` stored inline, unlike `ArrayList` which stores `object` and boxes every element). Boxing can still sneak back in inside a generic method if the value is cast to a non-generic interface or `object` — e.g., calling `.ToString()` is fine (it's overridden per-struct and doesn't box), but passing a generic `T` value to a method expecting `object`, or storing it in an `object`-typed field, still boxes it even inside generic code.

**Glossary:**
- **Boxing** — implicit conversion of a value type into a heap-allocated `object`.
- **Unboxing** — explicit conversion back from the boxed `object` to the value type.

**Mental model:**
Tests whether the candidate connects a language-level implicit conversion to its concrete performance cost (allocation + GC pressure), not just the mechanical definition.

**References:**
- [Boxing and Unboxing - C# | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)

---

### Q3. What does the compiler actually generate for an `async` method?

**Question:**
What does the compiler actually generate for an `async` method?

**Good answer:**
The compiler rewrites the method body into a compiler-generated state machine (a struct or class implementing `IAsyncStateMachine`), moving your code into its `MoveNext()` method. `MoveNext()` uses a saved integer state and a `switch` to jump back to the point right after the last `await`. When it hits an `await`, it checks whether the awaited task is already completed: if so, execution continues synchronously in the same `MoveNext()` call (no allocation, no yielding of the thread). If not, it registers a continuation with the awaiter (via `OnCompleted`), and `MoveNext()` returns — control goes back to the caller immediately (this is what makes it non-blocking). When the awaited task later completes, its continuation calls `MoveNext()` again, resuming exactly where it left off using the state machine's saved locals. If the whole method actually completes synchronously (never truly suspends), the state machine can stay a struct and avoid a heap allocation entirely — the compiler only "boxes" it onto the heap when the method genuinely needs to suspend and resume later.

**Follow-up question:**
Why does an `async` method's exception behave differently from a synchronous method's — i.e., why doesn't a `try/catch` around the *call site* always catch it the way you'd expect?

**Follow-up good answer:**
An exception thrown inside an `async Task`/`async Task<T>` method isn't propagated synchronously to the caller — it's captured and stored on the returned `Task` object (as a faulted state). It only surfaces as a thrown exception when that task is `await`ed (or `.Result`/`.Wait()` is called, wrapped in an `AggregateException` for the latter). So a `try/catch` wrapped around a call that isn't `await`ed (fire-and-forget) won't catch anything — the exception is sitting on a task nobody observed. This is compounded by `async void` methods, which have no `Task` to store the exception on, so their exceptions are thrown directly on whatever `SynchronizationContext` was active, often crashing the process instead of being catchable at all.

**Glossary:**
- **State machine** — the compiler-generated type that implements the suspend/resume logic of an `async` method.
- **Awaiter** — the object (from `GetAwaiter()`) that exposes `IsCompleted`, `OnCompleted`, and `GetResult()` to the `await` machinery.
- **Continuation** — the callback registered to resume the state machine once an awaited operation completes.

**Mental model:**
Distinguishes candidates who've memorized "async makes it non-blocking" from those who understand the actual mechanism — critical for reasoning about allocations, exception flow, and why blocking on async code deadlocks.

**References:**
- [How Async/Await Really Works in C# - .NET Blog](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/)
- [Asynchronous programming scenarios - C# | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/async-scenarios)

---

### Q4. What is `SynchronizationContext`, and what does `ConfigureAwait(false)` actually do?

**Question:**
What is `SynchronizationContext`, and what does `ConfigureAwait(false)` actually do?

**Good answer:**
`SynchronizationContext` is an abstraction representing "where should this continuation run?" — UI frameworks (WPF, WinForms, .NET MAUI) install one on the UI thread so that code after an `await` resumes back on that thread automatically, without you writing `Dispatcher.Invoke` everywhere. When an `await` captures a `SynchronizationContext` (the default behavior), the continuation is posted back to it instead of just running on whatever thread completed the task. `ConfigureAwait(false)` tells the awaiter not to capture the current context, so the continuation instead runs on whatever thread pool thread completed the underlying task — this avoids the overhead of marshaling back to the original context and, importantly, sidesteps a class of deadlocks. Library code should generally use `ConfigureAwait(false)` on every await, since library code has no business dictating what thread the caller resumes on.

**Code example:**
```csharp
public async Task<string> FetchDataAsync()
{
    var response = await httpClient.GetStringAsync(url).ConfigureAwait(false);
    // continues on a thread pool thread, not the original context
    return response.ToUpperInvariant();
}
```

**Follow-up question:**
Is `ConfigureAwait(false)` still necessary in an ASP.NET Core application?

**Follow-up good answer:**
No — ASP.NET Core doesn't install a `SynchronizationContext` at all (unlike classic ASP.NET, which used `AspNetSynchronizationContext`), so all continuations already run on arbitrary thread pool threads regardless of `ConfigureAwait`. It's harmless to keep using it (e.g., for library code shared with other host types), but it provides no deadlock-prevention or performance benefit specifically in ASP.NET Core request-handling code. It's still relevant in UI apps, or in libraries that might be consumed by UI apps.

**Glossary:**
- **SynchronizationContext** — abstraction for "where should code run after this await."
- **ExecutionContext** — a related but distinct concept: "what ambient environment (culture, security, `AsyncLocal` values) should flow with this code."

**Mental model:**
Probes whether the candidate can explain the deadlock/threading implications of async, not just recite "use ConfigureAwait(false) in libraries" as a rule without understanding why.

**References:**
- [ExecutionContext and SynchronizationContext - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/executioncontext-synchronizationcontext)
- [ConfigureAwait FAQ - .NET Blog](https://devblogs.microsoft.com/dotnet/configureawait-faq/)

---

### Q5. Why should you almost never use `async void`?

**Question:**
Why should you almost never use `async void`?

**Good answer:**
`async void` methods can't be composed or awaited by the caller — there's no `Task` returned, so the caller has no way to know when the operation finishes, chain `.ContinueWith`/`await` it, or combine it with `Task.WhenAll`/`WhenAny`. Worse, exceptions thrown inside an `async void` method aren't captured on a task; they're thrown directly on whatever `SynchronizationContext`/`ThreadPool` context was current when the method resumed, which in most hosts (including ASP.NET) crashes the process — it can't be caught by a `try/catch` around the call site. They're also hard to unit test, since the test can't await completion. The one legitimate use case is asynchronous event handlers (e.g., a WPF button click handler), which are required to return `void` by the delegate signature — even there, you should immediately wrap the actual logic in a `try/catch` inside the handler.

**Follow-up question:**
If you have an existing `async void` event handler and want it to be testable and safely error-handled, what's the standard pattern?

**Follow-up good answer:**
Keep a thin `async void` handler whose only job is to call an `async Task` method that does the real work, wrapped in a `try/catch` inside the `void` handler itself (since nothing downstream can catch it). This makes the actual logic (the `async Task` method) directly unit-testable/awaitable, while the `void` wrapper stays minimal and guarantees no unobserved exception escapes to crash the app.

```csharp
private async void Button_Click(object sender, EventArgs e)
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { HandleError(ex); }
}

public async Task DoWorkAsync() { /* real, testable logic */ }
```

**Glossary:**
- **Fire-and-forget** — calling an async operation without awaiting or otherwise observing its completion/exceptions.

**Mental model:**
Checks whether the candidate treats "avoid async void" as a memorized lint rule or actually understands the composability and exception-propagation failure modes it causes.

**References:**
- [Async/Await - Best Practices in Asynchronous Programming | Microsoft Learn](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)

---

### Q6. How does blocking on an async call (`.Result` / `.Wait()`) cause a deadlock?

**Question:**
How does blocking on an async call (`.Result` / `.Wait()`) cause a deadlock?

**Good answer:**
The deadlock needs a `SynchronizationContext` (or similar single-threaded context) plus code that synchronously blocks on a task. Sequence: a thread with such a context (e.g., a UI thread, or a classic ASP.NET request thread) calls an async method and immediately blocks on the result with `.Result` or `.Wait()`. Inside that async method, an `await` without `ConfigureAwait(false)` captures that same context, so its continuation is scheduled to run *on* that context. But the context's one thread is already blocked waiting for the task to finish — so the continuation can never run, the task never completes, and the blocking call waits forever. The fix is either "async all the way" (never block on a task with `.Result`/`.Wait()`), or use `ConfigureAwait(false)` throughout the async call chain so continuations don't try to marshal back to the blocked context.

**Follow-up question:**
Why doesn't this deadlock happen in a console app or in ASP.NET Core?

**Follow-up good answer:**
Because neither installs a single-threaded `SynchronizationContext` by default. A plain console app has no `SynchronizationContext` at all, so continuations just run on arbitrary thread pool threads — there's no single blocked thread that a continuation is stuck waiting for. ASP.NET Core deliberately doesn't install one either (unlike classic ASP.NET's `AspNetSynchronizationContext`), for the same reason — it removes an entire class of subtle deadlocks. Blocking on async code in ASP.NET Core can still hurt you (it ties up a thread pool thread and can cause starvation under load), but it won't deadlock the same way classic ASP.NET or a WPF/WinForms app can.

**Glossary:**
- **Deadlock** — two or more executions each waiting on the other, so none can proceed.

**Mental model:**
A staple senior-level question — tests real understanding of the SynchronizationContext + blocking interaction, not just "don't call .Result" as a rule of thumb.

**References:**
- [Common async/await bugs - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/common-async-bugs)

---

### Q7. What are the GC generations, and why does the CLR use them?

**Question:**
What are the GC generations, and why does the CLR use them?

**Good answer:**
The managed heap is divided into generations 0, 1, and 2 (plus the Large Object Heap for objects ≥ 85,000 bytes, collected alongside Gen2). Every new (small) object starts in Gen0. A Gen0 collection is fast because it only scans Gen0, and because most objects die young (the "generational hypothesis"), most of Gen0 turns out to be garbage — so these collections are frequent and cheap. Objects that survive a Gen0 collection get promoted to Gen1, which acts as a buffer between short-lived and truly long-lived objects. Objects that survive Gen1 get promoted to Gen2, which holds long-lived objects; Gen2 collections examine the entire heap, so they're the most expensive and happen least often. This design exists because compacting/scanning the whole heap on every allocation would be far too slow — generations let the GC spend most of its time on the cheap, high-yield Gen0 collections and only pay the full-heap cost rarely.

**Code example:**
```csharp
// diagnostic use only — not for production logic
Console.WriteLine(GC.CollectionCount(0));
Console.WriteLine(GC.GetGeneration(someObject));
```

**Follow-up question:**
If an app is allocating heavily and you suspect too many objects are being promoted to Gen2, what would that indicate, and how would you confirm it?

**Follow-up good answer:**
Frequent Gen2 promotions usually mean objects are living longer than they should — a common cause is objects held alive by an unintended reference (a cache with no eviction, a subscribed event handler, a static collection) that survive multiple Gen0/Gen1 collections and get promoted, or simply an allocation rate high enough that objects survive to the next Gen0 collection before dying. You'd confirm it with `dotnet-counters` (watching `gen-0-gc-count` vs. `gen-2-gc-count` and allocation rate) or by capturing a `dotnet-gcdump` and inspecting the retained-object graph in a memory profiler (e.g., which types dominate Gen2, and what's rooting them). Frequent Gen2 collections are expensive (full-heap scans) and are a classic cause of latency spikes/pauses in a service under load.

**Glossary:**
- **Generation** — an age-based bucket of the managed heap; younger generations are collected more often and more cheaply.
- **Large Object Heap (LOH)** — a separate heap segment for objects ≥ 85,000 bytes, collected along with Gen2.

**Mental model:**
Fundamentals of managed memory — checks whether the candidate can connect "why generations exist" to real GC-tuning/diagnosis reasoning, not just recite the three numbers.

**References:**
- [Fundamentals of garbage collection - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
- [dotnet-gcdump diagnostic tool - .NET CLI | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump)

---

### Q8. Workstation GC vs. Server GC — what's the difference, and when would you pick each?

**Question:**
Workstation GC vs. Server GC — what's the difference, and when would you pick each?

**Good answer:**
Workstation GC runs collections using a single dedicated GC thread (optionally with a concurrent/background mode to reduce pause times) and is tuned to minimize latency and memory footprint on client machines where CPU is shared with other work (like UI rendering). Server GC creates a separate heap and GC thread per logical core and runs collections in parallel across them, trading higher memory usage for much higher collection throughput — it's the default for ASP.NET Core apps because a server typically has CPU to spare and cares more about overall throughput than any single collection's latency. Rule of thumb from Microsoft's own guidance: on a typical server where CPU usage matters more than memory, use Server GC; if memory is constrained and CPU usage is relatively low (e.g., many small containers/processes on one host), Workstation GC (often with concurrent GC enabled) can be more appropriate.

**Code example:**
```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

**Follow-up question:**
You have many small containerized instances of a .NET service, each limited to 1 CPU core — would you use Server GC or Workstation GC, and why?

**Follow-up good answer:**
Workstation GC is usually the better fit here. Server GC allocates a heap and dedicated GC thread per logical core, so on a 1-core container it doesn't get the parallelism benefit it's designed for, and its per-heap memory overhead becomes pure waste multiplied across many container instances — this was a well-known pain point that pushed the runtime to make GC heap-count/CPU-limit awareness better in containers over time. Workstation GC's lower footprint and single-thread design match a CPU-constrained, memory-sensitive, high-density deployment much better; you'd validate the choice by comparing memory usage and P99 latency under load with `dotnet-counters` for both settings.

**Glossary:**
- **Workstation GC** — single-threaded (or concurrent/background) GC mode optimized for low footprint/latency.
- **Server GC** — multi-threaded, per-core GC mode optimized for throughput on multi-core servers.

**Mental model:**
Trade-off question — tests whether the candidate can reason about GC mode choice from actual deployment constraints (cores, memory, latency vs. throughput) rather than just picking "Server GC because it's faster."

**References:**
- [Workstation vs. server garbage collection (GC) - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)

---

### Q9. A .NET service is running slow in production and you suspect GC pressure — what's your diagnostic methodology?

**Question:**
A .NET service is running slow in production and you suspect GC pressure — what's your diagnostic methodology?

**Good answer:**
Start with `dotnet-counters` attached to the live process to get a first-level read without restarting anything: watch `% Time in GC`, Gen0/1/2 collection counts, and allocation rate (`alloc-rate`). A high `% Time in GC` combined with a high allocation rate points to excessive allocation somewhere in a hot path; a high Gen2 count specifically points to objects surviving too long. If that confirms GC is a real contributor, capture a `dotnet-trace` (an EventPipe-based CPU/GC trace) to see exactly where in the code the allocations are coming from, and optionally a `dotnet-gcdump` to snapshot the live object graph and see which types dominate the heap and what's rooting them (useful for finding leaks vs. just high churn). For CPU-side hotspots unrelated to GC, the same `dotnet-trace` capture can be viewed as a flame graph (e.g., in PerfView or Visual Studio) to find the actual expensive call path. Once you have a hypothesis (e.g., "this LINQ chain is allocating an enumerator + closures per request"), fix it and re-measure with the same counters/trace to confirm the regression is actually gone — not just "looks better."

**Follow-up question:**
What's the difference between `dotnet-trace` and `dotnet-counters`, and when would you reach for one over the other?

**Follow-up good answer:**
`dotnet-counters` is for lightweight, real-time first-level health monitoring — a rolling view of metrics like CPU%, GC counts, allocation rate, thread pool queue length, with very low overhead, good for "is something wrong and roughly what category." `dotnet-trace` captures a detailed EventPipe trace (CPU samples, GC events, exceptions, custom `EventSource` events) over a time window that you then analyze offline (in PerfView, Visual Studio, or `speedscope`) to root-cause exactly where time/allocations are going, down to the call stack — higher overhead and more detail, used once counters have told you roughly where to look.

**Glossary:**
- **EventPipe** — the cross-platform tracing mechanism .NET diagnostic tools use to collect events with low overhead.
- **Allocation rate** — bytes allocated per second, a leading indicator of GC pressure.

**Mental model:**
This is the core "performance diagnosis methodology" question for .NET — checks that the candidate has an actual detect → isolate → fix → verify loop with named tools, not just "I'd look at the logs."

**References:**
- [.NET Diagnostic tools overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/tools-overview)
- [dotnet-counters diagnostic tool - .NET CLI | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters)
- [dotnet-trace overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)

---

### Q10. What is tiered compilation, and how do ReadyToRun and Native AOT relate to it?

**Question:**
What is tiered compilation, and how do ReadyToRun and Native AOT relate to it?

**Good answer:**
Tiered compilation (default since .NET Core 3.0) lets the JIT trade off startup speed against steady-state throughput: methods are first compiled quickly at "Tier 0" (a fast, less-optimized JIT pass, or loaded from a precompiled ReadyToRun image if available) so the app starts responding sooner, and methods that turn out to be hot are recompiled in the background at "Tier 1" with full optimizations once they've been called enough times. ReadyToRun (R2R) is an ahead-of-time compilation format that ships precompiled native code alongside the IL in the assembly, so Tier 0 doesn't even need to JIT those methods at startup — it just loads the precompiled version, improving cold-start time; tiered compilation still promotes hot R2R methods to a JIT-optimized Tier 1 later. Native AOT goes further: it compiles the entire app to a single native executable ahead of time with no JIT and no IL at runtime at all — there's no tiering, no runtime code generation, and no `Assembly.LoadFile`/`Reflection.Emit` support, in exchange for the fastest possible startup and smallest memory footprint, at the cost of losing dynamic runtime features.

**Follow-up question:**
Why would you choose ReadyToRun over Native AOT, given Native AOT starts even faster?

**Follow-up good answer:**
ReadyToRun keeps the full .NET runtime and JIT available, so the app retains dynamic capabilities Native AOT doesn't support well or at all — runtime reflection-heavy scenarios, dynamic assembly loading, `Reflection.Emit`, and broader third-party library compatibility (since not every library is Native-AOT-compatible/trim-safe). It also still benefits from Tier 1's fully optimized steady-state code via tiered compilation, whereas Native AOT is compiled once at publish time with no further runtime specialization. In practice: use Native AOT when startup/footprint is critical and the app's dependency graph is verified AOT-compatible (e.g., minimal APIs, gRPC services with an audited dependency tree); use ReadyToRun when you want a faster cold start but still need full runtime flexibility or have libraries that aren't AOT-safe.

**Glossary:**
- **Tiered compilation** — JIT strategy of compiling fast-but-unoptimized first, then re-optimizing hot methods later.
- **ReadyToRun (R2R)** — AOT-precompiled native code embedded alongside IL, still runs on the full JIT-capable runtime.
- **Native AOT** — fully ahead-of-time compiled native executable with no JIT/runtime code generation.

**Mental model:**
Advanced/trade-off question that separates candidates who know .NET's compilation pipeline evolved beyond "it's just JIT'd" and can reason about startup-vs-flexibility trade-offs for real deployment decisions (e.g., serverless cold start).

**References:**
- [ReadyToRun deployment overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run)
- [Native AOT deployment overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

---

### Q11. What problem do `Span<T>` and `Memory<T>` solve?

**Question:**
What problem do `Span<T>` and `Memory<T>` solve?

**Good answer:**
They let you work with a contiguous region of memory (an array, a slice of an array, stack-allocated memory, or unmanaged memory) without copying it or allocating a new heap object to represent a "view" into it. `Span<T>` is a `ref struct` — stack-only, allocation-free — that wraps a pointer + length; slicing it (`.Slice()`, indexers) just adjusts the pointer/length, it never copies the underlying data. Before these types, doing something like "parse this substring without allocating a new string" wasn't possible in a type-safe, general way — you'd either accept the allocation (`string.Substring`) or drop into raw pointers. `Memory<T>` is the heap-allocatable counterpart (since `Span<T>` can't be a field of a class or crossed with `async`/`await`, both illegal for `ref struct`s), used when you need the same slicing capability but must store it in a field or pass it across an `await`.

**Code example:**
```csharp
ReadOnlySpan<char> line = "key=value".AsSpan();
int eq = line.IndexOf('=');
ReadOnlySpan<char> key = line[..eq];     // no allocation, just a view
ReadOnlySpan<char> value = line[(eq+1)..];
```

**Follow-up question:**
Why can't you use `Span<T>` as a field in a class or across an `await` boundary?

**Follow-up good answer:**
`Span<T>` is a `ref struct`, which the CLR restricts to stack-only usage specifically so it can safely wrap stack memory (like `stackalloc`) or a pointer whose lifetime is tied to the current stack frame — allowing it to escape onto the heap (as a class field) or survive across an `await` (where the async state machine may be heap-allocated and its execution suspended/resumed later, potentially outliving the original stack frame) would risk a dangling reference to memory that's no longer valid. `Memory<T>` doesn't have this restriction because it doesn't directly hold a raw pointer into stack memory — it holds a managed reference/array plus offset/length, and you convert it to a `Span<T>` transiently (via `.Span`) only when you're ready to actually use it synchronously.

**Glossary:**
- **`ref struct`** — a struct type restricted to stack allocation, cannot be boxed, cannot be a field of a non-ref-struct class, cannot cross `await`/`yield`.
- **Slicing** — creating a sub-view of a span/memory without copying the underlying data.

**Mental model:**
Advanced-feature question that tests whether the candidate understands span/memory as a *deliberate CLR safety trade-off* for zero-allocation code, not just "a faster array."

**References:**
- [Memory<T> and Span<T> usage guidelines - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)
- [Span<T> Struct (System) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.span-1)

---

### Q12. What is thread pool starvation, and how does it happen in a .NET service?

**Question:**
What is thread pool starvation, and how does it happen in a .NET service?

**Good answer:**
Thread pool starvation happens when all (or most) available thread pool worker threads are blocked or busy, so new queued work items have to wait — even though the process isn't doing much useful CPU work, latency spikes badly because work is stuck in the queue behind blocked threads. It commonly happens from calling blocking APIs (`.Result`, `.Wait()`, synchronous I/O like `File.ReadAllText` instead of the async version) inside code that's ultimately serving concurrent requests: each blocked call ties up a thread pool thread that could otherwise be doing real work. The thread pool does grow its thread count to compensate, but by design it only injects new threads slowly (roughly one every ~500ms historically) to avoid oversubscription — so a sudden burst of blocking calls can starve the pool faster than it can grow, causing a visible latency cliff until it catches up.

**Follow-up question:**
How would you detect thread pool starvation in a running production service, and how would you fix it without a code deploy?

**Follow-up good answer:**
Detect it with `dotnet-counters` watching `ThreadPool Queue Length` (rising) and `ThreadPool Thread Count` (not growing fast enough, or already at a cap) — a growing queue length with requests timing out despite low CPU usage is the signature. `dotnet-trace` with the `Microsoft-Windows-DotNETRuntime` provider can show thread pool worker thread adjustment events (`ThreadPoolWorkerThreadAdjustment`). As an immediate mitigation without a code change, you can raise `ThreadPool.SetMinThreads` to force more threads to be available immediately rather than waiting on the gradual injection rate — this is a stopgap, not a fix, since the real problem is blocking calls that should be async; over-raising min threads risks its own overhead from excessive context switching.

**Glossary:**
- **Thread pool starvation** — a state where queued work can't run because available worker threads are all blocked/busy.
- **Thread injection rate** — the rate at which the CLR thread pool adds new worker threads under sustained demand.

**Mental model:**
Real-world production-incident question — checks whether the candidate has actually diagnosed a live starvation issue (specific counters, specific tool) versus just knowing "avoid blocking calls" as a rule.

**References:**
- [Debug ThreadPool Starvation - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-threadpool-starvation)

---

### Q13. `Task` vs `ValueTask` — what's the difference, and when should you actually use `ValueTask`?

**Question:**
`Task` vs `ValueTask` — what's the difference, and when should you actually use `ValueTask`?

**Good answer:**
`Task`/`Task<T>` are reference types — every async call that returns one allocates an object on the heap (unless it's a cached completed task). `ValueTask`/`ValueTask<T>` are structs that can represent either a synchronously-available result (no allocation at all) or wrap an underlying `Task`/`IValueTaskSource` when the operation is genuinely asynchronous — so in the common case where a method's result is already available (e.g., served from a cache) most of the time, `ValueTask` avoids the allocation entirely. The official guidance is that `Task` should be the default for any async method; you should only switch to `ValueTask` after profiling shows the allocation is actually a measurable cost, because `ValueTask` has real constraints: it must not be awaited more than once, you shouldn't access `.Result` before checking completion, and because it's a larger struct than the single-field `Task` reference, passing it around by value copies more data — so it's not a free win, it's a targeted optimization.

**Follow-up question:**
What goes wrong if you `await` the same `ValueTask` twice?

**Follow-up good answer:**
It's undefined/unsupported behavior — a `ValueTask` may wrap a pooled/reusable backing object (an `IValueTaskSource`) whose state gets reset or reused once consumed, unlike `Task`, which is a plain immutable-after-completion object safe to await/observe multiple times from multiple places. Awaiting a `ValueTask` twice can throw, return a stale/incorrect result, or in the worst case corrupt the pooled backing object's state for an unrelated operation reusing it. If you need to await a result multiple times or hand it to multiple consumers, call `.AsTask()` once on the `ValueTask` to get a regular `Task` and share that instead.

**Glossary:**
- **`IValueTaskSource`** — the interface that lets `ValueTask` wrap a pooled, reusable backing implementation instead of always allocating a `Task`.

**Mental model:**
Trade-off/advanced-feature question — tests whether the candidate treats `ValueTask` as a drop-in "faster Task" (wrong, and dangerous) or understands it as a constrained optimization for a specific, profiled hot path.

**References:**
- [ValueTask Struct (System.Threading.Tasks) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask)

---

### Q14. Why does .NET have both `Dispose()` and finalizers, and how do they interact?

**Question:**
Why does .NET have both `Dispose()` and finalizers, and how do they interact?

**Good answer:**
The GC only knows how to reclaim *managed* memory — it has no idea about unmanaged resources (file handles, sockets, native memory, database connections) held by an object. `IDisposable.Dispose()` is the deterministic, explicit way for a consumer to say "release your resources now" — typically via a `using` statement. A finalizer (`~ClassName()`) is a safety net the GC calls if `Dispose()` was never called, so unmanaged resources still eventually get released — but finalization is non-deterministic (you don't control when, or if promptly, it runs) and finalizable objects survive at least one extra GC cycle (they get promoted and queued for the finalizer thread before their memory can actually be reclaimed), which is real overhead. The standard Dispose pattern has `Dispose()` call `GC.SuppressFinalize(this)` after cleanup, telling the GC "the finalizer's work is already done, don't bother queuing this object for finalization" — this is what makes explicit disposal cheaper than relying on the finalizer.

**Code example:**
```csharp
public class ResourceHolder : IDisposable
{
    private SafeHandle _handle;
    private bool _disposed;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing) { _handle?.Dispose(); }  // managed
        _disposed = true;
    }

    ~ResourceHolder() => Dispose(false);
}
```

**Follow-up question:**
Should you write a finalizer for every class that implements `IDisposable`?

**Follow-up good answer:**
No — only for a type that directly owns unmanaged resources (a raw handle, unmanaged memory pointer). If a class only holds *other* `IDisposable` managed objects (e.g., a `Stream` field), it should implement `Dispose()` to call `Dispose()` on those fields, but it should not have a finalizer itself — the managed objects it holds already have their own finalizers as a safety net if needed, and adding an unnecessary finalizer only adds GC overhead (extra generation survival, finalizer queue processing) with no benefit, since there's no unmanaged resource for it to protect.

**Glossary:**
- **Finalizer** — a method (`~ClassName()`) the GC calls before reclaiming an object's memory, used as a last-resort cleanup for unmanaged resources.
- **`GC.SuppressFinalize`** — tells the GC an object no longer needs finalization, typically called from `Dispose()`.

**Mental model:**
SE-theory-meets-practice question: connects the abstract idea of "deterministic vs. non-deterministic resource cleanup" to the concrete Dispose pattern, and checks the candidate doesn't over-apply finalizers.

**References:**
- [Implement a Dispose method - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)
- [Dispose Pattern - Framework Design Guidelines | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern)

---

### Q15. How can a .NET application "leak" memory even though it has a garbage collector?

**Question:**
How can a .NET application "leak" memory even though it has a garbage collector?

**Good answer:**
The GC only reclaims objects that are *unreachable* — it can't reclaim an object that's still referenced, even if your program logically no longer needs it. The classic culprit is event subscriptions: if a long-lived publisher object holds a reference to a subscriber's handler (e.g., a static event, or a singleton service raising an event), the subscriber stays reachable — and alive — through that reference chain for as long as the publisher lives, even if the subscriber itself should have been garbage a long time ago. Other common causes: static collections (caches, dictionaries) that keep adding entries and never evict; capturing a large object in a closure that outlives its intended scope; and `IDisposable` objects with unmanaged handles that are never disposed, which leaks the unmanaged resource even though the managed wrapper eventually gets collected. All of these are "leaks" in the sense of unbounded, unintended memory growth, even though technically nothing is *un*managed or corrupted — everything is exactly where a reference says it should be.

**Follow-up question:**
You suspect a memory leak from event subscriptions in a long-running service — how would you confirm it and find the root cause?

**Follow-up good answer:**
Take two `dotnet-gcdump` snapshots some time apart under normal load, and diff them — a genuine leak shows the same type(s) growing in count between snapshots without a corresponding drop after a full GC. In the dump, inspect the retained-object graph / "path to root" for one of the growing instances to see exactly what's holding the reference (a memory profiler like the one in Visual Studio, or `dotnet-gcdump`'s report, will show the chain — e.g., `StaticEventPublisher → EventHandler → MyViewModel`). Once you've found the rooting chain, the fix is usually either unsubscribing in `Dispose()`, switching to a weak-reference-based event pattern, or removing the unintended static reference.

**Glossary:**
- **Reachability** — an object is reachable if there's a chain of references from a GC root to it; the GC only collects unreachable objects.
- **GC root** — a starting reference the GC scans from (static fields, thread stacks, CPU registers, etc.).

**Mental model:**
Classic pitfall question — checks the candidate understands that "garbage collected" doesn't mean "leak-proof," and that leaks in managed languages are a reachability problem, not a missing-free problem.

**References:**
- [Memory leaks 101: Objects anchored by event generators | Microsoft Learn](https://learn.microsoft.com/en-us/archive/blogs/ricom/memory-leaks-101-objects-anchored-by-event-generators)
- [dotnet-gcdump diagnostic tool - .NET CLI | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump)

---

### Q16. What's your methodology for benchmarking a micro-optimization in .NET, and why not just wrap it in a `Stopwatch`?

**Question:**
What's your methodology for benchmarking a micro-optimization in .NET, and why not just wrap it in a `Stopwatch`?

**Good answer:**
A raw `Stopwatch` around a single run is unreliable for micro-benchmarks because it's dominated by noise you're not controlling for: JIT warm-up (the method may still be running unoptimized Tier 0 code), GC pauses landing mid-measurement, OS thread scheduling jitter, and whether the machine is under other load. BenchmarkDotNet is the standard tool for this in .NET: it runs the code through proper warm-up iterations (so you're measuring steady-state JIT-optimized code), runs many iterations to get a statistically meaningful distribution (mean, error, standard deviation), isolates each benchmark in its own process, and reports allocation counts per operation alongside timing — which matters because a "faster" version that allocates more can be a net loss under GC pressure at scale. The workflow: write competing implementations as `[Benchmark]` methods in a class, run in Release configuration, and compare the reported ns/op and allocated bytes, not just eyeball a stopwatch number once.

**Code example:**
```csharp
[MemoryDiagnoser]
public class StringConcatBenchmarks
{
    [Benchmark(Baseline = true)]
    public string Concat() => "a" + "b" + "c";

    [Benchmark]
    public string Interpolate() => $"{"a"}{"b"}{"c"}";
}
// BenchmarkRunner.Run<StringConcatBenchmarks>();
```

**Follow-up question:**
Why does BenchmarkDotNet insist on running in Release configuration, and what would go wrong benchmarking a Debug build?

**Follow-up good answer:**
Debug builds disable most JIT optimizations and insert extra debugging scaffolding (e.g., sequence points for the debugger, suppressed inlining) so that stepping through code in a debugger behaves predictably — this means a Debug build's performance characteristics don't represent what actually ships to production. Benchmarking a Debug build measures the cost of unoptimized, debugger-friendly code, which can be dramatically slower and allocate differently than the Release/Tier-1-JIT-optimized code path actually used in production, making the results meaningless (and BenchmarkDotNet will warn/refuse by default for exactly this reason).

**Glossary:**
- **Warm-up (benchmarking)** — initial iterations run and discarded so the JIT reaches steady-state optimized code before measurements are recorded.
- **MemoryDiagnoser** — a BenchmarkDotNet feature that reports allocated bytes per operation alongside timing.

**Mental model:**
Performance-methodology question specifically about *micro*-benchmarking correctness — tests whether the candidate knows naive timing is actively misleading in a JIT'd, GC'd runtime, and knows the standard tool that controls for it.

**References:**
- [Getting started | BenchmarkDotNet](https://benchmarkdotnet.org/articles/guides/getting-started.html)
- [Overview | BenchmarkDotNet](https://benchmarkdotnet.org/articles/overview.html)

---

### Q17. What's the N+1 query problem in EF Core, and how do you avoid it?

**Question:**
What's the N+1 query problem in EF Core, and how do you avoid it?

**Good answer:**
It happens when you fetch a list of N entities with one query, then access a navigation property on each of them in a loop — if that navigation property is lazily loaded, EF Core silently issues one additional query *per entity* to fetch its related data, turning what should be 1 (or 2) queries into N+1 round trips to the database. This is especially dangerous because lazy loading makes it invisible in the code — it just looks like a normal property access — and it typically isn't caught until the app is under real load with real data volumes. The fix is to control loading explicitly: use eager loading (`.Include()`/`.ThenInclude()`) to fetch the related data as part of the initial query via a `JOIN`, or explicit loading (`context.Entry(x).Collection(...).Load()`) when you need it conditionally — either way makes the extra round trip visible and intentional in the code rather than hidden behind a property getter.

**Code example:**
```csharp
// N+1: one query per order to load OrderItems
var orders = await context.Orders.ToListAsync();
foreach (var o in orders) { var items = o.OrderItems; }

// Fixed: single query with a JOIN
var orders = await context.Orders
    .Include(o => o.OrderItems)
    .ToListAsync();
```

**Follow-up question:**
Eager loading with `.Include()` fixes the N+1 problem — but what new performance problem can it introduce if overused?

**Follow-up good answer:**
Loading multiple collection navigations with `.Include()` in a single query (e.g., `Include(o => o.OrderItems).Include(o => o.Payments)`) causes EF Core to generate a query with multiple `JOIN`s that multiply rows together (a cartesian-product-like effect) — the flat result set duplicates parent columns once per combination of child rows, which can balloon the amount of data transferred and processed far beyond what's actually needed, especially with more than one collection include. EF Core mitigates this by default with "split queries" behavior in some cases, or you can explicitly request `AsSplitQuery()` to issue separate queries per included collection instead of one giant joined result — trading more round trips for avoiding the row-multiplication blowup, which is often the better trade-off for multiple large collections.

**Glossary:**
- **Lazy loading** — related data is fetched transparently, on first access to the navigation property, via an extra query.
- **Eager loading** — related data is fetched as part of the initial query, typically via `JOIN`.
- **Split query** — EF Core issuing separate queries per included collection instead of one joined query, to avoid row multiplication.

**Mental model:**
The signature "real situation this technology's design causes" question for ORMs — checks the candidate has actually hit this in production and knows the concrete EF Core APIs to fix it, not just the term "N+1."

**References:**
- [Loading Related Data - EF Core | Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/querying/related-data/)
- [Lazy Loading of Related Data - EF Core | Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/querying/related-data/lazy)

---

### Q18. What are the three DI service lifetimes in ASP.NET Core, and what's the most common lifetime-related bug?

**Question:**
What are the three DI service lifetimes in ASP.NET Core, and what's the most common lifetime-related bug?

**Good answer:**
**Transient** services are created fresh every time they're resolved — cheap to create, no shared state. **Scoped** services are created once per client request (per DI scope, which in ASP.NET Core aligns with an HTTP request) and reused for the duration of that scope. **Singleton** services are created once for the lifetime of the app and shared by everyone. The most common bug is *captive dependencies*: injecting a Scoped (or Transient-with-state) service into a Singleton. Because the Singleton is constructed once and holds onto the reference for the app's entire lifetime, the "scoped" service it captured effectively becomes a singleton too — it never gets a fresh instance per request, so any per-request state (like a `DbContext`, which is registered Scoped) ends up shared and mutated across unrelated requests, a serious and hard-to-diagnose correctness bug (and the DI container will throw a validation error for this exact case if scope validation is enabled).

**Code example:**
```csharp
builder.Services.AddSingleton<ICacheService, CacheService>();
builder.Services.AddScoped<AppDbContext>();
builder.Services.AddTransient<IEmailSender, EmailSender>();
```

**Follow-up question:**
If a singleton legitimately needs to use a scoped service (like `DbContext`) occasionally, what's the correct pattern?

**Follow-up good answer:**
Inject `IServiceScopeFactory` (or `IServiceProvider`) into the singleton instead of the scoped service directly, and create a new scope explicitly each time it's needed: `using var scope = scopeFactory.CreateScope(); var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();`. This gives the singleton a fresh, correctly-scoped instance on demand without ever holding onto a captive reference across requests, and the scope (and everything resolved from it) is disposed when the `using` block ends.

**Glossary:**
- **Captive dependency** — a shorter-lived service held by a longer-lived one, effectively extending its lifetime incorrectly.
- **DI scope** — a boundary (per-HTTP-request in ASP.NET Core) within which Scoped services are created once and reused.

**Mental model:**
Classic pitfall/fundamentals hybrid — tests whether the candidate has actually been bitten by captive dependencies (a very common real bug) and knows the concrete fix, not just the three lifetime definitions.

**References:**
- [Service lifetimes (dependency injection) - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/service-lifetimes)
- [Dependency injection guidelines - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines)

---

### Q19. How does the ASP.NET Core middleware pipeline work, and why does order matter?

**Question:**
How does the ASP.NET Core middleware pipeline work, and why does order matter?

**Good answer:**
Middleware components are chained request delegates, registered in `Program.cs` via `Use`/`Run`/`Map`. `Use` middleware can run code both before *and* after calling the next delegate in the chain (`await next(context)`), so the pipeline behaves like nested function calls — request flows down through each middleware in registration order, and the response flows back up through the same middlewares in reverse order after the innermost one completes. `Run` middleware is terminal — it doesn't call a next delegate, so nothing registered after it in the pipeline executes. Order matters because each middleware only sees what happens in the middlewares registered before it: e.g., `UseAuthentication()` must run before `UseAuthorization()` (you can't authorize an identity that hasn't been established yet), and exception-handling middleware (`UseExceptionHandler`) must be registered early so it can wrap everything downstream in a try/catch-like boundary. Static file middleware is typically placed early so static asset requests can short-circuit the pipeline without running through auth/routing at all.

**Code example:**
```csharp
app.UseExceptionHandler("/error");   // first: wraps everything below
app.UseStaticFiles();                 // short-circuits static asset requests
app.UseRouting();
app.UseAuthentication();              // must precede UseAuthorization
app.UseAuthorization();
app.MapControllers();
```

**Follow-up question:**
What happens if `UseAuthorization()` is accidentally registered before `UseRouting()`?

**Follow-up good answer:**
Authorization middleware needs the endpoint that routing has selected (specifically its authorization metadata/policies) to know what to check — if it runs before `UseRouting()`, there's no endpoint selected yet, so it has nothing to authorize against and effectively can't enforce any policy correctly (in practice this misconfiguration is flagged, and endpoint-specific authorization won't function as intended). This is exactly why the documented required order is routing, then authentication, then authorization, then endpoint execution — each stage depends on state the previous stage establishes.

**Glossary:**
- **Request delegate** — the function signature (`RequestDelegate`, `Func<HttpContext, Task>`) that each middleware implements to process (or pass along) a request.
- **Short-circuit** — a middleware that returns a response without calling the next delegate, ending the pipeline early.

**Mental model:**
Internals question for the web-framework-specific angle — checks the candidate understands middleware as literally nested delegate calls (explaining the before/after and reverse-order-response behavior), not just "a list of steps."

**References:**
- [ASP.NET Core Middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)

---

### Q20. What does Native AOT trade away, and when is it the wrong choice?

**Question:**
What does Native AOT trade away, and when is it the wrong choice?

**Good answer:**
Native AOT compiles your app and all its dependencies to a single native binary ahead of time, with no JIT at runtime — which means anything that relies on generating code or loading assemblies at runtime doesn't work: `Assembly.LoadFile`/plugin-style dynamic loading, `System.Reflection.Emit`, and unconstrained runtime reflection (reflection that the trimmer/AOT compiler can't statically analyze produces trim warnings and can fail at runtime instead of compile time). Not every runtime library or third-party NuGet package is annotated as AOT/trim-safe, so adopting it can mean auditing your whole dependency tree, and compiled AOT binaries also tend to be roughly double the size of the equivalent IL assembly (though still often smaller overall due to trimming unused code and not shipping the runtime). It's the wrong choice for apps that genuinely need runtime plugin loading, heavy dynamic reflection (e.g., certain ORMs' older reflection-based mapping, some DI containers' dynamic proxy generation), or where startup time and memory footprint simply aren't a constraint that matters — the migration cost isn't worth it there. It's the right choice for scenarios where cold start and density matter a lot: serverless functions, CLI tools, containerized microservices at high replica counts.

**Follow-up question:**
Your team wants to migrate an existing ASP.NET Core API to Native AOT — what would you actually check before committing to it?

**Follow-up good answer:**
Publish with AOT enabled and read the trim/AOT analyzer warnings first — they surface exactly which types/members the compiler couldn't statically analyze (often from reflection-based JSON serialization, dynamic DI proxies, or third-party libraries using `Reflection.Emit`). Cross-check every NuGet dependency against whether it's documented as trim/AOT-compatible; libraries that aren't will need a source-generator-based alternative (e.g., `System.Text.Json`'s source-generated serialization instead of reflection-based) or replacement. Also confirm the app doesn't rely on runtime code generation features minimal APIs and gRPC support well but some MVC/Razor scenarios and certain serializers historically needed extra work. Only after the warnings are resolved and dependencies are verified would you actually cut over, ideally behind a canary/staged rollout given how much of the runtime behavior changes under the hood.

**Glossary:**
- **Trimming** — removing unused code from the published app to reduce size; AOT compilation depends on trimming being accurate.
- **Trim warning** — a build-time warning that a reflection/dynamic-loading pattern can't be statically verified as safe for trimming/AOT.

**Mental model:**
Advanced/trade-off closer — checks whether the candidate can make a realistic go/no-go call on an exciting-sounding feature by actually naming its constraints, not just repeating the marketing benefit ("faster startup!").

**References:**
- [Native AOT deployment overview - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
- [Optimizing AOT deployments - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/optimizing)
