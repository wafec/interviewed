---
layout: default
title: "Python Interview — Decorators, Generators & Context Managers"
---

# Python Interview — Decorators, Generators & Context Managers

This set covers three of Python's signature control-flow features: decorators (function transformation via `@` syntax), generators (lazy, suspendable iteration via `yield`), and context managers (deterministic resource cleanup via `with`). Expect fundamentals, what's actually happening under the hood (closures, frame suspension, the `__enter__`/`__exit__` protocol), common footguns, and the modern async variants of all three.

### Q1. What is a decorator in Python, and what does `@decorator` actually do to the function below it? {#q1}

**Question:**
What is a decorator in Python, and what does `@decorator` actually do to the function below it?

**Good answer:**
A decorator is a callable that takes a function (or class) and returns a replacement — usually a wrapper function with extra behavior. `@decorator` above `def f(): ...` is pure syntactic sugar: it's exactly equivalent to writing `f = decorator(f)` after the function is defined. Nothing magical happens at the language level beyond "call this function with that function as an argument, and rebind the name to whatever comes back."

**Code example:**
```python
def shout(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs).upper()
    return wrapper

@shout
def greet(name):
    return f"hello, {name}"

# equivalent to: greet = shout(greet)
print(greet("wallace"))  # "HELLO, WALLACE"
```

**Follow-up question:**
Does the decorator run when the function is defined, or when it's called?

**Follow-up good answer:**
The decorator itself (`shout(greet)`) runs immediately at *definition* time, once, as soon as Python executes the `def greet` statement — that's when `greet` gets rebound to `wrapper`. The *wrapper's* body only runs later, each time you actually call `greet(...)`. This is a common confusion: people expect decoration to be lazy, but the "wrapping" happens eagerly at import/definition time; only the wrapped behavior is deferred to call time.

**Glossary:**
- **Decorator** — a callable that takes a function/class and returns a (usually different) callable, applied via `@` syntax.
- **Higher-order function** — a function that takes another function as an argument and/or returns one.

**Mental model:**
Tests whether the candidate sees through the `@` syntax to the plain function-call-and-rebind semantics underneath, rather than treating decorators as a mysterious separate language feature.

**TL;DR:**
`@decorator` above a function is just sugar for `f = decorator(f)`, executed once at definition time.

**References:**
- [Python Glossary — decorator](https://docs.python.org/3/glossary.html#term-decorator)
- [PEP 318 — Decorators for Functions and Methods](https://peps.python.org/pep-0318/)

---

### Q2. How does a decorator "remember" values from its enclosing scope across multiple calls to the wrapped function? {#q2}

**Question:**
How does a decorator "remember" values from its enclosing scope across multiple calls to the wrapped function?

**Good answer:**
Via closures. The inner `wrapper` function references names (like the original `func`, or any state such as a call counter) from the enclosing decorator function's scope. Python keeps those variables alive in "cells" — one per free variable — attached to the function object's `__closure__` tuple, rather than letting them be garbage-collected when the outer function returns. Every call to `wrapper` reads/writes through the same cell, which is exactly how something like a call-counting decorator can persist state between calls without using a global or a class.

**Code example:**
```python
def count_calls(func):
    calls = 0
    def wrapper(*args, **kwargs):
        nonlocal calls
        calls += 1
        print(f"call #{calls}")
        return func(*args, **kwargs)
    return wrapper

@count_calls
def ping():
    return "pong"

ping(); ping()  # call #1, call #2 — `calls` persists via the closure cell
```

**Follow-up question:**
Why is `nonlocal` required to mutate `calls` inside `wrapper`, but not required just to read `func`?

**Follow-up good answer:**
Reading a free variable from an enclosing scope works automatically — Python resolves the name via the closure. But *assigning* to a name inside a nested function makes Python treat it as local to that function by default (to avoid ambiguity), which would create a new local `calls` shadowing the outer one instead of mutating it. `nonlocal calls` tells Python explicitly "don't make a new local — bind this name to the existing variable in the nearest enclosing (non-global) scope," so `calls += 1` mutates the shared cell instead.

**Glossary:**
- **Closure** — a function object that retains references to variables from its enclosing lexical scope after that scope has exited.
- **Cell variable** — the mechanism CPython uses to keep a shared, mutable binding for a variable referenced by both an outer and inner function.
- **`nonlocal`** — a statement that binds an assignment target to a variable in the nearest enclosing function scope, instead of creating a new local.

**Mental model:**
Probes real understanding of Python's scoping rules (LEGB) and closures, not just "decorators use closures" as a memorized fact.

**TL;DR:**
Closures keep enclosing-scope variables alive via closure cells; `nonlocal` is required only when *assigning* to one, because assignment defaults to creating a new local.

**References:**
- [Python Language Reference — Naming and binding, nonlocal](https://docs.python.org/3/reference/executionmodel.html#resolution-of-names)
- [Python Data Model — `__closure__`](https://docs.python.org/3/reference/datamodel.html)

---

### Q3. What breaks if you write a decorator without `functools.wraps`, and why? {#q3}

**Question:**
What breaks if you write a decorator without `functools.wraps`, and why?

**Good answer:**
Without `functools.wraps`, the decorated function's identity gets replaced by the wrapper's: `func.__name__` becomes `"wrapper"` instead of the original name, `func.__doc__` is lost (`None`), `__module__`/`__qualname__`/`__annotations__` point at the wrapper, not the original. This breaks introspection-dependent tooling — debuggers, `help()`, Sphinx-style doc generation, and any framework that inspects `__name__` (e.g., some routing/DI frameworks key behavior off the function's name). `@functools.wraps(func)` applied to the wrapper copies `__module__`, `__name__`, `__qualname__`, `__annotations__`, `__doc__` from `func` onto `wrapper`, updates `wrapper.__dict__`, and sets `wrapper.__wrapped__ = func` so the original is still reachable.

**Code example:**
```python
import functools

def logged(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logged
def add(a, b):
    """Add two numbers."""
    return a + b

print(add.__name__, add.__doc__)  # "add" "Add two numbers." — preserved
```

**Follow-up question:**
How would you get back a reference to the *original*, undecorated function if you needed to bypass the decorator?

**Follow-up good answer:**
`functools.wraps` sets `wrapper.__wrapped__ = func` automatically, so `add.__wrapped__` gives you the original, undecorated `add`. This is exactly how `inspect.signature()` and `inspect.unwrap()` "see through" decorators to report the original function's real signature — they follow the `__wrapped__` chain.

**Glossary:**
- **`functools.wraps`** — a decorator that copies identity/metadata attributes from a wrapped function onto its wrapper.
- **`__wrapped__`** — an attribute set by `functools.wraps` pointing back to the original function.

**Mental model:**
Checks whether the candidate has actually been bitten by broken introspection in production (stack traces showing `wrapper` everywhere, docs tools failing) rather than just reciting "always use wraps."

**TL;DR:**
Skipping `functools.wraps` silently replaces the wrapped function's `__name__`/`__doc__`/etc. with the wrapper's, breaking introspection; `wraps` fixes this and adds `__wrapped__` back to the original.

**References:**
- [functools.wraps](https://docs.python.org/3/library/functools.html#functools.wraps)
- [functools.update_wrapper](https://docs.python.org/3/library/functools.html#functools.update_wrapper)

---

### Q4. How do you write a decorator that itself takes arguments, e.g. `@retry(times=3)`? {#q4}

**Question:**
How do you write a decorator that itself takes arguments, e.g. `@retry(times=3)`?

**Good answer:**
You need an extra level of nesting: an outer function that takes the decorator's own arguments (`times=3`) and *returns* the actual decorator, which in turn takes the function and returns the wrapper. `@retry(times=3)` first calls `retry(times=3)`, which returns a decorator function; that decorator is then applied to the target function, exactly as in the simple case. This is often called a "decorator factory."

**Code example:**
```python
import functools

def retry(times=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for _ in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exc = e
            raise last_exc
        return wrapper
    return decorator

@retry(times=5)
def flaky_call():
    ...
```

**Follow-up question:**
Can you make the decorator work both as `@retry` (no parens, using defaults) and `@retry(times=5)` (with arguments)?

**Follow-up good answer:**
Yes, but it requires detecting at call time whether the decorator was invoked with a single callable argument (bare `@retry`) or with keyword/other arguments (`@retry(times=5)`), typically by checking `len(args) == 1 and callable(args[0]) and not kwargs`. Libraries commonly implement this with a small dispatch at the top of the outer function, or by requiring all-keyword arguments for the parameterized form to make the check unambiguous. It adds real complexity, which is why many codebases simply require the parentheses always (`@retry()`) rather than supporting both forms.

**Glossary:**
- **Decorator factory** — a function that returns a decorator, used to make a decorator itself configurable via arguments.

**Mental model:**
Tests comfort with an extra layer of function nesting/currying, which is where many candidates who only know the basic decorator pattern start to struggle.

**TL;DR:**
A decorator that takes arguments is a function returning a function returning a function: outer args → decorator → wrapper.

**References:**
- [Python Glossary — decorator](https://docs.python.org/3/glossary.html#term-decorator)
- [PEP 318 — Decorators for Functions and Methods](https://peps.python.org/pep-0318/)

---

### Q5. Can a class be used as a decorator instead of a function? {#q5}

**Question:**
Can a class be used as a decorator instead of a function? How would that work?

**Good answer:**
Yes — anything callable can be a decorator, and a class instance is callable if the class defines `__call__`. A common pattern: `__init__` receives the function being decorated (and stores it, plus any state), and `__call__` implements the wrapper behavior, calling the stored function and adding logic before/after. This is useful when the decorator needs richer state or its own methods (e.g., a `.cache_clear()`-style method) rather than just closure variables.

**Code example:**
```python
import functools

class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        return self.func(*args, **kwargs)

@CountCalls
def ping():
    return "pong"

ping(); ping()
print(ping.calls)  # 2 — state lives on the instance, easy to inspect
```

**Follow-up question:**
If `CountCalls` decorates an instance *method* instead of a plain function, what goes wrong?

**Follow-up good answer:**
Because `CountCalls` is a class, not a function, its instances don't implement the descriptor protocol (`__get__`) the way plain functions do, so `self` won't be bound automatically when the method is looked up on an instance — the decorated "method" is just a `CountCalls` instance attribute on the class, and calling `obj.ping()` won't pass `obj` as `self` to the wrapped function correctly. Fixing this requires the decorator class to also implement `__get__` (making it a non-data descriptor that returns a bound-method-like callable), which is exactly why function-based decorators (which are already descriptors) are usually simpler for decorating methods.

**Glossary:**
- **`__call__`** — the dunder method that makes an object callable like a function.
- **Descriptor** — an object implementing `__get__`/`__set__`/`__delete__` that customizes attribute access; plain functions are non-data descriptors, which is how instance methods get `self` bound automatically.

**Mental model:**
Distinguishes candidates who understand decorators as "any callable" from those who've only seen the function-closure pattern, and probes deeper descriptor-protocol knowledge via the method-decoration edge case.

**TL;DR:**
A class with `__call__` can be a decorator, but decorating methods with a class-based decorator requires also implementing `__get__` to get correct `self` binding.

**References:**
- [Python Data Model — `object.__call__`](https://docs.python.org/3/reference/datamodel.html#object.__call__)
- [Python Data Model — Implementing Descriptors](https://docs.python.org/3/reference/datamodel.html#implementing-descriptors)

---

### Q6. If you stack two decorators, `@a` above `@b` above `def f(): ...`, what order do they run in? {#q6}

**Question:**
If you stack two decorators, `@a` above `@b` above `def f(): ...`, what order do they run in?

**Good answer:**
Decorators apply bottom-up but execute outside-in. Application order: `f = a(b(f))` — `b` wraps the original `f` first, then `a` wraps the result of that. So at *call* time, `a`'s wrapper logic runs first (it's the outermost layer), then it calls into `b`'s wrapper, which finally calls the original `f`. A common mnemonic: reading the stack of decorators bottom-to-top tells you the wrapping/application order; the *outermost* decorator (topmost in the source) is what actually executes first when you call the function.

**Code example:**
```python
def a(func):
    def wrapper(*a, **k):
        print("a: before")
        result = func(*a, **k)
        print("a: after")
        return result
    return wrapper

def b(func):
    def wrapper(*a, **k):
        print("b: before")
        result = func(*a, **k)
        print("b: after")
        return result
    return wrapper

@a
@b
def f():
    print("f")

f()
# a: before
# b: before
# f
# b: after
# a: after
```

**Follow-up question:**
Does decorator order matter for something like `@app.route(...)` combined with `@login_required` in a typical web framework — and if so, which should go on top?

**Follow-up good answer:**
Yes, it matters a lot in practice. Convention in most frameworks (e.g. Flask) is to put the routing decorator (`@app.route(...)`) *closest to the function* is wrong — actually the framework-specific route decorator is typically placed on top (outermost), with auth decorators like `@login_required` placed directly above the function (innermost, applied first), so that `login_required` wraps the raw view function and the routing decorator wraps that already-protected callable. Getting the order backwards can mean the route gets registered against the *unprotected* function if the auth decorator doesn't properly propagate metadata/behavior, effectively bypassing the check for some frameworks' introspection. The safe habit: read your specific framework's documented example order rather than assuming, since the failure mode (auth silently skipped) is a real security bug, not just a style nit.

**Glossary:**
- **Decorator stacking** — applying multiple decorators to one function; equivalent to nested function calls.

**Mental model:**
Tests whether the candidate can trace nested function application under pressure — a very common source of subtle bugs (wrong decorator order silently changing behavior, e.g. auth vs. logging vs. transaction boundaries).

**TL;DR:**
Stacked decorators apply bottom-up (`a(b(f))`) but execute outside-in at call time — the topmost decorator's code runs first.

**References:**
- [Python Glossary — decorator](https://docs.python.org/3/glossary.html#term-decorator)

---

### Q7. What actually happens when Python calls a function that contains a `yield` statement? {#q7}

**Question:**
What actually happens when Python calls a function that contains a `yield` statement?

**Good answer:**
Calling a generator function does *not* run its body at all — it immediately returns a generator-iterator object. The body only starts executing when you call `next()` (or `.send(None)`, or iterate it in a `for` loop) on that generator. Execution then proceeds until the first `yield`, at which point the value is handed back to the caller and the function's entire state — local variables, the instruction pointer, the evaluation stack, and any active `try` blocks — is frozen in place. Calling `next()` again resumes execution exactly where it left off, right after that `yield` expression, rather than starting over.

**Code example:**
```python
def counter():
    print("starting")
    n = 0
    while True:
        yield n
        n += 1

gen = counter()          # nothing printed yet — no code has run
print(next(gen))         # prints "starting", then yields 0
print(next(gen))         # resumes after yield, prints nothing, yields 1
```

**Follow-up question:**
What's the difference between what `__next__()` and `send(value)` do when resuming a paused generator?

**Follow-up good answer:**
Both resume execution at the point of the last `yield`, but they differ in what that `yield` *expression* evaluates to once resumed: `__next__()` (called implicitly by `for` loops and `next()`) makes the paused `yield` expression evaluate to `None`, while `send(value)` makes it evaluate to `value`. This is what lets generators act as simple coroutines — `x = yield foo` lets a caller push a value back in via `.send(x)`, whereas plain iteration with `next()`/`for` just drives the generator forward without feeding anything back into it.

**Glossary:**
- **Generator function** — a function containing `yield`, which returns a generator-iterator instead of executing immediately when called.
- **Generator-iterator** — the object returned by calling a generator function; implements the iterator protocol plus `send()`/`throw()`/`close()`.

**Mental model:**
Distinguishes candidates who think of generators as "just lazy lists" from those who understand the actual suspend/resume execution model, which matters for anything beyond trivial `for x in gen()` usage (coroutine-style patterns, `contextlib.contextmanager`, async generators).

**TL;DR:**
Calling a generator function returns a paused generator-iterator immediately; the body only runs (and freezes/resumes) as `next()`/`send()` drive it forward from each `yield`.

**References:**
- [Python Glossary — generator iterator](https://docs.python.org/3/glossary.html#term-generator-iterator)
- [Python Language Reference — Yield expressions](https://docs.python.org/3/reference/expressions.html#yield-expressions)

---

### Q8. What happens when a generator finishes without hitting another `yield` — how does the caller know it's done? {#q8}

**Question:**
What happens when a generator finishes without hitting another `yield` — how does the caller know it's done?

**Good answer:**
When a generator's code runs to completion (falls off the end, or hits a `return`), the generator-iterator raises `StopIteration` instead of returning normally. If the generator used a bare `return` with no value, `StopIteration` carries no payload; if it did `return some_value`, that value is attached as `StopIteration.value` — which is exactly the mechanism `yield from` uses to let a delegating generator retrieve the sub-generator's "return value." `for` loops and comprehensions handle `StopIteration` for you automatically (that's literally how `for` loop termination over any iterator works); calling `next()` manually past exhaustion means you need to catch `StopIteration` yourself, and any further calls to `next()`/`send()` on an already-exhausted generator keep raising it.

**Code example:**
```python
def one_two():
    yield 1
    yield 2
    return "done"

gen = one_two()
print(next(gen))  # 1
print(next(gen))  # 2
try:
    next(gen)
except StopIteration as e:
    print(e.value)  # "done"
```

**Follow-up question:**
Why is it a `RuntimeError` (not `StopIteration`) if a `StopIteration` accidentally escapes from *inside* a generator body in modern Python?

**Follow-up good answer:**
Before PEP 479 (enforced by default since Python 3.7), an accidental unhandled `StopIteration` raised inside a generator's body (e.g., from calling `next()` on an inner iterator without catching it) would silently propagate out and be indistinguishable from the generator legitimately finishing — a `for` loop consuming that generator would just quietly stop, hiding what was actually a bug. PEP 479 changed this: if a `StopIteration` bubbles up uncaught from inside a generator's frame, Python now converts it into a `RuntimeError`, so the bug becomes loud and visible instead of masquerading as normal generator exhaustion.

**Glossary:**
- **`StopIteration`** — the exception a generator/iterator raises to signal it has no more values.
- **PEP 479** — the change that turns an unhandled `StopIteration` inside a generator into a `RuntimeError`, to prevent it from being mistaken for normal termination.

**Mental model:**
Tests whether the candidate has actually debugged a "generator stopped early for no reason" bug — a classic PEP 479 motivating scenario — versus only knowing the happy-path iteration protocol.

**TL;DR:**
A generator signals completion by raising `StopIteration` (carrying its return value, if any); an *accidental* unhandled `StopIteration` from inside the generator is converted to `RuntimeError` since PEP 479 so it can't be silently mistaken for normal exhaustion.

**References:**
- [Python Language Reference — Yield expressions (StopIteration and yield from)](https://docs.python.org/3/reference/expressions.html#yield-expressions)
- [PEP 479 — Change StopIteration handling inside generators](https://peps.python.org/pep-0479/)

---

### Q9. What does `yield from subgenerator()` do that a plain `for x in subgenerator(): yield x` loop doesn't? {#q9}

**Question:**
What does `yield from subgenerator()` do that a plain `for x in subgenerator(): yield x` loop doesn't?

**Good answer:**
Both forward the values yielded by the subgenerator to the caller, but `yield from` additionally sets up full bidirectional delegation: any value sent into the outer generator via `.send(value)` is forwarded down into the subgenerator's own `send()`, and any exception thrown in via `.throw()` is forwarded into the subgenerator's `throw()` (or, if the subgenerator doesn't handle it, propagated appropriately) — a plain `for`/`yield` loop does neither, so `send()`/`throw()` calls on the outer generator would be silently swallowed or misdirected rather than reaching the subgenerator. `yield from` also captures the subgenerator's return value as the value of the `yield from` expression itself (via `StopIteration.value`), which a manual loop doesn't give you for free.

**Code example:**
```python
def inner():
    x = yield "start"
    yield f"got {x}"

def outer():
    result = yield from inner()
    yield f"inner said: {result}"

g = outer()
print(next(g))        # "start"
print(g.send("hi"))   # "got hi"
```

**Follow-up question:**
Why was `yield from` needed at all — what problem in delegating generators existed before PEP 380?

**Follow-up good answer:**
Before PEP 380 (Python 3.3), manually delegating to a subgenerator with a `for`/`yield` loop required hand-writing the forwarding logic for `send()` and `throw()` yourself if you wanted full coroutine-style two-way communication to pass through the delegation layer correctly — easy to get subtly wrong (e.g., forgetting to also forward `close()`/`GeneratorExit` correctly on early termination). `yield from` was introduced specifically to make that delegation "transparent" — the caller of the outer generator can interact with it exactly as if it were talking directly to the innermost subgenerator, with all three of iteration, `send()`, and `throw()`/`close()` correctly relayed.

**Glossary:**
- **`yield from`** — an expression that delegates iteration, `send()`, and `throw()`/`close()` to a subiterator/subgenerator.
- **PEP 380** — the proposal that introduced `yield from` for subgenerator delegation.

**Mental model:**
Separates candidates who've only used `yield from` for flattening iterables from those who understand it's really about full coroutine-style delegation (send/throw included), which matters for anyone who's touched pre-`async`/`await` coroutine code (e.g. older `asyncio`/`tornado` codebases).

**TL;DR:**
`yield from` isn't just value-forwarding sugar — it also transparently delegates `send()`/`throw()`/`close()` to the subgenerator and surfaces its return value, none of which a manual `for`/`yield` loop does.

**References:**
- [Python Language Reference — Yield expressions (yield from)](https://docs.python.org/3/reference/expressions.html#yield-expressions)
- [PEP 380 — Syntax for Delegating to a Subgenerator](https://peps.python.org/pep-0380/)

---

### Q10. Why does iterating over the same generator object a second time produce nothing? {#q10}

**Question:**
Why does iterating over the same generator object a second time produce nothing?

**Good answer:**
A generator-iterator is single-use and stateful: once it has run to completion (raised `StopIteration`), its internal frame is gone — there's no code left to re-run, no state to "rewind" to the start. Calling `next()` (or looping) on an already-exhausted generator just raises `StopIteration` again immediately, every time. This is different from something like a list, which you can iterate as many times as you like because `iter(list)` produces a fresh iterator each time; a generator function has to be *called again* to get a new, independent generator-iterator if you want to iterate the sequence from scratch.

**Code example:**
```python
def gen():
    yield 1
    yield 2

g = gen()
print(list(g))  # [1, 2]
print(list(g))  # [] — already exhausted, nothing left

# to iterate again, call the generator function again:
print(list(gen()))  # [1, 2]
```

**Follow-up question:**
If a function takes a generator as an argument and needs to iterate over it twice internally, what should it do?

**Follow-up good answer:**
Either require the caller to pass something re-iterable (like a list, or a factory function/generator function it can call again each time), or the function itself should materialize the generator into a list/tuple once at the top (`items = list(gen)`) if the full sequence is known to fit in memory, then iterate that list as many times as needed. There's no way to "reset" an exhausted generator-iterator in place — the fix is architectural (don't rely on re-iterating a one-shot iterator), not a method call.

**Glossary:**
- **Iterator** — an object with `__next__()` that is generally single-pass/exhaustible.
- **Iterable** — an object with `__iter__()` that can produce a *new* iterator each time, allowing repeated iteration.

**Mental model:**
A very common real-world bug source (e.g. accidentally consuming a generator in a debug `print(list(gen))` before the "real" consumption happens, leaving nothing for the actual logic) — tests whether the candidate has hit this in practice.

**TL;DR:**
Generators are single-use iterators, not re-iterable sequences — once exhausted they always raise `StopIteration`; getting fresh values means calling the generator function again.

**References:**
- [Python Glossary — generator iterator](https://docs.python.org/3/glossary.html#term-generator-iterator)
- [Python Glossary — iterator](https://docs.python.org/3/glossary.html#term-iterator)

---

### Q11. When would using a generator instead of building a list actually matter for performance? {#q11}

**Question:**
When would using a generator instead of building a list actually matter for performance?

**Good answer:**
Mainly memory, not raw CPU speed. A list comprehension over a large or unbounded source builds the *entire* sequence in memory before you can use any of it; a generator expression/function produces one item at a time, using roughly constant memory regardless of how many items the source has — so processing a huge file line-by-line, or an unbounded stream, is only practical with a generator (a list version might exhaust memory or simply never finish if the source is infinite). There's also a latency benefit for pipelines: a generator lets the first result flow through as soon as it's produced, instead of waiting for the whole list to be built first. It's not generally about the per-item CPU cost of `yield` vs. list appends — generator overhead per item is comparable to (or sometimes slightly more than) list iteration; the win is memory footprint and streaming behavior, which matters enormously once "large or unbounded input" is in play.

**Code example:**
```python
# builds the entire 10M-line list in memory first
lines = [line.strip() for line in open("huge.log")]

# processes one line at a time, ~constant memory
for line in (line.strip() for line in open("huge.log")):
    process(line)
```

**Follow-up question:**
How would you actually measure whether swapping a list comprehension for a generator helped in a specific case?

**Follow-up good answer:**
For memory, use `tracemalloc` or a process-level memory profiler (e.g. `memory_profiler`, or just watching RSS via `psutil`/the OS) around the two versions with a realistically large input, comparing peak memory rather than final memory. For latency/throughput in a pipeline, measure time-to-first-result and total wall-clock time, since a generator's win there is about when data becomes available, not total CPU work — a naive `timeit` comparing `sum(list_comp)` vs `sum(gen_expr)` for a fully-consumed, in-memory-sized dataset will often show generators as roughly the same speed or marginally slower per item, which is the correct (if initially surprising) result: the benefit is memory/streaming, not "generators are faster."

**Glossary:**
- **Generator expression** — a lazily-evaluated comprehension-like expression using `()` instead of `[]`, producing an iterator.
- **`tracemalloc`** — the standard-library module for tracing Python memory allocations.

**Mental model:**
Checks that the candidate doesn't overstate generators as a general "performance" trick — the real, defensible claim is memory/streaming, and a good answer should resist the urge to claim generators are simply "faster."

**TL;DR:**
Generators win on memory (constant footprint) and streaming latency for large/unbounded data, not on raw per-item CPU speed versus an equivalent list.

**References:**
- [Python HOWTO — Functional Programming HOWTO, generator expressions](https://docs.python.org/3/howto/functional.html)
- [tracemalloc — Trace memory allocations](https://docs.python.org/3/library/tracemalloc.html)

---

### Q12. How would you tell whether `functools.lru_cache` is actually helping a specific function in production? {#q12}

**Question:**
How would you tell whether `functools.lru_cache` is actually helping a specific function in production?

**Good answer:**
Call `.cache_info()` on the decorated function — it returns a named tuple with `hits`, `misses`, `maxsize`, and `currsize`. A high hit ratio (`hits / (hits + misses)`) confirms the cache is earning its keep; a low ratio suggests either the arguments are too varied to repeat (cache mostly misses, wasted memory/bookkeeping overhead) or `maxsize` is too small and useful entries are being evicted before they'd be reused (LRU eviction under `maxsize` pressure). You'd typically sample `cache_info()` periodically or at shutdown, and consider bumping `maxsize` (or switching to `maxsize=None` for unbounded caching, e.g. in memoized recursive algorithms) if eviction is the bottleneck, or removing the cache entirely if hits stay near zero.

**Code example:**
```python
from functools import lru_cache

@lru_cache(maxsize=256)
def expensive(x):
    ...

# later, in production diagnostics:
info = expensive.cache_info()
print(info)  # CacheInfo(hits=8341, misses=212, maxsize=256, currsize=256)
```

**Follow-up question:**
Why should you be careful applying `lru_cache` to a method on a class instance, or to a function with side effects?

**Follow-up good answer:**
On an instance method, `lru_cache` keys the cache on *all* positional/keyword arguments including `self` — since the cache holds a reference to `self` as a cache key, this keeps every instance that's ever called the cached method alive for as long as the cache entry exists, which can create a memory leak (objects that should have been garbage-collected are pinned by the cache). For side-effecting or non-pure functions (anything depending on mutable state, current time, randomness, or I/O), caching means later calls silently return a stale result instead of re-executing — correct only if the function's output genuinely depends solely on its arguments.

**Glossary:**
- **`cache_info()`** — the `lru_cache` introspection method returning hit/miss/size stats.
- **Pure function** — a function whose output depends only on its inputs and has no observable side effects, the precondition for safe caching.

**Mental model:**
Tests whether the candidate treats caching as "free" or understands its real costs (memory pinning via `self`, correctness risk on impure functions) — a frequent source of subtle production bugs and leaks.

**TL;DR:**
Use `cache_info()`'s hit/miss ratio to validate `lru_cache` is paying off, and watch for two hazards: pinning `self` alive on cached instance methods, and silently stale results on non-pure functions.

**References:**
- [functools.lru_cache](https://docs.python.org/3/library/functools.html#functools.lru_cache)

---

### Q13. What is a context manager, and what two methods does an object need to work with `with`? {#q13}

**Question:**
What is a context manager, and what two methods does an object need to work with `with`?

**Good answer:**
A context manager is any object implementing the context management protocol: `__enter__(self)` and `__exit__(self, exc_type, exc_val, exc_tb)`. `with obj as x:` calls `obj.__enter__()` at the start of the block, binding its return value to `x`; when the block exits — whether normally or via an exception — `obj.__exit__(...)` is called with the exception info (or `(None, None, None)` if no exception occurred). This guarantees cleanup code runs even if the block raises, which is the whole point: acquiring a resource in `__enter__` and reliably releasing it in `__exit__` regardless of how the block exits.

**Code example:**
```python
class Timer:
    def __enter__(self):
        import time
        self.start = time.perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        print(f"elapsed: {time.perf_counter() - self.start:.3f}s")
        return False  # don't suppress exceptions

with Timer():
    do_work()
```

**Follow-up question:**
What does the return value of `__exit__` control?

**Follow-up good answer:**
If `__exit__` returns a truthy value (e.g. `True`), any exception that occurred in the `with` block is *suppressed* — execution continues after the `with` statement as if nothing happened. If it returns a falsy value (including the implicit `None` from a function with no explicit `return`), the exception (if any) propagates normally after `__exit__` finishes. This is deliberate: `__exit__` can selectively swallow specific exception types (e.g. by checking `exc_type`) while letting others propagate, but a lazily-written `__exit__` that returns `True` unconditionally will silently eat *every* exception raised in the block — a dangerous footgun.

**Glossary:**
- **`__enter__`/`__exit__`** — the dunder methods implementing the context management protocol.
- **Context management protocol** — the informal name for the `__enter__`/`__exit__` interface consumed by `with`.

**Mental model:**
Fundamental protocol knowledge, but the follow-up on `__exit__`'s return value separates people who've memorized the method names from those who understand the exception-suppression footgun it enables.

**TL;DR:**
`with` calls `__enter__` on entry and always calls `__exit__` on exit (passing exception info if any); `__exit__` returning truthy suppresses that exception, falsy lets it propagate.

**References:**
- [Python Data Model — With Statement Context Managers](https://docs.python.org/3/reference/datamodel.html#context-managers)
- [PEP 343 — The "with" Statement](https://peps.python.org/pep-0343/)

---

### Q14. How does `@contextlib.contextmanager` let you write a context manager as a generator function instead of a class? {#q14}

**Question:**
How does `@contextlib.contextmanager` let you write a context manager as a generator function instead of a class?

**Good answer:**
`@contextmanager` wraps a generator function that must yield *exactly once*. Calling the decorated function returns a real context manager object; entering the `with` block runs the generator up to its `yield`, and the yielded value becomes the `as` target. Code after the `yield` (typically in a `finally`) runs when the `with` block exits. Under the hood, if an exception occurs inside the `with` block, `@contextmanager` re-raises it *inside the generator*, at the point of the `yield` — so a `try/except/finally` wrapped around the `yield` in the generator behaves like a real `__enter__`/`__exit__` pair, letting you write both halves of the resource lifecycle as one linear, readable function instead of a class with two separate methods.

**Code example:**
```python
from contextlib import contextmanager

@contextmanager
def managed_resource(name):
    print(f"acquiring {name}")
    resource = acquire(name)
    try:
        yield resource
    finally:
        print(f"releasing {name}")
        release(resource)

with managed_resource("db") as db:
    use(db)
```

**Follow-up question:**
Inside a `@contextmanager` generator, if you catch an exception from the `with` block but don't re-raise it, what happens — and why is that easy to get wrong?

**Follow-up good answer:**
Not re-raising it means the context manager *suppresses* the exception — execution resumes after the `with` block as if nothing went wrong, exactly like `__exit__` returning `True`. This is easy to get wrong because it's a very natural-looking mistake: writing `except SomeError as e: log(e)` around the `yield` (intending only to *log* the error) silently swallows it unless you add an explicit `raise` at the end of that `except` block — the official docs explicitly warn about this, since forgetting the re-raise is one of the most common `@contextmanager` bugs.

**Glossary:**
- **`contextlib.contextmanager`** — a decorator turning a single-yield generator function into a context manager.
- **Exception suppression** — an `__exit__` (or equivalent generator logic) preventing an exception from propagating past the `with` block.

**Mental model:**
Tests whether the candidate knows this specific, well-documented footgun — accidentally swallowing exceptions by forgetting to re-raise inside a `@contextmanager` generator's `except` clause.

**TL;DR:**
`@contextmanager` turns a single-yield generator into a context manager by resuming it past the `yield` on block exit and re-raising `with`-block exceptions at that `yield` point — catching without re-raising there silently suppresses the exception.

**References:**
- [contextlib.contextmanager](https://docs.python.org/3/library/contextlib.html#contextlib.contextmanager)

---

### Q15. You need to open a variable, data-dependent number of files and guarantee all of them get closed, even if one fails partway through. How? {#q15}

**Question:**
You need to open a variable, data-dependent number of files and guarantee all of them get closed, even if one fails partway through. How?

**Good answer:**
`contextlib.ExitStack` is built exactly for this: it lets you push an arbitrary, runtime-determined number of context managers (or plain cleanup callbacks) onto a stack, and guarantees they're all unwound — in reverse (LIFO) order — when the `ExitStack` itself exits, whether normally or via exception. You call `stack.enter_context(cm)` for each context manager as you discover it (e.g. in a loop over filenames); if opening the fifth file raises partway through, the first four that were already entered still get cleanly closed as the `ExitStack`'s own `__exit__` unwinds.

**Code example:**
```python
from contextlib import ExitStack

def process_all(filenames):
    with ExitStack() as stack:
        files = [stack.enter_context(open(fn)) for fn in filenames]
        # if opening file #5 raises, files 1-4 are still closed correctly
        for f in files:
            process(f)
```

**Follow-up question:**
Besides context managers, what else can you register on an `ExitStack`, and in what order do multiple registrations run relative to each other?

**Follow-up good answer:**
You can also register plain cleanup callbacks via `stack.callback(func, *args, **kwargs)` (or `stack.push(...)` for pre-built exit-like callables) — these don't need to be full context managers, just something to run on unwind. All registrations, whether `enter_context()` or `callback()`, share one stack and run in strict LIFO order — reverse of the order they were registered — exactly as if they'd been nested `with` statements in that reverse order. This also means an inner callback that suppresses or replaces an exception affects what outer callbacks see, mirroring how nested `with` blocks would behave.

**Glossary:**
- **`contextlib.ExitStack`** — a context manager for combining a dynamic number of other context managers/cleanup callbacks with guaranteed LIFO unwinding.
- **LIFO (last-in, first-out)** — the order in which `ExitStack` unwinds registered callbacks/context managers.

**Mental model:**
Tests knowledge of a less commonly known but genuinely useful standard-library tool for a real, recurring problem (dynamic resource lists) that a naive nested-`with` or manual `try/finally` approach handles poorly.

**TL;DR:**
`contextlib.ExitStack` lets you register a runtime-determined number of context managers/callbacks and guarantees they unwind in reverse (LIFO) order, even on partial failure.

**References:**
- [contextlib.ExitStack](https://docs.python.org/3/library/contextlib.html#contextlib.ExitStack)

---

### Q16. How does Python's context manager protocol relate to the classic "RAII" (Resource Acquisition Is Initialization) idea from C++? {#q16}

**Question:**
How does Python's context manager protocol relate to the classic "RAII" (Resource Acquisition Is Initialization) idea from C++?

**Good answer:**
Both solve the same problem — coupling a resource's lifetime to a scope so cleanup is automatic and exception-safe — but via different mechanisms. RAII in C++ ties cleanup to an object's *destructor*, run deterministically when the object goes out of scope (stack unwinding), with no separate "block" syntax needed. Python has no deterministic destructors in general (CPython's refcounting often frees objects promptly, but that's an implementation detail, not a language guarantee — and other implementations like PyPy don't behave that way), so Python instead makes the scope *explicit* via the `with` statement and the `__enter__`/`__exit__` protocol: cleanup is guaranteed by the `with` block's structure, not by watching an object's refcount hit zero. It's "RAII made explicit and syntactic" rather than "RAII via the object lifecycle."

**Follow-up question:**
Why can't you just rely on `__del__` (Python's closest thing to a destructor) for resource cleanup instead of `with`/`__exit__`?

**Follow-up good answer:**
`__del__` is called when an object is garbage-collected, but *when* that happens is not guaranteed by the language: CPython's reference counting usually collects unreferenced objects promptly, but reference cycles are only cleaned up by the periodic cyclic garbage collector (not immediately), and other Python implementations (PyPy, Jython) may defer collection arbitrarily. Relying on `__del__` for something time-sensitive — closing a file promptly, releasing a lock, committing/rolling back a transaction — can leave the resource held far longer than intended, or in pathological cases (e.g. exceptions inside `__del__`, or objects resurrected during finalization) not released in the way you'd expect at all. `with`/`__exit__` guarantees cleanup exactly at the end of the block, deterministically and immediately, regardless of the object's actual garbage-collection timing.

**Glossary:**
- **RAII** — "Resource Acquisition Is Initialization," a C++ idiom tying resource cleanup to object destruction.
- **`__del__`** — Python's finalizer method, called when an object is about to be destroyed, with timing that is not language-guaranteed.

**Mental model:**
Tests whether the candidate can connect a Python-specific idiom to broader software-engineering theory (deterministic resource management), and understands *why* Python's GC model makes `with` necessary rather than optional style preference.

**TL;DR:**
Context managers give Python RAII-like guaranteed cleanup, but via explicit `with`/`__exit__` scoping rather than C++-style deterministic destructors, because Python's garbage-collection timing (especially for cycles) isn't guaranteed.

**References:**
- [Python Data Model — With Statement Context Managers](https://docs.python.org/3/reference/datamodel.html#context-managers)
- [Python Data Model — `object.__del__`](https://docs.python.org/3/reference/datamodel.html#object.__del__)

---

### Q17. When would you reach for a context manager instead of a plain `try/finally`, given that they solve the same underlying problem? {#q17}

**Question:**
When would you reach for a context manager instead of a plain `try/finally`, given that they solve the same underlying problem?

**Good answer:**
`try/finally` is fine for a one-off, single-use cleanup inline in a function. A context manager earns its keep when the acquire/release pattern is reused across many call sites — packaging it once as a context manager (class or `@contextmanager` generator) means every caller just writes `with resource(...):` instead of repeating the same `try/finally` boilerplate, which also reduces the chance any individual call site forgets it or gets it slightly wrong. Context managers also compose more cleanly (nesting multiple `with` statements, or using `ExitStack` for a dynamic number of them) and integrate with tooling that expects the protocol (e.g. `contextlib.suppress`, `unittest.mock.patch` as a context manager). The underlying guarantee (cleanup on exception) is identical either way — it's a reuse/composability/readability trade-off, not a correctness one.

**Follow-up question:**
Is there a case where `try/finally` is actually *more* correct or safer than wrapping the same logic in a context manager?

**Follow-up good answer:**
Not more "correct" in terms of the cleanup guarantee — both are equally exception-safe. But `try/finally` can be clearer when the acquire/release logic is genuinely one-off and tightly coupled to the surrounding function's other state (extracting it into a separate context-manager object/generator would add indirection without any reuse benefit), or when the cleanup needs to inspect a lot of local state from the function that would be awkward to thread through `__enter__`/`__exit__` or a generator's closure. In short: reach for a context manager when there's reuse or composability to gain; a single well-placed `try/finally` is not a code smell on its own.

**Glossary:**
- **`try/finally`** — the base language construct guaranteeing cleanup code runs regardless of how a block exits; context managers are built on the same guarantee, packaged for reuse.

**Mental model:**
Software-engineering judgment question: tests whether the candidate treats context managers as always strictly "better" (cargo-culting) or understands the actual trade-off (reuse/composability vs. simplicity for one-off cases).

**TL;DR:**
Both guarantee cleanup identically; prefer a context manager when the acquire/release pattern is reused or composed across call sites, and a plain `try/finally` for genuinely one-off, tightly-coupled cleanup.

**References:**
- [Python Tutorial — Predefined Clean-up Actions (with vs try/finally)](https://docs.python.org/3/tutorial/errors.html#predefined-clean-up-actions)

---

### Q18. What's an async generator, and why couldn't you just put `yield` inside an `async def` function before Python 3.6? {#q18}

**Question:**
What's an async generator, and why couldn't you just put `yield` inside an `async def` function before Python 3.6?

**Good answer:**
An async generator is an `async def` function containing `yield`, consumed with `async for` instead of a plain `for`. It implements `__aiter__`/`__anext__` (the asynchronous iteration protocol) the same way a regular generator implements `__iter__`/`__next__`, letting you write a lazily-produced, `await`-capable stream of values — e.g. yielding rows as they arrive from an async database cursor, without buffering everything into a list first. Before PEP 525 (Python 3.6), combining `async def` with `yield` was a `SyntaxError` — PEP 492 (3.5) introduced native coroutines and `async`/`await` but explicitly disallowed `yield` inside them, since a coroutine (something you `await` once for a single result) and a generator (something you iterate for a stream of values) were kept as separate concepts until PEP 525 unified "stream of values" with "asynchronous."

**Code example:**
```python
async def stream_rows(cursor):
    async for row in cursor:
        yield transform(row)

async def consume():
    async for row in stream_rows(cursor):
        process(row)
```

**Follow-up question:**
Can you use `yield from` inside an async generator to delegate to another async generator, the way regular generators delegate with `yield from`?

**Follow-up good answer:**
No — `yield from` is not supported inside async generators (as of the current language spec); delegation has to be done manually with an `async for` loop that re-yields each item (`async for x in sub_agen(): yield x`), which — like the pre-PEP-380 situation for regular generators — doesn't automatically forward `asend()`/`athrow()` through to the delegated sub-generator the way `yield from` does for synchronous generators. This is a known asymmetry between sync and async generators that callers sometimes trip over, expecting `yield from` semantics to "just work" asynchronously.

**Glossary:**
- **Async generator** — an `async def` function containing `yield`, consumed via `async for`.
- **`__aiter__`/`__anext__`** — the asynchronous iteration protocol methods, analogous to `__iter__`/`__next__`.

**Mental model:**
Tests whether the candidate has actually written async streaming code (not just `async`/`await` for single-result coroutines), and whether they know the sync/async generator feature gap around `yield from`.

**TL;DR:**
Async generators (`async def` + `yield`, consumed via `async for`) were introduced in PEP 525 (3.6) because PEP 492's coroutines explicitly forbade `yield`; unlike sync generators, they have no `yield from` for sub-generator delegation.

**References:**
- [PEP 525 — Asynchronous Generators](https://peps.python.org/pep-0525/)
- [PEP 492 — Coroutines with async and await syntax](https://peps.python.org/pep-0492/)

---

### Q19. How does `async with` differ from a regular `with`, and what methods does an async context manager need? {#q19}

**Question:**
How does `async with` differ from a regular `with`, and what methods does an async context manager need?

**Good answer:**
An async context manager implements `__aenter__` and `__aexit__` instead of `__enter__`/`__exit__` — both are coroutine functions (defined with `async def`), so entering and exiting the block can itself `await` other coroutines (e.g. asynchronously acquiring a network connection or a lock without blocking the event loop). `async with cm as x:` awaits `cm.__aenter__()` for the enter value, and on exit awaits `cm.__aexit__(exc_type, exc_val, exc_tb)`, with the same exception-info-passing and suppress-via-truthy-return semantics as the synchronous protocol. You need `async with` specifically because a plain `with` would call synchronous `__enter__`/`__exit__`, which can't `await` anything — if the acquire/release logic needs to do async I/O, it has no choice but to be a coroutine, which only `__aenter__`/`__aexit__` support.

**Code example:**
```python
class AsyncConnection:
    async def __aenter__(self):
        self.conn = await connect_async()
        return self.conn

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.conn.close()
        return False

async def main():
    async with AsyncConnection() as conn:
        await conn.query(...)
```

**Follow-up question:**
Is there a decorator-based shortcut for writing async context managers, analogous to `@contextlib.contextmanager` for sync ones?

**Follow-up good answer:**
Yes — `contextlib.asynccontextmanager`, which does the async equivalent: it wraps a single-yield *async generator* function, and the resulting object works with `async with` the same way `@contextmanager`'s output works with plain `with`. The pattern is identical (acquire before `yield`, release in a `finally` after it, exceptions from the `async with` block re-raised at the `yield` point) — just with `async def`/`yield` and awaited I/O allowed in both halves.

**Glossary:**
- **`__aenter__`/`__aexit__`** — the coroutine-based dunder methods implementing the asynchronous context management protocol.
- **`contextlib.asynccontextmanager`** — the async analogue of `@contextmanager`, built on a single-yield async generator function.

**Mental model:**
Confirms the candidate can extend the sync context-manager mental model to the async world instead of treating `async with` as an unrelated new thing to memorize separately.

**TL;DR:**
`async with` uses awaitable `__aenter__`/`__aexit__` instead of synchronous `__enter__`/`__exit__`, needed whenever entering/exiting must itself perform async I/O; `contextlib.asynccontextmanager` mirrors `@contextmanager` for this case.

**References:**
- [contextlib.asynccontextmanager](https://docs.python.org/3/library/contextlib.html#contextlib.asynccontextmanager)
- [PEP 492 — Coroutines with async and await syntax](https://peps.python.org/pep-0492/)

---

### Q20. You need to add logging, timing, and retry-on-failure to a dozen unrelated functions across a codebase without duplicating that logic in each one. How do decorators solve this, and what's the trade-off versus baking that logic into each function directly? {#q20}

**Question:**
You need to add logging, timing, and retry-on-failure to a dozen unrelated functions across a codebase without duplicating that logic in each one. How do decorators solve this, and what's the trade-off versus baking that logic into each function directly?

**Good answer:**
This is the textbook case for the decorator pattern applied to cross-cutting concerns: logging, timing, and retry logic are orthogonal to what each function actually does, so writing them once as decorators (`@logged`, `@timed`, `@retry(times=3)`) and applying them declaratively lets every function stay focused on its own logic while still getting consistent, centrally-maintained behavior — fix a bug in the retry logic once, and every decorated function benefits immediately, instead of hunting down a dozen copy-pasted `try/except` blocks. The trade-off: stacking several decorators makes the *effective* behavior of a function harder to see at a glance from its own body (you have to mentally compose `@logged`, `@timed`, `@retry` in the right order to know what actually happens on a call), debugging/stack traces get an extra frame or two per decorator (mitigated but not eliminated by `functools.wraps`), and decorator order becomes a real correctness concern (e.g., does `@retry` wrap `@timed`, timing each individual retry attempt, or does `@timed` wrap `@retry`, timing the whole retry loop as one span? — those are different, both plausible, and easy to get backwards).

**Follow-up question:**
For something like "retry on failure," why might a dedicated library decorator (e.g. `tenacity.retry`) be preferable to writing your own from scratch?

**Follow-up good answer:**
A hand-rolled retry decorator tends to accumulate edge cases over time that are easy to get wrong on a first pass: exponential backoff with jitter (to avoid thundering-herd retries across many callers), which exceptions are worth retrying vs. which should fail fast, a maximum total wait time or attempt count, and correctly preserving the *original* exception/traceback if all retries are exhausted rather than masking it. A mature library like `tenacity` has already solved these edge cases, is tested against them, and lets you configure the policy declaratively (`@retry(stop=stop_after_attempt(3), wait=wait_exponential())`) instead of re-deriving backoff math and exception-filtering logic in-house — the classic build-vs-buy trade-off, where "buy" wins once the seemingly-simple utility (retry) turns out to have a surprising number of correctness-affecting details.

**Glossary:**
- **Cross-cutting concern** — behavior (logging, timing, auth, retries) that applies across many otherwise-unrelated parts of a codebase, a good fit for decorators/aspect-oriented patterns.
- **Decorator pattern (GoF)** — the general object-oriented design pattern of wrapping an object to add behavior without modifying its class; Python's `@` syntax is a lightweight, function-level realization of the same idea.

**Mental model:**
A synthesis question testing whether the candidate can connect the mechanical "how decorators work" knowledge from earlier questions to a real architectural decision, including honestly naming the readability/debuggability trade-offs rather than presenting decorators as strictly free wins.

**TL;DR:**
Decorators let you factor cross-cutting concerns (logging, timing, retry) out of individual functions into reusable, centrally-fixable units, at the cost of making a function's effective behavior less visible from its own body and making decorator *order* a real correctness concern.

**References:**
- [Python Glossary — decorator](https://docs.python.org/3/glossary.html#term-decorator)
- [functools.wraps](https://docs.python.org/3/library/functools.html#functools.wraps)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=decorators-generators-and-context-managers&autostart=1" | relative_url }})
