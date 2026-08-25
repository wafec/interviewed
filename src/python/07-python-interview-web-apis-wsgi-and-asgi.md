---
layout: default
title: "Python Interview — Web APIs (WSGI/ASGI)"
---

# Python Interview — Web APIs (WSGI/ASGI)

This set covers building APIs in Python specifically: the WSGI and ASGI
server/application interfaces, how Flask and FastAPI actually work on top
of them, production deployment with gunicorn/uvicorn, and the performance
diagnosis and pitfalls that come up when these apps don't scale the way
people expect.

### Q1. What is WSGI, and what problem does it solve? {#q1}

**Question:**
What is WSGI, and what problem does it solve?

**Good answer:**
WSGI (Web Server Gateway Interface, PEP 3333) is a standard interface
between a Python web server and a Python web application/framework. Before
WSGI, every framework (Zope, early Django, etc.) had its own bespoke way of
talking to a server, so a given framework only worked with servers that
specifically supported it. WSGI defines one simple contract — an
application is a callable that takes `environ` and `start_response` and
returns an iterable of bytestrings — so any WSGI-compliant server (Apache's
mod_wsgi, gunicorn, uWSGI, the stdlib's `wsgiref`) can run any
WSGI-compliant application or framework (Flask, Django, Pyramid) without
either side knowing about the other's internals.

**Code example:**
```python
def application(environ, start_response):
    status = "200 OK"
    headers = [("Content-Type", "text/plain")]
    start_response(status, headers)
    return [b"Hello, WSGI"]
```

**Follow-up question:**
What can't WSGI do, and why?

**Follow-up good answer:**
WSGI is fundamentally synchronous and tied to a single request/response
cycle: the server calls the application once per request, waits for it to
return an iterable, and that's it. It has no facility for a connection that
outlives a single response — no WebSockets, no server-sent events held
open, no async I/O within the request. That gap is exactly what ASGI was
created to fill.

**Glossary:**
- **WSGI** — Web Server Gateway Interface, PEP 3333's spec for the
  callable contract between a Python server and application.
- **`environ`** — the dict of request data (CGI-style variables, headers,
  the input stream) passed to the application callable.

**Mental model:**
Tests whether the candidate understands WSGI as an *interface/contract*
(like a plug standard) rather than a framework or library — the classic
"why does this abstraction exist" question.

**TL;DR:**
WSGI is the standard callable contract (PEP 3333) that lets any Python web
server run any Python web framework, but it's strictly synchronous
request/response — no WebSockets or long-lived connections.

**References:**
- [PEP 3333 — Python Web Server Gateway Interface v1.0.1](https://peps.python.org/pep-3333/)

---

### Q2. What is ASGI, and how does it differ from WSGI? {#q2}

**Question:**
What is ASGI, and how does it differ from WSGI?

**Good answer:**
ASGI (Asynchronous Server Gateway Interface) is the async successor/superset
to WSGI. Instead of a plain callable that gets `environ`/`start_response`
and returns once, an ASGI application is a single **async callable** that
takes `scope`, `receive`, and `send`: `scope` describes the connection
(HTTP request, WebSocket, lifespan event), `receive` is an awaitable the
app calls to get the next incoming event, and `send` is an awaitable the
app calls to emit outgoing events. Because it's event-based rather than
"one call in, one return out," the same interface naturally supports
HTTP request/response, WebSockets, and long-lived connections, and lets the
application do async I/O (`await` a database call) without blocking the
server process.

**Code example:**
```python
async def app(scope, receive, send):
    assert scope["type"] == "http"
    event = await receive()
    await send({"type": "http.response.start", "status": 200,
                "headers": [(b"content-type", b"text/plain")]})
    await send({"type": "http.response.body", "body": b"Hello, ASGI"})
```

**Follow-up question:**
How does a framework like FastAPI relate to ASGI — is FastAPI itself the
server?

**Follow-up good answer:**
No. FastAPI (built on Starlette) is an ASGI *application/framework* — it
implements the ASGI callable interface and adds routing, dependency
injection, and request/response validation on top. It still needs an ASGI
*server* (uvicorn, hypercorn, daphne) to actually accept TCP connections,
speak HTTP, and translate that into `scope`/`receive`/`send` calls into the
app. The same separation of concerns as WSGI: the server handles the
network, the app/framework handles the logic.

**Glossary:**
- **ASGI** — Asynchronous Server Gateway Interface, the async
  successor/superset of WSGI.
- **`scope`** — the dict describing one connection's type and metadata for
  its lifetime.

**Mental model:**
Tests whether the candidate can articulate *why* ASGI exists as more than
"the async version of WSGI" — specifically the event-based model that
makes WebSockets and streaming possible.

**TL;DR:**
ASGI is an async, event-based interface (`scope`/`receive`/`send`) that
generalizes WSGI's request/response model to support WebSockets and
long-lived connections; frameworks like FastAPI implement it but still need
an ASGI server like uvicorn to run.

**References:**
- [ASGI specification — main.html](https://asgi.readthedocs.io/en/latest/specs/main.html)

---

### Q3. Walk through exactly what a WSGI server passes into the application, and what the application must return. {#q3}

**Question:**
Walk through exactly what a WSGI server passes into the application, and
what the application must return.

**Good answer:**
The server calls `application(environ, start_response)`. `environ` is a
plain `dict` (not a subclass) containing CGI-style variables
(`REQUEST_METHOD`, `PATH_INFO`, `QUERY_STRING`, `HTTP_*` headers) plus
WSGI-specific keys like `wsgi.input` (a file-like object for the request
body), `wsgi.url_scheme`, and `wsgi.multithread`/`wsgi.multiprocess` flags.
`start_response` is a callable the app must invoke with the HTTP status
line (e.g. `"200 OK"`) and a list of `(name, value)` header tuples before
producing any body output; it optionally returns a `write()` callable for
imperative output, though returning/yielding an iterable of bytestrings is
the preferred style. The application's return value must be an iterable
yielding zero or more bytestrings, which the server iterates over and
streams to the client; if that iterable has a `close()` method, the server
must call it once done.

**Follow-up question:**
Why does the spec insist `environ` be a real `dict` and not a subclass or
custom mapping?

**Follow-up good answer:**
So every WSGI server and every middleware/application in the chain can rely
on exactly the same, predictable interface — plain dict semantics, no
surprises from an overridden `__getitem__` or custom behavior in a
subclass. WSGI middleware frequently reads and mutates `environ` to pass
information between layers (e.g. adding `wsgi.input`-related keys or
routing metadata); requiring a plain dict keeps that composition simple and
interoperable across unrelated implementations.

**Glossary:**
- **`start_response`** — the callable the WSGI app must call with status
  and headers before returning its body iterable.
- **`wsgi.input`** — the file-like object in `environ` used to read the
  request body.

**Mental model:**
Tests whether the candidate has actually read/implemented the WSGI contract
rather than only used it through a framework — a strong signal of someone
who understands what their framework is doing underneath.

**TL;DR:**
A WSGI app is called as `application(environ, start_response)`; it must
call `start_response(status, headers)` then return an iterable of
bytestrings as the body.

**References:**
- [PEP 3333 — Python Web Server Gateway Interface v1.0.1](https://peps.python.org/pep-3333/)

---

### Q4. Walk through the ASGI `scope`/`receive`/`send` contract for an HTTP request. {#q4}

**Question:**
Walk through the ASGI `scope`/`receive`/`send` contract for an HTTP
request.

**Good answer:**
An ASGI application is `async def app(scope, receive, send)`. `scope` is a
dict set once per connection describing its type (`"http"`, `"websocket"`,
or `"lifespan"`) and metadata (method, path, headers, ASGI version). To read
the incoming request, the app `await`s `receive()`, which yields the next
event dict as it becomes available (e.g. an `http.request` event carrying
the body). To send a response, the app `await`s `send(event)` — first an
`http.response.start` event with the status and headers, then one or more
`http.response.body` events carrying the body bytes. Because both
`receive` and `send` are awaitables rather than a single synchronous
call/return, the application can interleave I/O with the framework doing
other work, and the same shape extends naturally to WebSockets (repeated
`receive()`/`send()` calls over the connection's lifetime) rather than a
single request/response round-trip.

**Follow-up question:**
Why is `receive` a function the app calls, rather than the body just being
handed to the app directly like WSGI's `wsgi.input`?

**Follow-up good answer:**
Because the connection can be long-lived and multi-event (a WebSocket
sends many messages, an HTTP request can be chunked), the app needs to pull
events *as they arrive* rather than assume the whole body is available
up-front. Making `receive` an awaitable callable lets the app `await` the
next event whenever it's ready to process it, cooperating with the event
loop instead of blocking on a synchronous read — which is exactly the
capability WSGI's synchronous, single-shot model doesn't have.

**Glossary:**
- **`receive`/`send`** — the two awaitable callables an ASGI app uses to
  pull incoming events and push outgoing events.
- **Lifespan protocol** — the ASGI scope type used for startup/shutdown
  events, separate from a per-request scope.

**Mental model:**
Same internals-probing intent as the WSGI version of this question, but for
ASGI — checking the candidate understands the event-driven shape, not just
"it's async now."

**TL;DR:**
An ASGI app calls `await receive()` to pull incoming events and
`await send(event)` to emit outgoing ones (start-of-response, then
body chunks), which generalizes to any connection type, not just one HTTP
round trip.

**References:**
- [ASGI specification — main.html](https://asgi.readthedocs.io/en/latest/specs/main.html)

---

### Q5. How does FastAPI's `Depends()` dependency injection actually resolve a dependency? {#q5}

**Question:**
How does FastAPI's `Depends()` dependency injection actually resolve a
dependency?

**Good answer:**
You declare a parameter typed as `Annotated[SomeType, Depends(some_func)]`
(or the older `param: SomeType = Depends(some_func)` form). FastAPI
inspects the path operation's signature at route-registration time, sees
the `Depends()` marker, and builds a dependency tree — `some_func`'s own
parameters are resolved the same way (request query params, path params,
or further nested `Depends()`), recursively. At request time, FastAPI walks
that tree, calls each dependency function with its resolved arguments
(supporting both `def` and `async def` dependencies), and injects each
one's return value into the parameter of whatever depends on it — all the
way up to the path operation function itself.

**Code example:**
```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

async def common_parameters(q: str | None = None, limit: int = 100):
    return {"q": q, "limit": limit}

@app.get("/items/")
async def read_items(commons: Annotated[dict, Depends(common_parameters)]):
    return commons
```

**Follow-up question:**
If two different parameters in the same path operation depend (directly or
indirectly) on the same dependency function, is it called twice?

**Follow-up good answer:**
No — by default FastAPI caches a dependency's result per request: the
first time it's resolved, the return value is stored, and every other
dependant needing that same dependency in that request gets the cached
value instead of a fresh call. This is controlled by `use_cache`, which
defaults to `True`; passing `Depends(fn, use_cache=False)` forces that
specific usage to call `fn` again even within the same request (useful when
the dependency has side effects or must be re-evaluated).

**Glossary:**
- **`Depends()`** — FastAPI's marker for declaring a dependency to be
  resolved and injected.
- **`use_cache`** — the `Depends()` parameter controlling per-request
  dependency-result caching (default `True`).

**Mental model:**
Tests whether the candidate understands dependency injection as *resolved
by introspecting the function signature*, not "magic," and whether they
know the caching behavior well enough to avoid surprising duplicate-call
bugs.

**TL;DR:**
FastAPI resolves `Depends()` by introspecting each function's signature
into a dependency tree and calling each node once per request by default
(cached via `use_cache=True`), injecting results up to the path operation.

**References:**
- [FastAPI — Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI — Sub-dependencies (caching)](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/)

---

### Q6. What is Flask's application/request context, and why does it exist? {#q6}

**Question:**
What is Flask's application/request context, and why does it exist?

**Good answer:**
Flask needs objects like `current_app`, `g`, `request`, and `session` to be
reachable from anywhere in a view function or helper without threading them
through every function call as explicit arguments — and without importing a
global `app` instance directly, which causes circular imports and breaks
patterns like the application factory or reusable blueprints. Flask solves
this with "context locals": when a request comes in, Flask pushes a
*request context* (which layers on top of an *application context*) onto a
stack; `request`/`session` are then available for that request's duration,
and `current_app`/`g` for the app context's duration. When the
request/activity ends, the context is popped and those proxies stop
resolving.

**Follow-up question:**
How is this actually implemented under the hood, and is it thread-safe?

**Follow-up good answer:**
Context locals are implemented using Python's `contextvars` plus
Werkzeug's `LocalProxy`, which wraps a `contextvar` so attribute/method
access on the proxy forwards to whatever object is currently stored in that
context variable. Because `contextvars` are inherently per-thread (and
per-`asyncio`-task via context propagation), each worker/thread sees only
its own active context — the proxy itself can't be shared across workers,
each of which has its own separate context space. That's what makes it
safe under Flask's typical multi-threaded/multi-process deployment.

**Glossary:**
- **Context local** — data scoped to the currently active request/app
  activity rather than truly global.
- **`LocalProxy`** — Werkzeug's wrapper that forwards access to whatever
  object lives in the active `contextvar`.

**Mental model:**
Tests whether the candidate understands *why* a "global-looking" object
(`request`) is actually safe under concurrency — a common point of
confusion for people used to genuinely global state in other languages.

**TL;DR:**
Flask's `request`/`g`/`current_app`/`session` are `contextvars`-backed
proxies pushed onto a per-activity context stack, giving global-looking
access that's actually safely scoped per thread/task, not truly shared
state.

**References:**
- [Flask — The App and Request Context](https://flask.palletsprojects.com/en/latest/appcontext/)

---

### Q7. What happens if you access `flask.request` or `flask.g` outside of an active context? {#q7}

**Question:**
What happens if you access `flask.request` or `flask.g` outside of an
active context?

**Good answer:**
Flask raises a `RuntimeError`. Accessing `current_app` or `g` outside an
app context raises `RuntimeError: Working outside of application context.`
with guidance to push one manually via `with app.app_context():`.
Accessing `request` or `session` outside a request context raises
`RuntimeError: Working outside of request context.` — this most commonly
happens in tests or scripts that call view-adjacent code directly without
going through `app.test_client()` or `app.test_request_context()`.

**Code example:**
```python
from flask import request

def some_helper():
    return request.args.get("q")  # RuntimeError if called outside a request
```

**Follow-up question:**
You're seeing this error in what looks like normal request-handling code,
not a test. What's the most likely cause?

**Follow-up good answer:**
The most likely cause is that the code path runs before a request context
exists — commonly, initialization code (e.g. inside an extension's `init_app`,
or module-level code run at import time) trying to use `current_app`/`g`
before any request has come in and before an app context was manually
pushed. The fix is either to defer that access until inside a view/request,
or to explicitly wrap the initialization code in `with app.app_context():`.

**Glossary:**
- **App context** — the context that makes `current_app`/`g` available,
  active during a request or e.g. a CLI command.
- **Request context** — the context that additionally makes
  `request`/`session` available, active only during an HTTP request.

**Mental model:**
A very common real debugging scenario — tests whether the candidate has
actually hit and diagnosed this error, not just memorized what the context
objects are for.

**TL;DR:**
Accessing `request`/`g`/`current_app` outside an active context raises a
`RuntimeError` telling you exactly which context is missing; it's usually
caused by init-time code running before any context was pushed.

**References:**
- [Flask — The App and Request Context](https://flask.palletsprojects.com/en/latest/appcontext/)

---

### Q8. Why doesn't a Flask app deployed with gunicorn's default sync workers scale under high concurrent load? {#q8}

**Question:**
Why doesn't a Flask app deployed with gunicorn's default sync workers scale
under high concurrent load?

**Good answer:**
Gunicorn's default `sync` worker handles exactly one request at a time per
worker process — it's a pre-fork model where the arbiter manages N worker
processes, but within a single sync worker, a second request simply waits
until the first one finishes (and sync workers don't support keep-alive, so
each connection is closed after the response). If your handlers do any
blocking I/O (a slow downstream API, a synchronous DB query), that worker
is stuck for the whole duration, unable to serve anyone else. The fix is
either running more worker *processes* (bounded by CPU/memory — good for
CPU-bound work) or switching worker *class* to something that can juggle
many in-flight I/O-bound requests per process, like `gthread` (a thread
pool per worker) or `gevent` (greenlet-based, thousands of concurrent
connections per process).

**Follow-up question:**
If the app is CPU-bound rather than I/O-bound, would switching to gevent
workers help?

**Follow-up good answer:**
No — gevent's concurrency comes from cooperative greenlet-switching during
I/O waits (via monkey-patched sockets, etc.), which does nothing for a
request that's pegging the CPU the whole time; a CPU-bound greenlet won't
yield control until it's done, blocking every other greenlet in that
worker. For CPU-bound workloads, the right lever is more sync/gthread
*worker processes* (up to roughly the number of CPU cores, since the GIL
limits true parallelism within one process anyway) rather than an
async/greenlet worker class.

**Glossary:**
- **Pre-fork model** — gunicorn's arbiter process manages a pool of worker
  processes, each independently accepting connections.
- **`gthread`** — gunicorn's threaded worker class, running a thread pool
  per worker process.

**Mental model:**
This is the classic performance-diagnosis question for this topic: does
the candidate know to ask "I/O-bound or CPU-bound?" before reaching for a
worker-class change, rather than cargo-culting "switch to async/gevent."

**TL;DR:**
Sync workers handle one request at a time per process, so slow I/O in a
handler blocks that whole worker; fix I/O-bound bottlenecks with
gthread/gevent workers, and CPU-bound ones with more worker processes (up
to core count).

**References:**
- [Gunicorn — Design](https://gunicorn.org/design/)

---

### Q9. How would you diagnose why a specific FastAPI endpoint is slow in production? {#q9}

**Question:**
How would you diagnose why a specific FastAPI endpoint is slow in
production?

**Good answer:**
First isolate *where* the time goes rather than guessing: add request
timing (middleware or an APM/OpenTelemetry integration) that breaks the
request into phases — time spent in dependency resolution, time spent in
the handler body, time spent waiting on the database/downstream calls, and
serialization time for the response model. If most of the time is in a
downstream call, that's a database/network problem, not a FastAPI problem
(check DB query plans, connection pool exhaustion, N+1 queries). If the
time is inside the handler itself with no obvious I/O, profile it directly
with `py-spy` (can attach to a running production process without
restarting it) or `cProfile` locally to find hot functions. Also check
whether a supposedly-async endpoint is actually blocking the event loop
(see the next question) — that shows up as *all* concurrent requests
slowing down together, not just the one endpoint, which is a strong
diagnostic signal on its own.

**Follow-up question:**
You add timing and find the handler's own code is fast, but the endpoint's
*end-to-end* latency as seen by the client is still high. What do you check
next?

**Follow-up good answer:**
Look outside the handler: time spent in FastAPI's request validation
(large/complex Pydantic models can add measurable overhead), time spent
queued before a worker/thread even picks up the request (worker pool
saturation — check gunicorn/uvicorn worker counts and queue depth), and
network-level latency (TLS handshake, proxy/load-balancer hops, client-side
network conditions) that request-scoped timing inside the app never sees.
Distributed tracing across the whole request path (client → LB → app →
DB) is what actually separates these from in-process profiling.

**Glossary:**
- **`py-spy`** — a sampling profiler that can attach to a running Python
  process without instrumenting or restarting it.
- **APM** — Application Performance Monitoring; tooling (often
  OpenTelemetry-based) that traces a request across service boundaries.

**Mental model:**
Tests methodology, not tool trivia: does the candidate isolate before
guessing, and do they know to look both inside and outside the process for
latency.

**TL;DR:**
Diagnose FastAPI latency by timing each phase (dependencies, handler,
downstream I/O, serialization) before profiling, and treat "everything
slows down together" as a signal of a blocked event loop rather than a
single slow endpoint.

**References:**
- [FastAPI — Concurrency and async/await](https://fastapi.tiangolo.com/async/)

---

### Q10. A teammate wraps a route in `async def` expecting a speedup, but the app gets *slower* under load. What's the likely cause? {#q10}

**Question:**
A teammate wraps a route in `async def` expecting a speedup, but the app
gets *slower* under load. What's the likely cause?

**Good answer:**
They almost certainly put blocking, synchronous code (a `requests.get()`
call, a synchronous DB driver, `time.sleep()`, heavy CPU-bound computation)
directly inside the `async def` function. FastAPI runs `async def` path
operations directly on the single event loop thread — it does *not* put
them in a threadpool the way it does for plain `def` functions. A blocking
call inside `async def` therefore blocks that entire event loop, stalling
*every other concurrent request* the process is handling, not just this
one. The fix is either to use a genuinely async client library (`await`
based, e.g. `httpx.AsyncClient`, an async DB driver), or — if the work must
stay synchronous — to declare the route as plain `def` (FastAPI then runs
it in an external threadpool) or explicitly offload it via
`run_in_threadpool`/a process pool for CPU-bound work.

**Follow-up question:**
So is `def` always safer than `async def` for FastAPI routes?

**Follow-up good answer:**
Not "safer" in general — it's a different concurrency model with its own
limits (a threadpool has a bounded number of threads, so enough concurrent
blocking `def` routes will eventually queue too, and threads carry more
overhead than async tasks). The rule of thumb from FastAPI's own docs is:
use `async def` when everything inside it is genuinely awaitable
(async-native libraries), use plain `def` when the code is synchronous and
you can't avoid it, and when in doubt, `def` is the safer default because
FastAPI's threadpool isolation prevents one slow synchronous call from
freezing the whole process the way it would inside `async def`.

**Glossary:**
- **Event loop** — the single-threaded loop that drives all `async def`
  coroutines in the process; blocking it stalls everything scheduled on it.
- **External threadpool** — the thread pool FastAPI runs plain `def` path
  operations/dependencies in, so blocking code there doesn't stall the loop.

**Mental model:**
The single most common real-world FastAPI performance bug — tests whether
the candidate actually understands the `def` vs `async def` execution
model rather than treating "async" as a keyword that just makes things
faster.

**TL;DR:**
Blocking code inside `async def` stalls FastAPI's whole event loop for
every concurrent request; use real async libraries there, or declare the
route as plain `def` so FastAPI runs it in a threadpool instead.

**References:**
- [FastAPI — Concurrency and async/await](https://fastapi.tiangolo.com/async/)

---

### Q11. How should you decide gunicorn's worker count and worker class for a given workload? {#q11}

**Question:**
How should you decide gunicorn's worker count and worker class for a given
workload?

**Good answer:**
Start from the workload's bottleneck. For **CPU-bound** work, more worker
*processes* (roughly `(2 × cpu_cores) + 1` is gunicorn's commonly cited
starting heuristic) using the `sync` or `gthread` class helps, since
CPython's GIL means true parallel CPU work needs separate processes, not
more threads/greenlets in one process. For **I/O-bound** work (waiting on
databases, HTTP calls, etc.), a smaller number of workers using `gthread`
(thread pool per worker) or `gevent` (greenlet-based) can each juggle many
concurrent in-flight requests without needing a process per request, since
those requests spend most of their time waiting, not computing. For an
**ASGI** app (FastAPI/Starlette), you'd instead run gunicorn with a
Uvicorn worker class, letting each worker process run its own async event
loop and handle many concurrent requests cooperatively via `await`, which
suits I/O-bound async workloads especially well.

**Follow-up question:**
Your CPU-bound sync app is under-provisioned — would adding `gevent`
workers fix it?

**Follow-up good answer:**
No, for the same reason raised earlier: gevent's concurrency model relies
on cooperative yielding during I/O, and a CPU-bound task doesn't yield —
it keeps the single OS thread inside that greenlet busy until finished,
starving every other greenlet scheduled on that worker. The correct lever
for CPU-bound underprovisioning is more worker *processes* (up to roughly
the core count), not a different single-process concurrency model.

**Glossary:**
- **`(2 × cpu_cores) + 1`** — gunicorn's commonly cited starting-point
  heuristic for sync/gthread worker count.
- **Uvicorn worker** — a gunicorn worker class that runs an ASGI app's own
  event loop inside each managed process.

**Mental model:**
Reinforces the I/O-bound-vs-CPU-bound diagnostic lens from the earlier
questions, now applied as a *provisioning* decision rather than a
debugging one.

**TL;DR:**
Scale CPU-bound work with more gunicorn worker processes (sync/gthread, up
to core count); scale I/O-bound work with gthread/gevent/Uvicorn workers
that juggle many in-flight requests per process.

**References:**
- [Gunicorn — Design](https://gunicorn.org/design/)

---

### Q12. In what sense is WSGI/ASGI an example of good interface design, from a software-engineering theory standpoint? {#q12}

**Question:**
In what sense is WSGI/ASGI an example of good interface design, from a
software-engineering theory standpoint?

**Good answer:**
WSGI/ASGI are a textbook case of the dependency inversion principle applied
at the ecosystem level: instead of every server depending on every
framework's specific internals (or vice versa), both depend only on a
shared, minimal abstraction — the callable contract. This decouples
implementation from interface: you can swap gunicorn for uWSGI, or Flask
for Django, without either side needing to change, as long as both honor
the spec. It's the same principle behind JDBC in Java or a device driver
interface in an OS — a narrow, stable contract in the middle that lets
implementations on both sides evolve and be swapped independently.

**Follow-up question:**
What's the trade-off of standardizing on such a minimal interface?

**Follow-up good answer:**
Minimalism buys interoperability but pushes convenience elsewhere: WSGI/ASGI
themselves don't give you routing, request parsing, or session handling —
frameworks built on top (Flask, FastAPI) have to supply all of that, and
different frameworks can still be incompatible with *each other* at the
framework-feature level even though they're both WSGI/ASGI-compliant at the
transport level. The interface solves server/framework interoperability,
not framework/framework interoperability.

**Glossary:**
- **Dependency inversion** — depending on abstractions rather than
  concrete implementations, so either side can vary independently.

**Mental model:**
Checks whether the candidate can connect a concrete Python-ecosystem detail
to a general design principle they'd recognize in any language — a
"do they actually understand the theory" question, not a WSGI trivia one.

**TL;DR:**
WSGI/ASGI apply dependency inversion at the ecosystem level — a minimal
shared contract that decouples servers from frameworks — at the cost of
leaving all higher-level conveniences (routing, sessions) to the frameworks
built on top.

**References:**
- [PEP 3333 — Python Web Server Gateway Interface v1.0.1](https://peps.python.org/pep-3333/)

---

### Q13. What real-world problem forced the creation of ASGI, specifically — why wasn't extending WSGI enough? {#q13}

**Question:**
What real-world problem forced the creation of ASGI, specifically — why
wasn't extending WSGI enough?

**Good answer:**
WSGI's contract is "call once, return an iterable" — it has no concept of a
connection that receives multiple independent messages over time in either
direction, which is exactly what WebSockets and other long-lived,
bidirectional protocols need. You can't retrofit that onto WSGI without
breaking its core assumption, because the entire spec (and every server and
middleware built against it) assumes a single input stream and a single
output iterable per call, not an open channel you push/pull events on
indefinitely. The ASGI spec explicitly frames this as WSGI's design being
"irrevocably tied to the HTTP-style request/response cycle" — hence a new,
event-based spec rather than a WSGI extension.

**Follow-up question:**
Given that, could a WSGI app handle WebSockets at all, even with a
framework's help?

**Follow-up good answer:**
Not through the standard WSGI callable contract — any "WebSocket support"
bolted onto a WSGI stack works by escaping WSGI entirely for that
connection (e.g. hijacking the raw socket via server-specific extensions,
as some WSGI servers historically allowed), which is inherently
non-portable across servers since it's outside the spec. That's precisely
the interoperability problem ASGI fixes by making long-lived,
multi-message connections part of the standard interface itself.

**Glossary:**
- **Long-lived connection** — a connection (WebSocket, SSE) that stays open
  and exchanges multiple messages, rather than one request/response pair.

**Mental model:**
Tests understanding of *why* a new spec was necessary rather than just
"ASGI is newer/better" — the ability to trace a technical limitation to its
root cause.

**TL;DR:**
WSGI's one-call-in/one-iterable-out contract has no way to represent a
long-lived, multi-message connection, so WebSockets couldn't be added to it
without a new, event-based spec — which is what ASGI is.

**References:**
- [ASGI specification — main.html](https://asgi.readthedocs.io/en/latest/specs/main.html)

---

### Q14. What's wrong with calling a blocking, synchronous database driver from inside an `async def` FastAPI route, even if it "works" in testing? {#q14}

**Question:**
What's wrong with calling a blocking, synchronous database driver from
inside an `async def` FastAPI route, even if it "works" in testing?

**Good answer:**
It works under low/no concurrency because there's nothing else competing
for the event loop, so the bug is invisible until production load exposes
it. Under real concurrent traffic, that blocking DB call freezes the event
loop for its entire duration, and *every other in-flight request on that
worker* stalls too — not just the one hitting the database. This is a
classic pitfall precisely because it passes casual testing and only shows
up as mysterious, correlated latency spikes across unrelated endpoints in
production, which is much harder to diagnose after the fact than to avoid
up front by using an async-native driver or declaring the route as plain
`def`.

**Follow-up question:**
How would you catch this class of bug before it reaches production?

**Follow-up good answer:**
Load-test with realistic concurrency (not single-request smoke tests) so
the stall actually manifests as latency correlated across endpoints. You
can also proactively audit for it: grep for known-blocking calls
(`requests.`, `time.sleep`, synchronous DB driver imports) inside `async
def` functions, or use a lint/runtime tool that detects blocking calls on
the event loop (some APM/async debugging tools flag long
"loop iterations" or blocked-loop time directly).

**Glossary:**
- **Async-native driver** — a database/HTTP client library built to be
  `await`-ed, so it yields control instead of blocking the thread.

**Mental model:**
A deliberately "gotcha" pitfall question — tests whether the candidate has
actually been burned by this (or thought it through), since it's exactly
the kind of bug that survives naive testing.

**TL;DR:**
A blocking DB call inside `async def` stalls the whole event loop for every
concurrent request, not just the one making the call — and it often only
surfaces under real production concurrency, not in testing.

**References:**
- [FastAPI — Concurrency and async/await](https://fastapi.tiangolo.com/async/)

---

### Q15. What's the risk of running `flask run` or `uvicorn --reload` as your production server? {#q15}

**Question:**
What's the risk of running `flask run` or `uvicorn --reload` as your
production server?

**Good answer:**
Both are development servers, explicitly documented as not meant for
production: they typically run single-threaded/single-process (so no real
concurrency or fault isolation — one slow or crashing request can affect
everything), lack the process-management robustness of gunicorn/uvicorn's
production mode (automatic worker restart on crash, graceful reloads,
resource limits), and `--reload`/debug-mode file-watching adds overhead and
is sometimes bundled with a debugger that can leak sensitive information or
even allow remote code execution if inadvertently exposed. Production
deployments should run behind a process manager like gunicorn (optionally
with Uvicorn workers for ASGI apps) or uvicorn's own multi-worker mode,
typically behind a reverse proxy (nginx) that also handles TLS termination,
static files, and buffering slow clients away from the app workers.

**Follow-up question:**
If someone insists "it's fine, we only get low traffic," what's still a
concrete risk?

**Follow-up good answer:**
Low traffic doesn't remove the fault-isolation problem: a single
unhandled exception or a slow/hanging client connection on a
single-process dev server can take the *entire* app down or unresponsive
for all users, not just degrade gracefully. There's also no restart-on-crash
supervision, so an unhandled error can mean real, human-noticed downtime
until someone manually restarts the process — a risk that has nothing to
do with request volume.

**Glossary:**
- **Reverse proxy** — a front-line server (e.g. nginx) that terminates
  TLS, buffers slow clients, and forwards requests to app workers.

**Mental model:**
A practical, "have you actually deployed this" pitfall question rather
than a purely theoretical one.

**TL;DR:**
Dev servers (`flask run`, `uvicorn --reload`) lack production process
management and fault isolation — one crash or slow client can take the
whole thing down regardless of traffic volume, so always run behind
gunicorn/uvicorn's production mode and a reverse proxy.

**References:**
- [Flask — Deploying to Production](https://flask.palletsprojects.com/en/latest/deploying/)

---

### Q16. How does FastAPI generate interactive API documentation (Swagger UI) automatically? {#q16}

**Question:**
How does FastAPI generate interactive API documentation (Swagger UI)
automatically?

**Good answer:**
FastAPI builds an OpenAPI schema by introspecting your path operations at
startup: it reads each route's declared path/query/body parameters (their
Python type hints and any `Field`/`Query`/`Path` metadata), the Pydantic
models used for request bodies and response models, and any `Depends()`
sub-dependencies' own parameters — all of that gets folded into one
OpenAPI JSON document. It then serves interactive docs (Swagger UI at
`/docs`, ReDoc at `/redoc`) that render straight from that schema. Because
the schema comes directly from the same type hints and Pydantic models
that also drive runtime request validation, the documentation can't drift
out of sync with the actual validation behavior the way hand-written API
docs commonly do.

**Follow-up question:**
What's the practical benefit of the docs being generated from the same
source as the validation, rather than written separately?

**Follow-up good answer:**
It eliminates an entire class of "the docs lied" bugs: if a field is
required, has a specific type, or has a length constraint, that's enforced
at request time and shown in the docs from the exact same declaration —
there's no second place for it to become stale. Changing a Pydantic model
or adding a dependency's parameter updates both behavior and documentation
simultaneously, by construction, rather than by developer discipline to
remember to update docs separately.

**Glossary:**
- **OpenAPI schema** — the machine-readable API description (JSON/YAML)
  FastAPI generates, that Swagger UI/ReDoc render into interactive docs.

**Mental model:**
Tests whether the candidate understands *why* this feature is valuable
(single source of truth) rather than just knowing it exists as a checkbox
feature.

**TL;DR:**
FastAPI derives its OpenAPI schema (and therefore Swagger UI/ReDoc) directly
from the same type hints, Pydantic models, and dependencies that drive
runtime validation, so the docs can't drift out of sync with actual
behavior.

**References:**
- [FastAPI — First Steps (interactive docs)](https://fastapi.tiangolo.com/tutorial/first-steps/)
- [FastAPI — Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)

---

### Q17. You add Middleware A then Middleware B to a Starlette/FastAPI app. Which one sees the incoming request first? {#q17}

**Question:**
You add Middleware A then Middleware B to a Starlette/FastAPI app. Which
one sees the incoming request first?

**Good answer:**
Middleware B — the one added *last* — sees the request first on the way
in, because middleware wraps in an "onion" model: each middleware you add
wraps around the previous stack, so the most recently added middleware is
the outermost layer for the *response* but becomes the layer the request
passes through *last* before reaching the actual application on the way
in... concretely: requests flow through middleware in the reverse order
they were added (last-added runs first), and responses flow back out
through them in the order they were originally added (first-added's
post-processing runs last, since it's the outermost wrapper). If you think
of it as nested function calls, each middleware wraps the next, so
execution nests from the last-added inward for the request, then unwinds
back outward for the response.

**Code example:**
```python
app.add_middleware(MiddlewareA)
app.add_middleware(MiddlewareB)
# Request flow:  ... -> B -> A -> endpoint
# Response flow: endpoint -> A -> B -> ...
```

**Follow-up question:**
Why does this ordering matter in practice — give a concrete example where
getting it backwards causes a bug.

**Follow-up good answer:**
A common example is authentication vs. logging: if a logging middleware is
added *after* (i.e., needs to run outside) an authentication middleware so
it can log the authenticated user's identity, but it's actually registered
in the wrong order, the logging middleware may run before authentication
has populated request state, logging requests as anonymous even when
they're authenticated. Similarly, a CORS or exception-handling middleware
generally needs to be one of the *outermost* layers (added last, or via a
mechanism that guarantees outermost placement) so it can catch/handle
things happening in every inner layer, including other middleware —
getting the order backwards means it misses errors raised inside layers
that were "outside" it.

**Glossary:**
- **Onion model** — the nested-wrapping mental model for middleware, where
  each layer wraps the next.

**Mental model:**
A concrete "did you actually configure middleware and hit an ordering bug"
question rather than an abstract one.

**TL;DR:**
The last middleware added is the outermost on the way out but the
innermost-before-the-app on the way in — i.e., last-added runs first on
requests and last on responses, so ordering mistakes commonly break
auth/logging/error-handling middleware that assumes a particular layer.

**References:**
- [Starlette — Middleware](https://www.starlette.io/middleware/)

---

### Q18. What are FastAPI `BackgroundTasks` for, and when should you *not* use them? {#q18}

**Question:**
What are FastAPI `BackgroundTasks` for, and when should you *not* use
them?

**Good answer:**
`BackgroundTasks` let you schedule a function to run *after* the response
has already been sent to the client, without making them wait for it —
useful for small, quick side-effects like writing a log entry or sending a
notification email that shouldn't add to the request's latency. Critically,
the task still runs *in the same process*, on the same event loop/worker —
it is not a separate queue or separate process. That means it's the wrong
tool for heavy CPU-bound work (it'll compete with and block other requests
on that worker, defeating the whole point) or for work that must survive a
process restart or run on a different machine — for that, FastAPI's own
docs point to a real task queue like Celery with a broker (Redis/RabbitMQ).

**Follow-up question:**
If a background task raises an exception, does the client ever find out?

**Follow-up good answer:**
No — by the time the background task runs, the response has already been
sent and the client connection may be long closed, so an exception there
can't be surfaced back to that request's caller at all. It needs its own
error handling/logging inside the task function; an unhandled exception in
a background task is a silent failure from the client's point of view
unless you've built separate observability (logging, alerting) around it.

**Glossary:**
- **`BackgroundTasks`** — FastAPI's mechanism for scheduling in-process
  work to run after the response is sent.

**Mental model:**
Tests whether the candidate knows the boundary between "convenience
feature for small side-effects" and "you actually need a task queue" —
a common over-reach when people first discover `BackgroundTasks`.

**TL;DR:**
`BackgroundTasks` run after the response but in the same process — fine
for small side-effects like logging/emails, wrong for heavy CPU work or
anything that needs its own retry/durability, which calls for Celery-style
queues instead.

**References:**
- [FastAPI — Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)

---

### Q19. How would you decide between Flask, FastAPI, and Django REST Framework for a new API? {#q19}

**Question:**
How would you decide between Flask, FastAPI, and Django REST Framework for
a new API?

**Good answer:**
Flask is a minimal WSGI microframework — you get routing and little else
by default, so it's a good fit for small services or when you want full
control over which pieces (ORM, validation, auth) to bring in, at the cost
of assembling more of that yourself. FastAPI is ASGI-native, built around
Python type hints for automatic request validation and OpenAPI docs
generation, and is the natural choice when you want async I/O support,
strong typing, and free interactive docs without extra libraries — it's
newer and has a smaller (though very active) plugin ecosystem than Django's.
Django REST Framework sits on top of Django, so it's the right call when
you already want (or have) Django's batteries — its ORM, admin site,
auth system, migrations — and are building a REST API as part of a larger
Django application; it's WSGI-based by default (Django added async views
later, but the ecosystem is still primarily sync-oriented), so it's less
naturally suited to a heavily async, high-concurrency I/O workload than
FastAPI.

**Follow-up question:**
Your team is building a small internal CRUD API with no unusual
performance requirements. Does the "best" technical choice above still
apply?

**Follow-up good answer:**
Not necessarily — for a low-stakes internal CRUD API, team familiarity and
existing conventions usually outweigh marginal technical advantages: if the
team already knows Django well, DRF's batteries (auth, admin, ORM) will
ship faster than adopting FastAPI's patterns from scratch, even though
FastAPI might be the "better" async-first framework in the abstract. The
right choice depends on the actual constraints (team skill, existing
codebase, real performance needs) more than a one-size-fits-all ranking.

**Glossary:**
- **Django REST Framework (DRF)** — the standard toolkit for building REST
  APIs on top of Django.

**Mental model:**
A classic trade-off question — checks whether the candidate can reason
about fit-for-context rather than reciting "FastAPI is the best/newest."

**TL;DR:**
Flask is a minimal WSGI microframework for full control, FastAPI is
ASGI-native with type-hint-driven validation/docs and strong async support,
and DRF is the right call when you want Django's full batteries — the
"best" choice depends on team/ecosystem fit, not just technical merit.

**References:**
- [FastAPI — Features](https://fastapi.tiangolo.com/features/)

---

### Q20. Design the deployment topology (server + workers) for a FastAPI service that mixes fast async I/O endpoints with a few CPU-heavy synchronous endpoints. {#q20}

**Question:**
Design the deployment topology (server + workers) for a FastAPI service
that mixes fast async I/O endpoints with a few CPU-heavy synchronous
endpoints.

**Good answer:**
Run the ASGI app under gunicorn with Uvicorn worker processes (gunicorn for
process management/restarts, Uvicorn workers for the actual async event
loop per process) — this handles the fast async I/O endpoints well, since
each worker's event loop can juggle many concurrent `await`s. For the
CPU-heavy endpoints, make sure they're declared as plain `def` (so FastAPI
runs them in that worker's external threadpool) or, better, offload the
actual CPU-bound computation to a separate process pool
(`concurrent.futures.ProcessPoolExecutor`, or a proper task queue like
Celery if the work is substantial/needs retries), so it doesn't compete
with the async event loop or tie up threadpool capacity needed by other
`def` routes. Put the whole thing behind a reverse proxy (nginx) for TLS
termination and to buffer slow clients away from the app workers, and size
the Uvicorn worker *count* based on CPU cores available (since each is a
separate process, same GIL-bypassing logic as gunicorn's own
process-count guidance), independent of how many concurrent I/O requests
each single worker's event loop can additionally handle.

**Follow-up question:**
Why not just put the CPU-heavy work in a regular `async def` route with
`run_in_threadpool`?

**Follow-up good answer:**
`run_in_threadpool` moves work off the event loop, which is correct for
*blocking I/O*, but for genuinely CPU-bound work it doesn't help nearly as
much: threads in CPython still contend for the GIL, so heavy CPU work in a
thread still largely serializes with other CPU work in that same process
and can still degrade the async event loop's responsiveness under
contention. A separate process pool actually achieves parallel CPU
execution across cores, which a threadpool within the same GIL-bound
process cannot.

**Glossary:**
- **`ProcessPoolExecutor`** — offloads CPU-bound work to separate OS
  processes, bypassing the GIL for true parallelism.

**Mental model:**
A synthesis question pulling together WSGI/ASGI internals, the `def`
vs `async def` execution model, and worker provisioning into one design
decision — checks whether the candidate can combine everything from this
set rather than only answering each piece in isolation.

**TL;DR:**
Serve async I/O endpoints via gunicorn+Uvicorn workers sized to CPU cores,
and offload genuinely CPU-heavy work to a separate process pool or task
queue rather than the event loop or even a threadpool, since threads still
contend for the GIL.

**References:**
- [Gunicorn — Design](https://gunicorn.org/design/)
- [FastAPI — Concurrency and async/await](https://fastapi.tiangolo.com/async/)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=web-apis-wsgi-and-asgi&autostart=1" | relative_url }})
