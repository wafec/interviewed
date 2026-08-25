---
layout: default
title: "Python Interview — Concurrency, GIL & Async"
---

# Python Interview — Concurrency, GIL & Async

This set covers Python's three concurrency models (threading, multiprocessing,
asyncio), what the Global Interpreter Lock actually is and how it behaves
under the hood, how the asyncio event loop schedules coroutines, the
performance-diagnosis methodology for telling I/O-bound from CPU-bound code,
common pitfalls in both threaded and async code, and where CPython is headed
with free-threaded (no-GIL) builds.

### Q1. When would you reach for threading, multiprocessing, or asyncio in Python, and why can't you just always use one? {#q1}

**Question:**
When would you reach for threading, multiprocessing, or asyncio in Python, and why can't you just always use one?

**Good answer:**
Pick based on what's actually the bottleneck. `threading` gives you concurrency for **I/O-bound** work (network calls, file I/O, waiting on a subprocess) because the GIL is released during blocking I/O, so other threads can run while one waits — but it gives you no real parallelism for CPU-bound work, since only one thread executes Python bytecode at a time. `multiprocessing` gives you true **parallelism for CPU-bound** work by running separate OS processes, each with its own interpreter and GIL, at the cost of higher memory usage, slower startup, and the requirement that anything crossing the process boundary (arguments, return values) be picklable. `asyncio` gives you very high-concurrency **I/O-bound** work with far less overhead than threads (thousands of coroutines vs. a practical limit of maybe a few hundred OS threads), but only if your entire I/O stack is async-aware — a single blocking call in a coroutine stalls the whole event loop. You can't use one for everything because they solve different bottlenecks: threads/asyncio don't help CPU-bound work, and multiprocessing is overkill (and slower, due to IPC/serialization) for I/O-bound work that's mostly waiting.

**Follow-up question:**
Your service does both — it awaits a lot of network calls and occasionally has to run a CPU-heavy parsing step. How do you combine these models without one blocking the other?

**Follow-up good answer:**
Keep the async event loop for the I/O-bound majority of the work, and offload the CPU-heavy step so it doesn't block the loop. The standard pattern is `loop.run_in_executor()`: hand the CPU-bound function to a `concurrent.futures.ProcessPoolExecutor` (bypasses the GIL, true parallelism) if it's heavy enough to justify process overhead, or a `ThreadPoolExecutor` if it's more moderate or the work itself releases the GIL (e.g. calling into a C extension like NumPy). Either way, `run_in_executor` returns an awaitable, so the coroutine calling it doesn't block the loop — the executor's worker thread/process does the blocking work, and the loop keeps servicing other coroutines in the meantime.

**Code example:**
```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy_parse(data: bytes) -> dict:
    ...  # expensive parsing

async def handle_request(data: bytes):
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy_parse, data)
    return result
```

**Glossary:**
- **GIL (Global Interpreter Lock)** — CPython's lock ensuring only one thread executes Python bytecode at a time.
- **Event loop** — asyncio's single-threaded scheduler that runs coroutines cooperatively.
- **Executor** — a thread or process pool asyncio can offload blocking work to via `run_in_executor`.

**Mental model:**
Tests whether the candidate reasons about concurrency in terms of the actual bottleneck (I/O wait vs. CPU work) rather than treating "concurrency" as one undifferentiated tool, and whether they know these models compose rather than being mutually exclusive.

**TL;DR:**
Use threading/asyncio for I/O-bound waiting (GIL is released), multiprocessing for CPU-bound parallelism (separate GILs per process) — and combine them via `run_in_executor` when you need both.

**References:**
- [`concurrent.futures.Executor.run_in_executor` / asyncio-eventloop](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor)
- [asyncio-dev: Running Blocking Code](https://docs.python.org/3/library/asyncio-dev.html#running-blocking-code)

---

### Q2. What exactly is the GIL, and why does CPython have one? {#q2}

**Question:**
What exactly is the GIL, and why does CPython have one?

**Good answer:**
The Global Interpreter Lock is a mutex in the CPython interpreter that ensures only one thread executes Python bytecode at any given instant, even on a multi-core machine. It exists because CPython uses non-atomic reference counting for memory management: every time an object's reference count changes, that increment/decrement has to be safe against concurrent access, or you get corrupted counts, use-after-free bugs, or double-frees. Rather than putting a fine-grained lock on every object (which would be slow and extremely easy to get wrong, and could deadlock), CPython uses one global lock around bytecode execution. This "simplifies the CPython implementation by making the object model...implicitly safe against concurrent access," per the official glossary, at the cost of preventing true multi-core parallelism for pure-Python CPU-bound code.

**Follow-up question:**
If the GIL is so limiting, why hasn't CPython just removed it long before PEP 703?

**Follow-up good answer:**
Because removing it isn't just "delete the lock" — every C extension in the ecosystem (NumPy, and thousands of others) has relied for decades on the GIL implicitly protecting their internal state and Python's object model, so an unsynchronized removal would silently reintroduce races everywhere. Early attempts (e.g. the "Gilectomy" project) found that fine-grained locking made single-threaded performance meaningfully worse, which was unacceptable since most Python code is single-threaded. PEP 703 (accepted for CPython 3.13+ as an opt-in free-threaded build) only became viable once it paired per-object/biased reference counting, a different memory allocator (mimalloc), and per-object locks with "critical sections" — a coordinated set of changes to keep single-threaded overhead low (roughly 5–8%) while still being C-extension compatible via updated APIs.

**Glossary:**
- **Reference counting** — CPython's primary memory-management mechanism; each object tracks how many references point to it.
- **PEP 703** — the accepted proposal for an optional, free-threaded (no-GIL) CPython build.
- **Biased reference counting** — a technique giving each thread a private refcount for objects it "owns," reducing atomic-operation overhead.

**Mental model:**
Checks whether the candidate understands the GIL as a specific engineering trade-off tied to CPython's memory model, not a mysterious or arbitrary limitation — and whether they grasp why "just remove it" undersells the actual difficulty.

**TL;DR:**
The GIL exists because CPython's reference counting isn't otherwise thread-safe; removing it (PEP 703) needed a coordinated redesign of refcounting, allocation, and locking to avoid regressing single-threaded performance.

**References:**
- [Python glossary: Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)
- [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)

---

### Q3. What controls how often CPython switches between threads, and does that mean threads are preempted mid-bytecode? {#q3}

**Question:**
What controls how often CPython switches between threads, and does that mean threads are preempted mid-bytecode?

**Good answer:**
`sys.setswitchinterval()` sets the interpreter's thread switch interval in seconds (default 0.005s / 5ms) — the target duration of the "timeslice" a thread holding the GIL gets before being asked to release it so another waiting thread can run. It's a target, not a hard guarantee: the docs note the actual interval can be longer if a long-running internal function (e.g. a big regex match or a C extension call) doesn't check for the request, and which thread actually gets scheduled next after the GIL is released is up to the OS scheduler, not CPython. A single bytecode instruction itself is not preempted mid-execution — the interpreter checks for a pending switch/signal between bytecode instructions (specifically CPython uses "eval breaker" checks), so switches happen at instruction boundaries, not inside one.

**Follow-up question:**
Given that, can two threads still race on something like `counter += 1` even though "only one thread runs Python bytecode at a time"?

**Follow-up good answer:**
Yes — `counter += 1` compiles to multiple bytecode instructions (load, add, store), and the GIL can be released between any of them. So thread A can load the old value, get switched out, thread B loads the same old value, increments and stores, then A resumes, increments its stale copy, and stores it — overwriting B's update. This is the classic lost-update race, and it happens *despite* the GIL because "only one thread executes bytecode at a time" is a statement about a single instruction, not about a whole logical operation like `+=`. Any compound read-modify-write on shared mutable state still needs an explicit `threading.Lock` (or an atomic primitive) to be safe.

**Code example:**
```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:          # without this, += is not atomic across bytecode ops
            counter += 1
```

**Glossary:**
- **Switch interval** — the target duration a thread holds the GIL before being asked to yield it, set via `sys.setswitchinterval()`.
- **Eval breaker** — CPython's mechanism for checking, between bytecode instructions, whether the current thread should yield the GIL, handle a signal, etc.
- **Lost update** — a race where one thread's write silently overwrites another's due to an interleaved read-modify-write.

**Mental model:**
Probes whether the candidate has the precise (not folk) understanding of what the GIL guarantees — atomicity of single bytecode instructions, not of Python-level statements or operators.

**TL;DR:**
The GIL switches at a configurable interval (default 5ms) between bytecode instructions, not inside one — so multi-instruction operations like `+=` are still race-prone and need explicit locking.

**References:**
- [`sys.setswitchinterval`](https://docs.python.org/3/library/sys.html#sys.setswitchinterval)
- [Python glossary: Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)

---

### Q4. Why does I/O-bound work benefit from threads even with the GIL, but CPU-bound Python code doesn't? {#q4}

**Question:**
Why does I/O-bound work benefit from threads even with the GIL, but CPU-bound Python code doesn't?

**Good answer:**
The GIL is explicitly released around blocking I/O calls (socket reads, file reads, `time.sleep`, etc.) and around long-running calls into C extensions that opt to release it (e.g. heavy NumPy operations, `zlib` compression). While a thread is blocked waiting on the OS for I/O, it's not holding the GIL at all, so another thread is free to acquire it and run Python bytecode — that's real, useful concurrency, because the "work" is happening outside the interpreter (in the kernel/network stack) anyway. Pure-Python CPU-bound code, by contrast, is continuously executing bytecode, so it holds the GIL the entire time except for the brief windows at switch-interval boundaries — meaning multiple CPU-bound threads mostly take turns on a single core rather than running in parallel, and you get no speedup (often a slight slowdown from switching overhead) versus doing the work sequentially.

**Follow-up question:**
How would you empirically prove to a skeptical teammate that adding threads to their CPU-bound function isn't helping?

**Follow-up good answer:**
Benchmark it directly: run the CPU-bound function once sequentially and time it, then run N of them concurrently via `threading.Thread`/`ThreadPoolExecutor` and time the wall-clock for all N to finish. For genuinely CPU-bound pure-Python work, the threaded wall-clock time will be roughly the same as (or worse than) N times the single-run time, whereas the same experiment with `ProcessPoolExecutor` should show close-to-linear speedup up to the core count. You can also watch CPU utilization during the threaded run (e.g. via `top`/Activity Monitor) — it'll hover around one core's worth (~100% on an N-core machine) instead of scaling toward `N * 100%`, which is the visible signature of GIL contention rather than a genuinely slow algorithm.

**Glossary:**
- **CPU-bound** — work whose runtime is dominated by computation (interpreter bytecode execution), not waiting on external resources.
- **I/O-bound** — work whose runtime is dominated by waiting on external resources (network, disk, other processes).

**Mental model:**
Tests whether the candidate can connect the abstract GIL rule to concrete, measurable behavior, and whether they'd reach for a benchmark instead of asserting an answer from memory.

**TL;DR:**
Threads release the GIL during blocking I/O so other threads can run, but a CPU-bound thread holds the GIL almost continuously — so threading helps I/O concurrency but not CPU parallelism, which you can prove by comparing wall-clock time and CPU utilization for threaded vs. process-based CPU-bound work.

**References:**
- [Python glossary: Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)
- [`concurrent.futures.ProcessPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor)

---

### Q5. How does `await` actually suspend and resume a coroutine inside the asyncio event loop? {#q5}

**Question:**
How does `await` actually suspend and resume a coroutine inside the asyncio event loop?

**Good answer:**
A coroutine function is built on generators under the hood — `await`ing something suspends execution at that point and yields control back to the event loop, the same way `yield` suspends a generator. Concretely: when you `await` a `Future` (or a lower-level awaitable), the coroutine registers a callback on that future and returns control to the loop; the loop then goes on to run other ready callbacks/coroutines. When the awaited operation completes (e.g. a socket becomes readable, a timer fires), the loop schedules the callback, which resumes the coroutine exactly where it left off — the interpreter restores that coroutine's frame/locals just like resuming a generator. This is all single-threaded cooperative scheduling: nothing preempts a running coroutine; it only yields control at explicit `await` points, which is why one coroutine can starve the loop if it never awaits.

**Follow-up question:**
What's the difference, mechanically, between `asyncio.sleep(0)` and just not awaiting anything for a while inside a coroutine?

**Follow-up good answer:**
`await asyncio.sleep(0)` explicitly yields control back to the event loop for one iteration, giving other pending callbacks/coroutines a chance to run, then reschedules itself to resume on the next loop iteration — it's a deliberate cooperative yield point, commonly used to break up a long-running-but-technically-async loop so it doesn't starve other tasks. Not awaiting anything (running a tight synchronous loop inside a coroutine) never yields to the loop at all — the coroutine holds the single thread of execution until it returns or hits an actual `await`, blocking every other task and any I/O callbacks from running in the meantime, exactly like a blocking call would.

**Glossary:**
- **Awaitable** — an object (coroutine, `Task`, or `Future`) that can appear after `await`.
- **Cooperative scheduling** — a model where tasks voluntarily yield control (via `await`) rather than being preempted by a scheduler.
- **Future** — a low-level awaitable representing an eventual result the event loop resolves.

**Mental model:**
Checks whether the candidate has an accurate mental model of asyncio as cooperative and generator-based, rather than treating `async`/`await` as "magic parallelism" — which is the root cause of most asyncio bugs.

**TL;DR:**
`await` suspends a coroutine like a generator yield, handing control back to the single-threaded event loop until the awaited operation completes and the loop resumes the coroutine's frame — nothing preempts it in between.

**References:**
- [PEP 3156 – Asynchronous IO Support Rebooted](https://peps.python.org/pep-3156/)
- [`asyncio.sleep`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)

---

### Q6. How does `multiprocessing` sidestep the GIL, and what does that cost you compared to threading? {#q6}

**Question:**
How does `multiprocessing` sidestep the GIL, and what does that cost you compared to threading?

**Good answer:**
`multiprocessing` spawns separate OS processes, each running its own instance of the Python interpreter with its own memory space and, crucially, its own GIL — so N processes really can execute Python bytecode in parallel across N cores, unlike N threads inside one process sharing a single GIL. The cost is that processes don't share memory: anything crossing the process boundary — arguments to the worker, return values, exceptions — has to be serialized (pickled), sent through an OS pipe/socket, and deserialized on the other end. That means every value passed has to be picklable (plain functions/classes defined at module level, not lambdas or closures, in general), IPC has real latency and throughput cost compared to passing a reference within one process, and process startup itself is far heavier than thread startup (a full interpreter boot, larger memory footprint per worker).

**Follow-up question:**
A teammate passes a `lambda` as the target function to a `multiprocessing.Pool.map` call and it fails. Why, and what's the fix?

**Follow-up good answer:**
With the default `spawn` start method (used on macOS/Windows always, and on Linux by default since Python 3.14), the child process doesn't inherit the parent's memory — it starts a fresh interpreter and has to *unpickle* the target function to know what to run, which means the function must be importable by reference from a module. A `lambda` (and closures, and functions defined inside another function or interactively in a REPL) have no stable, importable name to pickle by reference, so pickling fails, typically with a `PicklingError`/`AttributeError`. The fix is to define the target as a plain function at module level (or a `functools.partial` of one) so it can be imported by name in the child process; alternatively, on POSIX, the `fork` start method avoids the pickling requirement for the target itself (it inherits parent memory via fork) but has its own hazards around inherited locks/threads and is no longer the default anywhere as of 3.14.

**Code example:**
```python
# BAD: lambda can't be pickled for spawn-based multiprocessing
# pool.map(lambda x: x * x, data)

# GOOD: module-level function
def square(x):
    return x * x

if __name__ == "__main__":
    from multiprocessing import Pool
    with Pool() as pool:
        results = pool.map(square, data)
```

**Glossary:**
- **Spawn** — a process start method that boots a fresh interpreter in the child and requires pickling the target/arguments.
- **Fork** — a POSIX process start method that clones the parent's memory into the child (no pickling of the target needed, but inherited state can be hazardous).
- **IPC (inter-process communication)** — the mechanism (pipes, sockets, shared memory) processes use to exchange data.

**Mental model:**
Tests understanding of the actual cost side of the threading-vs-multiprocessing trade-off — candidates often know "multiprocessing avoids the GIL" without grasping why that isn't free.

**TL;DR:**
Multiprocessing sidesteps the GIL because each process has its own interpreter/GIL, but pays for it with pickling/IPC overhead and heavier process startup — and objects like lambdas/closures can't cross that boundary because they aren't picklable by reference.

**References:**
- [multiprocessing: Programming guidelines — picklability](https://docs.python.org/3/library/multiprocessing.html#multiprocessing-programming)
- [`concurrent.futures.ProcessPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor)

---

### Q7. How do you determine whether a slow Python service is I/O-bound or CPU-bound before deciding how to fix it? {#q7}

**Question:**
How do you determine whether a slow Python service is I/O-bound or CPU-bound before deciding how to fix it?

**Good answer:**
Start with system-level signals before reaching for a Python profiler: check CPU utilization for the process during the slow period. Near-100%-of-one-core (or pegged across cores if already multi-process) sustained utilization strongly suggests CPU-bound work; low CPU utilization with high latency suggests the process is mostly waiting (I/O-bound) — network calls, DB queries, disk, or lock contention. From there, use `cProfile` (deterministic, function-level `tottime`/`cumtime` breakdown) to see where wall-clock time is actually going — if the top entries by cumulative time are your own computation functions, it's CPU-bound; if they're `socket.recv`, DB driver calls, or `time.sleep`-like blocking waits, it's I/O-bound. A sampling profiler like `py-spy` (attaches to a running process without code changes, low overhead, works in production) is useful to confirm this on a live process without restarting it with profiling enabled.

**Follow-up question:**
`cProfile` shows most time in a function that just calls a downstream HTTP API. Is that actually a *bug* in your code?

**Follow-up good answer:**
Not necessarily — `cProfile`'s `cumtime` for that function will include time spent waiting on the network call, which is expected and often not fixable by optimizing your own code at all. The real question is whether that wait time is *necessary and overlappable* or not: if you're making that HTTP call sequentially before other independent calls that could run concurrently, that's the actual fix (parallelize with `asyncio`/threads, not "optimize the function"). If the call itself is just slow (large payload, slow downstream service, no caching), the fix is on the I/O side — caching, batching requests, reducing payload size, or improving the downstream service — not CPU profiling of your own function, since there's no CPU-bound bottleneck to profile away.

**Glossary:**
- **`tottime`** — time spent in a function itself, excluding time in functions it calls.
- **`cumtime`** — cumulative time in a function including all functions it calls (so it includes I/O wait time).
- **py-spy** — a sampling profiler that can attach to a running Python process without modifying it.

**Mental model:**
This is the "how do you diagnose performance" question in disguise — it tests methodology (system signals → profiler → correct interpretation) rather than rote tool names, and whether the candidate knows profiler output needs interpretation, not just reading.

**TL;DR:**
Check CPU utilization first (pegged = CPU-bound, idle-but-slow = I/O-bound), then use `cProfile`'s tottime/cumtime (or a live sampling profiler like py-spy) to confirm where time actually goes — and remember cumtime on an I/O call includes wait time, which usually isn't a code-optimization problem.

**References:**
- [The Python Profilers — cProfile](https://docs.python.org/3/library/profile.html)
- [py-spy (GitHub)](https://github.com/benfred/py-spy)

---

### Q8. You add more `threading.Thread` workers to a CPU-bound image-processing function and throughput doesn't improve. Walk through how you'd confirm the GIL is the cause. {#q8}

**Question:**
You add more `threading.Thread` workers to a CPU-bound image-processing function and throughput doesn't improve. Walk through how you'd confirm the GIL is the cause.

**Good answer:**
First confirm it's actually CPU-bound in pure Python and not a library that releases the GIL internally (e.g. some NumPy/Pillow operations release the GIL during their C-level work, in which case threading *would* help). Then run a controlled comparison: time N sequential calls, N threaded calls, and N `ProcessPoolExecutor` calls, all doing identical work. If threaded throughput matches sequential (or is worse due to context-switch overhead) while process-based throughput scales toward linear with core count, that's the signature of GIL contention specifically — because processes have separate GILs and threads don't. You can further confirm by watching CPU utilization: the threaded run staying near one core's worth of usage on a multi-core box, versus the process run spreading across cores, is direct evidence the GIL — not the algorithm — is the bottleneck.

**Follow-up question:**
Given that confirmation, what are your options to actually fix it, in order of how much you'd disrupt the existing codebase?

**Follow-up good answer:**
Least disruptive: check if the CPU-heavy step can be pushed into a library that already releases the GIL for that operation (many NumPy/SciPy/Pillow/compiled-extension calls do) — if so, threading starts working with no architecture change. Next: swap the thread pool for a `ProcessPoolExecutor`/`multiprocessing.Pool` doing the same work — moderate change, mainly needs picklable inputs/outputs and accepting IPC overhead. Next: move the hot path into a C extension or Cython/`ctypes`/`cffi` call that releases the GIL explicitly during its computation. Most disruptive (and currently experimental for production use): run on the free-threaded (`--disable-gil`) CPython 3.13+ build, which removes the GIL constraint for threads entirely but requires the whole dependency chain (including C extensions) to be free-threading-compatible.

**Glossary:**
- **GIL contention** — the symptom of multiple threads competing for the same lock instead of running in parallel.
- **Free-threaded build** — the PEP 703 CPython build variant where the GIL can be disabled.

**Mental model:**
Wants a methodical diagnose-then-fix narrative, not just "use multiprocessing" — checks whether the candidate actually verifies the hypothesis before prescribing a fix, and knows fixes exist at multiple levels of disruption.

**TL;DR:**
Confirm GIL contention by comparing threaded vs. process-based throughput and CPU utilization for identical work; fix it (in increasing order of disruption) via a GIL-releasing library call, multiprocessing, a C extension, or ultimately a free-threaded CPython build.

**References:**
- [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)
- [`concurrent.futures.ProcessPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor)

---

### Q9. Besides `cProfile`, what tools would you reach for to diagnose a slow or "stuck" asyncio application specifically? {#q9}

**Question:**
Besides `cProfile`, what tools would you reach for to diagnose a slow or "stuck" asyncio application specifically?

**Good answer:**
Enable **asyncio debug mode** (`asyncio.run(main(), debug=True)` or `PYTHONASYNCIODEBUG=1`) — it makes the loop log a warning when a callback/step takes "too long" (blocking the loop), detect coroutines that were never awaited, and enables extra sanity checks like flagging non-threadsafe calls from the wrong thread. For a live/production process, `py-spy dump` or `py-spy top` can show you exactly which coroutine/frame every thread is sitting in without restarting the process, which is often faster than trying to reproduce a hang locally. For finding coroutines that leaked without being awaited, `RuntimeWarning: coroutine '...' was never awaited` messages (visible with `-W default` or debug mode) point directly at the bug. For understanding task scheduling and pending tasks, `asyncio.all_tasks()` can be inspected at runtime (e.g. via a debug endpoint) to see what's queued and for how long.

**Follow-up question:**
Debug mode logs "Executing <Task ...> took 0.150 seconds." What does that actually tell you, and what's a common cause?

**Follow-up good answer:**
That message means a single step of a task/callback ran for 150ms without yielding control back to the event loop — well above the default slow-callback threshold (100ms) — which blocks every other coroutine and any I/O callbacks from running during that window, i.e. it's evidence of exactly the "blocking call inside a coroutine" problem. The most common cause is calling a synchronous, blocking function directly inside an `async def` — a synchronous `requests.get()` instead of an async HTTP client, blocking file I/O, `time.sleep()` instead of `asyncio.sleep()`, or a genuinely CPU-heavy computation with no `await` points to let the loop breathe. The fix is either switching to an async-native equivalent of that call, or offloading it to `run_in_executor` so it runs off the event-loop thread.

**Glossary:**
- **Debug mode** — an asyncio runtime mode that adds extra checks and logging for slow callbacks, unawaited coroutines, and non-threadsafe API misuse.
- **Slow callback** — a task/callback step that runs long enough to noticeably delay the event loop (100ms by default in debug mode).

**Mental model:**
Checks familiarity with asyncio-specific tooling beyond generic profilers, since asyncio bugs (stalls, leaked coroutines) often don't show up as classic "hot function" profiles.

**TL;DR:**
Use asyncio debug mode to catch slow callbacks and never-awaited coroutines, and `py-spy` to inspect a live/stuck process without restarting it — both point directly at blocking calls hiding inside coroutines.

**References:**
- [asyncio-dev: Debug Mode](https://docs.python.org/3/library/asyncio-dev.html)
- [asyncio-dev: Running Blocking Code](https://docs.python.org/3/library/asyncio-dev.html#running-blocking-code)

---

### Q10. What's the difference between cooperative and preemptive multitasking, and which one is asyncio? {#q10}

**Question:**
What's the difference between cooperative and preemptive multitasking, and which one is asyncio?

**Good answer:**
In **preemptive** multitasking, a scheduler (the OS, for threads/processes) can interrupt a running task at essentially any point to give another task a turn, without that task's cooperation — this is how OS threads and processes work, and why you need locks to protect shared state even for "simple" operations. In **cooperative** multitasking, a running task keeps control until it voluntarily yields it back to the scheduler; nothing else runs in between unless the task itself hands off control. asyncio is cooperative: a coroutine runs uninterrupted from one `await` to the next, and it's only at an `await` point that the event loop gets a chance to run something else. This is why asyncio avoids a whole class of race conditions on ordinary Python objects between `await` points (no other coroutine can interleave), but also why a coroutine that never awaits can starve every other task in the process.

**Follow-up question:**
Given that asyncio is cooperative, do you still need `asyncio.Lock` for shared state between coroutines?

**Follow-up good answer:**
Often not for simple state accessed only between `await` points, precisely because nothing else runs while a coroutine executes synchronous code — a sequence of plain reads/writes with no `await` in between is effectively atomic from other coroutines' perspective. You do still need `asyncio.Lock` (or equivalent coordination) when a critical section itself contains an `await` — e.g. "check a value, await something, then update based on the earlier check" — because another coroutine can run during that awaited gap and change the state out from under you (a classic check-then-act race, just at `await` granularity instead of bytecode granularity). The rule of thumb: if there's no `await` between reading and writing shared state, you're safe under asyncio's cooperative model; if there is, you need explicit synchronization.

**Glossary:**
- **Cooperative multitasking** — a scheduling model where a task holds control until it explicitly yields.
- **Preemptive multitasking** — a scheduling model where the scheduler can interrupt a task at any time.
- **`asyncio.Lock`** — an async-aware mutex for coordinating coroutines across `await` boundaries.

**Mental model:**
Wants the candidate to connect the cooperative-scheduling theory to a concrete, correct answer about when synchronization is or isn't needed in async code — a common source of both over- and under-locking in real asyncio codebases.

**TL;DR:**
asyncio is cooperative — a coroutine only yields at `await` points — so plain state changes without an intervening `await` are safe, but a critical section spanning an `await` still needs an `asyncio.Lock` to avoid a check-then-act race.

**References:**
- [PEP 3156 – Asynchronous IO Support Rebooted](https://peps.python.org/pep-3156/)
- [`asyncio.Lock`](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Lock)

---

### Q11. How does asyncio's task/queue model relate to the theoretical CSP (Communicating Sequential Processes) or actor models of concurrency? {#q11}

**Question:**
How does asyncio's task/queue model relate to the theoretical CSP (Communicating Sequential Processes) or actor models of concurrency?

**Good answer:**
In both CSP and the actor model, independent units of execution don't share mutable state directly — they communicate by passing messages through channels (CSP) or mailboxes (actors), which sidesteps shared-memory races by construction. asyncio's common pattern of independent `Task`s communicating via `asyncio.Queue` maps closely onto CSP: each task is a sequential process, the queue is the channel, and `await queue.get()`/`await queue.put()` are the blocking (here, cooperatively-suspending) send/receive operations. It's not a strict, enforced implementation of either model — Python doesn't stop two tasks from sharing a mutable object directly if you choose to — but architecturally, "many small coroutines, communicating only via queues, each owning its own state" gives you most of CSP's safety benefits (no shared-state races within a single queue-based pipeline) using asyncio's primitives, which is why it's the recommended pattern for structuring non-trivial asyncio pipelines (producer/consumer, worker pools) rather than sharing dicts/lists across tasks directly.

**Follow-up question:**
Why is "queue-based tasks" specifically recommended over having tasks mutate a shared dict directly, if asyncio is cooperative and races need an `await` in between anyway?

**Follow-up good answer:**
It's about maintainability and correctness at scale, not just point-in-time safety. A shared dict mutated from multiple tasks still requires every developer touching that code to correctly reason about every `await` point across the whole codebase to know if a race is possible — that reasoning gets fragile as the code grows and changes. Queue-based message passing localizes state ownership to one task/coroutine, so you only have to reason about correctness within that one place, and the queue itself already handles the coordination (it's implemented with `asyncio.Lock`/condition variables internally) — you get a design that's robust to future changes in *other* parts of the code, not just correct for the current code as written.

**Glossary:**
- **CSP (Communicating Sequential Processes)** — a formal model of concurrency where processes communicate exclusively via channels rather than shared memory.
- **Actor model** — a concurrency model where independent actors communicate via asynchronous messages to mailboxes.
- **`asyncio.Queue`** — an async-aware FIFO queue used to coordinate producer/consumer coroutines.

**Mental model:**
This is a "connect the technology to the underlying theory" question — it separates candidates who've only used asyncio's syntax from those who understand why its idiomatic patterns look the way they do.

**TL;DR:**
asyncio's Task+Queue pattern is a practical instance of the CSP model (independent sequential processes communicating via channels instead of shared state), which is why it's recommended over direct shared-state mutation between coroutines.

**References:**
- [`asyncio.Queue`](https://docs.python.org/3/library/asyncio-queue.html)
- [PEP 3156 – Asynchronous IO Support Rebooted](https://peps.python.org/pep-3156/)

---

### Q12. What real production problem does asyncio actually solve that threads couldn't already solve? {#q12}

**Question:**
What real production problem does asyncio actually solve that threads couldn't already solve?

**Good answer:**
High-concurrency I/O at scale — the classic "C10K problem" of handling tens of thousands of simultaneous, mostly-idle connections (long-polling clients, WebSocket connections, many concurrent outbound API calls). Threads *can* solve I/O-bound concurrency in principle (the GIL is released during I/O), but each OS thread costs real memory (a default stack size in the megabytes) and has real OS scheduling overhead, so a process is practically limited to on the order of hundreds to low thousands of threads before that overhead dominates. asyncio coroutines are userspace objects scheduled by a single-threaded event loop — no OS thread per connection — so a single process can hold tens or hundreds of thousands of concurrent coroutines waiting on I/O with comparatively tiny memory/scheduling overhead per one. It solves the same *kind* of problem as threads (I/O-bound concurrency), just at a scale threads become impractical for.

**Follow-up question:**
If a service only ever needs to handle, say, 50 concurrent connections, is there a compelling reason to still choose asyncio over a simple threaded model?

**Follow-up good answer:**
Not on raw scalability grounds — 50 threads is well within comfortable OS limits, so asyncio's main scaling advantage isn't the deciding factor here. The remaining reasons to prefer asyncio at that scale are more about ecosystem/architecture fit: if you're already integrating with async-native libraries (an async web framework, async DB driver), staying in one model avoids awkward sync/async bridging code; and asyncio's explicit `await` points make control flow and cancellation easier to reason about precisely because nothing runs concurrently except at those points (no need to reach for locks for simple state). If neither applies — a small, mostly-synchronous codebase talking to blocking libraries — a threaded (or even purely synchronous) model is often simpler to write and debug, and "use asyncio because it's modern" isn't a strong enough reason on its own.

**Glossary:**
- **C10K problem** — the historical challenge of a single server handling ten thousand concurrent connections efficiently.
- **Userspace scheduling** — task switching managed by a language runtime/library rather than the OS kernel.

**Mental model:**
Tests whether the candidate can articulate asyncio's actual value proposition (scale/overhead) rather than reflexively recommending it, and whether they'd right-size the tool to the problem instead of defaulting to "async is better."

**TL;DR:**
asyncio's core win is handling very large numbers of concurrent I/O-bound connections with far less per-connection overhead than one-OS-thread-per-connection — at small scale, threads are often simpler and just as adequate.

**References:**
- [asyncio — Asynchronous I/O](https://docs.python.org/3/library/asyncio.html)
- [PEP 3156 – Asynchronous IO Support Rebooted](https://peps.python.org/pep-3156/)

---

### Q13. A coroutine calls a synchronous `time.sleep(2)` instead of `await asyncio.sleep(2)`. What actually happens to the rest of the application during those two seconds? {#q13}

**Question:**
A coroutine calls a synchronous `time.sleep(2)` instead of `await asyncio.sleep(2)`, inside an otherwise async web service. What actually happens to the rest of the application during those two seconds?

**Good answer:**
`time.sleep()` is a blocking call with no `await` point — it doesn't hand control back to the event loop, it just occupies the single thread the loop runs on for the full two seconds. Since asyncio is cooperative and single-threaded, that means *every other coroutine, every pending callback, and every I/O event* (other requests, timers, other tasks) is frozen for those two seconds too — a service that might normally handle thousands of concurrent requests effectively stalls completely, not just the one request that called `sleep`. This is different from a genuinely slow operation done correctly with `await asyncio.sleep(2)`, which yields control back to the loop immediately and lets everything else keep running while that one coroutine waits.

**Follow-up question:**
Your code doesn't call `time.sleep` directly, but a third-party library function it calls internally does a blocking network call with no async support. How do you use it safely inside an asyncio app?

**Follow-up good answer:**
Wrap the call in `loop.run_in_executor()` with a `ThreadPoolExecutor`, so the blocking library call runs on a separate OS thread instead of the event-loop thread — the coroutine `await`s the executor's future, which *is* a proper yield point, so the rest of the event loop keeps running normally while that thread blocks. This doesn't make the underlying call itself faster or non-blocking; it just moves the blocking off the thread the event loop depends on, which is the standard way to use blocking-only libraries (some DB drivers, some SDKs) from otherwise-async code without a full rewrite.

**Code example:**
```python
import asyncio

async def bad():
    import time
    time.sleep(2)  # blocks the ENTIRE event loop for 2s

async def good():
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, blocking_library_call)  # runs on a thread
```

**Glossary:**
- **Blocking call** — a call that occupies the calling thread until it completes, with no cooperative yield point.
- **`run_in_executor`** — the asyncio API for offloading blocking work to a thread/process pool without blocking the event loop.

**Mental model:**
A very common real-world bug pattern — tests whether the candidate viscerally understands "single-threaded cooperative" as meaning one bad blocking call takes down the whole app's concurrency, not just the one request.

**TL;DR:**
A blocking call with no `await` freezes the entire single-threaded event loop for its duration, stalling every other coroutine — fix it by offloading the blocking call to `run_in_executor` so it runs off the loop's thread.

**References:**
- [asyncio-dev: Running Blocking Code](https://docs.python.org/3/library/asyncio-dev.html#running-blocking-code)
- [`asyncio.sleep`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)

---

### Q14. You call an `async def` function without `await`ing it and nothing seems to happen. What's going on? {#q14}

**Question:**
You call an `async def` function without `await`ing it and nothing seems to happen — no error, no output. What's going on?

**Good answer:**
Calling an `async def` function doesn't execute its body at all — it immediately returns a **coroutine object**, the same way calling a generator function returns a generator object without running any of its code. The body only starts executing once that coroutine is driven forward, via `await`, `asyncio.create_task()`, or being passed to `asyncio.gather()`/a `TaskGroup`. If you just call it and discard the result, Python detects the coroutine object was garbage-collected without ever being awaited and emits a `RuntimeWarning: coroutine '...' was never awaited` — but only when the object is actually garbage-collected, which can be delayed, so it can look like "nothing happened, not even a warning" for a moment.

**Follow-up question:**
Why doesn't Python just raise this as a hard error immediately at the call site, instead of a delayed warning?

**Follow-up good answer:**
Because at the call site, Python can't yet know whether you intend to use the coroutine object later — you might be about to pass it to `asyncio.gather(coro1, coro2)` or store it in a list to schedule afterward, both perfectly valid uses of "call now, drive later." The warning is deliberately deferred to garbage-collection time because that's the only point where Python can be sure the coroutine was *never* going to be awaited — by then, if it's being destroyed without having run, that's unambiguously a bug. The trade-off is exactly the delay you noticed: the warning doesn't fire at the moment of the mistake, only whenever the garbage collector happens to reclaim that object.

**Glossary:**
- **Coroutine object** — the object returned by calling an `async def` function, representing suspended, not-yet-run code.
- **`RuntimeWarning: coroutine was never awaited`** — Python's diagnostic for a coroutine object that was garbage-collected without ever running.

**Mental model:**
Tests whether the candidate understands coroutines as lazy objects (like generators) rather than something that starts running on call — a foundational misunderstanding that causes silent bugs.

**TL;DR:**
Calling an `async def` function only creates a coroutine object — it doesn't run until awaited/scheduled — and an unawaited one only triggers a `RuntimeWarning` once it's garbage-collected, which can make the mistake look silent for a while.

**References:**
- [Coroutines and Tasks — asyncio-task](https://docs.python.org/3/library/asyncio-task.html)

---

### Q15. Two threads both do `if key not in cache: cache[key] = compute()`. Even with the GIL, why is this still a race? {#q15}

**Question:**
Two threads both run `if key not in cache: cache[key] = compute()` against a shared dict. Even with the GIL, why is this still a race, and what does it cost you?

**Good answer:**
This is a classic check-then-act race: "check if key is missing" and "compute and insert" are two separate operations spanning multiple bytecode instructions and — critically — the `compute()` call itself can be arbitrarily long and will release/re-acquire the GIL many times during its execution (at switch-interval boundaries, or immediately if it does any I/O). So thread A can check that `key` is missing, start computing, get switched out mid-computation, thread B checks the same key (still missing, since A hasn't inserted yet), and B *also* starts computing the same expensive value — you get redundant, wasted computation (and if `compute()` has side effects, like writing to a database, you can get duplicate writes), even though no single dict operation itself was corrupted.

**Follow-up question:**
How would you fix this cache-stampede pattern correctly?

**Follow-up good answer:**
Wrap the whole check-compute-insert sequence in a `threading.Lock` so it's atomic as a unit, not just each individual dict access: acquire the lock, check again inside it (a "double-checked" pattern — another thread might have filled it in while you were waiting for the lock), compute and insert only if still missing, then release. For higher-throughput scenarios, a per-key lock (e.g. a `defaultdict` of locks, or a proper caching library) avoids serializing unrelated keys behind one global lock — you only want to block threads racing for the *same* key, not all cache access. The general principle: any check-then-act sequence on shared state needs the whole sequence, not just each read/write, to be under one lock.

**Code example:**
```python
import threading

cache = {}
lock = threading.Lock()

def get_or_compute(key):
    with lock:
        if key not in cache:
            cache[key] = compute(key)  # protected as one atomic unit
        return cache[key]
```

**Glossary:**
- **Check-then-act race** — a race condition where a check and a subsequent action aren't atomic together, even if each is individually safe.
- **Cache stampede** — redundant concurrent recomputation of the same cache entry due to a check-then-act race.

**Mental model:**
Reinforces that "the GIL protects individual operations" doesn't mean "the GIL protects your logic" — a very common source of subtle production bugs in multi-threaded Python.

**TL;DR:**
"Check if missing, then compute and insert" is a check-then-act race even under the GIL, because `compute()` can be preempted between the check and the insert — fix it by holding a lock around the whole check-compute-insert sequence, not just the individual dict accesses.

**References:**
- [`threading.Lock`](https://docs.python.org/3/library/threading.html#lock-objects)
- [Python glossary: Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)

---

### Q16. You need to pass a closure (a function that captures a local variable) as the worker function to a `multiprocessing.Pool`. It fails — why, and what are your options? {#q16}

**Question:**
You need to pass a closure (a function that captures a local variable) as the worker function to a `multiprocessing.Pool`. It fails — why, and what are your options?

**Good answer:**
With the default `spawn` start method, the child process boots a fresh interpreter and must *unpickle* the target function — which for a plain function means importing it by its module-level qualified name. A closure isn't defined at module level and has no such stable importable name (and it also carries captured variables from its enclosing scope that pickling a plain function reference can't reconstruct), so `pickle` raises an error (typically `AttributeError` or `PicklingError`) trying to serialize it. This is the same root cause as failing to pickle a `lambda`: both are "not importable by reference."

**Follow-up question:**
What are your concrete options to actually get that per-call variable data into the worker function?

**Follow-up good answer:**
Use `functools.partial` with a module-level function instead of a closure — `partial` is itself picklable as long as the underlying function and the bound arguments are, so `functools.partial(worker_fn, captured_value)` carries the "captured" data explicitly as a picklable argument rather than via closure capture. Alternatively, pass the captured value as an extra argument to `pool.map`/`apply` directly (e.g. via `starmap` with tuples), or use the third-party `multiprocess`/`pathos` libraries, which use `dill` instead of `pickle` and *can* serialize closures and lambdas — useful when you don't control the function signature. If neither is workable and you're on POSIX, the `fork` start method (where available) avoids needing to pickle the target at all, since the child inherits the parent's memory directly — but it's no longer the default anywhere as of Python 3.14, and it comes with its own hazards (inheriting locks/threads from the parent at the moment of fork).

**Code example:**
```python
from functools import partial
from multiprocessing import Pool

def worker(multiplier, x):
    return x * multiplier

if __name__ == "__main__":
    with Pool() as pool:
        # instead of a closure like `lambda x: x * multiplier`
        results = pool.map(partial(worker, 3), [1, 2, 3])
```

**Glossary:**
- **Closure** — a function that captures variables from its enclosing scope.
- **`functools.partial`** — a picklable wrapper that pre-binds some arguments of a function.
- **`dill`** — a third-party serialization library (used by `multiprocess`/`pathos`) that can pickle closures and lambdas, unlike the standard `pickle`.

**Mental model:**
A very practical, commonly-hit pitfall — tests whether the candidate has an actual fix in their toolkit versus just recognizing "picklability is required."

**TL;DR:**
Closures/lambdas aren't picklable by reference, so spawn-based multiprocessing can't ship them to workers — use `functools.partial` on a module-level function (or `dill`-based alternatives) to pass the captured data explicitly instead.

**References:**
- [multiprocessing: Programming guidelines — picklability](https://docs.python.org/3/library/multiprocessing.html#multiprocessing-programming)
- [`functools.partial`](https://docs.python.org/3/library/functools.html#functools.partial)

---

### Q17. What's the practical difference between `asyncio.gather()` and `asyncio.TaskGroup` when one of several concurrent coroutines raises an exception? {#q17}

**Question:**
What's the practical difference between `asyncio.gather()` and `asyncio.TaskGroup` when one of several concurrent coroutines raises an exception?

**Good answer:**
With `asyncio.gather()` (default settings), if one coroutine raises, that exception propagates immediately to the caller of `gather` — but the *other* still-running coroutines are **not** automatically cancelled; they keep running in the background unless you explicitly handle that (e.g. via `return_exceptions=True` and manual cleanup). `asyncio.TaskGroup` (Python 3.11+) provides stronger structured-concurrency guarantees: if any task inside the group raises (other than `CancelledError`), the group cancels all the other still-running tasks in that group and waits for them to finish before propagating — you don't get orphaned background tasks. It also changes the exception shape: if multiple tasks fail, `TaskGroup` collects all of them into a single `ExceptionGroup` (`except*` in 3.11+ or iterating `.exceptions`) rather than surfacing only the first one, the way `gather` does by default.

**Follow-up question:**
If `TaskGroup` is the safer choice, why would you ever still reach for `gather()`?

**Follow-up good answer:**
`gather()` is still useful when you specifically want `return_exceptions=True` semantics — collecting a mix of results and exceptions for *every* coroutine without cancelling the others, e.g. "try N independent operations and report which succeeded/failed" rather than "abort everything if any one fails." It's also the only option pre-3.11 (or in codebases that need to support older Python versions), and its simpler single-exception-propagation model is sometimes exactly what you want for a small number of tightly-coupled calls where "first failure aborts everything, single stack trace" is fine and you don't need the structured-concurrency cancellation guarantees.

**Code example:**
```python
import asyncio

async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(risky_call_1())
            tg.create_task(risky_call_2())
    except* ValueError as eg:
        for exc in eg.exceptions:
            print("handled:", exc)
```

**Glossary:**
- **Structured concurrency** — a discipline where a group of concurrent tasks has a well-defined lifetime and failure/cancellation propagates predictably through the group.
- **`ExceptionGroup`** — a container (Python 3.11+) holding multiple exceptions raised concurrently, unpacked with `except*`.

**Mental model:**
Tests knowledge of a genuinely important, relatively recent API change — and whether the candidate understands *why* TaskGroup exists (orphaned tasks were a real footgun with gather) rather than just knowing it as a newer alternative.

**TL;DR:**
`gather()` propagates the first exception without cancelling sibling coroutines by default; `TaskGroup` (3.11+) cancels all sibling tasks on any failure and collects every exception into an `ExceptionGroup` — safer by default, at the cost of "abort-all" semantics.

**References:**
- [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup)
- [`asyncio.gather`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)

---

### Q18. What does Python 3.13's free-threaded build actually change, and is it something you'd deploy to production today? {#q18}

**Question:**
What does Python 3.13's free-threaded build actually change, and is it something you'd deploy to production today?

**Good answer:**
The free-threaded build (from PEP 703, available as an opt-in build starting in 3.13, `python3.13t`) removes the GIL as a hard requirement, so multiple threads can execute Python bytecode truly in parallel across cores. To make that safe without the GIL's implicit protection, it changes CPython's internals significantly: reference counting becomes "biased" (each thread has a private refcount for objects it owns, avoiding atomic ops on the common case), the memory allocator switches to a thread-safe design (mimalloc-based) enabling lock-free access patterns for lists/dicts in common cases, and per-object locks with "critical sections" replace the GIL's coarse protection for the cases that do need synchronization. As of 3.13, this build has roughly 5–8% single-threaded overhead versus the standard GIL build, and — more importantly for adoption — C extensions need to be updated/audited for thread-safety under this model; many popular extensions weren't yet fully compatible at 3.13's release.

**Follow-up question:**
Given that overhead and the ecosystem-compatibility gap, what would actually make you consider adopting the free-threaded build for a real service?

**Follow-up good answer:**
It only makes sense if the workload is genuinely GIL-bound on CPU-heavy, mostly-pure-Python code where multiprocessing's IPC/serialization overhead is itself a real cost you're trying to avoid (e.g. sharing large in-memory data structures across "threads" without copying/pickling them for each worker) — that's the specific case free-threading targets. You'd want to confirm every dependency in the critical path (especially C extensions) is verified free-threading-safe, since a single unsafe extension can silently reintroduce the exact races the GIL used to prevent, and benchmark the real single-threaded overhead against your actual workload rather than trusting the general 5–8% figure. Given it was still a relatively new, evolving option as of 3.13, most teams would pilot it carefully on a specific hot path rather than switching an entire production service to it outright.

**Glossary:**
- **Free-threaded build** — the CPython build variant, opt-in as of 3.13, where the GIL can be disabled.
- **Critical section** — a per-object (or per-group-of-objects) lock used by the free-threaded build in place of the GIL's global lock.

**Mental model:**
Tests whether the candidate can evaluate a genuinely new feature's trade-offs (overhead, ecosystem maturity) rather than treating "no GIL" as an unconditional win, which is exactly the kind of judgment a senior engineer needs before adopting bleeding-edge runtime changes.

**TL;DR:**
The 3.13+ free-threaded build removes the GIL via biased refcounting, a new allocator, and per-object locking, at ~5–8% single-threaded overhead and an ecosystem-compatibility gap — worth piloting for genuinely GIL-bound, shared-memory CPU workloads, not a default production choice yet.

**References:**
- [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)
- [Python glossary: free threading](https://docs.python.org/3/glossary.html#term-free-threading)

---

### Q19. When would you pick `ThreadPoolExecutor` over raw `threading.Thread`, and `ProcessPoolExecutor` over raw `multiprocessing.Process`? {#q19}

**Question:**
When would you pick `ThreadPoolExecutor`/`ProcessPoolExecutor` (from `concurrent.futures`) over raw `threading.Thread`/`multiprocessing.Process`?

**Good answer:**
Almost always prefer the executor-based APIs unless you need very fine-grained control the pool abstraction doesn't give you. `ThreadPoolExecutor`/`ProcessPoolExecutor` manage a bounded pool of reusable workers, give you a uniform `Future`-based API (`.submit()`, `.map()`, `.result()`, exception propagation through the future) regardless of whether you're using threads or processes, and handle worker lifecycle (creation, reuse, shutdown) for you — with raw `Thread`/`Process` you're manually creating, starting, and joining each one, and building your own result/exception plumbing if you need it. The executor's `max_workers` also gives you natural backpressure/concurrency limiting (e.g. "run at most 10 downloads at once") that's easy to get wrong by hand with raw threads. Reach for raw `Thread`/`Process` when you need a long-lived, uniquely-behaved worker (not part of a generic pool of interchangeable tasks) — e.g. a single background thread running its own loop for the life of the process.

**Follow-up question:**
Your `ProcessPoolExecutor.submit()` call for a CPU-heavy function returns a `Future` that raises `BrokenProcessPool` when you call `.result()`. What does that mean and how do you investigate it?

**Follow-up good answer:**
`BrokenProcessPool` means one of the pool's worker processes died unexpectedly (crashed, was killed, or raised something unpicklable/unrecoverable) in a way that leaves the pool unable to guarantee correctness for any pending or future work, so the executor deliberately fails fast for *all* outstanding futures rather than risk silently losing or corrupting results. To investigate, run the same function directly (not through the pool) with the same input to see if it segfaults or raises an exception that doesn't serialize cleanly (some C-extension crashes, or exceptions with non-picklable attributes, can manifest this way); check whether the worker was killed by an out-of-memory condition (common if each worker copies large data and you've undersized your worker count for available RAM); and check the exit code/logs of the worker process if your environment surfaces them, since the executor itself often can't tell you *why* the worker died, only that it did.

**Glossary:**
- **`Future`** — an object representing the eventual result (or exception) of a submitted task, used by both `concurrent.futures` and `asyncio`.
- **`BrokenProcessPool`** — an exception raised when a `ProcessPoolExecutor` worker terminates unexpectedly, invalidating the pool.

**Mental model:**
Tests practical, everyday API knowledge (most real code should use these higher-level pools, not raw threads/processes) plus whether the candidate has actually debugged a real-world executor failure mode, not just the happy path.

**TL;DR:**
Prefer `ThreadPoolExecutor`/`ProcessPoolExecutor` for their uniform Future-based API, worker reuse, and built-in concurrency limits; reach for raw `Thread`/`Process` only for a uniquely-behaved long-lived worker — and treat `BrokenProcessPool` as a signal a worker crashed (OOM, segfault, unpicklable failure), not a normal exception.

**References:**
- [`concurrent.futures` — ThreadPoolExecutor and ProcessPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html)

---

### Q20. Design the concurrency model for a service that: fetches data from 5 external APIs concurrently, then runs a CPU-heavy transformation on the combined result, for every incoming request. {#q20}

**Question:**
Design the concurrency model for a service that: fetches data from 5 external APIs concurrently, then runs a CPU-heavy transformation on the combined result, for every incoming request.

**Good answer:**
Two distinct bottlenecks need two distinct treatments. The 5 concurrent API calls are I/O-bound — use `asyncio` with an async HTTP client, firing all 5 with `asyncio.gather()` (or a `TaskGroup` if you want automatic cancellation of the others on any single failure) so they run concurrently instead of sequentially; this is the highest-leverage, lowest-overhead choice for "many concurrent waits" versus spinning up 5 threads or processes for that alone. The CPU-heavy transformation on the combined result is a different bottleneck entirely — running it directly inside the coroutine would block the event loop for every other concurrent request being served, so offload it via `loop.run_in_executor()` to a `ProcessPoolExecutor` (bypasses the GIL for true parallelism across requests) sized to the available CPU cores, or a `ThreadPoolExecutor` if the transformation is dominated by calls into a GIL-releasing C extension (e.g. NumPy) rather than pure Python. The overall per-request flow stays a single coroutine: `await gather(...)` the 5 calls, then `await run_in_executor(...)` the transform — cooperative the whole way, with the two different bottlenecks each routed to the concurrency primitive suited to it.

**Follow-up question:**
Under load, you notice the CPU-heavy transformation step is now the throughput bottleneck for the whole service. How do you decide how many worker processes to allocate to it?

**Follow-up good answer:**
Size the process pool to the number of available CPU cores (or slightly fewer, leaving headroom for the event-loop thread and other OS processes) since that's the point beyond which additional CPU-bound workers can't run in true parallel anyway — more workers than cores just adds context-switching and memory overhead without more real throughput. Measure per-transformation CPU time and memory footprint under realistic load to check the pool isn't causing memory pressure (each process has its own copy of whatever data it needs) or being starved by other CPU-heavy processes on the same host; if the transformation is short and frequent, also weigh the fixed per-call overhead of dispatching to a process pool (serialization + IPC) against just how much wall-clock time you're actually saving — for very small tasks, that overhead can eat the benefit, in which case batching multiple requests' transformations into fewer, larger process-pool calls can help.

**Glossary:**
- **`asyncio.gather`** — runs multiple awaitables concurrently and collects their results.
- **Worker pool sizing** — choosing the number of pool workers to match the resource (cores, for CPU-bound; a higher multiple, for I/O-bound) the work is actually bound by.

**Mental model:**
A synthesis question — tests whether the candidate can correctly decompose a mixed I/O-bound/CPU-bound workload into the right concurrency primitive for each part, which is exactly the kind of design decision that shows up in real backend services.

**TL;DR:**
Handle the 5 concurrent API calls with `asyncio.gather`/`TaskGroup` (I/O-bound), and offload the CPU-heavy transform to a `ProcessPoolExecutor` via `run_in_executor` (CPU-bound) — one coroutine per request coordinating both, each bottleneck routed to the primitive suited to it, with the process pool sized to available cores.

**References:**
- [`asyncio.gather`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)
- [asyncio-dev: Running Blocking Code](https://docs.python.org/3/library/asyncio-dev.html#running-blocking-code)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=concurrency-gil-and-async&autostart=1" | relative_url }})
