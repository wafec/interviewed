---
layout: default
title: "Python Interview — Core Language and Data Model"
---

# Python Interview — Core Language and Data Model

This set covers Python's object model: how names, assignment, and mutability
actually work; the data-model protocol (dunder methods) that powers
operator overloading, iteration, and context managers; attribute-lookup
internals (`__dict__`/`__slots__`/descriptors); multiple-inheritance
resolution; and the classic pitfalls (mutable defaults, `is` vs `==`,
shallow vs deep copy) that separate candidates who've internalized the
model from those who've memorized syntax.

### Q1. Is a Python variable a box that holds a value, or something else? {#q1}

**Question:**
Is a Python variable a box that holds a value, or something else?

**Good answer:**
A Python variable is a name bound to an object, not a box containing a
value. `x = 5` doesn't put `5` "into" `x` — it makes the name `x` in the
current namespace refer to an existing (in this case, cached) `int` object
`5`. Assignment is just name binding: `y = x` binds `y` to the *same
object* `x` refers to, it does not copy anything. Reassignment (`x = 6`)
just rebinds the name to a different object; it never mutates the object
`x` used to point to. This is why Python is often described as "pass by
object reference" rather than "pass by value" or "pass by reference" —
every variable, argument, list element, and dict value is a reference to
an object living on the heap.

**Code example:**
```python
x = [1, 2, 3]
y = x          # y is bound to the SAME list object as x
y.append(4)
print(x)       # [1, 2, 3, 4] — x sees the mutation, because x and y are the same object

x = "changed"  # this REBINDS x to a new string object; it doesn't touch the list
print(y)       # [1, 2, 3, 4] — y is unaffected, it was never tied to the name x
```

**Follow-up question:**
If that's true, why does `def f(x): x = x + 1` not change the caller's variable, but `def f(lst): lst.append(1)` does change the caller's list?

**Follow-up good answer:**
Both cases pass the same way — the parameter is bound to the same object
the caller's argument refers to. The difference is what happens *inside*
the function. `x = x + 1` creates a brand-new `int` object and rebinds the
local name `x` to it; the caller's name still points at the original
object, untouched. `lst.append(1)` doesn't rebind anything — it calls a
mutating method on the object itself, and since the local name `lst` and
the caller's name refer to the *same* list object, the caller observes the
mutation. The rule is: rebinding a name inside a function only affects that
function's local namespace; mutating the object a name points to affects
everyone holding a reference to that object.

**Glossary:**
- **Name binding** — associating a name in a namespace with an object; what `=` does.
- **Mutable/immutable** — whether an object's internal state can change after creation (lists/dicts/sets are mutable; int/str/tuple/frozenset are immutable).

**Mental model:**
Tests whether the candidate has the correct mental model of Python's
execution model (names/objects/namespaces) or is still thinking in
C-style "variables are memory slots holding values," which produces wrong
predictions about aliasing and function-argument behavior.

**TL;DR:**
Python variables are names bound to objects, not value-holding boxes — assignment binds/rebinds a name, it never copies; mutation vs. rebinding inside a function is what determines whether the caller sees the change.

**References:**
- [Python Language Reference §3.1, Objects, values and types](https://docs.python.org/3/reference/datamodel.html)
- [Python FAQ: How do I write a function with output parameters (call by reference)?](https://docs.python.org/3/faq/programming.html#how-do-i-write-a-function-with-output-parameters-call-by-reference)

---

### Q2. What's the practical difference between a mutable and an immutable type, beyond just "can it change"? {#q2}

**Question:**
What's the practical difference between a mutable and an immutable type, beyond just "can it change"?

**Good answer:**
Immutability guarantees an object's value can never change after
construction — every "modification" (e.g. `s = s + "x"` on a string)
actually builds and returns a new object, leaving the original untouched.
This has concrete consequences: immutable objects are safe to share freely
across functions/threads without defensive copying (no one can mutate your
copy out from under you), they can be used as dict keys / set members
(which requires a stable hash for the object's lifetime), and CPython can
safely cache/intern small immutable values (small ints, short strings)
since identical-looking instances are interchangeable. Mutable objects
(list, dict, set, and any custom class without `frozen`/immutability
discipline) can be changed in place, which is efficient (no realloc) but
means every alias to the object sees every mutation — the source of the
aliasing bugs in Q1's follow-up, and why they can't be dict keys (their
hash could change while they sit in the table, corrupting the hash
invariant).

**Follow-up question:**
Why exactly can't a list be used as a dictionary key, but a tuple can?

**Follow-up good answer:**
A dict/set relies on an object's `__hash__()` returning the same value for
the object's entire lifetime in the container — that's how it finds the
right bucket on lookup. `list` is mutable and deliberately does not define
`__hash__` (it's set to `None`, making lists explicitly unhashable) because
its contents — and therefore any hash you could compute from them — can
change after insertion, which would silently corrupt the dict's internal
bucket structure (the object would become unfindable at its original
bucket, or worse, collide incorrectly). `tuple` is immutable, so *if* all
its elements are themselves hashable, its hash can be computed once and is
guaranteed stable forever, making it a valid key. `hash((1,2))` works;
`hash((1,[2]))` raises `TypeError` because the nested list is unhashable.

**Glossary:**
- **Hashable** — implements `__hash__()` (and consistently `__eq__()`) such that the hash never changes while the object is in a hash-based container.
- **Interning** — CPython's implementation detail of reusing a single object for some small ints and string literals rather than allocating a new one each time.

**Mental model:**
Probes whether the candidate understands *why* the mutable/immutable split
exists mechanically (hash-table correctness, aliasing safety) rather than
reciting "lists are mutable, tuples aren't" as a memorized fact.

**TL;DR:**
Immutability buys hashability and safe sharing without defensive copies; lists can't be dict keys because a hash-based container needs an object's hash to stay constant for its lifetime in the table, and a mutable object can't promise that.

**References:**
- [Python Glossary: hashable](https://docs.python.org/3/glossary.html#term-hashable)
- [Python Data Model §3.3.1, object.__hash__()](https://docs.python.org/3/reference/datamodel.html#object.__hash__)

---

### Q3. What's the difference between `__new__` and `__init__`, and when would you actually need to override `__new__`? {#q3}

**Question:**
What's the difference between `__new__` and `__init__`, and when would you actually need to override `__new__`?

**Good answer:**
`__new__(cls, ...)` is a static method responsible for *creating* and
returning the new instance — it's called first, receives the class as its
first argument, and its return value becomes `self` for `__init__`.
`__init__(self, ...)` runs afterward, on the already-created instance, and
is responsible only for *initializing* its state; it must return `None`
(returning anything else raises `TypeError`). For an ordinary class you
never need to touch `__new__` — `object.__new__` allocates the instance
and `__init__` does the rest. You override `__new__` when you need control
over instance creation itself: subclassing an immutable built-in (`int`,
`str`, `tuple`) where the value must be fixed at creation time (since
`__init__` runs too late to change an already-immutable value), or
implementing patterns like a singleton or object pool/cache where you might
return an *existing* instance instead of a fresh one, or return an instance
of a different class entirely.

**Code example:**
```python
class PositiveInt(int):
    def __new__(cls, value):
        if value <= 0:
            raise ValueError("must be positive")
        return super().__new__(cls, value)  # int's value is fixed HERE, not in __init__

p = PositiveInt(5)   # works
PositiveInt(-1)      # raises ValueError, before any instance exists
```

**Follow-up question:**
If `__new__` returns an instance of a different, unrelated class, does `__init__` still run?

**Follow-up good answer:**
No. Python only calls `__init__` on the result of `__new__` if that result
is an instance of `cls` (or a subclass of it). If `__new__` returns an
object of an unrelated type, `__init__` is skipped entirely for that call —
the constructor call just returns the object `__new__` produced, uninitialized
by the class you "constructed." This is precisely how patterns like
returning a cached/pooled instance, or a completely different
representation, work without accidentally re-running initialization logic
on an object that's already fully set up.

**Glossary:**
- **`__new__`** — static method that allocates/creates the instance; called before `__init__`.
- **Immutable built-in subclassing** — subclassing `int`/`str`/`tuple` where the underlying value must be baked in at `__new__` time.

**Mental model:**
Tests whether the candidate understands object construction as a two-phase
process (create, then initialize) rather than treating `__init__` as "the
constructor" — a distinction that matters for immutable subclassing,
singletons, and metaclass work.

**TL;DR:**
`__new__` creates the instance and runs first; `__init__` only initializes an already-created instance and runs second (and only if `__new__` returned an instance of the right class) — override `__new__` only when creation itself needs control, like immutable subclassing or instance caching.

**References:**
- [Python Data Model §3.3.1, object.__new__()](https://docs.python.org/3/reference/datamodel.html#object.__new__)
- [Python Data Model §3.3.1, object.__init__()](https://docs.python.org/3/reference/datamodel.html#object.__init__)

---

### Q4. If you override `__eq__` on a class, why might instances suddenly become unusable as dict keys or set members? {#q4}

**Question:**
If you override `__eq__` on a class, why might instances suddenly become unusable as dict keys or set members?

**Good answer:**
Python enforces the invariant that objects which compare equal must have
the same hash — otherwise a hash-based container can't find a key it
already holds. To protect that invariant, Python's default behavior is:
as soon as a class defines `__eq__()` without also defining `__hash__()`,
that class's `__hash__` is implicitly set to `None`, making instances
explicitly unhashable (`TypeError: unhashable type` on `hash(obj)` or
inserting into a set/dict). This is deliberate — your custom `__eq__` might
compare objects in a way that's inconsistent with the inherited
`__hash__` (which by default hashes on identity), so Python refuses to
guess and instead forces you to explicitly opt back in.

**Code example:**
```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
    # no __hash__ defined -> __hash__ is now None

p = Point(1, 2)
hash(p)          # TypeError: unhashable type: 'Point'

class Point2:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))   # consistent with __eq__

{Point2(1, 2)}    # works fine
```

**Follow-up question:**
Is it ever correct to define `__hash__` on a class whose objects are mutable?

**Follow-up good answer:**
Generally no, and the documentation explicitly discourages it: if a class
defines mutable state and implements `__eq__`, it should not implement
`__hash__`, because a hashable collection requires a key's hash to stay
constant while it's stored — if you mutate the object's fields after
inserting it into a set/dict, its hash changes, and the container can no
longer find it in its own bucket (it appears "lost" even though `in`
checks against a freshly-constructed equal object might still pass by
accident, or fail entirely). The safe pattern is: mutable classes should
leave `__hash__` as `None` (the default once `__eq__` is defined) and
simply not be used as dict keys/set members; only make a type hashable
once it's genuinely immutable, or the hash is based on an immutable
identity field, or you can guarantee the hashed fields never change while
the object is stored.

**Glossary:**
- **Hash invariant** — `a == b` implies `hash(a) == hash(b)`, required for correct behavior in dict/set.
- **`__hash__ = None`** — Python's way of marking a type explicitly unhashable.

**Mental model:**
Tests whether the candidate understands the `__eq__`/`__hash__` contract as
a correctness requirement enforced by the language, not an arbitrary
restriction — and whether they know the implicit `__hash__ = None`
behavior, a frequent surprise in real codebases.

**TL;DR:**
Defining `__eq__` without `__hash__` makes instances unhashable by design, because Python can't assume your custom equality is still consistent with the inherited identity-based hash — only re-add `__hash__` (based on immutable fields) if you can guarantee the hash never changes while the object is stored in a set/dict.

**References:**
- [Python Data Model §3.3.1, object.__hash__()](https://docs.python.org/3/reference/datamodel.html#object.__hash__)

---

### Q5. What's the actual difference between `__repr__` and `__str__`, and which one does `print()` use? {#q5}

**Question:**
What's the actual difference between `__repr__` and `__str__`, and which one does `print()` use?

**Good answer:**
`__repr__` is meant to produce an unambiguous, developer-facing
representation — ideally one that, if fed back into the interpreter, would
recreate an equal object (`eval(repr(x)) == x` as a goal, not a strict
requirement). `__str__` is meant to produce a readable, user-facing string.
`print()` and `str()` call `__str__`; but if a class doesn't define
`__str__`, Python falls back to `__repr__` (since `object.__str__` is
implemented in terms of `__repr__`). The interactive interpreter, and
containers like lists printing their elements, always use `repr()` on
elements, not `str()` — so if you only override `__str__`, `print([my_obj])`
still shows the default `<MyClass object at 0x...>` repr, which surprises
people who assumed overriding one covers both.

**Code example:**
```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):
        return f"Point({self.x!r}, {self.y!r})"   # unambiguous, eval-able
    def __str__(self):
        return f"({self.x}, {self.y})"            # friendly

p = Point(1, 2)
print(p)          # (1, 2)          <- __str__
print([p])        # [Point(1, 2)]   <- list repr calls repr() on elements, not str()
repr(p)           # 'Point(1, 2)'
```

**Follow-up question:**
If you define only `__repr__` and not `__str__`, what happens when you call `str(obj)`?

**Follow-up good answer:**
It works and returns the same output as `repr(obj)`. `object.__str__`'s
default implementation simply calls `self.__repr__()`, so any class that
defines `__repr__` but not `__str__` automatically gets a reasonable
`str()`/`print()` behavior for free — this is why the common minimal advice
is "always define `__repr__`; only add `__str__` if you specifically want a
different, friendlier user-facing form."

**Glossary:**
- **`__repr__`** — unambiguous, developer-oriented representation; used by the REPL, `repr()`, and container printing.
- **`__str__`** — readable, user-facing representation; used by `str()` and `print()`, falls back to `__repr__` if undefined.

**Mental model:**
Tests precise knowledge of the data model's fallback chain (not just "one
is for users, one is for devs") and whether the candidate knows the
container-printing gotcha, a common source of "why isn't my `__str__`
showing up" bugs.

**TL;DR:**
`__str__` is the user-facing string (used by print/str, with `object.__str__` falling back to `__repr__` if undefined) and `__repr__` is the unambiguous developer-facing one always used by the REPL and inside container printing — define `__repr__` first since `__str__` inherits from it for free.

**References:**
- [Python Data Model §3.3.1, object.__repr__()](https://docs.python.org/3/reference/datamodel.html#object.__repr__)
- [Python Data Model §3.3.1, object.__str__()](https://docs.python.org/3/reference/datamodel.html#object.__str__)

---

### Q6. How does `@property` actually work under the hood? {#q6}

**Question:**
How does `@property` actually work under the hood?

**Good answer:**
`property` is a descriptor — a class implementing `__get__`, `__set__`, and
`__delete__`. When you write `@property def x(self): ...`, Python creates a
`property` object wrapping your getter function and binds it to the class
attribute `x`. Because `property` is a *data descriptor* (it defines both
`__get__` and `__set__`), it takes priority over instance `__dict__` entries
during attribute lookup: accessing `instance.x` doesn't look in
`instance.__dict__` at all — it finds the `property` object on the class
and calls `property.__get__(instance, type(instance))`, which internally
invokes your getter function with `instance` as `self`. `@x.setter` adds a
setter function to the *same* property object (creating a new property
object under the hood, replacing the class attribute), so `instance.x = v`
calls `property.__set__`, which calls your setter. This is exactly the
mechanism that lets attribute access (`obj.x`) trigger arbitrary code
without changing the caller's syntax from plain attribute access to a
method call.

**Code example:**
```python
class Celsius:
    def __init__(self, value=0):
        self._value = value

    @property
    def value(self):
        return self._value

    @value.setter
    def value(self, v):
        if v < -273.15:
            raise ValueError("below absolute zero")
        self._value = v

c = Celsius()
c.value = 25        # looks like plain attribute access, actually calls the setter
print(c.value)      # calls the getter -> 25
c.value = -300       # raises ValueError
```

**Follow-up question:**
Why does a data descriptor like `property` take priority over an instance's `__dict__`, but a non-data descriptor (like a plain function/method) doesn't?

**Follow-up good answer:**
CPython's attribute lookup order is deliberately: (1) data descriptors
found on the class/its MRO, (2) the instance's own `__dict__`, (3) non-data
descriptors and other class attributes found on the class/MRO. A "data
descriptor" defines `__set__` (and/or `__delete__`) in addition to
`__get__`; a "non-data descriptor" (like an ordinary function, which
implements only `__get__` to produce bound methods) defines only `__get__`.
The reasoning is: if a descriptor can be *set*, it's declaring ownership of
that attribute's storage and behavior (like `property` routing all
reads/writes through your getter/setter), so it must not be silently
shadowed by an instance dict entry of the same name — that would let
`instance.__dict__['x'] = ...` bypass your setter's validation entirely.
Non-data descriptors like methods don't have that concern (there's nothing
to protect), so an instance attribute of the same name is allowed to
shadow them — which is exactly how per-instance attribute overrides of
methods work.

**Glossary:**
- **Descriptor** — an object implementing `__get__`/`__set__`/`__delete__` that customizes attribute access when placed as a class attribute.
- **Data descriptor** — one that defines `__set__` and/or `__delete__`; takes priority over instance `__dict__`.

**Mental model:**
Tests whether the candidate can explain a commonly-used feature (`property`)
in terms of the underlying protocol rather than treating it as magic syntax
— and whether they know the data-vs-non-data descriptor priority rule,
which explains several "why didn't my instance attribute override this"
surprises.

**TL;DR:**
`@property` is a data descriptor object bound to the class; because data descriptors (defining `__set__`) take priority over instance `__dict__` in attribute lookup, `obj.x` always routes through your getter/setter instead of being silently shadowed by an instance attribute.

**References:**
- [Python Data Model §3.3.2, Implementing Descriptors](https://docs.python.org/3/reference/datamodel.html#implementing-descriptors)
- [Python Data Model §3.3.2.2, Invoking Descriptors](https://docs.python.org/3/reference/datamodel.html#invoking-descriptors)
- [Python built-in functions, property()](https://docs.python.org/3/library/functions.html#property)

---

### Q7. In `class D(B, C):` where both `B` and `C` inherit from `A`, what determines which `method()` runs when `D` doesn't define one, and how does `super()` fit in? {#q7}

**Question:**
In `class D(B, C):` where both `B` and `C` inherit from `A`, what determines which `method()` runs when `D` doesn't define one, and how does `super()` fit in?

**Good answer:**
Python resolves this via the Method Resolution Order (MRO), computed with
the C3 linearization algorithm and stored on the class as `D.__mro__`. For
`class A: ...`, `class B(A): ...`, `class C(A): ...`, `class D(B, C): ...`,
the MRO is `[D, B, C, A, object]` — C3 guarantees a class always appears
before its parents, and preserves the left-to-right order that bases were
listed in, while resolving the diamond so `A` is only visited once, after
*both* `B` and `C`. Attribute/method lookup walks this list in order and
uses the first match. `super()` doesn't mean "my parent class" — it means
"the next class after the current one in the *instance's* MRO," which is
why cooperative multiple inheritance (`super().method()` in every class in
the hierarchy) correctly visits `B` then `C` then `A` exactly once each,
even though `B.method` and `C.method` were both written with no knowledge
of each other.

**Code example:**
```python
class A:
    def method(self): print("A")
class B(A):
    def method(self): print("B"); super().method()
class C(A):
    def method(self): print("C"); super().method()
class D(B, C):
    pass

print(D.__mro__)   # (D, B, C, A, object)
D().method()       # B, C, A  -- NOT B, A (super() follows the MRO, not "my direct parent")
```

**Follow-up question:**
What happens if two base classes have MROs that can't be consistently linearized — for example, they disagree on the relative order of two shared ancestors?

**Follow-up good answer:**
Python raises `TypeError: Cannot create a consistent method resolution
order` at class-definition time. C3 linearization has a "monotonicity"
requirement: the final MRO must respect the local precedence order
declared in every base class's own MRO and the order bases were listed in
the subclass. If two bases fundamentally disagree about which of two
common ancestors should come first, no single linear ordering can satisfy
both constraints simultaneously, so C3 fails outright rather than silently
picking an arbitrary (and therefore surprising) order — this is by design,
since a silently "guessed" MRO could resolve `super()` calls inconsistently
with what either base class's author intended.

**Glossary:**
- **MRO (Method Resolution Order)** — the linear order Python uses to search a class hierarchy for attributes/methods.
- **C3 linearization** — the specific deterministic algorithm Python uses to compute the MRO, guaranteeing monotonicity and local precedence.

**Mental model:**
Tests whether the candidate actually understands multiple inheritance
resolution (a frequent weak spot) versus assuming naive depth-first
left-to-right lookup (Python 2 classic-class behavior, which produces
wrong answers for diamonds) — and whether they know `super()` is MRO-based
cooperative dispatch, not "call my parent."

**TL;DR:**
Python uses C3-linearized MRO (`D.__mro__`) to resolve multiple inheritance, guaranteeing each ancestor in a diamond is visited exactly once in a consistent order, and `super()` walks that same MRO rather than jumping straight to a "parent" class — which is what makes cooperative multiple inheritance work.

**References:**
- [Python 2.3 Method Resolution Order (the canonical C3 linearization writeup, still the authoritative reference cited by the docs)](https://docs.python.org/3/howto/mro.html)
- [Python built-in functions, super()](https://docs.python.org/3/library/functions.html#super)

---

### Q8. What actually changes at the memory-layout level when a class defines `__slots__`? {#q8}

**Question:**
What actually changes at the memory-layout level when a class defines `__slots__`?

**Good answer:**
By default, every instance of a Python class carries its own `__dict__` —
a per-instance dictionary that stores its attributes, which is flexible
(you can add any attribute at runtime) but costly (a full hash table per
instance). Declaring `__slots__ = ('x', 'y')` tells Python to allocate
fixed-size, fixed-offset storage slots for exactly those named attributes
instead of creating a per-instance `__dict__` — attribute access becomes a
direct slot read at a known offset rather than a dict lookup, and no
`__dict__` object is allocated at all (unless a base class in the MRO
doesn't declare `__slots__`, in which case a `__dict__` still gets created
and the memory savings are lost). Consequences: instances can no longer
have arbitrary attributes added at runtime — assigning anything not listed
in `__slots__` raises `AttributeError`. For multiple inheritance, all
non-`object` base classes generally need `__slots__` defined (or you'll
still get a `__dict__` from whichever base lacks it).

**Follow-up question:**
If `__slots__` saves memory and speeds up attribute access, why isn't it the default for every class?

**Follow-up good answer:**
Because it trades away flexibility that a lot of ordinary Python code
relies on: monkey-patching instances with new attributes at runtime,
frameworks that attach arbitrary metadata to objects, `pickle`/`copy`
support that historically assumed `__dict__` existed (modern Python
handles `__slots__` fine but it requires the class to cooperate), and
`weakref` support (which requires explicitly adding `'__weakref__'` to
`__slots__` if you need weak references, since it's not included by
default). It also complicates multiple inheritance, as noted above. For
most application code the memory/speed win is negligible; it matters when
you're instantiating millions of small, fixed-shape objects (e.g. a graph
node, a point, a parsed token) where the per-instance `__dict__` overhead
dominates memory usage — which is exactly the profile `dataclasses`
supports via `@dataclass(slots=True)` (Python 3.10+).

**Glossary:**
- **`__slots__`** — a class attribute listing the only instance attributes allowed, replacing the per-instance `__dict__` with fixed slots.
- **`__weakref__`** — the slot needed to support `weakref` on a `__slots__`-using class; not included automatically.

**Mental model:**
Tests understanding of the actual memory/lookup mechanism behind
`__slots__` (not just "it saves memory") and whether the candidate can
reason about the real trade-off (flexibility vs. footprint) rather than
recommending `__slots__` everywhere as a cargo-culted optimization.

**TL;DR:**
`__slots__` replaces a per-instance `__dict__` with fixed-offset attribute storage, cutting memory and speeding up attribute access, at the cost of disallowing arbitrary new attributes and complicating multiple inheritance/weak references — worth it for millions of small fixed-shape objects, not for general-purpose classes.

**References:**
- [Python Data Model §3.3.2.4, __slots__](https://docs.python.org/3/reference/datamodel.html#slots)
- [dataclasses — slots parameter](https://docs.python.org/3/library/dataclasses.html)

---

### Q9. You suspect a service is using too much memory because it holds millions of small objects. How would you confirm `__slots__` would actually help, and by how much? {#q9}

**Question:**
You suspect a service is using too much memory because it holds millions of small objects. How would you confirm `__slots__` would actually help, and by how much?

**Good answer:**
Don't guess — measure. First confirm the object count and per-object
overhead with `sys.getsizeof()` on a representative instance (note this
only measures the object's own footprint, not what its attributes point
to, so use `pympler.asizeof` or `tracemalloc` for a fuller picture of
actual retained memory across the whole object graph). Compare
`getsizeof()` of the dict-based class vs. a `__slots__` version of the same
class with the same fields on real data — the per-instance `__dict__`
typically costs a few hundred bytes on top of the object header, which
`__slots__` eliminates, so the win scales linearly with instance count
(negligible for thousands of objects, very real for tens of millions).
`tracemalloc` (stdlib) can snapshot allocations and show you exactly which
line/class is responsible for the largest share of live memory, which
tells you whether these objects are actually the bottleneck before you
spend effort converting them.

**Code example:**
```python
import sys

class Point:            # dict-based
    def __init__(self, x, y): self.x, self.y = x, y

class SlottedPoint:      # slots-based
    __slots__ = ('x', 'y')
    def __init__(self, x, y): self.x, self.y = x, y

print(sys.getsizeof(Point(1, 2).__dict__))   # instance dict overhead
print(sys.getsizeof(SlottedPoint(1, 2)))     # slotted instance, no separate dict
```

**Follow-up question:**
Besides memory, could `__slots__` meaningfully change CPU time in a hot loop, and how would you verify that?

**Follow-up good answer:**
Yes — attribute access on a slotted instance is a direct, fixed-offset
read rather than a dictionary hash/lookup, so tight loops that do heavy
attribute get/set (e.g. numeric simulation touching `.x`/`.y` millions of
times) can see a measurable speedup, though it's usually smaller than the
memory win and is implementation/version dependent. Verify it the same way
you'd verify any performance claim: write a microbenchmark with `timeit`
comparing the slotted and non-slotted versions doing the exact same
attribute-heavy workload, run it multiple times to account for noise, and
only trust the result on the actual Python version/interpreter (CPython
vs. PyPy behave very differently here) you'll deploy — never assume a
"well-known" optimization transfers to your specific workload without
measuring it there.

**Glossary:**
- **`tracemalloc`** — stdlib module for tracing memory allocations by source line.
- **`timeit`** — stdlib module for accurate microbenchmarking of small code snippets.

**Mental model:**
This is fundamentally a performance-diagnosis question wearing a
`__slots__` costume: tests whether the candidate reaches for measurement
tools (`sys.getsizeof`, `tracemalloc`, `timeit`) before optimizing, rather
than applying `__slots__` everywhere on faith.

**TL;DR:**
Confirm the memory/CPU win with real measurement — `sys.getsizeof`/`tracemalloc` for the per-instance `__dict__` overhead at your actual object count, and `timeit` for any attribute-access speedup in your actual hot loop — rather than assuming `__slots__` helps without profiling first.

**References:**
- [tracemalloc — Trace memory allocations](https://docs.python.org/3/library/tracemalloc.html)
- [timeit — Measure execution time of small code snippets](https://docs.python.org/3/library/timeit.html)
- [sys.getsizeof()](https://docs.python.org/3/library/sys.html#sys.getsizeof)

---

### Q10. A hot loop calling `obj.method()` millions of times is slower than expected. What tools would you use to find out why, and what's actually expensive about a Python attribute/method access? {#q10}

**Question:**
A hot loop calling `obj.method()` millions of times is slower than expected. What tools would you use to find out why, and what's actually expensive about a Python attribute/method access?

**Good answer:**
Start with `cProfile` (stdlib) to get function-level call counts and
cumulative time and identify which function actually dominates — don't
optimize blind. For line-level detail within a hot function, use
`line_profiler` or sampling profilers like `py-spy` (attaches to a running
process with minimal overhead, good for production). To understand *why* a
particular call is expensive, `dis.dis()` shows the bytecode: a plain
attribute/method access like `obj.method()` compiles to `LOAD_FAST`,
`LOAD_ATTR` (or `LOAD_METHOD`/`CALL_METHOD` on newer versions, an
optimization that avoids creating an intermediate bound-method object),
then `CALL`. Each `LOAD_ATTR` on a dict-based instance walks the MRO for
class-level lookups and/or does a dict lookup on `instance.__dict__` —
which is why `__slots__` (Q8/Q9), caching a method reference in a local
variable before a loop, or restructuring to avoid repeated attribute
traversal (`self.a.b.c` inside a loop) are all real, measurable levers —
but only after profiling confirms attribute lookup, not something else
(I/O, an accidentally-quadratic algorithm), is the actual bottleneck.

**Code example:**
```python
import cProfile

def hot():
    class P:
        def __init__(self): self.x = 0
        def inc(self): self.x += 1
    p = P()
    for _ in range(2_000_000):
        p.inc()

cProfile.run("hot()")   # shows time spent in inc() vs elsewhere
```

**Follow-up question:**
The profiler shows most time is in `inc()` itself, not in getting to it. What would you look at next to actually reduce that per-call cost?

**Follow-up good answer:**
At that point the bottleneck is genuine per-call Python-level overhead
(bytecode dispatch, attribute read/write, function-call machinery), not a
design mistake elsewhere — so the next lever is whether the operation
needs to run in pure Python at that frequency at all. Options in order of
typical effort: (1) hoist repeated attribute lookups to locals inside the
loop (`x = self.x; for _ in range(n): x += 1; self.x = x` — local variable
access is faster than attribute access since it's a direct frame-slot
read), (2) use `__slots__` if not already, (3) vectorize with NumPy if the
operation is numeric and batchable, replacing the Python-level loop
entirely with a single C-level call, or (4) push the hot path to a
C-extension/`Cython`/`numba`-jitted function if it's inherently scalar and
can't be vectorized. Which lever is worth it depends entirely on whether
this loop is actually the application's bottleneck end-to-end — confirmed
by the profiler, not assumed.

**Glossary:**
- **`cProfile`** — stdlib deterministic profiler; function-level call counts/timings.
- **`py-spy`** — third-party sampling profiler that can attach to a live process without code changes.
- **`LOAD_METHOD`/`CALL_METHOD`** — CPython bytecode optimization (3.7+) that avoids materializing an intermediate bound method object for `obj.method()` calls.

**Mental model:**
Tests whether the candidate has a real performance-diagnosis workflow
(profile first, hypothesize from data, verify the fix) for Python
specifically, and whether they understand attribute access isn't free —
it's a lookup, not a memory offset, unless `__slots__` is involved.

**TL;DR:**
Profile with cProfile/py-spy before optimizing; a Python attribute/method access is a real lookup (MRO walk and/or dict lookup) compiled to LOAD_ATTR/LOAD_METHOD bytecode, not a free memory read, which is why hoisting to locals, __slots__, or vectorization are the levers once profiling confirms that's actually the bottleneck.

**References:**
- [cProfile and profile — Python Profilers](https://docs.python.org/3/library/profile.html)
- [dis — Disassembler for Python bytecode](https://docs.python.org/3/library/dis.html)

---

### Q11. What's the difference between duck typing and structural typing, and how does `typing.Protocol` relate to both? {#q11}

**Question:**
What's the difference between duck typing and structural typing, and how does `typing.Protocol` relate to both?

**Good answer:**
Duck typing is a *runtime* philosophy: Python doesn't check an object's
declared type before calling a method on it — "if it walks like a duck and
quacks like a duck, treat it as a duck." Any object with a `.read()` method
works wherever file-like behavior is expected, regardless of its class
hierarchy; the check (if any) only happens when the method is actually
called and either succeeds or raises `AttributeError`. Structural typing is
the *static* analogue of the same idea: a type checker (mypy, pyright)
decides whether a type is compatible with another based on the *shape* of
its interface (which methods/attributes it has), not on explicit
inheritance. Historically Python's type hints only supported *nominal*
typing (compatibility required explicit subclassing of an ABC like
`collections.abc.Sized`), which felt distinctly un-Pythonic. `typing.Protocol`
(PEP 544) closes that gap: a class inheriting from `Protocol` and declaring
some methods becomes a structural type — any class implementing those
methods satisfies it for a type checker, with zero inheritance required,
making static type-checking match how duck typing actually behaves at
runtime.

**Code example:**
```python
from typing import Protocol

class Closable(Protocol):
    def close(self) -> None: ...

class FileHandle:                 # note: does NOT inherit from Closable
    def close(self) -> None:
        print("closed")

def cleanup(resource: Closable) -> None:
    resource.close()

cleanup(FileHandle())   # type-checks fine: FileHandle structurally satisfies Closable
```

**Follow-up question:**
Does using `typing.Protocol` give you any runtime enforcement, or is it purely a static-analysis tool?

**Follow-up good answer:**
By default it's purely static — at runtime, Python doesn't check anything
about `Protocol` classes; `cleanup(42)` above would run without error until
`.close()` is actually attempted and fails, exactly like ordinary duck
typing. You *can* opt into limited runtime checking by decorating the
protocol with `@runtime_checkable`, which allows `isinstance(obj, MyProtocol)`
to check for the *presence* of the required method names (not their
signatures, argument types, or return types — just that the attributes
exist). This is intentionally shallow: full runtime structural type
checking (verifying signatures match) isn't something the typing module
provides, since it would conflict with Python's dynamic nature and add
real runtime cost to what's meant to be a static-analysis feature.

**Glossary:**
- **Duck typing** — runtime behavior compatibility based on what an object can do, not its declared type.
- **Structural (subtyping)** — static type-checker compatibility based on interface shape rather than explicit inheritance; "static duck typing."
- **`@runtime_checkable`** — decorator enabling shallow `isinstance()` checks (attribute presence only) against a `Protocol`.

**Mental model:**
Tests whether the candidate can connect a very Python-specific runtime
philosophy (duck typing) to its formalization in the modern type system
(`Protocol`/PEP 544), and understands the boundary between static analysis
and runtime behavior — a common point of confusion once type hints entered
the language.

**TL;DR:**
Duck typing is Python's runtime "call it and see" behavior; structural typing is its static-analysis equivalent, and `typing.Protocol` (PEP 544) lets a type checker treat any class with a matching method shape as compatible without requiring explicit inheritance — with `@runtime_checkable` offering only shallow, attribute-presence-only runtime checks.

**References:**
- [typing — Protocol classes](https://docs.python.org/3/library/typing.html#typing.Protocol)
- [PEP 544 – Protocols: Structural subtyping (static duck typing)](https://peps.python.org/pep-0544/)

---

### Q12. How does the Liskov Substitution Principle apply to Python, given that the language doesn't enforce method signatures at all? {#q12}

**Question:**
How does the Liskov Substitution Principle apply to Python, given that the language doesn't enforce method signatures at all?

**Good answer:**
LSP says subtypes must be substitutable for their base type without
breaking correctness — callers relying on the base type's contract
shouldn't be surprised by a subclass's behavior. Python doesn't check this
mechanically (no compiler enforces it), so violating LSP in Python doesn't
raise an error at the point of violation — it surfaces later, as a runtime
bug somewhere a caller trusted the contract. Common Python-specific ways to
violate it: overriding a method and narrowing the accepted input types or
widening the raised exceptions beyond what the base class documents,
overriding `__eq__`/`__hash__` inconsistently with the base class's
semantics, or overriding a method to have side effects the base class's
docstring/behavior didn't promise (e.g. a subclass's `append()` that
also logs to a network call, breaking a caller's assumption about
performance/side-effect-freedom). Since nothing enforces this
mechanically, it becomes a *design discipline* problem — the fact Python
lets you technically override anything with any signature makes LSP
violations easier to introduce and harder to catch than in a statically
enforced language, which is exactly why interviewers ask about it: it
reveals whether a candidate designs contracts deliberately.

**Follow-up question:**
Give a concrete example of an LSP violation using Python's dunder methods specifically.

**Follow-up good answer:**
A classic one: a `Rectangle` class supports `__eq__` comparing width and
height. You create a `Square(Rectangle)` subclass whose `__init__` forces
width and height to always match, and you don't override `__eq__` — so far
so good, it inherits sensible behavior. The violation appears if `Square`
instead overrides `width`'s setter so that setting `width` *also* silently
changes `height` (to keep it "square"), while `Rectangle`'s public contract
(implied by having independent width/height setters) is that they vary
independently. Code written against `Rectangle` — e.g. `rect.width = 5;
rect.height = 10; assert rect.area() == 50` — silently breaks when handed
a `Square`, not because Python raised an error, but because the subclass's
narrowed behavior violates an assumption the base type's interface implied.
This is the canonical LSP teaching example precisely because "a square is
a rectangle" is true geometrically but false as an LSP-compliant subtype
relationship.

**Glossary:**
- **LSP (Liskov Substitution Principle)** — the "L" in SOLID: subtypes must be usable anywhere their base type is expected without altering correctness.
- **Contract** — the behavioral guarantees (accepted inputs, exceptions, side effects) a class's interface implies, whether or not it's type-checked.

**Mental model:**
Tests whether the candidate can apply a classic OO design principle in a
language that doesn't mechanically enforce it — separating "the type
checker would catch this" from "this is still a design bug even though
Python will happily run it."

**TL;DR:**
Python doesn't enforce LSP mechanically, so violations (narrowed accepted inputs, inconsistent __eq__/__hash__, unexpected side effects in an override) don't error at the violation site — they surface later as subtle bugs wherever a caller trusted the base type's contract, making LSP purely a design discipline in Python rather than something the language protects for you.

**References:**
- [PEP 8 style guide references consistent, documented method contracts as part of API design (general good-practice reference; LSP itself is a general OO principle, not Python-specific — no single official CPython doc defines it)](https://peps.python.org/pep-0008/)

---

### Q13. What real problem do dunder methods like `__add__` and `__len__` actually solve, beyond "syntax sugar for operators"? {#q13}

**Question:**
What real problem do dunder methods like `__add__` and `__len__` actually solve, beyond "syntax sugar for operators"?

**Good answer:**
They let user-defined types participate in Python's built-in operator and
protocol syntax as first-class citizens, indistinguishable from built-in
types at the call site. Without them, every custom numeric-like or
container-like type would need its own bespoke method names
(`vector.add(other)`, `matrix.getLength()`), forcing every piece of generic
code that wants to work with "anything addable" or "anything with a
length" to special-case each type by name. By implementing `__add__`, a
`Vector` class makes `v1 + v2` work identically in syntax to `1 + 2`; by
implementing `__len__`, any container works with the built-in `len()`
function and, by extension, anywhere Python's runtime checks "does this
have a length" (e.g. `bool()` on an object without `__bool__` falls back to
`__len__() != 0`). This is the mechanism behind Python's broader design
philosophy that generic functions (`len`, `+`, `for...in`, `with`) should
work uniformly across built-in and user types, rather than requiring
special-casing — the data model protocol *is* Python's extensibility story
for its own syntax.

**Code example:**
```python
class Vector:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)
    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1, v2 = Vector(1, 2), Vector(3, 4)
print(v1 + v2)     # Vector(4, 6) -- '+' dispatches to __add__
```

**Follow-up question:**
If `a + b` fails because `type(a).__add__` returns `NotImplemented`, what happens next, and why does that matter for supporting mixed-type operations?

**Follow-up good answer:**
Python then tries the *reflected* method on the other operand:
`type(b).__radd__(b, a)`. Only if that also returns `NotImplemented` (or
doesn't exist) does Python raise `TypeError: unsupported operand type(s)`.
This two-sided dispatch is what makes `my_vector + 5` or `5 + my_vector`
both possible to support correctly even though `int` knows nothing about
`Vector`: `int.__add__(5, my_vector)` correctly returns `NotImplemented`
(ints don't know how to add a Vector), which signals Python to try
`Vector.__radd__(my_vector, 5)` instead. Returning `NotImplemented` (not
raising an exception directly) is the crucial detail — it's a cooperative
signal meaning "I don't know how to handle this combination, ask the other
operand," which is what enables correct mixed-type arithmetic without
either type needing to know about the other in advance.

**Glossary:**
- **`NotImplemented`** — the sentinel a dunder method returns to signal "I can't handle this operand combination, try the reflected method."
- **Reflected method** — `__radd__`, `__rsub__`, etc., tried when the left operand's forward method declines.

**Mental model:**
Tests whether the candidate sees the data model as Python's actual
extensibility mechanism for built-in syntax (not decoration) and
understands the `NotImplemented`/reflected-method dance, which is
essential for correctly supporting mixed-type operator overloading.

**TL;DR:**
Dunder methods let custom types plug directly into Python's built-in operator/protocol syntax instead of needing bespoke method names, and mixed-type operations work correctly because a dunder method can return NotImplemented to hand off to the other operand's reflected method (`__radd__`, etc.) rather than failing outright.

**References:**
- [Python Data Model §3.3.8, Emulating numeric types](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types)
- [Python Data Model §3.3.7, Basic customization / object.__len__()](https://docs.python.org/3/reference/datamodel.html#object.__len__)

---

### Q14. What problem does the context manager protocol (`with` statement) actually solve that a plain `try`/`finally` doesn't? {#q14}

**Question:**
What problem does the context manager protocol (`with` statement) actually solve that a plain `try`/`finally` doesn't?

**Good answer:**
Functionally, `with obj: body` is close to `try: __enter__(); body;
finally: __exit__()` — but the protocol solves two things `try`/`finally`
alone doesn't give you for free: reusability and correctness at scale.
Implementing `__enter__`/`__exit__` once on a resource type (a DB
connection, a lock, a temp file) means every caller gets guaranteed cleanup
with a single `with conn:` line, instead of every call site having to
remember to hand-write a correct `try`/`finally` — and "every call site
remembers" is exactly the kind of discipline that erodes at scale. It also
gives you a single, structured place (`__exit__`) to inspect *whether* an
exception occurred (`exc_type`/`exc_val`/`exc_tb` arguments) and decide
whether to suppress it — a `try`/`finally` block's `finally` clause runs
regardless of exceptions but has no clean built-in way to say "and also
swallow this specific exception," whereas `__exit__` returning `True` does
exactly that, deliberately.

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
        return False   # don't suppress exceptions

with Timer():
    do_work()
```

**Follow-up question:**
What does `contextlib.contextmanager` let you do instead of writing a class, and what's the trade-off?

**Follow-up good answer:**
`@contextlib.contextmanager` lets you write a context manager as a single
generator function: code before `yield` runs as `__enter__` (the yielded
value becomes the `as` target), code after `yield` runs as `__exit__`
(wrapped in `try`/`finally` so cleanup still runs on exception), and an
exception from the `with` block is raised at the `yield` point inside the
generator, letting you `try`/`except` around it there if you want to
suppress or transform it. The trade-off is expressiveness vs. simplicity:
the class-based form is more verbose but supports being instantiated once
and entered multiple times, and makes the state explicit as attributes;
the generator form is much more concise for simple setup/teardown pairs
but the single-function shape makes it awkward for context managers with
complex reentrancy or state-inspection needs. For the common case
(acquire something, always release it), the generator form is idiomatic
and far less boilerplate.

**Glossary:**
- **`__enter__`/`__exit__`** — the two methods implementing the context manager protocol.
- **`contextlib.contextmanager`** — decorator turning a single generator function into a context manager.

**Mental model:**
Tests whether the candidate sees `with` as solving a real reliability
problem (guaranteed, reusable, exception-aware cleanup) rather than as
"nicer syntax," and whether they know the generator-based shortcut and its
trade-offs versus the full class-based protocol.

**TL;DR:**
The context manager protocol turns "remember to write correct try/finally cleanup" into "implement it once, get it everywhere for free," and __exit__'s exception-inspection + suppress-via-return-True capability gives structured exception handling that bare try/finally doesn't — contextlib.contextmanager gets you the same behavior from a single generator function for simple cases.

**References:**
- [Python Data Model §3.3.9, With Statement Context Managers](https://docs.python.org/3/reference/datamodel.html#with-statement-context-managers)
- [contextlib — @contextmanager](https://docs.python.org/3/library/contextlib.html#contextlib.contextmanager)

---

### Q15. Why does Python have a separate iterator protocol (`__iter__`/`__next__`) instead of just letting `for` loops index into anything with `__getitem__` and a length? {#q15}

**Question:**
Why does Python have a separate iterator protocol (`__iter__`/`__next__`) instead of just letting `for` loops index into anything with `__getitem__` and a length?

**Good answer:**
(Python actually supports both — `for` falls back to the old
`__getitem__`-with-increasing-integers protocol if `__iter__` is absent,
for backward compatibility — but the iterator protocol is the primary,
recommended mechanism.) The iterator protocol exists because it decouples
*how you produce values* from *needing a container with a known length and
random-access indexing at all*. An object implementing `__iter__` (which
returns an iterator, an object with `__next__` that raises `StopIteration`
when exhausted) can represent something that has no fixed size and no
meaningful index — an infinite sequence, a stream read lazily from a
network socket or file line-by-line, or values computed on demand rather
than precomputed and stored. This is the foundation generator functions
build on: a `yield`-based generator function automatically implements the
full iterator protocol, letting you write lazy, memory-efficient sequences
(processing a 10GB file line by line without loading it all into memory)
using ordinary control-flow syntax instead of hand-writing a state
machine.

**Code example:**
```python
def read_large_file(path):
    with open(path) as f:
        for line in f:          # file object IS an iterator; lines produced lazily
            yield line.strip()

for line in read_large_file("huge.log"):   # never loads the whole file into memory
    process(line)
```

**Follow-up question:**
What's actually different, mechanically, between a generator *function* and a generator *object*, and why does calling a generator function not run any of its code immediately?

**Follow-up good answer:**
A generator function is any function containing `yield` — calling it does
not execute the function body at all; it immediately returns a generator
object, which is a paused coroutine-like frame holding the function's
local state (its bytecode instruction pointer, local variables, and
evaluation stack) frozen at the top. Execution only actually happens when
you call `next()` on that generator object (directly, or implicitly via a
`for` loop) — the function body runs until the next `yield`, produces that
value, and suspends again with its entire local state intact, to be
resumed exactly where it left off on the next `next()` call. This is why
generator functions are "lazy by construction": defining `def gen(): ...
yield ...` and even calling `gen()` does zero work up front — all the real
computation happens incrementally, driven by whoever consumes the
iterator, which is precisely what makes them suitable for infinite or
arbitrarily large lazy sequences.

**Glossary:**
- **Iterator** — an object with `__next__()` (and `__iter__()` returning itself) that produces a sequence of values on demand, raising `StopIteration` when done.
- **Generator function** — a function containing `yield`; calling it returns a generator object without running any code, deferring execution to each `next()` call.

**Mental model:**
Tests whether the candidate understands laziness as a mechanical property
of the iterator protocol (deferred, resumable execution) rather than a
vague performance claim, and can explain why generators specifically are
"free" to define but incremental to run.

**TL;DR:**
The iterator protocol (`__iter__`/`__next__`, raising `StopIteration` on exhaustion) lets a `for` loop consume values lazily from something with no fixed size or index at all, and generator functions build on it by turning ordinary control flow into a resumable paused frame — calling a generator function does zero work, all computation happens incrementally as `next()` is called.

**References:**
- [Python Data Model §3.3.9, The iterator protocol section is under "Basic customization" — see object.__iter__() / object.__next__()](https://docs.python.org/3/reference/datamodel.html#object.__iter__)
- [Python Language Reference §6.2.9, Yield expressions](https://docs.python.org/3/reference/expressions.html#yield-expressions)

---

### Q16. What's wrong with `def add_item(item, target=[]):` as a function signature, and why does the bug only show up intermittently? {#q16}

**Question:**
What's wrong with `def add_item(item, target=[]):` as a function signature, and why does the bug only show up intermittently?

**Good answer:**
Default argument values are evaluated exactly once, at function *definition*
time, not once per call — the `[]` literal creates a single list object
that becomes part of the function object itself, and every call that
doesn't supply `target` explicitly reuses that *same* list. If the function
mutates it (`target.append(item)`), the mutation persists across calls,
silently accumulating state that has nothing to do with the current call —
"intermittent" because the bug is invisible as long as every caller
happens to pass their own `target` explicitly; it only surfaces once
someone relies on the "default empty list" behavior across two separate
calls and is surprised the second call already sees the first call's item.
The fix is the standard idiom: default to `None`, and create a fresh
mutable object inside the function body when the default is used.

**Code example:**
```python
def add_item(item, target=[]):     # BUG: one shared list for every call
    target.append(item)
    return target

add_item(1)     # [1]
add_item(2)     # [1, 2]  <- surprise! not a fresh list

def add_item_fixed(item, target=None):
    if target is None:
        target = []      # fresh list every call that omits target
    target.append(item)
    return target

add_item_fixed(1)   # [1]
add_item_fixed(2)   # [2]  <- correct, independent
```

**Follow-up question:**
Is this behavior a bug in Python, or is there a reason the language works this way?

**Follow-up good answer:**
It's an intentional, documented consequence of the language's
evaluation model, not a bug — default values are ordinary expressions
evaluated once when the `def` statement executes (the same time the
function object itself is created), consistent with how Python treats
everything else at module/class-body execution time. Re-evaluating
defaults on every call would actually be inconsistent with that model and
would also be strictly less useful for legitimate uses of mutable-looking
defaults that are never mutated, or intentional "compute once, reuse many
times" patterns (e.g. a default that's an expensive-to-construct but
never-mutated lookup table). The behavior only becomes a footgun when the
default is both mutable and gets mutated by the function body — the fix is
a matter of knowing this Python-specific evaluation-timing rule, not
Python being broken.

**Glossary:**
- **Default argument evaluation** — default values are computed once, at `def` time, and reused for every call that doesn't override them.

**Mental model:**
One of the most common real-world Python bugs; tests whether the candidate
has actually been bitten by this (or deeply understands why it happens) as
opposed to only knowing the surface-level "don't use mutable defaults"
rule without being able to explain the mechanism or why the fix works.

**TL;DR:**
Default argument values are created once at function-definition time, not per call, so a mutable default like `[]` is silently shared and accumulates state across every call that omits it — fix by defaulting to `None` and creating a fresh object inside the function body.

**References:**
- [Python FAQ: Why are default values shared between objects?](https://docs.python.org/3/faq/programming.html#why-are-default-values-shared-between-objects)

---

### Q17. What's the difference between `copy.copy()` and `copy.deepcopy()`, and when would a shallow copy silently produce a bug? {#q17}

**Question:**
What's the difference between `copy.copy()` and `copy.deepcopy()`, and when would a shallow copy silently produce a bug?

**Good answer:**
`copy.copy()` constructs a new outer object but populates it with
*references* to the same nested objects the original holds — the new
container is independent, but anything inside it that's itself mutable is
still shared. `copy.deepcopy()` recursively copies everything, so the
entire object graph becomes independent, at the cost of more time/memory
and needing to handle cycles (which `deepcopy` does via an internal `memo`
dict tracking already-copied objects, to avoid infinite recursion on
self-referential structures). The silent bug case: you shallow-copy a
list-of-lists (or any nested mutable structure) expecting full
independence, mutate an inner list through the copy, and discover the
"original" changed too — because the inner lists were never copied, only
referenced. This is a frequent source of confusing bugs precisely because
the top-level object *does* behave independently (append/remove on the
outer list doesn't affect the original), which makes it easy to assume the
whole structure is independent until a nested mutation reveals otherwise.

**Code example:**
```python
import copy

original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
shallow[0].append(99)        # mutates the SHARED inner list
print(original)              # [[1, 2, 99], [3, 4]] -- surprise, original changed too

deep = copy.deepcopy(original)
deep[0].append(100)
print(original)              # [[1, 2, 99], [3, 4]] -- unaffected, fully independent copy
```

**Follow-up question:**
For a custom class, how would you control exactly how `copy.deepcopy()` copies your object, if the default recursive behavior isn't right for it?

**Follow-up good answer:**
Implement `__deepcopy__(self, memo)` (and/or `__copy__(self)` for the
shallow case) on the class — when `copy.deepcopy()` encounters an object
that defines `__deepcopy__`, it delegates to that method instead of its
default reflection-based recursive copying. This matters for objects
holding resources that shouldn't be blindly duplicated (a database
connection, a file handle, a cache that should stay shared) — you can
write `__deepcopy__` to skip copying that particular attribute (keep the
shared reference) while still deep-copying everything else, and you must
pass the `memo` dict through to any nested `copy.deepcopy(value, memo)`
calls you make internally, so cycles and shared sub-objects within the
larger copy operation are still handled correctly instead of being
duplicated redundantly or causing infinite recursion.

**Glossary:**
- **`memo` dict** — the id-to-copy mapping `deepcopy` threads through recursive calls to handle shared references and cycles correctly.
- **`__copy__`/`__deepcopy__`** — hooks a class can define to override the default copying behavior for its instances.

**Mental model:**
Tests whether the candidate has actually debugged this exact class of bug
(shared nested mutable state after a "copy") rather than only knowing the
one-line definitions of shallow vs. deep copy in the abstract.

**TL;DR:**
`copy.copy()` only copies the outer container — nested mutable objects stay shared, which silently breaks any code assuming full independence — while `copy.deepcopy()` recursively copies the whole graph (using a memo dict to handle cycles/shared refs), and a class can override `__copy__`/`__deepcopy__` to customize or skip copying specific attributes like open resources.

**References:**
- [copy — Shallow and deep copy operations](https://docs.python.org/3/library/copy.html)

---

### Q18. Why does `a is b` sometimes return `True` for two separately-written integer literals, but `False` for two separately-written string concatenations that produce "the same" string? {#q18}

**Question:**
Why does `a is b` sometimes return `True` for two separately-written integer literals, but `False` for two separately-written string concatenations that produce "the same" string?

**Good answer:**
`is` checks object *identity* (same object in memory), not value equality
— for value comparison you always want `==`. CPython, as an
*implementation detail* (not a language guarantee), caches and reuses
certain small immutable objects: small integers in a fixed range
(historically -5 to 256) are pre-allocated singletons, so `a = 100; b =
100; a is b` happens to be `True` because both names are bound to the same
cached object. String literals that look identical *at compile time* may
also be interned (deduplicated to one object) by the compiler for
efficiency. But an integer built at runtime outside that cached range
(`a = 10_000_000; b = 5_000_000 + 5_000_000`), or a string built by runtime
concatenation, is not guaranteed to be interned — each expression can
produce a genuinely distinct object even though the *values* are equal, so
`is` returns `False` while `==` correctly returns `True`. The official
guidance is explicit: identity tests should never be used for value types
like `int`/`str`, which are not guaranteed to be singletons — this is a
CPython implementation quirk, not something correct code should ever rely
on.

**Code example:**
```python
a = 100
b = 100
print(a is b)          # True (small-int cache, CPython implementation detail)

x = 1_000
y = 1_000
print(x is y)          # NOT guaranteed True -- and often False outside the cached range

s1 = "hello"
s2 = "hello"
print(s1 is s2)        # often True (compile-time literal interning)

s3 = "".join(["h", "e", "l", "l", "o"])
print(s3 == s1, s3 is s1)   # True, False -- equal value, different object
```

**Follow-up question:**
Given this, when is `is` actually the correct thing to use instead of `==`?

**Follow-up good answer:**
`is` is correct precisely when you care about object identity itself, not
value equality — the canonical case is comparing against `None`
(`if x is None:`), which is explicitly recommended by PEP 8 because `None`
*is* a true singleton (there's only ever one `None` object) and using `is`
avoids accidentally matching some custom `__eq__` that made another object
compare equal to `None`. The same reasoning applies to `True`/`False` as
singletons, and to sentinel objects you deliberately create for the sole
purpose of identity comparison (e.g. a private `_MISSING = object()` used
to distinguish "argument not supplied" from "argument explicitly passed as
`None`"). Outside of singleton/sentinel comparisons, reaching for `is`
instead of `==` is very likely a bug waiting to surface the moment the
underlying CPython caching behavior doesn't happen to apply to your
specific values.

**Glossary:**
- **Identity vs. equality** — `is` compares object identity (same object); `==` compares value equality (calls `__eq__`).
- **Interning** — CPython's implementation-detail practice of reusing a single object for certain small ints/string literals; not a language guarantee.

**Mental model:**
Tests whether the candidate understands this as a CPython implementation
detail rather than a language feature to rely on — a very common
real-world gotcha for anyone coming from languages where `==` and identity
are more tightly coupled, or where such caching is guaranteed rather than
incidental.

**TL;DR:**
`is` checks identity, not value equality, and CPython's small-int/string-literal caching is an implementation detail that makes `is` "accidentally" work for some literals but not others with equal values — use `==` for value comparison always, and reserve `is` for true singletons like `None` or deliberately-created sentinel objects.

**References:**
- [Python FAQ: Why does Python use `is` for identity comparisons?](https://docs.python.org/3/faq/programming.html#id27)
- [PEP 8 — Programming Recommendations (identity tests for None)](https://peps.python.org/pep-0008/#programming-recommendations)

---

### Q19. What is a metaclass, and when (if ever) would you actually reach for one instead of a simpler alternative? {#q19}

**Question:**
What is a metaclass, and when (if ever) would you actually reach for one instead of a simpler alternative?

**Good answer:**
A metaclass is "the class of a class" — it defines how a class *itself*
behaves and is constructed, the same way an ordinary class defines how its
instances behave. Every class in Python is an instance of some metaclass;
by default that's `type` (`type(MyClass) is type` for an ordinary class).
Writing `class MyClass(metaclass=Meta):` tells Python to use `Meta`
(itself a subclass of `type`) to construct `MyClass`, letting you hook into
and customize the class-creation process itself — inspecting/modifying the
namespace of methods being defined, auto-registering every subclass in a
registry, enforcing that certain methods are implemented, or injecting
methods/attributes into every class using that metaclass. In practice,
metaclasses are a last resort: most of what people reach for a metaclass to
do is better solved with a class decorator (simpler, composable, doesn't
affect `isinstance`/inheritance semantics), `__init_subclass__` (a hook
introduced specifically to cover the common "do something when a subclass
is defined" case without needing a full metaclass), or Abstract Base
Classes (`abc.ABC`) for enforcing required methods. The famous quote
(commonly attributed to Tim Peters) — "metaclasses are deeper magic than
99% of users should ever worry about... if you wonder whether you need
them, you don't" — reflects real community consensus that they're
overused relative to how rarely they're the *right* tool.

**Code example:**
```python
class Registry(type):
    registry = {}
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        Registry.registry[name] = cls   # auto-register every subclass
        return cls

class Plugin(metaclass=Registry):
    pass

class MyPlugin(Plugin):
    pass

print(Registry.registry)   # {'Plugin': <class '...Plugin'>, 'MyPlugin': <class '...MyPlugin'>}
```

**Follow-up question:**
The example above (auto-registering subclasses) is one of the most common reasons people reach for a metaclass. What's the simpler, more modern alternative for exactly that use case?

**Follow-up good answer:**
`__init_subclass__`, a special classmethod (added in Python 3.6, PEP 487)
that Python automatically calls on the *parent* class whenever a subclass
is defined — no metaclass required. It gives you the same "run code when a
subclass is created" hook without any of the added complexity, `isinstance`
surprises, or metaclass-conflict risk (two base classes with unrelated
custom metaclasses can't always be combined; two base classes with
`__init_subclass__` hooks compose fine via normal MRO/`super()`). The
registry pattern above becomes dramatically simpler with it:

```python
class Plugin:
    registry = {}
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry[cls.__name__] = cls

class MyPlugin(Plugin):
    pass

print(Plugin.registry)   # {'MyPlugin': <class '...MyPlugin'>}
```

This is a strong signal in an interview: knowing to reach for
`__init_subclass__` (or a class decorator) over a metaclass for the common
cases shows the candidate understands metaclasses' actual cost, not just
their existence.

**Glossary:**
- **Metaclass** — a class of a class; `type` is the default, controls how classes themselves are constructed and behave.
- **`__init_subclass__`** — a classmethod hook (PEP 487) run on subclass creation, covering the most common metaclass use case without one.

**Mental model:**
This is as much a judgment question as a knowledge question: tests whether
the candidate reaches for the simplest tool that solves the problem
(class decorator / `__init_subclass__` / ABC) rather than defaulting to the
most powerful, most complex one (metaclass) because it's impressive.

**TL;DR:**
A metaclass (default `type`) controls how a class itself is constructed and behaves, but it's usually the wrong tool — `__init_subclass__` (PEP 487) or a class decorator covers the common cases (like auto-registering subclasses) with far less complexity and no metaclass-conflict risk, so reach for a metaclass only when those don't suffice.

**References:**
- [Python Data Model §3.3.3, Customizing class creation](https://docs.python.org/3/reference/datamodel.html#customizing-class-creation)
- [PEP 487 – Simpler customization of class creation](https://peps.python.org/pep-0487/)

---

### Q20. When would you reach for `@dataclass` instead of writing `__init__`/`__eq__`/`__repr__` by hand, and where does it stop being the right tool? {#q20}

**Question:**
When would you reach for `@dataclass` instead of writing `__init__`/`__eq__`/`__repr__` by hand, and where does it stop being the right tool?

**Good answer:**
`@dataclass` is for classes whose primary job is to hold a fixed, typed set
of fields with straightforward value semantics — it generates `__init__`,
`__repr__`, and (by default) `__eq__` from your type-annotated field
declarations, eliminates the boilerplate of writing all three by hand and
keeping them in sync as fields change, and — importantly — protects you
from the mutable-default-argument bug (Q16) at the class level: assigning
a bare mutable literal (`x: list = []`) as a field default raises
`ValueError` at class-definition time, forcing you to use
`field(default_factory=list)` instead, which correctly creates a fresh list
per instance. `frozen=True` gives you immutability (raises on attribute
assignment) plus, combined with `eq=True`, an auto-generated `__hash__`,
making instances usable as dict keys — a combination that's tedious to get
right by hand. It stops being the right tool once a class's job is
fundamentally *behavioral* rather than data-holding — heavy custom
validation logic in `__init__` beyond what `__post_init__` comfortably
expresses, complex invariants between fields, or a class that's mostly
methods with only incidental state — at which point a dataclass's
generated methods either don't fit or you end up overriding most of them
anyway, at which point you're paying the abstraction's conceptual overhead
without its convenience.

**Code example:**
```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Point:
    x: float
    y: float

@dataclass
class Polygon:
    points: list[Point] = field(default_factory=list)   # NOT points: list = []

p1, p2 = Point(1, 2), Point(1, 2)
print(p1 == p2)     # True -- auto-generated field-by-field __eq__
print(hash(p1))     # works -- frozen=True + eq=True auto-generates __hash__
```

**Follow-up question:**
How does choosing `@dataclass` relate to the broader dynamic-vs-static-typing trade-off in Python?

**Follow-up good answer:**
`@dataclass` leans into Python's gradual-typing story: its field
declarations use type annotations (`x: float`), but those annotations are
*not* enforced at runtime by `dataclass` itself — you get the
*documentation* and static-type-checker benefits of declared types (mypy
will flag `Point("a", "b")`) without giving up Python's dynamic flexibility
(nothing stops you from actually passing a string at runtime; `dataclass`
won't validate it). This mirrors the general Python trade-off: static
typing via annotations + a checker catches a large class of bugs *before*
runtime at zero runtime cost, but only as much as you annotate and only
if you actually run a checker in CI — it's opt-in, layered on top of a
fundamentally dynamic runtime, rather than a compiler-enforced guarantee
like in a statically-typed language. Teams that want actual runtime
validation on top of dataclass-style declarations reach for libraries like
`pydantic`, which do validate types at construction time, trading some of
`dataclass`'s zero-runtime-cost simplicity for that guarantee.

**Glossary:**
- **`field(default_factory=...)`** — the dataclasses mechanism for safely giving a field a fresh mutable default per instance.
- **Gradual typing** — Python's model of optional, checker-enforced (not runtime-enforced) type annotations layered on a dynamically-typed runtime.

**Mental model:**
Tests whether the candidate can judge when a convenience abstraction
(`dataclass`) fits a class's actual responsibility (data holder vs.
behavior-heavy object), and whether they understand type hints in Python as
a static-analysis layer rather than a runtime guarantee — a frequent
source of confusion for people newer to the language.

**TL;DR:**
`@dataclass` is the right tool for classes whose job is holding a fixed set of typed fields with value semantics — it removes `__init__`/`__eq__`/`__repr__` boilerplate and guards against the mutable-default bug via `default_factory` — but stops being appropriate once a class is fundamentally behavioral, and its type annotations are static-analysis documentation only, not runtime-enforced, unlike a library such as pydantic.

**References:**
- [dataclasses — Data Classes](https://docs.python.org/3/library/dataclasses.html)
- [typing — Support for type hints (annotations are not enforced at runtime)](https://docs.python.org/3/library/typing.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=core-language-and-data-model&autostart=1" | relative_url }})
