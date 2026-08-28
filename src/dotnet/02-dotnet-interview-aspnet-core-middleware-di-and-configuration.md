---
layout: default
title: ".NET Interview — ASP.NET Core Middleware, DI & Configuration"
---

# .NET Interview — ASP.NET Core Middleware, DI & Configuration

This set covers the request pipeline (middleware ordering, `Use`/`Run`/`Map`, endpoint routing), the built-in dependency injection container (service lifetimes, captive dependencies, scope validation), and the options pattern for strongly-typed configuration — the plumbing every ASP.NET Core app is built on, and a frequent source of subtle production bugs.

### Q1. What is middleware in ASP.NET Core, and what classic design pattern does the request pipeline implement? {#q1}

**Question:**
What is middleware in ASP.NET Core, and what classic design pattern does the request pipeline implement?

**Good answer:**
Middleware is software assembled into an app's request pipeline to handle requests and responses — each component decides whether to pass the request to the next component in the pipeline, and can do work both before and after that next call. This is a direct implementation of the **Chain of Responsibility** pattern: each handler either processes the request itself or forwards it to the next handler, and the caller doesn't need to know which handler in the chain actually deals with it. In ASP.NET Core, the chain is built explicitly in code (`Program.cs`) as a sequence of `RequestDelegate`s composed via `IApplicationBuilder`.

**Code example:**
```csharp
app.Use(async (context, next) =>
{
    // work before
    await next(context);
    // work after
});
```

**Follow-up question:**
Can a middleware component stop the request from reaching the rest of the pipeline entirely?

**Follow-up good answer:**
Yes — that's called short-circuiting. If a middleware doesn't call `next`, none of the downstream middleware runs, and the middleware that short-circuits is called *terminal middleware* since it fully handles the response itself. Static file middleware does this for matched files: it serves the file and returns without invoking the rest of the pipeline.

**Glossary:**
- **Middleware** — a component in the request pipeline that can inspect/modify the request or response and choose whether to call the next component.
- **RequestDelegate** — the delegate signature (`Task Invoke(HttpContext)`) that represents one step in the pipeline.
- **Short-circuiting** — a middleware handling the request and not calling `next`, stopping the pipeline there.

**Mental model:**
Tests whether the candidate sees middleware as an implementation of a well-known composability pattern rather than "magic ASP.NET stuff" — this framing is what makes ordering rules (Q4, Q10) intuitive instead of memorized trivia.

**TL;DR:**
ASP.NET Core middleware is Chain of Responsibility made concrete: each component decides whether to handle a request or hand it to the next one, and can short-circuit the chain entirely.

**References:**
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)
- [Chain of Responsibility — Refactoring.Guru](https://refactoring.guru/design-patterns/chain-of-responsibility)

---

### Q2. What are the three built-in DI service lifetimes in ASP.NET Core, and what does each one guarantee? {#q2}

**Question:**
What are the three built-in DI service lifetimes in ASP.NET Core, and what does each one guarantee?

**Good answer:**
**Transient** services are created every time they're requested from the container — no sharing at all. **Scoped** services are created once per client request (per HTTP request in a web app) and disposed at the end of that request — everything resolved within the same request shares the same instance. **Singleton** services are created once (the first time they're requested, or eagerly if you register an existing instance) and that same instance is reused for the lifetime of the application.

**Code example:**
```csharp
builder.Services.AddTransient<IEmailSender, EmailSender>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddSingleton<IClock, SystemClock>();
```

**Follow-up question:**
Why is mixing lifetimes dangerous, specifically injecting a scoped service into a singleton?

**Follow-up good answer:**
That's the "captive dependency" problem (Q7) — the singleton captures the scoped instance the first time it's constructed and holds onto it for the app's entire lifetime, so every subsequent "request" for that scoped service through the singleton actually gets the same stale instance instead of a fresh one per HTTP request. This can leak state across requests and cause hard-to-reproduce bugs since it depends on which singleton happened to be resolved first.

**Glossary:**
- **Transient** — new instance every resolution.
- **Scoped** — one instance per request/scope.
- **Singleton** — one instance for the app's lifetime.

**Mental model:**
Baseline knowledge check — but the interviewer is really listening for whether the candidate can reason about *why* the lifetimes exist (state sharing boundaries), not just recite the three names.

**TL;DR:**
Transient = new every time, Scoped = one per request, Singleton = one for the app's life — mixing them incorrectly (scoped inside singleton) is the classic DI bug.

**References:**
- [Dependency injection in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0)
- [Service lifetimes (dependency injection) - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/service-lifetimes)

---

### Q3. How do `Use`, `Run`, and `Map` differ when building the middleware pipeline? {#q3}

**Question:**
How do `Use`, `Run`, and `Map` differ when building the middleware pipeline?

**Good answer:**
`Use` registers a middleware that receives a `next` delegate — it can do work, then call `next` to continue the pipeline, or skip calling it to short-circuit. `Run` registers *terminal* middleware: it doesn't receive a `next` parameter at all, and it's meant to be the last delegate in the pipeline — anything registered after the first `Run` never executes. `Map` (and `MapWhen`) branches the pipeline based on the request path (or a predicate): if the path matches, execution diverts into that separate branch's own mini-pipeline.

**Code example:**
```csharp
app.Map("/admin", adminApp => adminApp.Run(async ctx => await ctx.Response.WriteAsync("admin branch")));
app.Run(async ctx => await ctx.Response.WriteAsync("default branch"));
```

**Follow-up question:**
If you accidentally place `app.Run(...)` in the middle of your pipeline, what happens to the middleware registered after it?

**Follow-up good answer:**
It never runs. The first `Run` delegate always terminates the pipeline for any request that reaches it, so any `Use` or `Run` calls registered afterward in the source are simply dead code for that path — a subtle bug since the app compiles fine and nothing indicates the later middleware is unreachable.

**Glossary:**
- **Terminal middleware** — middleware that doesn't call `next`, ending the pipeline.
- **Branch** — a sub-pipeline created by `Map`/`MapWhen` for requests matching a condition.

**Mental model:**
Checks whether the candidate actually understands pipeline construction mechanics rather than treating `Program.cs` middleware registration as boilerplate they copy-paste.

**TL;DR:**
`Use` can continue or short-circuit, `Run` always terminates the pipeline, `Map` branches it by path — anything registered after the first `Run` is unreachable.

**References:**
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)

---

### Q4. Why must `UseRouting` come before authorization middleware, and what do `UseRouting`/`UseEndpoints` actually each do? {#q4}

**Question:**
Why must `UseRouting` come before authorization middleware, and what do `UseRouting`/`UseEndpoints` actually each do?

**Good answer:**
Endpoint routing splits routing into two separate steps via two middleware components: `UseRouting` matches the incoming request against the set of registered endpoints and selects the best match, storing it on the `HttpContext` — but doesn't execute it. `UseEndpoints` (or, since .NET 6+, the implicit endpoint execution at the end of the pipeline) actually invokes the delegate for the selected endpoint. Authorization middleware needs to know *which* endpoint was matched to check that endpoint's `[Authorize]` metadata — so if `UseAuthorization` runs before `UseRouting`, there's no selected endpoint yet to check policies against, and authorization can't do its job correctly.

**Code example:**
```csharp
app.UseRouting();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers(); // implicit UseEndpoints
```

**Follow-up question:**
What can any middleware placed between `UseRouting` and endpoint execution do that middleware placed before `UseRouting` cannot?

**Follow-up good answer:**
It can call `HttpContext.GetEndpoint()` to inspect the endpoint that was selected — including reading its metadata (like `[Authorize]` attributes, allowed HTTP methods, or custom endpoint metadata) — before the endpoint actually runs. Middleware registered before `UseRouting` has no endpoint to inspect yet, since matching hasn't happened.

**Glossary:**
- **Endpoint routing** — the two-phase model (match, then execute) introduced to decouple route matching from execution.
- **`GetEndpoint()`** — the `HttpContext` extension that returns the endpoint selected by `UseRouting`.

**Mental model:**
This is the single most common real-world middleware-ordering bug interviewers probe for — it tests whether the candidate understands routing as two distinct phases rather than "routing just works."

**TL;DR:**
`UseRouting` selects an endpoint without running it; `UseEndpoints`/endpoint execution runs it — authorization has to sit after routing because it needs to know which endpoint (and its `[Authorize]` metadata) was matched.

**References:**
- [Routing in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-10.0)
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)

---

### Q5. What's the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`? {#q5}

**Question:**
What's the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?

**Good answer:**
`IOptions<T>` is registered as a singleton and its `Value` is computed once and cached for the app's lifetime — it never reflects configuration changes after startup, and it doesn't support named options. `IOptionsSnapshot<T>` is scoped: it recomputes the options once per request (or per scope) when first accessed, so it picks up configuration changes on the *next* request, and it does support named options — but that scoped lifetime means it can't be injected into a singleton. `IOptionsMonitor<T>` is itself a singleton, so it can be injected anywhere (including other singletons), exposes the always-current value via `CurrentValue`, and can notify subscribers of changes via `OnChange`.

**Code example:**
```csharp
public class Worker(IOptionsMonitor<MySettings> settingsMonitor)
{
    public MySettings Current => settingsMonitor.CurrentValue;
}
```

**Follow-up question:**
You need live-updating configuration inside a singleton background service. Which of the three would you use, and why can't you use the others?

**Follow-up good answer:**
`IOptionsMonitor<T>`, because it's registered as a singleton and can therefore be injected into another singleton, and its `CurrentValue` reflects the latest configuration without needing a new request/scope. `IOptions<T>` is out because it's frozen at startup. `IOptionsSnapshot<T>` is out because it's scoped — the DI container will throw (in `Development`, per `ValidateScopes`) or silently misbehave if you try to inject a scoped service into a singleton's constructor.

**Glossary:**
- **`CurrentValue`** — `IOptionsMonitor<T>`'s always-up-to-date snapshot of the bound options.
- **Named options** — multiple distinct configurations of the same options type, distinguished by a string name.

**Mental model:**
Tests precise understanding of how DI lifetime rules interact with the options pattern — a very common "gotcha" question because picking the wrong one either throws at runtime or silently serves stale config.

**TL;DR:**
`IOptions<T>` is frozen at startup, `IOptionsSnapshot<T>` refreshes per-request but is scoped, `IOptionsMonitor<T>` is a singleton that's always current and supports change notifications.

**References:**
- [Options pattern in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-10.0)
- [Options pattern - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/options)

---

### Q6. How does a scoped service injected into convention-based middleware behave differently depending on whether it's injected via the constructor or via `InvokeAsync`'s parameters? {#q6}

**Question:**
How does a scoped service injected into convention-based middleware behave differently depending on whether it's injected via the constructor or via `InvokeAsync`'s parameters?

**Good answer:**
Convention-based middleware (the `Use<T>` pattern without implementing `IMiddleware`) is constructed exactly once, when the app starts — so it behaves like a singleton at the application level. If you inject a scoped service through its *constructor*, that scoped instance is captured once at startup and effectively becomes a captive dependency, shared across every request instead of being fresh per request. To get a genuinely per-request scoped service, you must add it as a parameter to the `InvokeAsync`/`Invoke` method instead — the framework resolves those parameters from the current request's DI scope on every call.

**Code example:**
```csharp
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    public AuditMiddleware(RequestDelegate next) => _next = next;

    // Resolved fresh per request from the current scope:
    public Task InvokeAsync(HttpContext context, IAuditService auditService)
    {
        auditService.Record(context.Request.Path);
        return _next(context);
    }
}
```

**Follow-up question:**
Is there a way to get constructor injection of scoped services for middleware without this per-app-lifetime pitfall?

**Follow-up good answer:**
Yes — implement `IMiddleware` and register the middleware type in DI (as scoped or transient). The `IMiddlewareFactory` then activates a new instance of that middleware *per request*, so scoped services can safely be injected via its constructor rather than being restricted to `InvokeAsync` parameters.

**Glossary:**
- **Convention-based middleware** — middleware added via `UseMiddleware<T>()` without implementing `IMiddleware`; instantiated once at startup.
- **`IMiddleware`/`IMiddlewareFactory`** — the factory-based activation model that constructs middleware per request.

**Mental model:**
A deep-cut internals question that separates candidates who've actually debugged a captive-dependency-in-middleware bug from those who've only used the framework at a surface level.

**TL;DR:**
Convention-based middleware is built once at startup, so constructor-injected scoped services become accidentally singleton-scoped — inject scoped dependencies via `InvokeAsync` parameters instead, or use `IMiddleware` for real per-request constructor injection.

**References:**
- [Write custom ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write?view=aspnetcore-10.0)
- [Factory-based middleware activation in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility?view=aspnetcore-10.0)

---

### Q7. What is a "captive dependency," and how does it happen in ASP.NET Core's DI container? {#q7}

**Question:**
What is a "captive dependency," and how does it happen in ASP.NET Core's DI container?

**Good answer:**
A captive dependency (a term coined by Mark Seemann) is when a longer-lived service holds a reference to a shorter-lived service, extending that shorter service's effective lifetime beyond what its registration promises. The classic case: a singleton takes a scoped service as a constructor dependency. The scoped service is resolved once, when the singleton is first constructed, and the singleton then holds that single instance forever — so every "scoped" operation performed through the singleton actually reuses the same instance across every request, defeating the purpose of scoping it (e.g., a scoped `DbContext` now being shared and mutated concurrently across unrelated requests).

**Code example:**
```csharp
// Anti-pattern: OrderRepository is Scoped, CacheWarmer is Singleton
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddSingleton<CacheWarmer>(); // captures IOrderRepository forever
```

**Follow-up question:**
How do you fix a singleton that genuinely needs to use a scoped service occasionally, without making the scoped service itself a singleton?

**Follow-up good answer:**
Inject `IServiceScopeFactory` (or `IServiceProvider`) into the singleton instead of the scoped service directly, and call `CreateScope()` (or `CreateAsyncScope()`) each time you need it, resolving the scoped service from that fresh scope's `ServiceProvider` and disposing the scope afterward (typically with a `using` block). This is exactly the pattern `BackgroundService`/`IHostedService` implementations use, since hosted services are themselves registered as singletons.

**Glossary:**
- **Captive dependency** — a shorter-lived service held alive beyond its intended scope by a longer-lived consumer.
- **`IServiceScopeFactory`** — the singleton-safe way to create a new DI scope on demand.

**Mental model:**
Tests whether the candidate has actually hit and diagnosed this bug in production, not just memorized the lifetime definitions from Q2.

**TL;DR:**
A captive dependency is a scoped (or transient) service accidentally kept alive for the app's lifetime because a singleton captured it via constructor injection — fix it by resolving the scoped service through a fresh `IServiceScopeFactory`-created scope instead.

**References:**
- [Dependency injection guidelines - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines)
- [Using Scoped Services From Singletons in ASP.NET Core — Milan Jovanović](https://milanjovanovic.tech/blog/using-scoped-services-from-singletons-in-aspnetcore)

---

### Q8. What happens if code resolves a scoped service directly from the root `IServiceProvider`, and does the behavior differ between environments? {#q8}

**Question:**
What happens if code resolves a scoped service directly from the root `IServiceProvider`, and does the behavior differ between environments?

**Good answer:**
By default, the `ServiceProviderOptions.ValidateScopes` flag is set based on `IsDevelopment()` — enabled in Development, disabled in Production. With validation enabled (Development), resolving a scoped service from the root provider throws an `InvalidOperationException` immediately, which is exactly the point: it surfaces the bug loudly during development/testing. With validation disabled (Production, by default), no exception is thrown — the service is instead silently created once from the root container and effectively promoted to singleton lifetime, which can quietly cause shared-state bugs that only show up in production and are hard to reproduce locally.

**Code example:**
```csharp
// silently becomes a "singleton" in Production, throws in Development
var repo = app.Services.GetRequiredService<IOrderRepository>(); // IOrderRepository is Scoped
```

**Follow-up question:**
Given that Production doesn't validate scopes by default, how would you make sure this bug is caught before it ships?

**Follow-up good answer:**
Explicitly enable scope validation in all environments (or at least in a staging/test environment configured like production) by passing `new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true }` to `CreateDefaultBuilder`/`Host.ConfigureServices`'s provider factory, or via `UseDefaultServiceProvider(options => ...)`. `ValidateOnBuild` additionally catches missing/misconfigured registrations at container-build time rather than lazily on first resolution.

**Glossary:**
- **`ValidateScopes`** — the `ServiceProviderOptions` flag controlling whether resolving a scoped service outside a scope throws.
- **`ValidateOnBuild`** — a related flag that validates the entire service graph can be constructed at startup, not lazily.

**Mental model:**
Probes whether the candidate knows this safety net exists at all, and — more importantly — that it silently disappears in Production, which is exactly when the bug becomes dangerous.

**TL;DR:**
Resolving a scoped service from the root provider throws in Development (scope validation is on by default there) but silently promotes it to singleton behavior in Production — enable `ValidateScopes`/`ValidateOnBuild` everywhere to catch this before it ships.

**References:**
- [ServiceProviderOptions.ValidateScopes Property | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.serviceprovideroptions.validatescopes?view=net-8.0)

---

### Q9. What's the risk of misusing `IHttpContextAccessor`, and how does it actually track "the current" `HttpContext`? {#q9}

**Question:**
What's the risk of misusing `IHttpContextAccessor`, and how does it actually track "the current" `HttpContext`?

**Good answer:**
`IHttpContextAccessor` is registered as a singleton, but the `HttpContext` it hands back is always the one for the *currently executing request*, because internally it relies on `AsyncLocal<T>` to flow the context through the async call chain of that specific request — not because the accessor itself stores per-request state directly. The misuse pattern is capturing the `HttpContext` once (e.g., assigning `accessor.HttpContext` to a field during a singleton's construction/initialization) instead of reading `accessor.HttpContext` fresh every time it's needed — the captured reference can end up stale, null, or pointing at the wrong request depending on when it was captured. There's also a real performance cost: `AsyncLocal<T>` has a non-trivial overhead on async flows, which is why it isn't registered by default and must be added explicitly via `AddHttpContextAccessor()`.

**Code example:**
```csharp
// Correct: read it fresh at the point of use, every time
public class AuditLogger(IHttpContextAccessor accessor)
{
    public void LogCurrentUser() =>
        Console.WriteLine(accessor.HttpContext?.User.Identity?.Name);
}
```

**Follow-up question:**
Since `IHttpContextAccessor` is a singleton, is it safe to inject it into another singleton service?

**Follow-up good answer:**
Yes — that's exactly the supported use case (it's precisely why it's a singleton itself: so it can be injected anywhere). The singleton holding the accessor just has to always read `.HttpContext` at the moment it needs it (per call), rather than caching the value, since the `AsyncLocal`-backed context changes per request regardless of the accessor's own singleton lifetime.

**Glossary:**
- **`AsyncLocal<T>`** — a .NET primitive that flows ambient data through an async call chain without explicit parameter passing.
- **`IHttpContextAccessor`** — the singleton service exposing the current request's `HttpContext` via ambient (`AsyncLocal`) state.

**Mental model:**
Tests whether the candidate understands *why* a singleton can safely expose "current request" data (ambient async-flowed state) versus assuming singleton automatically means "one fixed value forever."

**TL;DR:**
`IHttpContextAccessor` is a singleton, but its `HttpContext` property is backed by `AsyncLocal<T>` and reflects the currently executing request — the bug is capturing that value once instead of reading it fresh at each use.

**References:**
- [IHttpContextAccessor Interface | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.ihttpcontextaccessor?view=aspnetcore-10.0)

---

### Q10. What breaks if exception-handling middleware is registered at the end of the pipeline instead of the start, or `UseCors` is placed after `UseAuthentication`? {#q10}

**Question:**
What breaks if exception-handling middleware is registered at the end of the pipeline instead of the start, or `UseCors` is placed after `UseAuthentication`?

**Good answer:**
Exception-handling middleware (`UseExceptionHandler`) has to wrap every other middleware to catch what they throw — since a `try/catch`-style wrapper can only catch exceptions from code that runs *after* it in the call chain, registering it late means exceptions thrown by earlier middleware propagate unhandled instead of being caught and turned into a proper error response. For CORS: browsers send a preflight `OPTIONS` request and expect CORS headers on it regardless of whether the actual request would be authenticated; if `UseCors` runs after `UseAuthentication`/`UseAuthorization`, an unauthenticated cross-origin request gets rejected by auth *before* CORS ever adds the headers the browser needs to even see a proper error — the browser just reports a generic CORS failure instead of the real 401/403.

**Code example:**
```csharp
app.UseExceptionHandler("/error"); // first
app.UseRouting();
app.UseCors();                     // before auth
app.UseAuthentication();
app.UseAuthorization();
```

**Follow-up question:**
Static file middleware is also recommended early in the pipeline — why, and is that for the same reason as exception handling?

**Follow-up good answer:**
No, it's a different reason: performance/simplicity, not correctness. Static files are typically meant to be served unconditionally without going through auth, routing, or other heavier middleware, so placing `UseStaticFiles` early lets it short-circuit matched requests immediately, avoiding the cost of running the rest of the pipeline for something as simple as serving a `.css` file.

**Glossary:**
- **Preflight request** — the browser's automatic `OPTIONS` request checking CORS permissions before the real cross-origin request.
- **`UseExceptionHandler`** — middleware that catches unhandled exceptions from everything registered after it and produces an error response.

**Mental model:**
Directly tests whether "middleware order matters" is understood as a causal mechanism (what each piece needs from what ran before it) rather than a memorized checklist to follow blindly.

**TL;DR:**
Exception handling must wrap everything else (register it first) to catch downstream exceptions; CORS must run before auth so preflight/cross-origin requests get proper CORS headers even when the real request would fail authentication.

**References:**
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)

---

### Q11. How would you diagnose that a production bug is being caused by middleware ordering rather than application logic? {#q11}

**Question:**
How would you diagnose that a production bug is being caused by middleware ordering rather than application logic?

**Good answer:**
Start by logging (or attaching a debugger/breakpoint to) a temporary diagnostic middleware placed at the very start and very end of the pipeline, timestamping entry/exit — this tells you whether the request even reaches later middleware, and in what order components actually execute versus what you assumed from reading `Program.cs` top to bottom. For routing/auth-specific issues, inspect `HttpContext.GetEndpoint()` at different points in the pipeline to see exactly when (and whether) an endpoint gets selected. `ASP.NET Core`'s built-in logging (set the `Microsoft.AspNetCore` category to `Debug` or `Trace`) also logs which middleware and endpoint handled each request, which quickly reveals ordering issues like auth running before routing or CORS never getting a chance to add headers.

**Code example:**
```csharp
app.Use(async (ctx, next) =>
{
    Console.WriteLine($"before: {ctx.GetEndpoint()?.DisplayName ?? "none"}");
    await next(ctx);
});
```

**Follow-up question:**
How would you validate a fix for a middleware-ordering bug without waiting for it to reproduce in production again?

**Follow-up good answer:**
Write an integration test using `WebApplicationFactory<T>` that exercises the exact scenario (e.g., an unauthenticated cross-origin request, or a request that should be caught by the global exception handler) and asserts on the response — status code, headers (like CORS headers being present even on a 401), or body. Because the test spins up the real configured pipeline, it will fail if the ordering regresses again, turning a one-off production incident into a permanent regression guard.

**Glossary:**
- **`WebApplicationFactory<T>`** — the ASP.NET Core test host that boots the real app pipeline in-process for integration testing.

**Mental model:**
This is the "performance/production-diagnosis" angle applied to correctness rather than raw speed — tests whether the candidate has a systematic method rather than "add print statements and guess."

**TL;DR:**
Diagnose middleware-ordering bugs by timestamping/logging at pipeline boundaries and inspecting `GetEndpoint()` at each stage, then lock the fix in with a `WebApplicationFactory`-based integration test asserting on the exact previously-broken scenario.

**References:**
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)
- [Routing in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-10.0)

---

### Q12. What would you check first if an ASP.NET Core app's first request after startup is dramatically slower than subsequent ones? {#q12}

**Question:**
What would you check first if an ASP.NET Core app's first request after startup is dramatically slower than subsequent ones?

**Good answer:**
This is the classic "cold start" symptom, and the usual suspects, roughly in order of likely impact: JIT compilation warming up hot paths for the first time (mitigated by ReadyToRun/AOT compilation or tiered PGO in modern .NET); the DI container building/validating its full service graph lazily on first resolution rather than at startup; EF Core's first query paying the cost of building and caching the model and compiling the query; and any middleware or service doing expensive one-time work (loading configuration, warming caches, opening connection pools) lazily on first use instead of eagerly at startup. The fix pattern is usually to move that expensive first-time work into an explicit startup step (e.g., an `IHostedService` that runs before the app starts accepting traffic, or `app.Services.GetRequiredService<T>()` calls at startup to force eager resolution) rather than letting the first real user pay for it.

**Follow-up question:**
How would you actually measure where the time goes in that first slow request, rather than guessing?

**Follow-up good answer:**
Use `dotnet-trace` to capture a trace across app startup through the first few requests and inspect it in PerfView or the Visual Studio profiler — this shows JIT time, GC pauses, and where wall-clock time is actually spent, broken down by method. For a narrower EF Core suspicion specifically, enable EF Core's logging/`IDbCommandInterceptor` to see how long model-building and the first compiled query take versus subsequent ones.

**Glossary:**
- **ReadyToRun (R2R)** — precompiling IL to native code ahead of time to reduce JIT work at startup.
- **Cold start** — the elevated latency of the first request(s) after an app starts, before caches/JIT/pools warm up.

**Mental model:**
The performance-diagnosis-methodology question for this file's subtopic — checks whether the candidate has a mental checklist for startup latency specifically, distinct from steady-state request latency.

**TL;DR:**
First-request slowness is usually JIT warmup, lazy DI container graph construction, or EF Core's first-query model-building cost — diagnose with `dotnet-trace`/PerfView, and fix by moving expensive first-time work into an explicit eager startup step.

**References:**
- [ASP.NET Core Performance Best Practices | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/performance-best-practices?view=aspnetcore-10.0)

---

### Q13. How does the Chain of Responsibility pattern specifically map onto ASP.NET Core's middleware pipeline, beyond just "requests pass through handlers"? {#q13}

**Question:**
How does the Chain of Responsibility pattern specifically map onto ASP.NET Core's middleware pipeline, beyond just "requests pass through handlers"?

**Good answer:**
In the classic Chain of Responsibility pattern, each handler holds a reference to the next handler and independently decides whether to process the request itself, pass it along unchanged, or do some work and then pass it along. ASP.NET Core's middleware pipeline implements exactly this: each `RequestDelegate` closes over a reference to the "next" delegate, can inspect/mutate the `HttpContext` before and after calling it, and can choose not to call it at all (short-circuiting, Q1). The key structural benefit the pattern gives here is the same one Chain of Responsibility always gives: senders (the framework dispatching a request) are decoupled from receivers (whichever middleware ultimately handles it), so middleware can be composed, reordered, and reused across different apps without any of them needing to know about the others.

**Follow-up question:**
What's a concrete downside of Chain of Responsibility (and therefore of the middleware model) that this decoupling introduces?

**Follow-up good answer:**
There's no built-in guarantee that *any* handler in the chain actually processes the request — if every middleware just calls `next` and nothing ever short-circuits with a real response, the request falls through with no meaningful handling (in ASP.NET Core this typically surfaces as a 404 once it reaches routing with no matching endpoint). The decoupling that makes the pattern flexible also means correctness depends entirely on the chain being assembled correctly, with no compiler-level check that some handler will actually respond.

**Glossary:**
- **Sender/receiver decoupling** — the core benefit of Chain of Responsibility: the code issuing a request doesn't need to know which handler will ultimately deal with it.

**Mental model:**
Pushes past "I've heard of Chain of Responsibility" to see if the candidate can connect the theoretical pattern's specific mechanics and trade-offs to the concrete framework feature.

**TL;DR:**
Middleware *is* Chain of Responsibility: each link decides independently whether to handle, pass along, or short-circuit, decoupling the framework's dispatch from whichever middleware ultimately responds — at the cost of no compile-time guarantee that something in the chain actually will.

**References:**
- [Chain of Responsibility — Refactoring.Guru](https://refactoring.guru/design-patterns/chain-of-responsibility)
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)

---

### Q14. What is Inversion of Control, and how does Dependency Injection implement it, per Martin Fowler's original formulation? {#q14}

**Question:**
What is Inversion of Control, and how does Dependency Injection implement it, per Martin Fowler's original formulation?

**Good answer:**
Inversion of Control is the general principle that a framework or container — not your own code — controls the flow of a program and decides when to call into application-supplied code (the "Hollywood Principle": don't call us, we'll call you). Fowler's 2004 article specifically clarifies "Dependency Injection" as the name for the more specific pattern where a component's dependencies are supplied to it from the outside (via constructor, setter, or interface injection) rather than the component constructing or looking them up itself — he introduced the more precise term partly because "Inversion of Control" alone was too generic to distinguish this pattern from other IoC-flavored techniques like the Service Locator pattern. In ASP.NET Core, the built-in container is exactly this: you register implementations centrally, and the framework supplies (injects) the right instance into each constructor that asks for it, rather than each class new-ing up or looking up its own dependencies.

**Follow-up question:**
Fowler's article also contrasts Dependency Injection with the Service Locator pattern. What's the key difference, and why does he generally favor DI?

**Follow-up good answer:**
With a Service Locator, a component actively asks a central registry for its dependencies at runtime (e.g., `serviceLocator.Get<IFoo>()` inside a method body) — the component still has an explicit, visible dependency on the locator itself, and its *actual* dependencies are hidden inside method bodies rather than declared in its constructor signature. With Dependency Injection, dependencies are declared explicitly as constructor (or setter) parameters, making them visible from the outside and testable by simply passing in different implementations — no framework/container awareness is required inside the component itself. Fowler favors DI because that explicitness makes a class's true dependencies obvious just by reading its signature, which Service Locator obscures.

**Glossary:**
- **Hollywood Principle** — "don't call us, we'll call you" — the framework calls your code, not the other way around.
- **Service Locator** — a pattern where components pull dependencies from a central registry at runtime, rather than having them pushed in.

**Mental model:**
Checks whether the candidate can articulate the theory precisely enough to distinguish DI from IoC-in-general and from Service Locator specifically — a common area where candidates use the terms interchangeably without understanding the distinction Fowler was making.

**TL;DR:**
Inversion of Control is the framework controlling program flow instead of your code; Dependency Injection is Fowler's more precise name for supplying dependencies explicitly (usually via constructor) instead of a component fetching them itself via a Service Locator.

**References:**
- [Inversion of Control Containers and the Dependency Injection pattern — Martin Fowler](https://martinfowler.com/articles/injection.html)

---

### Q15. What real-world problem does a composable middleware pipeline solve that one big monolithic request handler wouldn't? {#q15}

**Question:**
What real-world problem does a composable middleware pipeline solve that one big monolithic request handler wouldn't?

**Good answer:**
Cross-cutting concerns — logging, exception handling, authentication, CORS, response compression, static file serving — apply to most or all requests but aren't specific to any one endpoint's business logic. Without middleware, every action/handler would need to duplicate that logic (or rely on inheritance/copy-paste), making it easy for behavior to drift between endpoints and hard to change centrally. A pipeline lets each concern live in exactly one reusable, independently testable component, composed declaratively in one place (`Program.cs`), and reused unchanged across completely different apps (e.g., the same `UseCors()` middleware works regardless of what the app's actual business logic is).

**Follow-up question:**
Middleware handles *global* cross-cutting concerns well. What's a case where that global scope becomes a limitation, and how does ASP.NET Core address it?

**Follow-up good answer:**
When a concern needs to run only for some endpoints, or needs access to MVC-specific context (route values, model-bound parameters, the action result before it's written to the response), plain middleware is either too broad (applies to every request unless you manually branch with `MapWhen`) or lacks the right context. ASP.NET Core addresses this with **filters** (Q20), which run inside the MVC/Minimal API pipeline with access to action-level context and can be scoped to a specific controller, action, or endpoint rather than globally.

**Glossary:**
- **Cross-cutting concern** — behavior that spans many parts of an app (logging, auth, error handling) rather than being specific to one feature.

**Mental model:**
Tests the "why does this feature exist" angle directly — a candidate who can only describe *what* middleware does, not *why* it's structured as a pipeline, likely hasn't reasoned about the alternative (duplicated logic per handler).

**TL;DR:**
Middleware exists so cross-cutting concerns (logging, auth, CORS, exception handling) live in one reusable place instead of being duplicated across every request handler; when a concern needs action-level context or per-endpoint scoping, filters fill that gap instead.

**References:**
- [ASP.NET Core middleware | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)

---

### Q16. What problem does the Options pattern solve compared to reading `IConfiguration["Some:Key"]` directly wherever a setting is needed? {#q16}

**Question:**
What problem does the Options pattern solve compared to reading `IConfiguration["Some:Key"]` directly wherever a setting is needed?

**Good answer:**
Reading raw string keys scattered throughout the codebase means every access site has to know the exact key path and target type, there's no compile-time checking of key names or shapes, and typos in a key string fail silently at runtime by returning null rather than failing fast. The Options pattern binds a section of configuration into a strongly-typed POCO once, via `services.Configure<MySettings>(config.GetSection("MySettings"))`, so every consumer just injects `IOptions<MySettings>` (or the snapshot/monitor variants from Q5) and gets a typed, validated object — the binding logic and key names live in exactly one place, and typos in the config source produce a missing/default property value on a known type rather than a silent magic-string mismatch scattered across the app.

**Code example:**
```csharp
builder.Services.Configure<MySettings>(builder.Configuration.GetSection("MySettings"));
// consumer:
public class Worker(IOptions<MySettings> options)
{
    private readonly MySettings _settings = options.Value;
}
```

**Follow-up question:**
Where does that configuration value actually come from, if `appsettings.json` and an environment variable both define the same key?

**Follow-up good answer:**
Configuration providers are read in the order they're added, and later providers override earlier ones for the same key. The typical default order is `appsettings.json`, then `appsettings.{Environment}.json`, then user secrets (Development only), then environment variables, then command-line arguments — so an environment variable setting the same key as `appsettings.json` wins, and command-line arguments would win over everything, since they're added last.

**Glossary:**
- **POCO** — Plain Old CLR Object; here, a simple class with properties matching the configuration section's shape.
- **Configuration provider** — a source (JSON file, env vars, command line, etc.) that `IConfiguration` reads key-value pairs from, in a defined order.

**Mental model:**
Connects a "why does this abstraction exist" question to a very concrete, common follow-up (override precedence) that every candidate who's actually deployed a multi-environment app has had to reason about.

**TL;DR:**
The Options pattern replaces scattered magic-string `IConfiguration` lookups with one strongly-typed, centrally-bound settings object; later-added configuration providers (env vars, then command-line args) override earlier ones (appsettings.json) for the same key.

**References:**
- [Options pattern in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-10.0)
- [Configuration in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-10.0)

---

### Q17. What are "keyed services," and what problem do they solve that plain DI registration can't? {#q17}

**Question:**
What are "keyed services," and what problem do they solve that plain DI registration can't?

**Good answer:**
Keyed services (introduced in .NET 8) let you register multiple implementations of the same interface, each associated with a distinct key, and resolve a specific one by that key — via `AddKeyedSingleton`/`AddKeyedScoped`/`AddKeyedTransient` for registration, and the `[FromKeyedServices(key)]` attribute (or `IServiceProvider.GetKeyedService`) for resolution. Before this existed, having multiple implementations of `INotificationSender` (say, one for email, one for SMS) that a consumer needed to pick between at construction time required workarounds like a custom factory, an `IEnumerable<INotificationSender>` filtered by some discriminator property, or the Service Locator anti-pattern (Q14) — keyed services make "give me *this specific* implementation" a first-class DI feature.

**Code example:**
```csharp
builder.Services.AddKeyedSingleton<INotificationSender, EmailSender>("email");
builder.Services.AddKeyedSingleton<INotificationSender, SmsSender>("sms");

public class AlertService([FromKeyedServices("sms")] INotificationSender sender) { ... }
```

**Follow-up question:**
Before keyed services existed, what was the most common workaround, and what was wrong with it?

**Follow-up good answer:**
The most common workaround was injecting `IEnumerable<INotificationSender>` (the container resolves all registered implementations into a collection) and then selecting the right one at runtime based on some discriminator property or type check — this pushes selection logic into every consumer, is easy to get wrong (e.g., forgetting a case), and doesn't scale cleanly when a class only ever needs exactly one specific implementation rather than "all of them, then pick." Keyed services move that selection to the registration/injection point instead, where it belongs.

**Glossary:**
- **Keyed service** — a DI registration associated with a specific key, allowing multiple implementations of the same interface to coexist and be resolved individually.

**Mental model:**
An "advanced/recent feature" check that also reveals whether the candidate understands *why* it was added (i.e., what pain it removes), not just that the API exists.

**TL;DR:**
Keyed services (.NET 8+) let you register and resolve multiple implementations of the same interface by an explicit key, replacing older workarounds like filtering an `IEnumerable<T>` of all implementations at each call site.

**References:**
- [Dependency injection in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0)

---

### Q18. How does factory-based middleware (`IMiddleware`/`IMiddlewareFactory`) differ from convention-based middleware, and what does it unlock? {#q18}

**Question:**
How does factory-based middleware (`IMiddleware`/`IMiddlewareFactory`) differ from convention-based middleware, and what does it unlock?

**Good answer:**
Convention-based middleware (registered via `UseMiddleware<T>()` on a plain class following the `RequestDelegate`-constructor / `InvokeAsync`-method convention) is instantiated once at app startup and reused for every request (Q6). Factory-based middleware instead implements the `IMiddleware` interface and is registered in the DI container as a scoped or transient service; `UseMiddleware<T>()` detects that it implements `IMiddleware` and, instead of the convention-based activation logic, delegates creation to the registered `IMiddlewareFactory`, which calls `Create(Type)` to produce a *new instance per request* (and `Release(IMiddleware)` to clean it up afterward). This unlocks safe constructor injection of scoped services directly into the middleware, without needing the `InvokeAsync`-parameter workaround from Q6.

**Code example:**
```csharp
public class AuditMiddleware(RequestDelegate next, IAuditService auditService) : IMiddleware
{
    public Task InvokeAsync(HttpContext context, RequestDelegate _) =>
        next(context); // auditService is safely per-request here
}
builder.Services.AddScoped<AuditMiddleware>();
```

**Follow-up question:**
Given this benefit, why isn't `IMiddleware` the default recommendation for all custom middleware?

**Follow-up good answer:**
It requires explicit DI registration of the middleware type itself (as scoped or transient) in addition to the `UseMiddleware<T>()` call, adding a small amount of ceremony most middleware doesn't need — plenty of middleware has no dependencies at all, or only needs singleton dependencies, in which case convention-based activation with `InvokeAsync` parameters (Q6) is simpler and works fine. `IMiddleware` earns its extra setup specifically when a middleware genuinely needs scoped dependencies available via constructor injection.

**Glossary:**
- **`IMiddlewareFactory`** — the DI-integrated factory (`Create`/`Release`) responsible for activating `IMiddleware` instances per request.

**Mental model:**
Tests whether the candidate sees this as a deliberate trade-off (a bit more registration ceremony for safe constructor-injected scoped state) rather than "the fancy modern way to always do it."

**TL;DR:**
`IMiddleware` + `IMiddlewareFactory` creates a fresh middleware instance per request via DI, safely allowing constructor-injected scoped services — at the cost of needing explicit registration, which is why simpler convention-based middleware remains the default for dependency-light cases.

**References:**
- [Factory-based middleware activation in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility?view=aspnetcore-10.0)

---

### Q19. Minimal APIs vs. MVC controllers — what factors should actually drive that choice? {#q19}

**Question:**
Minimal APIs vs. MVC controllers — what factors should actually drive that choice?

**Good answer:**
Both sit on the same underlying fundamentals — hosting, endpoint routing, DI, middleware, auth — so the choice isn't about capability so much as structure and ergonomics at scale. Controllers bring built-in conventions (model binding, action filters, `[ApiController]` behaviors), strong discoverability for large teams/codebases, and mature extension points for enforcing cross-cutting policy consistently across many endpoints. Minimal APIs map routes directly to handlers (`MapGet`/`MapPost`/etc.) with less ceremony, favoring an explicit, endpoint-first style well suited to smaller focused services (microservices, internal APIs) — but that same flexibility means the team has to intentionally impose structure (route groups, endpoint filters, feature-based file organization) rather than getting it from framework conventions by default. For a large, long-lived API surface, controllers' conventions tend to pay for themselves; for small, tightly-scoped services, minimal APIs' lower ceremony tends to win.

**Follow-up question:**
Is there a meaningful runtime performance difference between the two that should factor into the decision?

**Follow-up good answer:**
Minimal APIs generally have somewhat lower per-request overhead since they skip some of the MVC pipeline machinery (like the full filter pipeline and some reflection-based model binding paths) that controllers always pay for — but for the overwhelming majority of real applications, this difference is dwarfed by I/O latency (database calls, downstream HTTP calls), so performance alone is rarely the deciding factor. It's a legitimate tiebreaker only in extremely high-throughput, latency-sensitive services where every microsecond of framework overhead is being actively measured and optimized.

**Glossary:**
- **`[ApiController]`** — the controller-base attribute enabling automatic model-validation responses, binding source inference, and other API-specific conventions.

**Mental model:**
A classic trade-off question — tests whether the candidate defaults to a dogmatic "always use X" answer or can articulate the actual factors (team size, API longevity, need for conventions vs. flexibility) that should drive the decision.

**TL;DR:**
Both share the same fundamentals; choose controllers for large/long-lived APIs that benefit from built-in conventions and consistency, and minimal APIs for small, focused services where lower ceremony matters more than default structure — raw performance rarely decides it.

**References:**
- [APIs overview | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis?view=aspnetcore-10.0)

---

### Q20. When would you reach for a middleware versus an action filter for a cross-cutting concern, and is the built-in DI container ever not enough? {#q20}

**Question:**
When would you reach for a middleware versus an action filter for a cross-cutting concern, and is the built-in DI container ever not enough?

**Good answer:**
Reach for middleware when the concern applies to the raw HTTP pipeline broadly and doesn't need MVC-specific context — logging, exception handling, CORS, authentication, response compression. Reach for a filter when the concern needs access to action-level context (route/model-bound parameters, the action result, controller metadata) or needs to apply to only specific controllers/actions rather than globally — e.g., custom authorization logic tied to a specific action's arguments, or shaping/validating a particular action's result. On the DI container question: the built-in container deliberately stays simple (constructor injection, three lifetimes, no property injection, no complex interception/AOP support) — it's enough for the large majority of apps, but teams that need property injection, more elaborate lifetime scopes, decorator/interceptor support, or assembly-scanning conventions often bring in a third-party container (e.g., Autofac) via ASP.NET Core's pluggable `IServiceProviderFactory` extension point, which the framework explicitly supports rather than treating the built-in container as the only option.

**Follow-up question:**
What's a concrete example of something the built-in container doesn't support that might push a team toward a third-party container?

**Follow-up good answer:**
Decorator registration — wrapping an existing registered implementation with a decorator that adds behavior (caching, retry, logging) around it while still satisfying the same interface — isn't natively supported by the built-in container without manual factory-lambda workarounds. Third-party containers like Autofac or Scrutor (a library that extends the built-in container) provide this as a first-class feature, which is a common concrete reason teams reach beyond the built-in container specifically for this pattern.

**Glossary:**
- **`IServiceProviderFactory`** — the extension point ASP.NET Core exposes to swap in a third-party DI container while keeping the rest of the hosting model unchanged.
- **Decorator registration** — wrapping a registered service with another implementation of the same interface that adds behavior around it.

**Mental model:**
A synthesis trade-off question closing out the set — checks whether the candidate can reason about tool selection (middleware vs. filter vs. even swapping the container) based on actual requirements rather than habit.

**TL;DR:**
Use middleware for pipeline-wide concerns without MVC context, filters for action-scoped concerns needing that context; the built-in DI container covers most needs but lacks things like native decorator support, which is the usual concrete reason teams adopt a third-party container via `IServiceProviderFactory`.

**References:**
- [Filters in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-10.0)
- [Dependency injection in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=dotnet&tags=aspnet-core-middleware-di-and-configuration&autostart=1" | relative_url }})
