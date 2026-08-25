---
layout: default
title: "Python Interview: Typing & Modern Python Features"
---

# Python Interview: Typing & Modern Python Features

Twenty questions on Python's gradual type system and the modern language
features built around it — type hints and why they're optional at runtime,
structural typing via `Protocol`, how `dataclasses` actually build your
class, `match`/`case` pattern matching, `TypedDict`/`ParamSpec`/`Self`, and
the practical trade-offs of adopting static typing (and libraries like
pydantic) in a dynamically-typed language.

### Q1. Are Python type hints enforced at runtime? What actually happens if you violate one? {#q1}

**Question:**
If I write `def add(x: int, y: int) -> int: return x + y` and call `add("a", "b")`, what happens? Does Python raise a `TypeError`?

**Good answer:**
No — nothing happens at the type level. CPython does not check annotations at call time; `add("a", "b")` runs, `x + y` becomes string concatenation, and you get `"ab"` back with zero complaints, because `str.__add__` is a perfectly valid operation. The official typing docs state this explicitly: "the Python runtime does not enforce function and variable type annotations." Annotations are purely metadata — stored in `__annotations__` — that static tools (mypy, pyright), IDEs, and linters read and reason about *offline*, before the code ever runs. This is the core of Python's **gradual typing** design (PEP 484): you opt into as much or as little static checking as you want, and the interpreter's dynamic behavior is completely unaffected either way. If you want actual runtime enforcement, you need a separate library (e.g. pydantic, or manual `isinstance` checks) that inspects the annotations and validates values explicitly.

**Code example:**
```python
def add(x: int, y: int) -> int:
    return x + y

add("a", "b")  # runs fine, returns "ab" — no TypeError from typing
```

**Follow-up question:**
If type hints have zero effect on the running program, what's actually catching bugs — the annotations themselves, or something else?

**Follow-up good answer:**
The annotations themselves catch nothing; the *type checker* you run against them does. `mypy`/`pyright` are separate, offline programs that parse your source, build a static model of what types flow where, and report a diagnostic before you ever execute the code — like a very sophisticated linter with a full type-inference engine. That's why the mypy docs recommend "run mypy as part of your Continuous Integration (CI) system as soon as possible" — the value only materializes if something is actually invoking the checker and failing a build on its output. Skip that step and annotations are just readable documentation with no enforcement at all.

**Glossary:**
- **Gradual typing** — a type-system design (PEP 484) where static type annotations are optional and can be adopted incrementally alongside untyped code.
- **`__annotations__`** — the dict on functions/classes/modules where Python stores annotation objects at definition time.

**Mental model:**
Tests whether the candidate understands type hints as a *tooling* layer bolted onto a dynamically-typed runtime, versus assuming Python type hints work like Java/TypeScript's compile-time enforcement.

**TL;DR:**
Python never checks type hints at runtime — they're metadata for external tools like mypy/pyright, so violating one produces no error unless a separate type checker (or library like pydantic) is actually run against the code.

**References:**
- [typing — Support for type hints](https://docs.python.org/3/library/typing.html)
- [PEP 484 – Type Hints](https://peps.python.org/pep-0484/)
- [mypy: Using mypy with an existing codebase — CI](https://mypy.readthedocs.io/en/stable/existing_code.html)

---

### Q2. What's the difference between nominal and structural typing, and which does Python's `typing.Protocol` provide? {#q2}

**Question:**
Explain nominal vs. structural typing. Where does `typing.Protocol` fit, and how is it different from just subclassing an abstract base class?

**Good answer:**
Nominal typing means compatibility is based on declared identity — a class is only considered a `Duck` if it explicitly inherits from `Duck` (or registers as one), regardless of what methods it actually has. That's what PEP 484's original type hints gave you: `isinstance`/inheritance-based compatibility. Structural typing means compatibility is based on shape — if an object has the right methods/attributes with the right signatures, it satisfies the type, no matter what it inherits from. That's exactly what `typing.Protocol` (PEP 544) adds: a `Protocol` class defines a set of methods, and any object that happens to implement them satisfies it for a type checker, with **no explicit inheritance required**. PEP 544 frames this directly: PEP 484 "only specifies the semantics of nominal subtyping," and protocols exist to "provide support for structural subtyping (static duck typing)." Concretely, a class with `def __len__(self) -> int` and `def __iter__(self)` satisfies a `SizedIterable` protocol automatically, without ever mentioning that protocol's name.

**Code example:**
```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

class FileLike:            # no inheritance from SupportsClose at all
    def close(self) -> None:
        print("closed")

def shut_down(x: SupportsClose) -> None:
    x.close()

shut_down(FileLike())      # type-checks fine — structural match
```

**Follow-up question:**
Can you use a `Protocol` with `isinstance()` at runtime the same way you'd use an ABC?

**Follow-up good answer:**
Only if you opt in explicitly. By default, protocols are a purely static-analysis construct — `isinstance()` against a plain `Protocol` doesn't work reliably. You have to decorate it with `@runtime_checkable`, per the PEP: "a protocol can be used as a second argument in `isinstance()` and `issubclass()` only if it is explicitly opt-in by `@runtime_checkable` decorator." Even then, it's a shallow check — it only verifies the *names* of the required methods exist on the object, not their signatures, so it's weaker than what the static type checker verifies at analysis time.

**Glossary:**
- **Nominal typing** — type compatibility based on declared class identity/inheritance.
- **Structural typing** — type compatibility based on an object's actual shape (methods/attributes), independent of inheritance. Sometimes called "static duck typing."
- **`@runtime_checkable`** — decorator that enables (shallow, presence-only) `isinstance()` checks against a `Protocol`.

**Mental model:**
Probes whether the candidate can connect Python's long-standing duck-typing culture to a formal type-system concept, and knows precisely where the formalization (Protocol) stops short of full runtime guarantees.

**TL;DR:**
Nominal typing checks declared inheritance; structural typing (Python's `Protocol`, PEP 544) checks whether an object merely has the right methods — formalizing duck typing for static checkers, with `isinstance()` support only via opt-in `@runtime_checkable`.

**References:**
- [PEP 544 – Protocols: Structural subtyping (static duck typing)](https://peps.python.org/pep-0544/)
- [typing — Protocol, runtime_checkable](https://docs.python.org/3/library/typing.html)

---

### Q3. What exactly does the `@dataclass` decorator generate for you? {#q3}

**Question:**
When you write `@dataclass class Point: x: int; y: int`, what does the decorator actually do to the class at class-creation time?

**Good answer:**
`@dataclass` inspects the class's type-annotated class variables (`x: int`, `y: int`) and, by default, synthesizes several dunder methods based on them, in the order the fields were declared: an `__init__(self, x, y)` that assigns each parameter to `self`; a `__repr__` that renders something like `Point(x=1, y=2)`; and an `__eq__` that compares instances field-by-field (and requires both sides be the exact same type). Per the docs: "If true (the default), an `__eq__() `method will be generated... This method compares the class as if it were a tuple of its fields, in order." None of this touches the class body's actual runtime type — it's still a normal class; the decorator just writes boilerplate methods you'd otherwise hand-write, driven purely by the annotations (which is a nice irony: this is one of the few places annotations *do* have a runtime effect, because `dataclass` explicitly reads `__annotations__` at decoration time to know what fields exist).

**Code example:**
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p1, p2 = Point(1, 2), Point(1, 2)
print(p1)          # Point(x=1, y=2)  <- generated __repr__
print(p1 == p2)    # True             <- generated __eq__
```

**Follow-up question:**
Why does `x: list = []` as a dataclass field raise `ValueError` instead of just silently doing the wrong thing like a regular default argument would?

**Follow-up good answer:**
Because `dataclass` specifically guards against the classic Python mutable-default footgun instead of reproducing it. A plain default argument (`def f(x=[])`) shares one list object across every call that doesn't pass `x`; `dataclass` refuses to let that happen silently for its generated `__init__` — the docs say it "will raise a `ValueError` if it detects a default parameter of type `list`, `dict`, or `set`" (generalized: any unhashable value, since "the assumption is that if a value is unhashable, it is mutable"). Instead you're required to use `field(default_factory=list)`, which stores a zero-argument callable and invokes it fresh for every new instance, so each instance gets its own list rather than sharing one.

**Glossary:**
- **`field()`** — a `dataclasses` helper for per-field configuration (defaults, `default_factory`, whether a field appears in `__init__`/`__repr__`/comparisons, etc.).
- **Mutable default argument pitfall** — the classic Python bug where a mutable default (e.g. `[]`) is created once at function-definition time and shared across all calls.

**Mental model:**
Tests whether the candidate has actually read past "dataclasses save boilerplate" to understand it's a compile-time-ish (decoration-time) code generator driven by annotations, and whether they know *why* one specific safety rail exists.

**TL;DR:**
`@dataclass` reads the class's annotated fields at decoration time and generates `__init__`/`__repr__`/`__eq__` from them, and it deliberately raises `ValueError` on mutable defaults like `[]`, forcing `field(default_factory=list)` instead to avoid the shared-mutable-default bug.

**References:**
- [dataclasses — Data Classes](https://docs.python.org/3/library/dataclasses.html)

---

### Q4. What's the difference between a bounded and a constrained `TypeVar`? {#q4}

**Question:**
`TypeVar('S', bound=str)` vs. `TypeVar('A', str, bytes)` — what's the practical difference for a type checker?

**Good answer:**
Both restrict what a generic parameter can be, but differently. A **bounded** TypeVar (`bound=str`) accepts `str` *or any subtype of it* — the checker infers the most specific actual type used at each call site, so passing a `str` subclass keeps that subclass's identity through the generic function. A **constrained** TypeVar (`str, bytes`) is stricter: per the docs, "using a constrained type variable... means that the TypeVar can only ever be solved as being exactly one of the constraints given" — the checker collapses the inferred type to exactly `str` or exactly `bytes`, not some subtype of either, and a single call can't mix both. Constrained TypeVars are for genuinely disjoint valid types where you want the checker to pick one lane; bounded TypeVars are for "anything that behaves like X (including subclasses)."

**Code example:**
```python
from typing import TypeVar

S = TypeVar("S", bound=str)          # str or any subclass of str
A = TypeVar("A", str, bytes)         # exactly str, or exactly bytes

def first_bounded(x: S) -> S: ...
def first_constrained(x: A) -> A: ...
```

**Follow-up question:**
When would you reach for a constrained TypeVar over just using a `Union[str, bytes]` parameter type directly?

**Follow-up good answer:**
Use the constrained TypeVar when the function's *return type depends on which branch of the union was passed in* — i.e., you need to preserve the relationship between an argument's specific type and the return type across the signature. `def f(x: Union[str, bytes]) -> Union[str, bytes]` loses that link entirely: the checker can't prove that passing `bytes` in guarantees `bytes` out. `def f(x: A) -> A` (with `A = TypeVar('A', str, bytes)`) does prove it, because the same TypeVar solved once applies to both the parameter and the return type. If the function doesn't correlate input and output types this way, a plain union is simpler and clearer.

**Glossary:**
- **`TypeVar`** — a placeholder representing a generic type parameter, later solved by the type checker per call site.
- **Bound** — restricts a TypeVar to a type or its subtypes.
- **Constraint** — restricts a TypeVar to being resolved as exactly one of an explicit list of types.

**Mental model:**
Checks whether the candidate understands generics well enough to know these aren't interchangeable spellings of "restrict the type" — each encodes a different relationship the checker can (or can't) prove.

**TL;DR:**
A bounded TypeVar accepts a type or any subtype of it and preserves that subtype through the function; a constrained TypeVar must resolve to exactly one of the listed types, useful when you need input/output types to stay correlated across a signature.

**References:**
- [typing — TypeVar](https://docs.python.org/3/library/typing.html)

---

### Q5. How does Python's `match`/`case` decide which branch runs — and how is that different from a C-style `switch`? {#q5}

**Question:**
Walk through how `match` evaluates its `case` blocks. Is it just comparing values like a `switch`?

**Good answer:**
No — `match` evaluates **patterns**, not just equality against literal values, and it stops at the first one that succeeds; there's no fallthrough to worry about. Per the language reference: each `case` pattern is attempted against the subject in source order, and if a pattern matches, its **optional guard** (`if` clause) is checked — the case only executes if the guard is true or absent, otherwise the engine moves on to the *next* case block entirely (not "falls through" the current one). Patterns can be far richer than equality: capture patterns bind a name unconditionally (`case x:` always matches and binds `x`), sequence/mapping patterns destructure structurally, and **class patterns** check `isinstance()` against a type and then match constructor-style sub-patterns against attributes — converting positional sub-patterns to keyword ones via the class's `__match_args__`. So `match` is closer to structural/algebraic pattern matching (as in Haskell/Rust) than to C's jump-table `switch`.

**Code example:**
```python
match command:
    case ("go", direction) if direction in ("n", "s", "e", "w"):
        move(direction)
    case ("go", _):
        print("unknown direction")
    case Point(x=0, y=0):        # class pattern via __match_args__
        print("origin")
    case _:
        print("unrecognized")
```

**Follow-up question:**
What does `__match_args__` do, and how does a class pattern like `Point(0, 0)` use it?

**Follow-up good answer:**
`__match_args__` is a class-level tuple of attribute names that tells `match` how to interpret *positional* arguments inside a class pattern. Per the reference, when a class pattern has positional sub-patterns, the engine does the equivalent of `getattr(cls, "__match_args__", ())` and converts positional pattern `i` into a keyword pattern using `__match_args__[i]` as the attribute name — so `case Point(0, 0)` becomes, in effect, `case Point(x=0, y=0)` if `Point.__match_args__ == ("x", "y")`. Dataclasses set this automatically (in field-declaration order), which is why `@dataclass class Point: x: int; y: int` supports positional matching for free; a hand-written class needs to define `__match_args__` itself to get the same behavior.

**Glossary:**
- **Subject** — the value being matched (the expression after `match`).
- **Guard** — an optional `if` condition attached to a `case` that must also be true for that case to run.
- **`__match_args__`** — class attribute listing which attributes positional sub-patterns in a class pattern map to.

**Mental model:**
Tests whether the candidate understands `match` as structural pattern matching with binding and destructuring, not a rebranded `switch` — a common surface-level misconception.

**TL;DR:**
`match` tries each `case` pattern in order and stops at the first structural match whose guard (if any) passes — no fallthrough — and class patterns convert positional sub-patterns to attribute checks via `__match_args__`, which dataclasses populate automatically.

**References:**
- [The match statement — Python Language Reference](https://docs.python.org/3/reference/compound_stmts.html#the-match-statement)
- [PEP 634 – Structural Pattern Matching: Specification](https://peps.python.org/pep-0634/)

---

### Q6. Static typing adds friction to a dynamic language. What real production problem does it actually solve? {#q6}

**Question:**
Given Python already worked for 25+ years without mandatory types, what's the concrete payoff that made PEP 484 worth adding?

**Good answer:**
The payoff scales with codebase size and team size, not with any individual script. On a large codebase, the failure mode static typing targets isn't "the code crashes" — dynamically-typed Python already surfaces plenty of `AttributeError`/`TypeError` at runtime — it's that those errors surface *late*, often in production, far from the line that caused them, because nothing stops a caller from passing the wrong shape of object until it's actually exercised. Type hints let IDEs and CI catch a whole class of these mismatches statically, before a human or a test even runs that path, and they double as accurate, checked documentation of a function's contract (a docstring can lie about types; a type checker won't let annotations drift the same way without complaining). PEP 484 is explicit that the goal was adding this *without* compromising what makes Python Python: "Python will remain a dynamically typed language, and the authors have no desire to ever make type hints mandatory, even by convention" — it's an opt-in safety net, not a rewrite of the language's semantics.

**Follow-up question:**
If a team already has 100% test coverage, does static typing still add value?

**Follow-up good answer:**
Yes, because tests and types catch different classes of problems and neither subsumes the other. Tests verify specific *behaviors* for the inputs you thought to write cases for; a type checker verifies a *contract* holds for every call site in the codebase, including ones nobody wrote a test for yet, and it does so in seconds on every keystroke in an IDE rather than requiring a test run. Types also catch a category tests routinely miss: internal refactors that silently break a caller's assumptions three modules away, which only a whole-codebase static pass (not a unit test scoped to one module) reliably flags immediately. In practice they're complementary risk-reduction layers, not substitutes — high test coverage reduces the marginal value of typing somewhat, but doesn't eliminate the argument for it on any codebase big enough to have callers a change's author isn't personally tracking.

**Glossary:**
- **Static analysis** — checking a program's properties without executing it (as opposed to runtime/dynamic checks like tests).

**Mental model:**
Looks for whether the candidate can articulate the actual engineering trade-off (catch-early vs. catch-late, contract-for-everyone vs. behavior-for-what-you-tested) rather than reciting "types are good" as dogma.

**TL;DR:**
Static typing's real payoff is catching contract violations across an entire codebase before runtime and keeping documentation honest at scale — a complement to tests, not a replacement, since PEP 484 deliberately kept it optional rather than changing Python's dynamic runtime semantics.

**References:**
- [PEP 484 – Type Hints: Rationale](https://peps.python.org/pep-0484/)

---

### Q7. Why does `Any` quietly undermine a codebase's type safety if it's overused? {#q7}

**Question:**
What does `typing.Any` mean precisely, and why is sprinkling it everywhere (to silence errors) worse than it looks?

**Good answer:**
`Any` is special-cased in the type system to be "consistent with... all types" — a value typed `Any` can be assigned to a variable of any type, and a value of any type can be assigned to an `Any`-typed variable, with the checker verifying nothing in either direction. That's precisely its danger: it isn't "unknown, please infer" — it's "stop checking this, trust me." Once a value flows through an `Any`-typed parameter, return type, or variable, the checker's guarantees don't just weaken locally, they can propagate: anything derived from that `Any` is often inferred as `Any` too, silently widening the untyped surface far past the one line where it was introduced. A codebase riddled with `Any` (often from untyped third-party libraries, or from developers reaching for it to make an error go away) can look fully type-annotated while actually verifying almost nothing — which is worse than having no annotations at all, because it gives false confidence.

**Follow-up question:**
What's a narrower, safer alternative to `Any` when you genuinely don't know or don't want to constrain a value's type yet?

**Follow-up good answer:**
`object` is the honest alternative when you truly want "anything" with the checker still enforcing something: unlike `Any`, `object` is a real type at the top of the hierarchy, so the checker *will* flag any operation on it that isn't valid for arbitrary objects (you can't call `.upper()` on something typed `object` without first narrowing it, e.g. via `isinstance`). If the goal is "accept several specific types," a `Union`/constrained `TypeVar` documents the actual contract instead of opting out of checking entirely. `Any` should be reserved for genuine escape hatches — interfacing with untyped code, or intentionally-dynamic patterns — not as a default reach when a real annotation is just inconvenient to write.

**Glossary:**
- **`Any`** — the type that is compatible with every other type in both directions, effectively disabling static checking for that value.
- **Type narrowing** — refining a broad static type (like `object`) to a more specific one within a code branch, typically via `isinstance` or similar checks.

**Mental model:**
Distinguishes candidates who've only used `Any` as an error-suppression reflex from those who understand it as a deliberate, bidirectional escape hatch with a real propagation cost.

**TL;DR:**
`Any` is bidirectionally compatible with every type, so overusing it doesn't just leave one value unchecked — it silently propagates "unchecked" through everything derived from it, creating false confidence; `object` or a proper `Union`/TypeVar is usually the safer choice.

**References:**
- [typing — The Any type](https://docs.python.org/3/library/typing.html)

---

### Q8. Why would `class Config: name: str; timeout: int = 30` (a plain annotated class, no decorator) behave surprisingly if you expected it to work like a dataclass? {#q8}

**Question:**
If I write a plain class with class-level type annotations but forget `@dataclass`, what actually happens when I try `Config(name="x")`?

**Good answer:**
It fails, and for a subtle reason: bare class-level annotations without `@dataclass` don't generate an `__init__` at all — `x: int` at class scope is *just an annotation*, recorded in `Config.__annotations__`, with no assignment and no generated constructor logic. So `Config(name="x")` raises `TypeError: Config() takes no arguments` (falling back to `object.__init__`), because nothing ever taught the class how to accept and store constructor arguments — the annotation looks like it's declaring a field, but on its own it's inert metadata, exactly like the annotations discussed in Q1. This is a common trap for developers coming from languages where field declarations imply storage and initialization automatically; in Python, `@dataclass` (or a hand-written `__init__`) is what actually wires annotations to real per-instance attributes.

**Code example:**
```python
class Config:
    name: str
    timeout: int = 30      # this IS a real class attribute (has a value)

Config(name="x")           # TypeError: Config() takes no arguments
# `name: str` alone created no instance attribute and no __init__ param
```

**Follow-up question:**
In that example, is `timeout` treated the same way as `name`?

**Follow-up good answer:**
No — `timeout: int = 30` has an actual assigned value, so unlike `name: str`, it *does* become a real class attribute (`Config.timeout == 30`), accessible via normal attribute lookup, exactly as if you'd written `timeout = 30` with no annotation at all; the `: int` part is still just metadata layered on top. `name: str` has no value, so it produces an annotation entry only — no attribute exists until something (like `@dataclass`, or a hand-written `__init__`) actually assigns to `self.name`. This is exactly why `@dataclass` needs to inspect `__annotations__` specifically to know which class variables are meant to be *fields* to synthesize into `__init__`, rather than just reading the class's existing attributes.

**Glossary:**
- **Class-level annotation** — a bare `name: type` statement at class body scope; adds an entry to `__annotations__` but does not create an attribute unless also assigned a value.

**Mental model:**
Surfaces whether the candidate actually understands the mechanics behind Q1/Q3 rather than having memorized "dataclasses use type hints" as a fact without knowing what a plain annotation does on its own.

**TL;DR:**
A bare class-level annotation with no assigned value (`name: str`) creates no instance attribute and no constructor logic on its own — it's inert metadata — which is exactly the gap `@dataclass` fills by reading `__annotations__` to generate a real `__init__`.

**References:**
- [The Python Language Reference — Annotated assignment statements](https://docs.python.org/3/reference/simple_stmts.html#annotated-assignment-statements)
- [dataclasses — Data Classes](https://docs.python.org/3/library/dataclasses.html)

---

### Q9. What problem does `typing.TYPE_CHECKING` solve, and why can't you just always import the type normally? {#q9}

**Question:**
Why would you write `if TYPE_CHECKING: import Foo` instead of a normal top-level `import Foo`?

**Good answer:**
`TYPE_CHECKING` is a special constant that is `False` at actual runtime but which static type checkers treat as `True` when analyzing your code. Guarding an import with it lets you make a name available *only for annotation purposes* — the type checker sees the import and resolves the annotation correctly, but the interpreter never executes that import at runtime, because the `if` branch is always false when the program actually runs. This solves circular-import problems: if module A needs a type from module B purely for annotations, and module B already imports module A for real logic, a normal top-level `import B` in A would create an import cycle that can crash at startup — but a `TYPE_CHECKING`-guarded import never executes, so the cycle never actually forms at runtime, only in the type checker's static view of the world (which handles cycles fine since it doesn't execute code).

**Code example:**
```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from heavy_module import HeavyType   # never actually imported at runtime

def process(x: "HeavyType") -> None:     # quoted forward reference
    ...
```

**Follow-up question:**
Why is the parameter annotated as the string `"HeavyType"` instead of bare `HeavyType`?

**Follow-up good answer:**
Because at the point the function is *defined*, `HeavyType` doesn't exist as a real name in that module's namespace — the import that would bind it never runs (it's inside the `TYPE_CHECKING`-guarded branch). If the annotation were bare `HeavyType`, Python would try to evaluate it immediately at function-definition time and raise `NameError`. Quoting it makes it a **forward reference**: a string that Python stores as-is without evaluating, which type checkers parse and resolve against their own (TYPE_CHECKING=True) view of the world, where the import did happen. Alternatively, `from __future__ import annotations` postpones evaluation of *all* annotations in a module automatically, making the quotes unnecessary — but the underlying reason (the name genuinely doesn't exist at runtime) is the same either way.

**Glossary:**
- **Forward reference** — a type annotation written as a string, deferring its resolution so it can refer to a name not yet defined (or never defined) at runtime.
- **Circular import** — a dependency cycle between modules that can fail depending on import order, since a module may be only partially initialized when re-entered.

**Mental model:**
Tests whether the candidate has hit this exact real-world pain point (typing-only circular imports) rather than only knowing `TYPE_CHECKING` exists as trivia.

**TL;DR:**
`TYPE_CHECKING` is `False` at runtime but treated as `True` by static checkers, so guarding an import with it makes a type available for annotations without ever executing that import at runtime — breaking circular-import chains that only exist for typing purposes.

**References:**
- [typing — TYPE_CHECKING](https://docs.python.org/3/library/typing.html#typing.TYPE_CHECKING)

---

### Q10. When would you choose a `dataclass` over a `NamedTuple`, or vice versa? {#q10}

**Question:**
Both give you a lightweight structured value with generated `__init__`/`__repr__`. What actually differs, and how does that drive the choice?

**Good answer:**
The fundamental difference is what they're built on: a `NamedTuple` *is* a tuple subclass, so instances are immutable, indexable/iterable like a tuple, unpackable (`x, y = point`), and — per the docs — memory-lean because "named tuple instances do not have per-instance dictionaries, so they are lightweight and require no more memory than regular tuples." A `dataclass` is a normal class (mutable by default, though `frozen=True` can lock it down) with a `__dict__` per instance unless you also add `__slots__`, and it doesn't behave like a tuple at all — no automatic unpacking or positional indexing. Practically: reach for `NamedTuple` when the value is a small, truly immutable, tuple-like bundle (e.g. a return value you want to unpack, coordinates, a cache key), and reach for `dataclass` when you want a mutable object with more fields, richer per-field configuration (`field(default_factory=...)`, `init=False` fields, etc.), or when you plan to add real methods/behavior — dataclasses read more like "this is a class," `NamedTuple`s read more like "this is a structured tuple."

**Code example:**
```python
from typing import NamedTuple
from dataclasses import dataclass

class PointT(NamedTuple):   # immutable, tuple-like
    x: int
    y: int

@dataclass
class PointD:                # mutable class, dict-backed by default
    x: int
    y: int

x, y = PointT(1, 2)          # tuple unpacking works
```

**Follow-up question:**
Does using `dataclass(frozen=True)` make it behave identically to a `NamedTuple`?

**Follow-up good answer:**
No — `frozen=True` only adds immutability (it makes `__setattr__`/`__delattr__` raise `FrozenInstanceError`); it doesn't turn the instance into a tuple. A frozen dataclass instance still isn't iterable or unpackable by position (`x, y = frozen_point` fails unless you implement `__iter__` yourself), still carries a `__dict__` unless you also declare `__slots__`, and still compares by field values via the generated `__eq__` rather than tuple identity semantics. So `frozen=True` closes the mutability gap but not the structural one — if you specifically need tuple behavior (unpacking, indexing, being usable as a dict key without extra plumbing), `NamedTuple` still gives you that for free where a frozen dataclass doesn't.

**Glossary:**
- **`NamedTuple`** — a `tuple` subclass with named, typed fields, generated via `typing.NamedTuple` or `collections.namedtuple`.
- **`frozen=True`** — a `dataclass` option that makes instances immutable by raising on attribute assignment after `__init__`.

**Mental model:**
Checks whether the candidate picks data structures based on actual behavioral requirements (mutability, tuple semantics, memory) rather than treating "dataclass vs NamedTuple" as an arbitrary style preference.

**TL;DR:**
`NamedTuple` is an immutable, memory-lean tuple subclass with automatic unpacking/indexing; `dataclass` is a normal (optionally frozen) class with richer per-field configuration — pick based on whether you actually need tuple semantics or class semantics.

**References:**
- [collections — namedtuple()](https://docs.python.org/3/library/collections.html#collections.namedtuple)
- [dataclasses — frozen instances](https://docs.python.org/3/library/dataclasses.html)

---

### Q11. `TypedDict` type-checks dictionary shapes, but what actually exists at runtime? {#q11}

**Question:**
If I define `class Point2D(TypedDict): x: int; y: int`, what kind of object do I get back when I create one?

**Good answer:**
A plain `dict` — nothing more. The docs are blunt about it: "at runtime, `TypedDict` instances are simply dicts." `TypedDict` exists purely as a static-typing construct so a type checker can verify that a dict literal or variable has exactly the keys and value types you declared (flagging a missing key, an extra key, or a wrong-typed value as a static error) — but there's no runtime class, no validation, no generated `__init__`, nothing intercepting dict operations. `{"x": 1, "y": 2}` and an object explicitly annotated as `Point2D` are the *exact same runtime object*; the only difference is what the type checker is willing to assume about its shape while analyzing your code.

**Code example:**
```python
from typing import TypedDict

class Point2D(TypedDict):
    x: int
    y: int

p: Point2D = {"x": 1, "y": 2}
print(type(p))     # <class 'dict'>  — not Point2D, there's no such runtime class
```

**Follow-up question:**
Since there's no runtime validation, what happens if untrusted data (e.g. parsed JSON) is assigned to a `TypedDict`-annotated variable but doesn't actually match the shape?

**Follow-up good answer:**
Nothing stops it — the type checker only verifies static assignments/literals it can see at analysis time; it has no way to validate data arriving dynamically at runtime (a JSON payload from a network call, for instance), so a malformed dict will happily flow through code annotated with `Point2D` with zero errors, static or dynamic, until something actually tries to use a missing key and raises `KeyError` deep in unrelated code. `TypedDict` is a documentation/checker-assistance tool for *code you control*, not a validation boundary for external input. For that, you need an actual runtime-validating library (e.g. pydantic, or manual key/type checks) at the boundary where untrusted data enters the system.

**Glossary:**
- **`TypedDict`** — a typing construct declaring the expected keys/value-types of a dict for static checking, with no runtime representation of its own.

**Mental model:**
Tests whether the candidate can generalize the "annotations aren't enforced" theme from Q1 to a construct that specifically *looks* like it should validate something, since it resembles a schema.

**TL;DR:**
A `TypedDict` produces a plain `dict` at runtime with no validation whatsoever — it only lets a static type checker flag shape mismatches it can see in your source, so it does nothing to guard against malformed data arriving dynamically.

**References:**
- [typing — TypedDict](https://docs.python.org/3/library/typing.html)
- [PEP 589 – TypedDict](https://peps.python.org/pep-0589/)

---

### Q12. What problem does `ParamSpec` solve that a plain `TypeVar` can't? {#q12}

**Question:**
Why do you need `ParamSpec` specifically for typing a generic decorator, instead of just using `Callable[..., T]`?

**Good answer:**
`Callable[..., T]` erases all information about the wrapped function's *parameters* — the `...` means "any arguments, we're not checking them," so a decorator typed that way loses the original function's signature entirely from the caller's point of view; a type checker can no longer flag a wrong-typed or missing argument on the decorated function. `ParamSpec` (PEP 612) exists to capture and forward an *entire parameter list* as a unit, not just a single type: it introduces `P.args`/`P.kwargs` to annotate `*args`/`**kwargs` inside the wrapper, and using the same `P` on both the inner callable's parameters and the wrapper's parameters tells the checker "whatever the original signature is, faithfully preserve it here." This is specifically for the shape of decorators that don't change the call signature — only wrapping behavior around it.

**Code example:**
```python
from typing import ParamSpec, TypeVar, Callable
P = ParamSpec("P")
T = TypeVar("T")

def add_logging(f: Callable[P, T]) -> Callable[P, T]:
    def inner(*args: P.args, **kwargs: P.kwargs) -> T:
        print(f"calling {f.__name__}")
        return f(*args, **kwargs)
    return inner
```

**Follow-up question:**
What does `Concatenate` add on top of `ParamSpec`?

**Follow-up good answer:**
`Concatenate` (introduced alongside `ParamSpec` in the same PEP) lets you prepend fixed parameter types *before* a captured `ParamSpec`, for decorators that inject an extra argument into the call rather than just forwarding everything unchanged — for example, a decorator that adds a `logger` argument as the wrapped function's first parameter. Without `Concatenate`, `ParamSpec` alone can only express "forward the exact same signature"; `Concatenate[LoggerType, P]` expresses "the same signature, but with one extra required parameter inserted at the front," which a plain `ParamSpec` has no syntax for.

**Glossary:**
- **`ParamSpec`** — a typing construct (PEP 612) that captures an entire callable's parameter list as a single generic unit, used via `P.args`/`P.kwargs`.
- **`Concatenate`** — lets fixed parameter types be prepended to a captured `ParamSpec` in a signature.

**Mental model:**
Distinguishes candidates who've only typed simple functions from those who've had to correctly type a decorator-heavy codebase, where signature-preservation is a real, recurring problem.

**TL;DR:**
`ParamSpec` (PEP 612) captures a whole parameter list as one unit via `P.args`/`P.kwargs`, letting a decorator forward the original function's exact signature to callers instead of erasing it to `Callable[..., T]`; `Concatenate` extends this to decorators that also inject an extra parameter.

**References:**
- [PEP 612 – Parameter Specification Variables](https://peps.python.org/pep-0612/)
- [typing — ParamSpec](https://docs.python.org/3/library/typing.html)

---

### Q13. What does `typing.Self` let you express that a regular `TypeVar` bound to the class couldn't already? {#q13}

**Question:**
Before PEP 673 added `Self`, people typed "returns an instance of my own class" with a TypeVar bound to the class. Why add a dedicated `Self` type at all?

**Good answer:**
`Self` doesn't add new expressive power over a class-bound `TypeVar` — the docs state it's "semantically equivalent to... using a TypeVar bound to the class, except in a more succinct fashion." The value is purely ergonomic and correctness-by-default: without `Self`, correctly typing "returns an instance of whatever concrete subclass called this" required manually declaring a `TypeVar`, binding it to the class, and threading it through the signature — easy to get subtly wrong (e.g. forgetting the bound, or accidentally reusing a shared TypeVar across unrelated methods) and verbose enough that many codebases just skipped it and returned the base class type, silently losing precision for subclasses. `Self` makes the pattern a single reserved name with no extra ceremony, so it actually gets used consistently.

**Code example:**
```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:
        self.name = name
        return self

class SpecialBuilder(Builder):
    pass

# reveal_type(SpecialBuilder().set_name("x")) -> "SpecialBuilder", not "Builder"
```

**Follow-up question:**
Give a concrete case where returning `Self` instead of the declared class name actually changes what a type checker will accept.

**Follow-up good answer:**
Fluent/chainable builder methods on a subclass, and alternative-constructor classmethods, are the classic cases. If `Builder.set_name` were annotated to return `Builder` (not `Self`), then `SpecialBuilder().set_name("x").special_method()` would fail static checking — the checker only knows the chain produced a `Builder`, which has no `special_method`. Annotated with `Self`, the checker correctly infers the return type as `SpecialBuilder` for that specific call, so the chained `.special_method()` call type-checks. The same applies to `__enter__(self) -> Self` on context managers with subclasses, and classmethod constructors like `def create(cls) -> Self: return cls()`, which need to return the *actual* subclass when called on a subclass, not the base class.

**Glossary:**
- **`Self`** — a typing construct (PEP 673) meaning "the type of the current class instance," correctly narrowing to a subclass when called on one.

**Mental model:**
Checks whether the candidate understands `Self` as solving a real, recurring subclassing correctness problem rather than being cosmetic sugar with no behavioral difference.

**TL;DR:**
`Self` (PEP 673) is shorthand for a class-bound `TypeVar`, but it matters in practice for chainable methods and alternative constructors on subclasses — annotating a return as the base class name (instead of `Self`) makes a type checker lose the subclass's type through method chains.

**References:**
- [PEP 673 – Self Type](https://peps.python.org/pep-0673/)
- [typing — Self](https://docs.python.org/3/library/typing.html)

---

### Q14. Where does pydantic's runtime validation actually cost you something that a pure `dataclass` + type hints wouldn't? {#q14}

**Question:**
Since type hints don't validate anything at runtime (Q1), why not just always reach for pydantic instead of a plain `dataclass`? What's the trade-off?

**Good answer:**
Pydantic deliberately trades runtime cost for runtime safety: unlike a plain `dataclass` (which just assigns whatever values it's given, unchecked), pydantic's `BaseModel` actively parses and validates every field against its declared type *at instantiation*, coercing where it can and raising a structured `ValidationError` where it can't. That validation is real work on every object creation — attribute lookups, type/coercion logic, potentially nested-model validation — which is measurable overhead a `dataclass` simply doesn't pay, since a dataclass's generated `__init__` is a straight assignment with no type-checking logic at all. The trade-off is where you want the cost to live: pydantic is the right choice at trust boundaries (API request bodies, config file parsing, anything from outside your process) where the validation *is* the point; a plain dataclass is the right, cheaper choice for internal, already-trusted data moving between functions you control, where you're relying on static type checking (mypy/pyright) rather than per-instantiation runtime checks.

**Follow-up question:**
If a team is CPU-bound and creates millions of pydantic model instances per second in a hot path, what would you actually check before assuming pydantic is the bottleneck?

**Follow-up good answer:**
Profile before concluding anything — measure with `cProfile`/`py-spy` to confirm model instantiation (not something else entirely, like I/O or serialization) is actually where time goes, since intuition about "validation must be slow" is often wrong relative to the real hotspot. If it *is* confirmed, check the pydantic version and config first: pydantic v2's core validation is implemented in Rust (`pydantic-core`), which is dramatically faster than v1's pure-Python validators, so an easy win is simply being on v2. Beyond that, look at whether the hot-path data genuinely needs re-validation on every call (data already validated once at the trust boundary doesn't need re-validating internally — that's a case for converting to a plain dataclass/plain object after the boundary) versus whether it's being re-parsed redundantly deeper in the pipeline.

**Glossary:**
- **`BaseModel`** — pydantic's base class for models that validates and coerces field values at instantiation time.
- **Trust boundary** — the point where data crosses from outside control (network, file, user input) into your program's internal logic.

**Mental model:**
Probes whether the candidate can reason about a real performance/safety trade-off with a methodology (measure, then decide) rather than picking a library by reputation alone.

**TL;DR:**
Pydantic pays real per-instantiation validation cost that a plain `dataclass` doesn't, which is worth it at trust boundaries (external data) but wasteful for already-trusted internal data-passing — and any suspected pydantic hot path should be profiled, not assumed, before optimizing.

**References:**
- [Pydantic — Models](https://docs.pydantic.dev/latest/concepts/models/)
- [Pydantic — Performance (pydantic-core)](https://docs.pydantic.dev/latest/concepts/performance/)

---

### Q15. What happens if a `Protocol`'s method has an incompatible signature on the implementing class — does structural matching just check method *names*? {#q15}

**Question:**
If `SupportsClose` declares `def close(self) -> None`, and my class defines `def close(self, force: bool) -> None` (an extra required argument), does it still satisfy the protocol?

**Good answer:**
No — for full static structural matching (as opposed to the shallow `@runtime_checkable` `isinstance` check from Q2), the type checker verifies full method compatibility, not just that a same-named method exists. That includes parameter compatibility (an implementation can't *require* an extra parameter the protocol's callers won't supply) and return-type compatibility. A `close(self, force: bool)` would fail to satisfy `SupportsClose`, because any code calling through the protocol type does `x.close()` with no arguments — that call would break against the real implementation. Compatible overrides are allowed to *widen* what they accept (e.g. add an optional parameter with a default) and *narrow* what they return, following the same variance rules as normal nominal-subtyping method overrides; they just can't demand something the protocol's contract doesn't promise to provide.

**Code example:**
```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

class Bad:
    def close(self, force: bool) -> None: ...   # extra required param

def shut_down(x: SupportsClose) -> None:
    x.close()

shut_down(Bad())   # type checker error: Bad doesn't satisfy SupportsClose
```

**Follow-up question:**
Does an *optional* extra parameter (with a default) fix this?

**Follow-up good answer:**
Yes — `def close(self, force: bool = False) -> None` satisfies `SupportsClose`, because every call the protocol promises (`x.close()`, no arguments) still works against that implementation; the extra parameter is never required. This mirrors ordinary method-override compatibility rules outside of protocols too: a subclass/implementation is free to accept a superset of calls (optional extra parameters) but never a subset (fewer accepted calls, or new required parameters), because callers relying on the narrower/original contract must keep working unchanged.

**Glossary:**
- **Structural matching (full)** — full static verification that an implementation's method signatures are call-compatible with a Protocol's declared methods, beyond just matching method names.

**Mental model:**
Checks whether the candidate assumes Protocol matching is a superficial "does this name exist" check (like the shallow runtime version) versus knowing static checkers actually verify call-compatibility.

**TL;DR:**
Static Protocol matching checks full signature compatibility, not just method names — an implementation can't add a *required* parameter the protocol doesn't promise callers will supply, though optional parameters with defaults are fine.

**References:**
- [PEP 544 – Protocols: Protocol members](https://peps.python.org/pep-0544/)

---

### Q16. How does adopting mypy on a large, previously untyped codebase actually happen in practice — do you annotate everything on day one? {#q16}

**Question:**
You've been asked to introduce mypy on a 300,000-line untyped Flask codebase. Do you turn on `--strict` and start fixing every error?

**Good answer:**
No — that's a recipe for the initiative stalling before it produces any value, given the sheer error volume `--strict` would surface immediately on fully-untyped code. The documented, practical path is incremental: pick a small, contained subset of the codebase first (the mypy docs suggest something like "5,000 to 50,000 lines") and get mypy running cleanly on *just that subset*, using per-module config (like `ignore_errors = True` for everything not yet in scope) rather than annotating the whole codebase up front. From there, get mypy wired into CI immediately for the covered subset — "run mypy as part of your Continuous Integration (CI) system as soon as possible" — so newly-written code in that scope can't regress, then expand coverage module by module, prioritizing widely-imported utility modules early since annotating them improves inference everywhere that imports them. `--strict` becomes a later-stage goal per module, not a day-one setting for the whole repo.

**Follow-up question:**
Why prioritize annotating "widely-imported utility modules" early instead of, say, the newest feature code?

**Follow-up good answer:**
Because type information composes through imports — if a shared utility function has no annotations, every caller of it gets weaker inference regardless of how well-annotated the caller's own code is (the call's return type is effectively unknown/`Any`-ish to the checker). Annotating a handful of heavily-imported low-level modules improves the checker's ability to catch real errors across a large surface of dependent code immediately, whereas annotating one isolated feature module only helps that module. It's a leverage argument: fix the parts of the dependency graph everything else flows through first, for the highest ratio of bugs-caught to annotation-effort-spent.

**Glossary:**
- **Incremental typing adoption** — rolling out static type checking across a codebase module-by-module rather than all at once.
- **`--strict`** — mypy's flag enabling its full, most rigorous set of checks; per the docs, it "ensures you will never have a type related error without an explicit circumvention."

**Mental model:**
Tests real-world adoption judgment (how do you actually roll this out on a legacy codebase without stalling) versus textbook knowledge of what mypy checks.

**TL;DR:**
Rolling out mypy on a large untyped codebase means starting with a small subset under CI enforcement and expanding module-by-module — prioritizing widely-imported utilities for maximum leverage — rather than flipping on `--strict` everywhere immediately.

**References:**
- [mypy — Using mypy with an existing codebase](https://mypy.readthedocs.io/en/stable/existing_code.html)

---

### Q17. Does `get_type_hints()` behave exactly like reading `__annotations__` directly? {#q17}

**Question:**
Why would you ever call `typing.get_type_hints(func)` instead of just accessing `func.__annotations__`?

**Good answer:**
They can return different things because `get_type_hints()` does extra resolution work that raw `__annotations__` doesn't. If a module uses string forward references (or `from __future__ import annotations`, which stringifies all annotations automatically), `func.__annotations__` may literally contain string objects like `"HeavyType"` rather than the real type object — `get_type_hints()` resolves those strings against the function's global/local namespace and returns actual type objects instead. It also, per the docs, "strips the metadata from annotations" by default for anything wrapped in `Annotated[...]`, unless you pass `include_extras=True` to preserve it. So `get_type_hints()` is the "give me what a type checker would actually see, fully resolved" API, while `__annotations__` is the raw, possibly-still-stringified storage.

**Code example:**
```python
from typing import get_type_hints, Annotated

def func(x: Annotated[int, "meta"]) -> None: ...

func.__annotations__            # {'x': Annotated[int, 'meta'], ...} (raw)
get_type_hints(func)            # {'x': <class 'int'>, ...} (metadata stripped)
get_type_hints(func, include_extras=True)  # metadata preserved
```

**Follow-up question:**
Why does resolving string forward references require passing namespaces, and what can go wrong if you call `get_type_hints()` from the wrong context?

**Follow-up good answer:**
A stringified annotation is just text until something `eval`s it against the right namespace — `get_type_hints()` does this using the function's `__globals__` (and can accept explicit `globalns`/`localns` overrides) to resolve names like `"HeavyType"` back into the real object. If the referenced name isn't actually importable/defined in that namespace at the time you call `get_type_hints()` — for instance, if it only ever existed inside a `TYPE_CHECKING` guard and was never really imported at runtime (see Q9) — resolution fails with a `NameError`, because there's no real object for the string to resolve to outside of the type checker's own (TYPE_CHECKING=True) static view. This is a real gotcha for any tool (like some validation libraries) that calls `get_type_hints()` at runtime against annotations that were only ever meant for static analysis.

**Glossary:**
- **`get_type_hints()`** — a `typing` function that returns a function/class/module's annotations with string forward references resolved to real objects.

**Mental model:**
Tests whether the candidate understands that annotations are sometimes literal strings at runtime, and that "resolving" them is an active, fallible operation, not a given.

**TL;DR:**
`get_type_hints()` resolves string forward references into real type objects and strips `Annotated` metadata by default (unlike raw `__annotations__`), but that resolution can raise `NameError` if a `TYPE_CHECKING`-only name was never actually importable at runtime.

**References:**
- [typing — get_type_hints()](https://docs.python.org/3/library/typing.html)

---

### Q18. In a `match` statement, why does `case Point(x, y):` risk a subtle bug if `Point` isn't actually a dataclass or doesn't define `__match_args__`? {#q18}

**Question:**
If someone writes a class pattern with positional arguments against a class that never set up `__match_args__`, what happens?

**Good answer:**
It fails outright rather than silently misbehaving — per the language reference, positional sub-patterns are converted using `getattr(cls, "__match_args__", ())`, and if there are more positional patterns than entries in `__match_args__` (which defaults to an empty tuple when absent), the match raises `TypeError` at the point that case is attempted. So `case Point(x, y):` against a `Point` with no `__match_args__` set (a plain hand-written class with no dataclass decorator and no explicit `__match_args__ = ("x", "y")`) doesn't just "not match" — it blows up with a `TypeError` the moment control flow reaches that case, which can be a nasty surprise if it's buried deep in a match chain that otherwise looked fine in testing (if that particular branch wasn't exercised). The fix is either using `@dataclass` (which sets `__match_args__` automatically, in field order) or defining `__match_args__` explicitly on the class.

**Code example:**
```python
class Point:                 # NOT a dataclass, no __match_args__
    def __init__(self, x, y):
        self.x, self.y = x, y

match Point(0, 0):
    case Point(x, y):         # TypeError: Point() accepts 0 positional sub-patterns
        print(x, y)
```

**Follow-up question:**
Does using keyword sub-patterns (`case Point(x=x, y=y):`) instead of positional ones avoid this failure mode entirely?

**Follow-up good answer:**
Yes, largely — keyword patterns match via ordinary attribute lookup (`hasattr`/`getattr` on the subject for each named keyword), with no dependency on `__match_args__` at all, so a plain hand-written class with `self.x`/`self.y` set works fine with `case Point(x=x, y=y):` even with zero special setup. The trade-off is verbosity: keyword patterns spell out every attribute name at every call site, while positional patterns (once `__match_args__` is set up once, centrally, on the class) let every `match` block using that class stay terser. For a class you control and use in many match statements, defining `__match_args__` once is usually worth it; for occasional matching against a class you don't control (and can't add `__match_args__` to), keyword patterns are the safe, always-available option.

**Glossary:**
- **Positional sub-pattern** — an argument in a class pattern matched by position, requiring `__match_args__` to map positions to attribute names.
- **Keyword sub-pattern** — an argument in a class pattern matched by explicit attribute name, requiring no `__match_args__`.

**Mental model:**
Probes for hands-on experience debugging a real `match` failure mode, not just textbook syntax familiarity.

**TL;DR:**
A positional class pattern (`case Point(x, y):`) raises `TypeError` at match time if the class has no `__match_args__` covering that many positions — keyword patterns (`case Point(x=x, y=y):`) sidestep this entirely since they use plain attribute lookup instead.

**References:**
- [The match statement — Python Language Reference](https://docs.python.org/3/reference/compound_stmts.html#the-match-statement)

---

### Q19. Why can a dataclass with `eq=True` (the default) not be safely used as a dict key or set member? {#q19}

**Question:**
`@dataclass class Point: x: int; y: int` — can you put a `Point` instance into a `set`, or use it as a `dict` key?

**Good answer:**
Not safely, by default — the generated `__eq__` (from `eq=True`, the default) makes instances compare by value, but `dataclass` also sets `__hash__` to `None` whenever `eq=True` and `frozen=False` (the defaults), which is exactly the combination you get by writing plain `@dataclass` with no arguments. An unhashable object can't go into a `set` or serve as a `dict` key — attempting it raises `TypeError: unhashable type`. This isn't an oversight; it enforces the same Python-wide invariant `list`/`dict` themselves respect: **mutable objects with value-based equality shouldn't be hashable**, because if you hashed a mutable object, changed a field, and its hash changed, it would become unreachable in whatever hash-bucket it was originally placed in — silently corrupting the container. Since a default (`frozen=False`) dataclass is mutable, `dataclass` disables hashing to prevent exactly that.

**Code example:**
```python
from dataclasses import dataclass

@dataclass                 # eq=True, frozen=False (defaults)
class Point:
    x: int
    y: int

{Point(1, 2)}               # TypeError: unhashable type: 'Point'
```

**Follow-up question:**
What do you actually need to set to make a dataclass hashable, and why does that combination make sense?

**Follow-up good answer:**
`@dataclass(frozen=True)` makes it hashable again, because `dataclass` specifically generates a `__hash__` (based on the same fields used in `__eq__`) when `eq=True` and `frozen=True` together — the exact combination where value-based hashing is actually safe, since a frozen instance's fields (and therefore its hash) can never change after construction. This mirrors why Python's own `tuple` is hashable but `list` isn't: both compare by value, but only the immutable one guarantees its hash stays stable for its entire lifetime. If you need mutability *and* hashability, `dataclass` won't give you both automatically — you'd have to explicitly opt in via `unsafe_hash=True` and personally guarantee you never mutate a field used in the hash after the object's been placed in a hash-based container, which is exactly the footgun the default behavior protects you from.

**Glossary:**
- **`frozen=True`** — makes a dataclass instance immutable and, combined with `eq=True`, hashable.
- **`unsafe_hash=True`** — a dataclass option that forces `__hash__` generation even for a mutable class, bypassing the default safety rule.

**Mental model:**
Tests whether the candidate connects a specific, memorable dataclass gotcha (unhashable by default) back to the general Python invariant about mutability and hashing, rather than memorizing it as an isolated fact.

**TL;DR:**
A default (`eq=True, frozen=False`) dataclass is unhashable on purpose — mutable value-equal objects can't safely hash — and becomes hashable again once `frozen=True` guarantees its fields (and hash) can never change.

**References:**
- [dataclasses — eq and hash](https://docs.python.org/3/library/dataclasses.html)

---

### Q20. Given all these modern features are optional, how would you decide how much of this toolkit (typing, dataclasses, Protocol, match) to actually adopt on a new project? {#q20}

**Question:**
You're starting a new mid-sized Python service. Do you type-annotate everything, use `match` everywhere it's syntactically possible, and make every data holder a `Protocol`-checked structure — or is that overkill?

**Good answer:**
Match the tool to the actual risk/complexity at each point, rather than maximizing feature usage for its own sake. Type hints on public function signatures and module boundaries earn their keep almost everywhere — cheap to write, and per PEP 484's own framing, they're an optional net that costs little and catches real mismatches; skipping them entirely on anything beyond a throwaway script is rarely the right call. `dataclass` is the default for any structured data holder unless you specifically need tuple semantics (`NamedTuple`) or runtime validation at a trust boundary (pydantic) — using a plain dataclass everywhere else avoids paying for validation you don't need. `Protocol` earns its place specifically where you want structural, duck-typed flexibility across unrelated class hierarchies (e.g. plugin-style interfaces) — it's overkill for a single concrete implementation with no need for substitutability. `match`/`case` is genuinely better than an `if`/`elif` chain specifically when you're destructuring/branching on *shape* (nested structures, tagged unions, class hierarchies) — using it to replace a simple three-way `if x == 1` chain adds syntax without adding clarity. The unifying principle: each of these exists to solve a specific pain (untyped bugs, boilerplate, rigid inheritance, structural branching) — reach for the one whose pain you actually have, not all of them by default.

**Follow-up question:**
How would you convince a skeptical teammate who thinks "this is all just Java creeping into Python" that this isn't over-engineering?

**Follow-up good answer:**
Point to what's *not* mandatory: none of these features change Python's runtime semantics or force anything — PEP 484 explicitly preserved that "Python will remain a dynamically typed language" with type hints never becoming mandatory, and every construct here (dataclasses, Protocol, TypedDict, match) is opt-in, degrades gracefully, and can be adopted incrementally in exactly the parts of a codebase that benefit. The Java comparison misses that Java's type system is a compiler-enforced runtime guarantee, while all of this is an optional, offline-checked layer with an escape hatch (`Any`, or just not annotating something) always available — it's additive tooling for humans and CI, not a constraint the interpreter imposes. The practical argument, not the philosophical one, usually lands better: show a real bug type hints or dataclasses would have caught in code review, on this specific codebase, rather than arguing the abstract case.

**Glossary:**
- **Trust boundary** — see Q14; the point where external, unvalidated data enters your program's control.

**Mental model:**
Tests engineering judgment and communication — whether the candidate can reason about tool selection by actual problem-fit, and defend adoption pragmatically rather than dogmatically, both signs of senior-level thinking.

**TL;DR:**
Adopt each modern Python feature (typing, dataclasses, Protocol, match) where it solves a specific pain you actually have — public-boundary typing almost always, dataclasses by default for data holders, Protocol for genuine structural flexibility, match for shape-based branching — rather than maximizing feature usage, since none of it is mandatory or changes Python's runtime semantics.

**References:**
- [PEP 484 – Type Hints: Rationale](https://peps.python.org/pep-0484/)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=typing-and-modern-python-features&autostart=1" | relative_url }})
