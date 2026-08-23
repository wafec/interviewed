---
layout: default
title: "Java Interview: Concurrency & the Java Memory Model"
---

# Java Interview: Concurrency & the Java Memory Model

Twenty questions on Java concurrency — from the fundamentals of threads and
locks, through JVM/JMM internals (monitors, `volatile`, AQS, `ConcurrentHashMap`
internals), to how you actually diagnose contention and GC-driven performance
problems in production, plus the modern `java.util.concurrent` toolkit
(`CompletableFuture`, `StampedLock`, virtual threads).

### Q1. What is the difference between a race condition and a data race?

**Question:**
What's the difference between a "race condition" and a "data race" in Java, and why does the distinction matter?

**Good answer:**
A **data race** is a specific, well-defined term from the Java Memory Model (JLS Chapter 17): it occurs when two threads access the same variable concurrently, at least one access is a write, and there is no happens-before ordering between the accesses. A data race means the JVM gives **no guarantees** about what value is observed — not just "maybe stale," but potentially any interleaving of partial writes, reordered instructions, or cached values.

A **race condition** is a broader software-engineering term: it's a bug where the correctness of the program depends on the relative timing/interleaving of operations, even if every individual memory access is technically safe (no data race). Example: `if (!map.containsKey(k)) map.put(k, v)` on a thread-safe `ConcurrentHashMap` has no data race (the map itself is safe), but it's still a race condition (check-then-act), because another thread can insert `k` between the check and the put.

So: every unsynchronized data race is a bug, but you can have a race condition (a logic bug) even with zero data races, if your synchronization is correct at the memory level but wrong at the algorithm level.

**Code example:**
```java
// Data race: no synchronization at all on `counter`
int counter = 0;
Runnable r = () -> counter++; // read-modify-write, not atomic, not visible

// Race condition WITHOUT a data race: map is thread-safe, but the
// check-then-act sequence is not atomic as a whole.
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
if (!map.containsKey("k")) {
    map.put("k", 1); // another thread may have inserted "k" here already
}
// Fix: use the atomic compound operation instead
map.putIfAbsent("k", 1);
```

**Follow-up question:**
How would `putIfAbsent` or `computeIfAbsent` fix the race condition example above, and why don't they suffer from the same issue?

**Glossary:**
- **Data race** — concurrent access to shared memory, one write, no happens-before edge between them (JLS §17.4.5).
- **Race condition** — outcome depends on timing/interleaving of operations; a correctness bug, not necessarily a memory-safety violation.
- **Happens-before** — a partial ordering the JMM guarantees between actions, ensuring visibility and ordering.
- **Check-then-act** — an anti-pattern where a condition is checked and then acted upon non-atomically.

**Mental model:**
This question tests whether the candidate conflates "thread-safe data structure" with "thread-safe algorithm using that data structure" — a very common source of real production bugs even among engineers who know the JMM basics.

**References:**
- [JLS §17.4 Memory Model](https://docs.oracle.com/javase/specs/jls/se24/html/jls-17.html)
- [ConcurrentHashMap javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

---

### Q2. What does "happens-before" mean in the Java Memory Model, and name three ways to establish it.

**Question:**
Explain the "happens-before" relationship in the JMM. Give at least three concrete ways a Java program establishes a happens-before edge between two threads.

**Good answer:**
Happens-before is a partial ordering defined by JLS Chapter 17 that guarantees two things when action A happens-before action B: (1) the effects of A (memory writes) are **visible** to B, and (2) the JVM/compiler cannot reorder A after B in a way that's observable. Without a happens-before edge between a write in one thread and a read in another, the reader has no guarantee it will ever see the write — not "probably will," genuinely undefined.

Concrete ways to establish happens-before in Java:
1. **Monitor lock/unlock**: an unlock of a monitor happens-before every subsequent lock of that same monitor (this is what makes `synchronized` blocks work for visibility, not just mutual exclusion).
2. **Volatile write/read**: a write to a `volatile` field happens-before every subsequent read of that same field.
3. **Thread.start()**: happens-before any action in the started thread.
4. **Thread termination**: all actions in a thread happen-before another thread successfully returns from `join()` on it.
5. **java.util.concurrent** utilities: e.g. placing an item in a `BlockingQueue` happens-before another thread retrieves it; a `CountDownLatch.countDown()` happens-before `await()` returns.

**Code example:**
```java
class Flag {
    private volatile boolean ready = false;
    private int data;

    void writer() {
        data = 42;       // (1) plain write
        ready = true;    // (2) volatile write — happens-before any subsequent read of `ready`
    }

    void reader() {
        if (ready) {      // (3) volatile read
            System.out.println(data); // guaranteed to see 42, NOT because data is volatile,
                                       // but because (1) happens-before (2) happens-before (3)
        }
    }
}
```

**Follow-up question:**
In the code example, why is it safe for the reader to see `data == 42` even though `data` itself is not `volatile`?

**Glossary:**
- **Happens-before** — JMM partial order guaranteeing visibility + no harmful reordering.
- **Program order** — within a single thread, happens-before is implied by sequential execution order.
- **Transitivity** — if A happens-before B and B happens-before C, then A happens-before C (used in the code example above).

**Mental model:**
Tests whether the candidate understands that `synchronized`/`volatile` are not just about "locking"/"not caching" but define a formal ordering contract — this is the theoretical foundation every other concurrency question in this set builds on.

**References:**
- [JLS §17.4.5 Happens-before Order](https://docs.oracle.com/javase/specs/jls/se24/html/jls-17.html)
- [JSR 133 (Java Memory Model) FAQ](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)

---

### Q3. What does `volatile` actually guarantee, and what does it NOT guarantee?

**Question:**
What does declaring a field `volatile` guarantee in Java? What common mistake do people make assuming `volatile` gives them more than it does?

**Good answer:**
`volatile` guarantees two things: **visibility** (every read sees the most recent write from any thread — no thread-local caching of the value) and a **happens-before ordering** (write happens-before subsequent read, plus the compiler/JIT cannot reorder other memory operations across the volatile access — this is what "no reordering" means in practice).

What it does **not** guarantee: **atomicity of compound operations**. `count++` on a `volatile int` is still a read-modify-write of three separate steps (read, add, write) and is not atomic — two threads can race and lose an increment. People frequently assume `volatile` makes counters or accumulations thread-safe; it doesn't. For that you need `synchronized`, a `Lock`, or `java.util.concurrent.atomic.AtomicInteger` (which uses CAS to make read-modify-write atomic).

**Code example:**
```java
private volatile int counter = 0;

void increment() { counter++; } // BUG under concurrency: not atomic, lost updates possible

// Correct fix:
private final AtomicInteger counter = new AtomicInteger();
void increment() { counter.incrementAndGet(); } // CAS loop, actually atomic
```

**Follow-up question:**
How does `AtomicInteger.incrementAndGet()` achieve atomicity without taking a lock? (Hint: CAS / `compareAndSwap`, and what happens on contention.)

**Glossary:**
- **Visibility** — a thread reading a variable sees the latest write from any thread.
- **Atomicity** — an operation appears indivisible; no other thread observes an intermediate state.
- **CAS (Compare-And-Swap)** — a hardware-supported atomic instruction: swap a value only if it still equals an expected value.

**Mental model:**
A classic trap question — separates candidates who've memorized "volatile = thread-safe" from those who understand the visibility vs. atomicity distinction, which is one of the most common real bugs in production Java code.

**References:**
- [JLS §17.4 Memory Model / volatile semantics](https://docs.oracle.com/javase/specs/jls/se24/html/jls-17.html)
- [AtomicInteger javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html)

---

### Q4. `synchronized` vs `ReentrantLock` — when would you choose one over the other?

**Question:**
Compare `synchronized` blocks and `java.util.concurrent.locks.ReentrantLock`. When would you pick `ReentrantLock` over plain `synchronized`?

**Good answer:**
`synchronized` uses the JVM's built-in intrinsic (monitor) lock: automatic acquire/release (even on exception, via the implicit `finally`), reentrant, but limited — no timeout on acquisition, no interruptible waiting, no ability to try-and-back-off, no fairness policy, and one condition-queue per monitor (`wait`/`notify`).

`ReentrantLock` (implementing `Lock`) is explicit: you must `lock()`/`unlock()` yourself (almost always in a `try/finally`), but you gain: `tryLock()` with optional timeout, `lockInterruptibly()`, an optional **fairness** policy (FIFO ordering of waiters, at a throughput cost), and multiple `Condition` objects per lock (instead of one implicit wait-set).

Pick `ReentrantLock` when you need: a timeout on acquisition, interruptible lock waits (to avoid a thread being stuck forever), multiple independent condition queues, or fairness guarantees. Otherwise, prefer `synchronized` — it's simpler, less error-prone (no risk of forgetting `unlock()`), and modern JVMs optimize uncontended `synchronized` extremely well (biased/lightweight locking), so there's no automatic performance win from `ReentrantLock`.

**Code example:**
```java
private final ReentrantLock lock = new ReentrantLock();

void doWork() {
    if (lock.tryLock()) {          // won't block forever
        try {
            // critical section
        } finally {
            lock.unlock();          // MUST be in finally — no implicit release
        }
    } else {
        // fallback: couldn't acquire, avoid blocking
    }
}
```

**Follow-up question:**
What happens if you forget the `finally` block and an exception is thrown inside the critical section while holding a `ReentrantLock`? Contrast with `synchronized`.

**Glossary:**
- **Reentrant** — a thread already holding the lock can re-acquire it without deadlocking itself.
- **Fairness** — FIFO granting of a lock to waiting threads, vs. default barging/unfair mode which favors throughput.
- **Condition** — `ReentrantLock`'s analogue to `Object.wait/notify`, but supporting multiple independent wait-sets per lock.

**Mental model:**
Tests whether the candidate can reason about trade-offs rather than reflexively reaching for "the more powerful API" — using `ReentrantLock` by default is itself often a code smell interviewers watch for.

**References:**
- [ReentrantLock javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
- [Lock Objects — Java Tutorials](https://docs.oracle.com/javase/tutorial/essential/concurrency/newlocks.html)

---

### Q5. What actually happens internally when a thread enters and exits a `synchronized` block?

**Question:**
Walk through what happens at the JVM level when a thread enters and exits a `synchronized` block — what is the "monitor," and what do `monitorenter`/`monitorexit` do?

**Good answer:**
Every Java object has an associated **monitor** (intrinsic lock), tracked via the object header. `synchronized` compiles to bytecode with `monitorenter`/`monitorexit` instructions (for blocks) or an `ACC_SYNCHRONIZED` method flag (for methods). On `monitorenter`, the thread attempts to acquire the monitor: if unheld, it becomes the owner (recording a hold count of 1, supporting reentrancy — the same thread can re-enter and increments the count); if held by another thread, the requesting thread blocks (parked) and is queued.

Modern HotSpot JVMs optimize this heavily rather than always taking a full OS-level mutex: **biased locking** (removed by default in modern JDKs, e.g. JEP 374 deprecated then disabled it) let an uncontended lock stay "biased" to the single thread using it with near-zero overhead; **lightweight/thin locking** uses a CAS on the object header to acquire the lock without OS involvement when uncontended; only under real contention does the JVM **inflate** to a heavyweight monitor backed by an OS mutex, which is when threads actually get descheduled and parked by the OS scheduler. On `monitorexit`, the hold count decrements; at zero, the lock is released and one waiting thread (if any) is unblocked.

Critically, unlock happens-before the next thread's lock of the same monitor (per the JMM), so exiting a synchronized block flushes writes that the next thread entering is guaranteed to see.

**Code example:**
```java
synchronized void increment() { // ACC_SYNCHRONIZED method
    count++;
}

void increment2() {
    synchronized (this) { // explicit monitorenter/monitorexit around this block
        count++;
    }
}
```

**Follow-up question:**
What is "lock inflation" and why does the JVM avoid starting every lock as a heavyweight OS mutex?

**Glossary:**
- **Monitor** — the object-associated lock structure supporting mutual exclusion + wait/notify.
- **monitorenter / monitorexit** — JVM bytecode instructions implementing `synchronized` blocks.
- **Lock inflation** — escalating a lightweight/CAS-based lock to a heavyweight OS-backed monitor under contention.

**Mental model:**
This is the canonical "what happens internally" question for Java — probes whether the candidate has any model of the JVM below the language keyword, not just "synchronized makes it thread-safe."

**References:**
- [JVM Specification §8.2.5 (monitorenter/monitorexit) via JLS Chapter 17](https://docs.oracle.com/javase/specs/jls/se24/html/jls-17.html)
- [Intrinsic Locks and Synchronization — Java Tutorials](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)

---

### Q6. Semaphore vs. Mutex/Lock — what's the actual difference?

**Question:**
What's the conceptual difference between a `Semaphore` and a mutual-exclusion lock (`Mutex`/`ReentrantLock`)? When would you use a `Semaphore` with a permit count greater than 1?

**Good answer:**
A **mutex/lock** enforces exclusive ownership: exactly one thread holds it, and (in Java's `ReentrantLock`/`synchronized`) typically only the owning thread can release it. A **semaphore** is a counting construct: it holds N permits; any thread can `acquire()` a permit (blocking if none available) and any thread can `release()` a permit — there's no notion of "ownership," so a thread other than the acquirer can legally release. A mutex is essentially a semaphore with exactly 1 permit, though Java's real mutex implementations add reentrancy and ownership checks that `Semaphore` deliberately lacks.

Use a `Semaphore` with permits > 1 to bound concurrent access to a limited resource — e.g. capping concurrent connections to a downstream service, limiting concurrent file handles, or throttling parallel task execution — anywhere you want "up to N threads at once," not "exactly one."

**Code example:**
```java
// Allow at most 5 concurrent calls to a rate-limited downstream API
private final Semaphore permits = new Semaphore(5);

Response call() throws InterruptedException {
    permits.acquire();
    try {
        return downstreamClient.call();
    } finally {
        permits.release();
    }
}
```

**Follow-up question:**
Since a `Semaphore` has no ownership, what bug class becomes possible that isn't possible with `ReentrantLock`?

**Glossary:**
- **Permit** — a semaphore's unit of availability; acquiring decrements the count, releasing increments it.
- **Mutex** — mutual-exclusion primitive limited to exactly one holder.
- **Binary semaphore** — a semaphore configured with 1 permit, behaviorally similar to (but not identical to) a mutex.

**Mental model:**
Checks whether the candidate understands synchronization primitives by their actual counting/ownership semantics rather than treating "lock," "mutex," and "semaphore" as interchangeable buzzwords.

**References:**
- [Semaphore javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Semaphore.html)

---

### Q7. What causes a deadlock, and how do you prevent one?

**Question:**
What are the necessary conditions for a deadlock? Give a concrete Java example and describe two concrete prevention strategies.

**Good answer:**
Deadlock requires (classically) four conditions simultaneously: **mutual exclusion** (resources can't be shared), **hold and wait** (a thread holds one resource while waiting for another), **no preemption** (a resource can't be forcibly taken from a thread), and **circular wait** (a cycle of threads each waiting on a resource held by the next). The textbook Java case: Thread A locks `objX` then tries to lock `objY`; Thread B locks `objY` then tries to lock `objX` — both block forever.

Prevention strategies:
1. **Lock ordering**: always acquire multiple locks in a fixed, global order (e.g. by object identity hash or a defined hierarchy) so a circular wait can't form.
2. **Timeouts / tryLock**: use `Lock.tryLock(timeout)` instead of unbounded blocking acquisition, so a thread backs off and retries instead of waiting forever, breaking "hold and wait" in practice.
3. Reduce the surface area: hold locks for the shortest possible time and avoid calling into unknown/external code while holding a lock (avoids surprising re-entrant lock acquisition).

**Code example:**
```java
// Deadlock-prone: order depends on call site
void transfer(Account a, Account b, int amt) {
    synchronized (a) {
        synchronized (b) { a.debit(amt); b.credit(amt); }
    }
}

// Fixed: always lock in a consistent global order
void transferSafe(Account a, Account b, int amt) {
    Account first = a.id() < b.id() ? a : b;
    Account second = a.id() < b.id() ? b : a;
    synchronized (first) {
        synchronized (second) { a.debit(amt); b.credit(amt); }
    }
}
```

**Follow-up question:**
How would you detect a live deadlock in a running production JVM without restarting it?

**Glossary:**
- **Circular wait** — a cycle in the "waiting-for" graph of threads and resources.
- **Lock ordering** — a discipline of always acquiring locks in the same relative order.
- **Livelock** — threads actively respond to each other but make no progress (contrast with deadlock's total blockage).

**Mental model:**
Tests both theoretical grounding (the four Coffman conditions) and whether the candidate has an actual practical fix, not just "avoid nested locks" hand-waving.

**References:**
- [Deadlock — Java Tutorials (Liveness)](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)

---

### Q8. How do you diagnose a production issue caused by thread contention?

**Question:**
A service's p99 latency has spiked and CPU usage looks normal (not maxed out) — you suspect thread contention/lock blocking rather than raw compute. What's your methodology and what tools do you reach for?

**Good answer:**
Methodology: first confirm the hypothesis (threads blocked, not CPU-bound) — normal/low CPU with high latency and possibly a growing thread count or queue depth is the classic signature of contention rather than compute-bound slowness. Then capture evidence, then narrow to root cause, then validate the fix.

Tools/steps:
1. **Thread dump**: `jcmd <pid> Thread.print` (preferred over the older `jstack <pid>`) captures all thread stacks at a point in time. Look for many threads `BLOCKED` on the same monitor/lock — the dump shows exactly which lock and which thread currently owns it.
2. Take **multiple dumps a few seconds apart** to see if the same threads are stuck in the same place (persistent contention) vs. just a snapshot artifact.
3. **Java Flight Recorder (JFR)**: attach with low overhead (`jcmd <pid> JFR.start`) to capture a continuous profile including lock contention events (`jdk.JavaMonitorEnter`/`jdk.JavaMonitorWait`), not just point-in-time snapshots — far better for intermittent contention.
4. Correlate with **GC logs/JFR GC events** — a long GC pause can look like "everything blocked" and be misdiagnosed as lock contention.
5. Once you identify the hot monitor/lock, look at the code path: is the critical section doing more work than necessary (I/O inside a lock, unnecessarily coarse-grained locking)? Fix by narrowing the critical section, switching to a finer-grained or lock-free structure (e.g. `ConcurrentHashMap` instead of a synchronized `HashMap`), or replacing a lock with a `Semaphore`/queue-based design.
6. **Validate**: re-run the same load test / re-capture JFR and confirm blocked-thread time and p99 latency actually dropped — don't ship a "should fix it" change without measurement.

**Code example:**
```bash
# capture a thread dump, including java.util.concurrent lock owners
jcmd <pid> Thread.print -l

# start a JFR recording for 60s focused on the running process
jcmd <pid> JFR.start duration=60s filename=recording.jfr
```

**Follow-up question:**
In a thread dump, how do you distinguish a thread that's `BLOCKED` on a monitor from one that's `WAITING` because of `Object.wait()` or a `Condition.await()`, and why does that distinction change your diagnosis?

**Glossary:**
- **jcmd** — the modern, recommended diagnostic-command tool; superset of `jstack`/`jmap`/`jinfo` functionality with lower overhead.
- **JFR (Java Flight Recorder)** — low-overhead, continuous JVM profiling/event framework built into HotSpot.
- **BLOCKED vs WAITING** — thread states: `BLOCKED` = waiting to acquire a monitor; `WAITING`/`TIMED_WAITING` = voluntarily waiting on a condition/notification/sleep.

**Mental model:**
This is the "performance methodology" question every interview is now asking in some form — tests whether the candidate has a real, tool-backed investigative process versus guessing-and-changing-code.

**References:**
- [Troubleshooting Guide — Diagnostic Tools (Java SE 21)](https://docs.oracle.com/en/java/javase/21/troubleshoot/diagnostic-tools.html)
- [jstack Command Reference](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jstack.html)

---

### Q9. What is `ThreadLocal`, and what's the classic pitfall with it in a thread-pooled environment?

**Question:**
What problem does `ThreadLocal` solve? What's the well-known memory-leak pitfall when using it inside an application running on a thread pool (e.g. a web server), and how do you avoid it?

**Good answer:**
`ThreadLocal<T>` gives each thread its own independent copy of a variable — useful for per-thread state that shouldn't be shared or synchronized, e.g. a per-request `SimpleDateFormat` (not thread-safe to share), a per-transaction context, or request-scoped tracing IDs.

The pitfall: in a **thread pool** (e.g. servlet container, `ExecutorService`), threads are long-lived and reused across many tasks/requests. If you `set()` a `ThreadLocal` value and never `remove()` it, the value stays attached to the pooled thread **after the task finishes**, because the thread itself isn't destroyed — it just goes back to the pool. This causes (a) memory leaks (the value, and anything it references, is retained indefinitely — a classic cause of `OutOfMemoryError` in long-running app servers) and (b) **data leakage** across unrelated requests, since the next task on that thread can see stale state left by a previous, unrelated task.

Fix: always `remove()` the `ThreadLocal` in a `finally` block once you're done with it (e.g. at the end of request processing), not just letting it go out of scope.

**Code example:**
```java
private static final ThreadLocal<String> requestId = new ThreadLocal<>();

void handleRequest(String id) {
    requestId.set(id);
    try {
        process();
    } finally {
        requestId.remove(); // mandatory in pooled-thread environments
    }
}
```

**Follow-up question:**
Why does this leak not show up as a `Thread`-level leak in a heap dump, but rather appears attached to `Thread.threadLocals`? How would you find it in a heap dump?

**Glossary:**
- **ThreadLocal** — per-thread variable storage, backed internally by a `ThreadLocalMap` referenced from the `Thread` object itself.
- **Thread pool** — a set of reused worker threads (e.g. `ThreadPoolExecutor`), meaning `Thread` objects outlive any single task.

**Mental model:**
This tests real production experience — `ThreadLocal` leaks in pooled environments are one of the most common "worked in dev, OOM'd in prod after N hours" bugs, and this question separates textbook knowledge from operational scars.

**References:**
- [ThreadLocal javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html)

---

### Q10. How does `ConcurrentHashMap` achieve thread safety without locking the whole table?

**Question:**
How does `ConcurrentHashMap` provide thread-safe access with much better concurrency than a synchronized `HashMap`? How did its internal locking strategy change between Java 7 and Java 8+?

**Good answer:**
In **Java 7 and earlier**, `ConcurrentHashMap` used **lock striping**: the table was divided into a fixed number of `Segment`s (default 16), each independently `synchronized`. Writes to different segments could proceed fully in parallel; only writes to the *same* segment contended.

From **Java 8 onward**, segments were removed. The table is now an array of bins (like `HashMap`), and thread safety is achieved with a mix of: **CAS** (`Unsafe`/`VarHandle` compare-and-swap) for adding the first node into an empty bin (no lock needed at all in the common case), and `synchronized` on the **first node of a bin** only when there's already a chain/tree there (so contention is per-bin, not per-segment — far finer-grained, since bin count scales with table size, not a fixed 16). Table resizing itself is done cooperatively — multiple threads can help transfer nodes to the new table concurrently.

Regardless of era, key guarantees stay the same: `get()` never blocks (fully lock-free, reads see a consistent snapshot via volatile/CAS-published references), iteration is weakly consistent (won't throw `ConcurrentModificationException`, but may or may not reflect concurrent updates during the iteration), and compound operations like `putIfAbsent`, `computeIfAbsent`, `merge` are atomic — which is exactly the tool for the check-then-act problem from Q1.

**Code example:**
```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
// atomic increment-or-initialize, no external locking needed
counts.merge("key", 1, Integer::sum);
```

**Follow-up question:**
Why does `ConcurrentHashMap.size()` being an approximate/eventually-consistent value under heavy concurrent modification not usually matter in practice — and when would it?

**Glossary:**
- **Lock striping** — dividing a structure into independently-lockable segments to reduce contention.
- **CAS** — compare-and-swap; lock-free atomic update.
- **Weakly consistent iterator** — an iterator that tolerates concurrent modification without throwing, without guaranteeing it reflects every update.

**Mental model:**
Probes genuine internals knowledge of a class nearly every Java engineer uses but few can explain — a strong signal for "has read source / understands trade-offs" vs. "knows the API surface only."

**References:**
- [ConcurrentHashMap javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

---

### Q11. What is `AbstractQueuedSynchronizer` (AQS), and why does it matter even if you never use it directly?

**Question:**
What is `AbstractQueuedSynchronizer`? Why is it significant to understand even though most developers never touch it directly?

**Good answer:**
`AbstractQueuedSynchronizer` (AQS) is a framework class in `java.util.concurrent.locks` for building blocking synchronization primitives. It provides the hard, easy-to-get-wrong plumbing — an internal FIFO wait queue of blocked threads, and atomic (CAS-based) management of a single `int` state — while leaving subclasses to define what "acquire" and "release" mean for their specific semantics (exclusive vs. shared).

It matters because it's the shared foundation underneath most of `java.util.concurrent`'s synchronizers: `ReentrantLock` (state = hold count, exclusive mode), `Semaphore` (state = permit count, shared mode), `CountDownLatch` (state = count, shared mode, release when it hits zero), and `ReentrantReadWriteLock`. Understanding AQS means understanding *why* these primitives behave the way they do (e.g. why `ReentrantLock`'s fairness option affects the FIFO queue, or why `Semaphore.release()` can be called by a non-acquiring thread) instead of memorizing each class's API in isolation.

**Follow-up question:**
Why does AQS use a single `int` for state rather than something more expressive, and how does `Semaphore` encode "permits available" into that single int?

**Glossary:**
- **AQS** — `AbstractQueuedSynchronizer`, the shared framework for blocking lock/synchronizer implementations.
- **Exclusive mode** — only one acquirer can hold the synchronizer at a time (used by `ReentrantLock`).
- **Shared mode** — multiple acquirers can hold it simultaneously (used by `Semaphore`, `CountDownLatch`).

**Mental model:**
An "advanced feature" question that tests whether the candidate has looked one layer below the `java.util.concurrent` API surface — a strong senior signal, since most mid-level candidates use these classes without knowing they share an implementation.

**References:**
- [AbstractQueuedSynchronizer javadoc (Java SE 8)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)

---

### Q12. `Future` vs `CompletableFuture` — what problem does `CompletableFuture` actually solve?

**Question:**
What limitation of the plain `Future` interface does `CompletableFuture` address? Walk through a realistic use case chaining async operations.

**Good answer:**
`Future<T>` (from `ExecutorService.submit`) only supports **blocking retrieval** via `get()` (with an optional timeout) — there's no way to attach a callback, compose it with another async operation, or combine multiple futures without blocking a thread to wait on each one. That's expensive at scale (thread-per-blocked-wait) and awkward to compose.

`CompletableFuture<T>` implements both `Future` and `CompletionStage`, letting you build **non-blocking pipelines**: `thenApply`/`thenCompose` to transform or chain, `thenCombine` to join two independent futures, `exceptionally`/`handle` for error handling, and `allOf`/`anyOf` to wait on multiple futures without blocking each individually. Each stage can run on a supplied `Executor` (via the `Async` variants), decoupling "what to run" from "which thread pool runs it" — critical for not accidentally starving a shared pool.

**Code example:**
```java
CompletableFuture<User> userFuture = fetchUserAsync(id);
CompletableFuture<Orders> ordersFuture = userFuture.thenCompose(u -> fetchOrdersAsync(u.id()));

ordersFuture
    .thenApply(orders -> orders.total())
    .exceptionally(ex -> { log.error("failed", ex); return BigDecimal.ZERO; })
    .thenAccept(total -> System.out.println("Total: " + total));
// non-blocking: no thread is parked waiting on either call
```

**Follow-up question:**
What's the difference between `thenApply` and `thenCompose`, and when would using the wrong one give you a `CompletableFuture<CompletableFuture<T>>`?

**Glossary:**
- **CompletionStage** — the interface defining composable async pipeline stages.
- **thenApply** — transform the result with a synchronous function (`T -> U`).
- **thenCompose** — flatten/chain into another async stage (`T -> CompletableFuture<U>`), avoiding nested futures.

**Mental model:**
Tests whether the candidate can reason about async composition, not just "I used `CompletableFuture.supplyAsync` once" — this is core to modern service-to-service code.

**References:**
- [CompletableFuture javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CompletableFuture.html)

---

### Q13. How do you size a thread pool (`ThreadPoolExecutor`), and what happens when the queue fills up?

**Question:**
How would you choose `corePoolSize`/`maximumPoolSize` for a `ThreadPoolExecutor`? What actually happens when submitted tasks exceed capacity, and what are the rejection options?

**Good answer:**
Sizing depends on the workload type. For **CPU-bound** work, a pool around `N_cpu (+1)` threads is close to optimal — more threads than cores just adds context-switch overhead. For **I/O-bound** work (blocking on network/disk), threads spend most of their time waiting, so a much larger pool is appropriate — a common heuristic is `N_cpu * (1 + waitTime/computeTime)`, tuned empirically with load testing, not guessed.

`ThreadPoolExecutor` behavior on submission: if fewer than `corePoolSize` threads exist, a new thread is started immediately (even if others are idle). Once at `corePoolSize`, new tasks go into the **work queue** first, not straight to new threads. Only once the queue is full does the pool grow threads up to `maximumPoolSize`. If the queue is full **and** the pool is already at `maximumPoolSize`, the configured `RejectedExecutionHandler` kicks in — default `AbortPolicy` throws `RejectedExecutionException`; `CallerRunsPolicy` runs the task on the submitting thread (a built-in backpressure mechanism); `DiscardPolicy`/`DiscardOldestPolicy` silently drop tasks. An **unbounded queue** (e.g. plain `LinkedBlockingQueue`) means `maximumPoolSize` is effectively never reached — the queue absorbs everything, which can hide overload until you run out of memory instead of getting controlled rejection.

**Code example:**
```java
new ThreadPoolExecutor(
    10,                              // corePoolSize
    50,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,           // keepAliveTime for threads above core
    new ArrayBlockingQueue<>(200),   // bounded queue -> real backpressure
    new ThreadPoolExecutor.CallerRunsPolicy() // apply backpressure to caller instead of dropping
);
```

**Follow-up question:**
Why is an unbounded work queue considered a production anti-pattern even though it "never rejects" tasks?

**Glossary:**
- **corePoolSize / maximumPoolSize** — minimum threads kept alive vs. the ceiling under load.
- **RejectedExecutionHandler** — the pluggable policy invoked when a task can't be queued or run.
- **Backpressure** — deliberately slowing/rejecting producers when a system is saturated, instead of unbounded buffering.

**Mental model:**
A very common real "system design meets Java" question — checks whether the candidate has actually tuned a thread pool under real load versus just calling `Executors.newFixedThreadPool(n)` and hoping.

**References:**
- [ThreadPoolExecutor javadoc (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)

---

### Q14. What problem do virtual threads (JEP 444, Java 21) solve, and what do they NOT fix?

**Question:**
What problem do Java's virtual threads (finalized in JDK 21 via JEP 444) solve? What limitations or gotchas remain, especially around `synchronized` and native code?

**Good answer:**
Traditional Java threads are thin wrappers around **OS threads** — expensive to create (megabyte-scale stacks) and limited in number (thousands, not millions), which forces high-throughput I/O-bound servers into thread-pool-and-reuse patterns or reactive/async programming models just to avoid running out of OS threads while waiting on I/O. **Virtual threads** are lightweight threads implemented by the JDK, not the OS: many virtual threads are multiplexed onto a small number of platform (carrier) threads. When a virtual thread blocks on supported blocking operations (e.g. blocking I/O, `Thread.sleep`), it's **unmounted** from its carrier thread, freeing the carrier to run another virtual thread — so you can write ordinary blocking, synchronous-style code (`InputStream.read()`, JDBC calls) and still get massive concurrency, without rewriting to reactive/async style.

What it doesn't fix: virtual threads are not faster for **CPU-bound** work (no more parallelism than you have cores) — they help *I/O-bound* concurrency, not compute throughput. Historically, `synchronized` blocks could "pin" a virtual thread to its carrier during blocking operations inside the block, defeating the benefit (this was significantly improved/removed for most cases by JEP 491 in later JDKs — worth knowing this evolved). Thread-local-heavy code can also behave differently since virtual threads are cheap and numerous, so patterns like large fixed-size thread pools designed around a small thread count need rethinking.

**Follow-up question:**
Why doesn't creating a million virtual threads by itself improve throughput for a CPU-bound task like a tight numerical loop?

**Glossary:**
- **Virtual thread** — a JDK-managed lightweight thread, not directly mapped 1:1 to an OS thread.
- **Carrier thread** — the platform (OS) thread a virtual thread is currently mounted on.
- **Pinning** — a virtual thread staying mounted to its carrier during a blocking operation (can't be unmounted), reducing the benefit.

**Mental model:**
A trending, current-events-style question (JDK 21 is now mainstream) that tests whether the candidate keeps up with the platform's evolution and can articulate the actual mechanism, not just "virtual threads = faster."

**References:**
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Virtual Threads — Java SE 21 documentation](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

---

### Q15. `CountDownLatch` vs `CyclicBarrier` — what's the difference and when would you use each?

**Question:**
What's the functional difference between `CountDownLatch` and `CyclicBarrier`? Give a realistic use case for each.

**Good answer:**
`CountDownLatch` is a **one-shot** gate: initialized with a count, any thread can call `countDown()` (they don't have to be "the waiting threads"), and once the count reaches zero it stays open forever — `await()` calls after that return immediately, and the count cannot be reset. Classic use: a main thread waits for N worker threads to finish initialization before proceeding (`new CountDownLatch(N)`, each worker calls `countDown()` on completion, main calls `await()`).

`CyclicBarrier` is **reusable**: a fixed number of threads must all call `await()` before any of them proceeds — once all N arrive, the barrier "breaks," optionally runs a supplied `Runnable`, and then **automatically resets** for the next round. Classic use: parallel divide-and-conquer computation where multiple worker threads must all finish phase 1 before any starts phase 2, repeated across many phases.

**Code example:**
```java
// CountDownLatch: one-time startup gate
CountDownLatch ready = new CountDownLatch(3);
for (int i = 0; i < 3; i++) new Thread(() -> { init(); ready.countDown(); }).start();
ready.await(); // proceeds once all 3 have counted down, never again after

// CyclicBarrier: repeated synchronization point across phases
CyclicBarrier barrier = new CyclicBarrier(4, () -> System.out.println("phase done"));
Runnable worker = () -> {
    for (int phase = 0; phase < 5; phase++) {
        doPhaseWork(phase);
        barrier.await(); // blocks until all 4 threads reach this point, then resets
    }
};
```

**Follow-up question:**
What happens to a `CyclicBarrier` if one of the participating threads throws an exception instead of calling `await()`?

**Glossary:**
- **Latch** — a one-time gate that opens permanently once triggered.
- **Barrier** — a reusable rendezvous point requiring all parties to arrive before any proceeds.

**Mental model:**
A fundamentals question that also tests API-choice judgment — picking the wrong one (e.g. `CountDownLatch` for a repeated phase barrier) is a real design mistake interviewers watch for.

**References:**
- [High Level Concurrency Objects — Java Tutorials](https://docs.oracle.com/javase/tutorial/essential/concurrency/highlevel.html)
- [CountDownLatch javadoc (Java SE 8)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html)

---

### Q16. What is `StampedLock`'s optimistic read mode, and why can it outperform `ReentrantReadWriteLock`?

**Question:**
What does `StampedLock`'s optimistic read mode do, and why can it be significantly faster than a traditional read-write lock under read-heavy workloads?

**Good answer:**
`StampedLock` offers three modes: writing (exclusive), reading (shared, like a normal read lock), and **optimistic reading** — the fast one. `tryOptimisticRead()` returns a stamp **without acquiring any lock at all** (no CAS, no blocking, no memory barrier for other readers). The caller reads the shared state, then calls `validate(stamp)` to check whether a write occurred in the meantime; if not, the read was safe and consistent; if a write happened, the caller must fall back to acquiring a real read lock and retrying.

This beats `ReentrantReadWriteLock` under heavy read concurrency because a traditional read lock still requires every reader to perform a shared-state update (incrementing a reader count) that all readers contend on, which limits scalability with many concurrent readers. Optimistic reads let the truly common case (no concurrent writer) proceed with **zero** contention.

The catch: code in an optimistic-read section must be side-effect-free (you might be reading inconsistent, torn state — you're gambling that it's fine because you'll validate before acting on it) and must not call anything that isn't safe to run against a possibly-inconsistent snapshot.

**Code example:**
```java
private final StampedLock lock = new StampedLock();
private double x, y;

double distanceFromOrigin() {
    long stamp = lock.tryOptimisticRead();
    double curX = x, curY = y; // may be torn/inconsistent
    if (!lock.validate(stamp)) {          // a write happened concurrently — fall back
        stamp = lock.readLock();
        try { curX = x; curY = y; }
        finally { lock.unlockRead(stamp); }
    }
    return Math.hypot(curX, curY);
}
```

**Follow-up question:**
Why does `StampedLock` not support reentrancy the way `ReentrantReadWriteLock` does, and what real bug can that cause if you're not careful?

**Glossary:**
- **Optimistic read** — an unlocked read followed by a validation check, retried under a real lock if invalidated.
- **Torn read** — observing a partially-updated, inconsistent combination of fields.
- **Stamp** — a version token `StampedLock` returns, used both to release/convert locks and to validate optimistic reads.

**Mental model:**
An "advanced feature" question distinguishing candidates who know the standard `java.util.concurrent` toolkit from those who've reached for the less common but high-value primitives under real read-heavy performance pressure.

**References:**
- [StampedLock javadoc (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/StampedLock.html)

---

### Q17. What is false sharing, and how would you detect and fix it?

**Question:**
What is "false sharing" in the context of multi-threaded performance? How would you detect it, and how do you fix it in Java?

**Good answer:**
CPUs move memory between cores in fixed-size **cache lines** (commonly 64 bytes), not individual variables. False sharing happens when two *logically unrelated* variables, written by *different threads*, happen to live on the **same cache line** — even though the threads never touch each other's variable, every write invalidates the whole cache line on other cores, forcing expensive cache-coherency traffic (MESI protocol invalidations) as if there were real contention. The result: severe throughput degradation with no visible logical race and no lock contention in a thread dump — this is exactly why it's a "performance internals" question, not a correctness one.

Detection is trickier than lock contention: a thread dump/JFR lock-contention view won't show it (there's no lock). Signs include unexplained scaling cliffs (throughput doesn't improve, or gets worse, when adding cores/threads writing to nearby fields) and can be confirmed with hardware performance counters (e.g. via `perf stat -e cache-misses,cache-references` on Linux, or JFR's native memory/hardware event profiling where available) showing abnormal cache-miss rates correlated with the suspect code path.

Fix: **padding** — separate the hot fields so each lands on its own cache line, e.g. by adding unused padding fields around a heavily-contended counter, or using `@jdk.internal.vm.annotation.Contended` (a JDK-internal annotation used for exactly this in `java.util.concurrent` classes like `Exchanger`/`ConcurrentHashMap` counters), or simply restructuring so per-thread counters are stored in separate array slots/objects far enough apart rather than adjacent fields of one shared object.

**Code example:**
```java
// Prone to false sharing: counter1 and counter2 likely share a cache line
class Counters {
    volatile long counter1;
    volatile long counter2;
}

// Padded to force separate cache lines (64 bytes = 8 longs on typical hardware)
class PaddedCounters {
    volatile long counter1;
    long p1, p2, p3, p4, p5, p6, p7; // padding
    volatile long counter2;
}
```

**Follow-up question:**
Why can padding-based fixes be fragile across different JVMs/hardware, and what's a more portable alternative in modern Java?

**Glossary:**
- **Cache line** — the fixed-size unit (typically 64 bytes) transferred between CPU cache and memory.
- **False sharing** — cache-line-level contention between logically independent variables.
- **MESI protocol** — the cache-coherency protocol CPUs use to keep caches consistent across cores.

**Mental model:**
A genuinely "advanced/trending" performance question — most candidates know lock contention, few can explain a slowdown with zero visible locking, which is exactly the kind of scenario current interviews probe for.

**References:**
- [JEP draft / OpenJDK discussion referencing @Contended (jdk.internal.vm.annotation.Contended)](https://bugs.openjdk.org/browse/JDK-8153057)
- [Troubleshooting Guide — Diagnostic Tools (Java SE 21)](https://docs.oracle.com/en/java/javase/21/troubleshoot/diagnostic-tools.html)

---

### Q18. How does GC pause time show up as a "performance problem," and how do you diagnose it separately from application-level slowness?

**Question:**
A service has occasional latency spikes that don't correlate with request volume. How would you determine whether garbage collection is the cause, and what would you look at in G1 GC specifically?

**Good answer:**
Any stop-the-world GC pause freezes **all** application threads for its duration, which shows up indistinguishable from "the app froze" at the request layer — a request in flight during the pause just stalls. The signature to look for: latency spikes with **no corresponding CPU/throughput spike**, often periodic or bursty, that align with GC log timestamps.

Diagnosis: enable and read **GC logs** (`-Xlog:gc*` on modern JDKs) or capture a JFR recording, which includes GC pause events (`jdk.GCPhasePause`, heap-before/after-size) correlated with the same timeline as your latency metrics — overlay them and look for coincidence. For **G1 GC** specifically (the default collector since JDK 9), look at: young-gen pause frequency/duration (should be short and frequent, tunable via `-XX:MaxGCPauseMillis` as a *target*, not a hard guarantee), and whether you're seeing rare-but-large **mixed collections** or, worse, a **Full GC** (G1's fallback when it can't keep up with allocation/reclaim rate, and by far the most expensive pause type — should essentially never happen in a healthy G1 setup).

Fix options depend on root cause: reduce allocation rate (object churn) in hot paths, tune heap size/region size, adjust `-XX:MaxGCPauseMillis` (with the understanding it's a soft target G1 tries to meet, trading off throughput), or consider a low-pause collector (ZGC/Shenandoah) if sub-millisecond pause requirements can't be met by G1 at all. Always validate with before/after GC logs and latency percentiles, not just "we changed a flag."

**Code example:**
```bash
# modern unified logging: log GC details with timestamps to a file
java -Xlog:gc*:file=gc.log:time,uptime,level,tags -jar app.jar
```

**Follow-up question:**
Why is a Full GC in G1 considered a "failure mode" rather than normal operation, unlike in the older CMS/Parallel collectors where full GCs were more routine?

**Glossary:**
- **Stop-the-world (STW) pause** — a GC phase where all application threads are suspended.
- **Mixed collection** — a G1 collection cycle that reclaims both young and some old-gen regions.
- **Full GC** — G1's fallback, full-heap, non-incremental collection; expensive and a sign of misconfiguration/overload.

**Mental model:**
Tests whether the candidate treats GC as an opaque black box ("just tune the heap") or as a diagnosable subsystem with its own tools and failure signatures — a very common real production performance investigation.

**References:**
- [Garbage-First Garbage Collector — HotSpot GC Tuning Guide (Java SE 17)](https://docs.oracle.com/en/java/javase/17/gctuning/hotspot-virtual-machine-garbage-collection-tuning-guide.pdf)
- [Getting Started with the G1 Garbage Collector](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)

---

### Q19. Why are immutable objects considered "automatically thread-safe," and what are the rules to actually make a class immutable in Java?

**Question:**
Why does immutability give you thread safety "for free"? What are the concrete rules for making a Java class truly immutable (not just having `final` fields)?

**Good answer:**
An immutable object's state can never change after construction, so there is no write to race against — every thread that can see a reference to it sees the same, permanently-valid state. No locking is needed for reads because there's nothing that can become stale or torn. This is why immutable value types are a go-to tool for concurrent code: pass them freely between threads with zero synchronization overhead.

Rules for real immutability in Java (having `final` fields alone is not sufficient):
1. All fields `private final`.
2. No setter methods, and no other methods that mutate state.
3. The class itself should be `final` (or otherwise ensure subclasses can't break invariants) to prevent a subclass from adding mutable behavior.
4. If a field holds a **mutable** object (e.g. a `Date`, an array, a mutable collection), you must **defensively copy** it on construction and on any getter that returns it — otherwise external code can mutate the "immutable" object's internal state through the reference.
5. Ensure proper construction so a partially-constructed object is never published to another thread before the constructor finishes (avoid leaking `this` during construction) — otherwise another thread could observe an inconsistent pre-final state due to reordering, even with all-final fields, in pre-JMM-updated (older) understandings; the JMM's special "final field" semantics (JLS §17.5) guarantee a thread that sees a reference to a correctly-constructed immutable object sees fully-initialized final fields, *provided the reference isn't leaked during construction*.

**Code example:**
```java
final class Point {
    private final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    int x() { return x; }
    int y() { return y; }
    // no setters, class is final, primitives need no defensive copy
}
```

**Follow-up question:**
If a class has a `final List<String> items` field but exposes it via a plain getter (`return items;`), is the class actually immutable? Why or why not?

**Glossary:**
- **Immutability** — state cannot change after construction.
- **Defensive copy** — copying a mutable input/output so external code can't reach into an object's internal state.
- **Final field semantics (JLS §17.5)** — special JMM guarantees for `final` fields, ensuring safe publication without explicit synchronization, if construction doesn't leak `this`.

**Mental model:**
This connects a Java-specific idiom to a general software-engineering theory principle (state-minimization = concurrency-safety-by-construction) — the exact "theory mixed with practice" angle this file's contract requires.

**References:**
- [JLS §17.5 final Field Semantics](https://docs.oracle.com/javase/specs/jls/se24/html/jls-17.html)
- [Immutable Objects — Java Tutorials](https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html)

---

### Q20. What's the difference between deadlock, livelock, and starvation?

**Question:**
Deadlock, livelock, and starvation are all "liveness" failures in concurrent programs. What distinguishes them, and give a realistic example of each.

**Good answer:**
All three mean threads fail to make useful progress, but for different reasons:
- **Deadlock**: threads are permanently blocked, each waiting on a resource held by another in a cycle — total halt, nothing executes (see Q7's account-transfer example).
- **Livelock**: threads are *not* blocked — they're actively running and responding to each other — but their combined behavior never converges to progress. Classic example: two threads each detect a potential conflict and both "politely" back off and retry at the same time, repeatedly colliding again (like two people repeatedly stepping the same way to avoid each other in a hallway). Livelock often results from overly-eager deadlock-avoidance logic.
- **Starvation**: a thread *can* make progress in principle, but is perpetually denied the resources/CPU time/lock it needs because other threads are unfairly favored — e.g. a non-fair lock repeatedly "barges" ahead of a long-waiting thread, or a low-priority thread never gets scheduled because higher-priority threads keep the CPU busy.

Prevention differs accordingly: deadlock via lock ordering/timeouts (Q7); livelock via randomized backoff (jitter) instead of deterministic simultaneous retry; starvation via fairness policies (e.g. `new ReentrantLock(true)`) or priority/queue design that guarantees eventual service.

**Code example:**
```java
// Starvation risk: default ReentrantLock is unfair (better throughput,
// but a waiting thread has no guarantee of eventually winning the race)
Lock unfair = new ReentrantLock();

// Fair variant: FIFO granting avoids starving long-waiting threads,
// at a real throughput cost under contention
Lock fair = new ReentrantLock(true);
```

**Follow-up question:**
Why does using a fair lock everywhere "by default" hurt overall system throughput, and when is that trade-off actually worth it?

**Glossary:**
- **Deadlock** — permanent mutual blocking via circular resource waits.
- **Livelock** — active but non-progressing thread interaction.
- **Starvation** — a thread indefinitely denied resources due to unfair scheduling/contention resolution.

**Mental model:**
Tests precise vocabulary and conceptual boundaries between related failure modes — interviewers use this to check whether "deadlock" is being used as a catch-all buzzword or the candidate actually distinguishes the failure mechanisms.

**References:**
- [Liveness — Java Tutorials (Deadlock, Starvation and Livelock)](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)
- [ReentrantLock javadoc — fairness policy (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
