---
layout: default
title: "Python Interview — Packaging, Testing & Tooling"
---

# Python Interview — Packaging, Testing & Tooling

Twenty questions on the Python ecosystem tooling that surrounds the language itself: virtual environments and packaging formats, how `pip`'s and `uv`'s resolvers actually work, `pytest`'s discovery/fixture machinery, diagnosing slow test suites and slow imports, and the trade-offs between the pip+venv, Poetry, and uv workflows.

### Q1. What is a Python virtual environment, and why does every project need one? {#q1}

**Question:**
What is a Python virtual environment, and why does every project need one?

**Good answer:**
A virtual environment is an isolated Python installation with its own `site-packages` directory, separate from the system/base Python and from other projects' environments. It exists to solve dependency conflicts: without it, every project on a machine shares one global `site-packages`, so two projects needing different versions of the same package (or even different Python versions) can't coexist. `venv` (stdlib, since Python 3.3) creates this isolation by writing a `pyvenv.cfg` pointing back at the base interpreter, copying/symlinking the Python executable, and giving the environment its own `site-packages` directory that Python's import system prefers over the system one.

**Code example:**
```bash
python -m venv .venv
source .venv/bin/activate     # POSIX
# .venv\Scripts\Activate.ps1  # Windows PowerShell
pip install requests
```

**Follow-up question:**
Technically, how does Python know it's running inside a virtual environment, and how does that change where it looks for packages?

**Follow-up good answer:**
Python exposes `sys.prefix` and `sys.base_prefix`: outside a venv they're equal; inside one, `sys.prefix` points at the venv directory while `sys.base_prefix` still points at the base install (`sys.prefix != sys.base_prefix` is the standard "am I in a venv" check). That prefix drives where the interpreter resolves `site-packages` — the venv's `lib/pythonX.Y/site-packages` takes precedence, and system packages are excluded entirely unless the environment was created with `--system-site-packages`. This is also why venvs are not portable between machines/paths: installed scripts get an absolute shebang like `#!/path/to/venv/bin/python`.

**Glossary:**
- **`site-packages`** — the directory where third-party packages are installed for a given Python environment.
- **`sys.prefix`** — the filesystem prefix the running interpreter uses to locate its standard library and site-packages.

**Mental model:**
Tests whether the candidate understands isolation as a *filesystem/path* mechanism, not magic — a common gap for people who only ever type `source venv/bin/activate` without knowing what it does.

**TL;DR:**
A venv isolates a project's packages by giving it its own `site-packages` and shifting `sys.prefix` to point at it, so `pip install` there never touches the system Python or other projects.

**References:**
- [Python docs — venv: Creation of virtual environments](https://docs.python.org/3/library/venv.html)

---

### Q2. What's the difference between a wheel and an sdist, and when does pip use each? {#q2}

**Question:**
What's the difference between a wheel and an sdist, and when does pip use each?

**Good answer:**
An **sdist** (source distribution) is a tarball of the raw source code plus metadata — installing it requires a build step (running the build backend to produce something installable, potentially compiling C extensions). A **wheel** is a *built* distribution: a zip-format archive of already-built files that pip can just unpack into place, with no build step. `pip install` prefers a compatible wheel when the index offers one (fast, no compiler needed); it falls back to the sdist (slower, requires a working build toolchain) when no matching wheel exists for the target platform/Python version.

**Code example:**
```bash
python -m build            # produces both:
# dist/mypkg-1.0.0.tar.gz       <- sdist
# dist/mypkg-1.0.0-py3-none-any.whl  <- wheel
```

**Follow-up question:**
Why would a package publish multiple wheels for the same version, and what do tags like `cp311-manylinux_x86_64` mean?

**Follow-up good answer:**
A single wheel isn't always portable across platforms/interpreters — packages with compiled C extensions need a separate wheel per Python ABI/OS/architecture combination. The filename's compatibility tags encode this: `cp311` means CPython 3.11 ABI, `manylinux_x86_64` means a Linux wheel built against a baseline glibc for broad compatibility on x86_64. Pure-Python packages instead publish one universal wheel tagged `py3-none-any` (any Python 3 interpreter, any ABI, any platform), since there's nothing platform-specific to compile.

**Glossary:**
- **ABI (Application Binary Interface)** — the compiled-code-level contract (e.g. CPython's C API) a wheel with native extensions is built against.
- **manylinux** — a PyPA standard defining a baseline Linux environment wheels can target for broad compatibility.

**Mental model:**
Checks whether the candidate has actually published or debugged a package with native dependencies, versus only ever consuming pure-Python packages.

**TL;DR:**
sdist = source, needs a build step; wheel = prebuilt, pip just unpacks it — pip prefers a matching wheel and only falls back to building the sdist when none exists.

**References:**
- [Python Packaging User Guide — Source distributions vs. wheels](https://packaging.python.org/en/latest/discussions/wheel-vs-sdist/)

---

### Q3. What's the difference between `requirements.txt` and the `[project]` table in `pyproject.toml`? {#q3}

**Question:**
What's the difference between `requirements.txt` and the `[project]` table in `pyproject.toml`?

**Good answer:**
`requirements.txt` is a plain, tool-specific list of pip install arguments (often pinned exact versions) with no standardized schema — it's not project metadata, just an install manifest, and different projects use it inconsistently (loose ranges vs. exact pins vs. a "lock" file). `pyproject.toml`'s `[project]` table, standardized by PEP 621, is the tool-agnostic way to declare a project's actual metadata — name, version, dependencies, entry points, etc. — so any PEP 517-compliant build backend (setuptools, Hatchling, Poetry-core...) can read the same fields instead of every tool inventing its own config file.

**Follow-up question:**
If `pyproject.toml` can declare dependencies too, why do many projects still keep a `requirements.txt`?

**Follow-up good answer:**
They serve different jobs: `[project.dependencies]` in `pyproject.toml` declares *abstract* requirements (e.g. `"requests>=2.28"`) for the package itself, while a `requirements.txt` is commonly used as a *lock artifact* — an exact, fully-pinned, reproducible set of versions (often generated by `pip freeze` or `pip-compile`) for deploying a specific environment. Many modern workflows keep the abstract constraints in `pyproject.toml` and generate a locked `requirements.txt` (or a tool-specific lockfile like `poetry.lock`/`uv.lock`) from it for reproducible installs.

**Glossary:**
- **PEP 621** — the standard defining the `[project]` table in `pyproject.toml` for static project metadata.
- **Lockfile** — a fully-pinned dependency manifest generated from abstract constraints, used to reproduce an exact environment.

**Mental model:**
Tests whether the candidate distinguishes "what my package needs" (abstract, in `pyproject.toml`) from "exactly what's installed right now" (concrete, in a lock artifact) — conflating the two is a common source of packaging confusion.

**TL;DR:**
`pyproject.toml`'s `[project]` table (PEP 621) is the standardized place for a package's abstract metadata/dependencies; `requirements.txt` is typically used as an unstandardized, often fully-pinned install/lock manifest.

**References:**
- [PEP 621 – Storing project metadata in pyproject.toml](https://peps.python.org/pep-0621/)

---

### Q4. Walk through what happens when you write `import foo` — how does Python actually find and load the module? {#q4}

**Question:**
Walk through what happens when you write `import foo` — how does Python actually find and load the module?

**Good answer:**
Python first checks `sys.modules` for an already-imported `foo` (imports are cached, so a second `import foo` anywhere is nearly free). If not cached, the import system walks `sys.meta_path` finders in order — the built-in finder for built-in modules, the frozen-module finder, then the path-based finder that searches `sys.path` (script directory or cwd, `PYTHONPATH`, install-dependent paths including the venv's `site-packages`) directory by directory, matching `foo.py`, a `foo/` package with `__init__.py`, or a compiled extension. Once found, the corresponding loader compiles/executes the module's code in a fresh module namespace, caches it in `sys.modules`, and binds the name in the importing scope.

**Code example:**
```python
import sys
print(sys.path)          # search order the path-based finder uses
print("foo" in sys.modules)  # True after first successful import
```

**Follow-up question:**
Why can adding a virtual environment's `site-packages` earlier or later in `sys.path` silently change which version of a package gets imported?

**Follow-up good answer:**
The path-based finder searches `sys.path` in order and stops at the *first* match, so if two directories on the path both contain a module named `foo` (e.g. a stray copy in the project directory vs. the real dependency in `site-packages`), whichever directory appears first wins — silently shadowing the other. This is exactly why running `python script.py` from inside a project directory containing a file that shares a stdlib/dependency name (like an accidental `random.py`) can produce baffling import errors: the local file, being earlier on `sys.path` (script directory is prepended), shadows the real one.

**Glossary:**
- **`sys.meta_path`** — the ordered list of finder objects Python's import system consults to locate a module.
- **Shadowing** — when an earlier entry on the search path supplies a module of the same name, hiding the intended one.

**Mental model:**
Distinguishes candidates who've only used imports from those who've actually debugged an import-resolution bug (wrong package version imported, mysterious `ImportError`, name-shadowing).

**TL;DR:**
Import checks `sys.modules` first, then walks `sys.meta_path` finders (built-in, then path-based over `sys.path` in order), and the first matching entry wins — so ordering and stray same-named files can silently shadow the real module.

**References:**
- [Python docs — The import system](https://docs.python.org/3/reference/import.html)

---

### Q5. How does `pip`'s dependency resolver work, and why can `pip install` sometimes hang for minutes? {#q5}

**Question:**
How does `pip`'s dependency resolver work, and why can `pip install` sometimes hang for minutes?

**Good answer:**
Since pip 20.3, pip uses a proper backtracking resolver: it picks candidate versions for each requirement, and when it later discovers a conflict (two already-chosen packages require incompatible versions of a shared dependency), it backtracks and tries a different candidate version instead of just failing immediately. The slowness comes from that backtracking search: when a package has many published versions and the constraint graph is large, pip may have to download and inspect metadata for many candidate versions before finding a consistent set, and this can take minutes on projects with deep, loosely-pinned dependency trees.

**Follow-up question:**
What can you actually do to speed up a slow `pip install` resolution, beyond just waiting?

**Follow-up good answer:**
Narrow the search space: add explicit version constraints on the packages causing the most backtracking (both a floor and, if needed, a ceiling) so pip doesn't have to try dozens of candidates; avoid over-pinning unrelated packages, since that can force pip into corners that require more backtracking to resolve; or sidestep resolution at install time entirely by using a precomputed lockfile (e.g. via `pip-tools`' `pip-compile`, or switching to a resolver built for speed like `uv`, which is written in Rust and resolves and installs dramatically faster).

**Glossary:**
- **Backtracking resolver** — a resolution algorithm that retries earlier decisions when a later constraint proves them wrong, rather than failing outright.

**Mental model:**
Tests real production experience — anyone who has run `pip install` on a large, loosely-pinned project has hit this, and the fix (constrain the culprit, or use a lockfile/faster resolver) is a genuinely useful piece of practical knowledge.

**TL;DR:**
pip's resolver backtracks through candidate versions when constraints conflict, which is slow on large/loose dependency graphs — fix it with tighter constraints on the worst offenders or by resolving once into a lockfile instead of at every install.

**References:**
- [pip docs — Dependency resolution](https://pip.pypa.io/en/stable/topics/dependency-resolution/)

---

### Q6. How does `pytest` decide which files and functions are tests, with zero configuration? {#q6}

**Question:**
How does `pytest` decide which files and functions are tests, with zero configuration?

**Good answer:**
By convention: starting from `testpaths` (if configured) or the current directory, pytest recursively walks directories (skipping ones matched by `norecursedirs`), collecting files named `test_*.py` or `*_test.py`. Within those files it collects any function prefixed `test`, and any method prefixed `test` inside a class prefixed `Test` (as long as that class has no `__init__`). It also recognizes `unittest.TestCase` subclasses for backward compatibility with the standard library's test style.

**Follow-up question:**
Why does a `TestCase`-like class with an `__init__` method get silently skipped by pytest's default collection instead of erroring?

**Follow-up good answer:**
pytest's naming-convention collector assumes test classes are simple containers it can instantiate itself with no arguments; a class with its own `__init__` breaks that assumption (pytest wouldn't know what to pass it), so rather than guessing or crashing collection for the whole file, pytest treats it as "not a test class" and quietly excludes it — which is precisely why a typo'd or refactored constructor can make an entire class of tests silently vanish from a run instead of failing loudly. Running with `-v` or checking `pytest --collect-only` output is the way to catch this, since a missing test won't show up as a failure.

**Glossary:**
- **Collection** — pytest's process of discovering and building the list of test items before running any of them.
- **`norecursedirs`** — a config option listing directory patterns pytest should not descend into while searching for tests.

**Mental model:**
Probes whether the candidate has been bitten by "my tests just didn't run and nothing told me" — a classic pytest gotcha that separates people who've read the collection rules from people who've only ever run `pytest` and hoped.

**TL;DR:**
pytest collects `test_*.py`/`*_test.py` files and `test`-prefixed functions/methods (in `Test`-prefixed classes without `__init__`) purely by naming convention — no explicit registration needed, but that also means a class with a constructor is silently excluded rather than erroring.

**References:**
- [pytest docs — Conventions for Python test discovery](https://docs.pytest.org/en/stable/explanation/goodpractices.html#conventions-for-python-test-discovery)

---

### Q7. How do pytest fixtures work as a dependency-injection mechanism? {#q7}

**Question:**
How do pytest fixtures work as a dependency-injection mechanism?

**Good answer:**
A function decorated with `@pytest.fixture` becomes a provider that test functions (or other fixtures) request simply by naming it as a parameter — pytest inspects each test's signature, matches parameter names against registered fixtures, and calls them (in dependency order, since fixtures can themselves request other fixtures) before running the test body, injecting each fixture's return/yield value as the matching argument. This is dependency injection in the classic sense: the test declares *what* it needs by name, and the framework is responsible for constructing and supplying it, including managing setup/teardown via `yield`-style fixtures.

**Code example:**
```python
import pytest

@pytest.fixture
def db_connection():
    conn = create_connection()
    yield conn          # setup done, value injected
    conn.close()         # teardown after the test

def test_query(db_connection):
    assert db_connection.execute("SELECT 1")
```

**Follow-up question:**
What's the practical benefit of fixture *scope* (e.g. `scope="session"` vs the default `scope="function"`), and what's the risk of overusing a broad scope?

**Follow-up good answer:**
Scope controls how often a fixture is torn down and recreated: `function` (default) reruns setup/teardown for every test, which is safest but can be slow for expensive resources (spinning up a database, a browser session); `session`/`module`/`class` scopes reuse the same fixture instance across many tests, trading isolation for speed. The risk of overusing a broad scope is state leakage between tests — if a `session`-scoped fixture's state is mutated by one test, later tests that assumed a clean instance can fail (or worse, pass incorrectly) depending on test execution order, which is a common source of flaky, order-dependent test suites.

**Glossary:**
- **Fixture scope** — how long a fixture instance is reused before pytest tears it down and creates a new one.

**Mental model:**
Tests understanding of fixtures beyond "the decorator that gives me test data" — specifically the trade-off between test isolation and setup cost, which matters once a suite has expensive fixtures.

**TL;DR:**
Fixtures are injected by parameter name and resolved in dependency order (with `yield` handling teardown); broadening their scope for speed trades away per-test isolation and can introduce order-dependent flakiness.

**References:**
- [pytest docs — About fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html)

---

### Q8. Your CI test suite takes 20 minutes and used to take 5. How do you find out what got slow? {#q8}

**Question:**
Your CI test suite takes 20 minutes and used to take 5. How do you find out what got slow?

**Good answer:**
Start with `pytest --durations=20` (optionally `--durations-min=1.0`), which reports the slowest individual test durations — this immediately tells you whether it's a handful of pathologically slow tests or a broad, even slowdown across the whole suite. For a handful of slow outliers, profile just those (e.g. `cProfile`/`py-spy` around the specific test, or add timing around suspect fixtures) to find whether it's a slow fixture (e.g. a `function`-scoped fixture doing expensive setup that should be `session`-scoped), an accidental real network/database call that should be mocked, or genuinely slow application code the test is exercising.

**Code example:**
```bash
pytest --durations=20 --durations-min=1.0
```

**Follow-up question:**
`--durations` shows the slowest tests, but the suite got slower *evenly* across almost every test — what else would you check?

**Follow-up good answer:**
An even, across-the-board slowdown often isn't in the tests themselves but in *import time* or shared setup that runs once per process but got slower — check `python -X importtime -m pytest` (or `PYTHONPROFILEIMPORTTIME=1`) to see if a newly-added dependency or an expensive module-level side effect (e.g. eager config loading, an ORM building its schema on import) is now adding real fixed cost to every test collection/run. Also check for a broad-scope fixture or `conftest.py` autouse fixture that now does more work per test than before — since it's invisible in `--durations`' per-test view but runs implicitly for everything.

**Glossary:**
- **`conftest.py`** — a pytest file providing fixtures and hooks automatically available to tests in its directory tree.
- **Autouse fixture** — a fixture applied to every applicable test automatically, without being named as a parameter.

**Mental model:**
This is the performance-diagnosis pattern applied to test infrastructure rather than production code — checking whether the candidate reaches for the right tool (per-test profiling vs. import-time profiling) based on the *shape* of the slowdown (isolated vs. systemic).

**TL;DR:**
Use `pytest --durations` to find slow individual tests; for an even, suite-wide slowdown, suspect import time (`python -X importtime`) or a broad-scope/autouse fixture doing more work than before.

**References:**
- [pytest docs — Profiling test execution duration](https://docs.pytest.org/en/stable/how-to/usage.html#profiling-test-execution-duration)
- [Python docs — `-X importtime`](https://docs.python.org/3/using/cmdline.html#cmdoption-X)

---

### Q9. Why is a lockfile necessary for reproducible deployments, even if `pyproject.toml` already lists your dependencies? {#q9}

**Question:**
Why is a lockfile necessary for reproducible deployments, even if `pyproject.toml` already lists your dependencies?

**Good answer:**
`pyproject.toml` dependencies are typically *ranges* (`"requests>=2.28"`), which is intentional — it lets your package remain installable alongside other packages with their own compatible ranges. But a range means "install could resolve to any version satisfying it," so two installs run weeks apart (or on two different machines) can silently pick different concrete versions if new releases were published in between — breaking the core reproducibility property that "the same install command produces the same environment." A lockfile pins every package (including transitive dependencies) to an exact, hash-verified version, so `pip install -r requirements.lock` (or `poetry install`/`uv sync` against `poetry.lock`/`uv.lock`) reproduces the *exact* environment every time, independent of what's been published since.

**Follow-up question:**
If lockfiles are strictly better for reproducibility, why doesn't a *library* (as opposed to an application) typically ship one?

**Follow-up good answer:**
A library's dependency ranges exist specifically so it can coexist with whatever versions its consumers already have pinned for their other dependencies — locking a library to exact versions would force every consumer to use those exact versions too, which reintroduces the version-conflict problem a resolver exists to solve. Lockfiles make sense for the *leaf* of the dependency tree (an application with a fixed, deployed set of dependents) where "reproduce this exact environment" is the actual goal, not for a package meant to be installed alongside arbitrary other packages.

**Glossary:**
- **Transitive dependency** — a dependency of one of your direct dependencies, not declared directly by your project.

**Mental model:**
Tests whether the candidate understands *why* the range-vs-pin distinction exists (library flexibility vs. application reproducibility) rather than just knowing "pin your deps" as a rule of thumb.

**TL;DR:**
Ranges in `pyproject.toml` let a package coexist with others' constraints but don't guarantee the same resolve twice; a lockfile pins every (including transitive) dependency to an exact version so an application's environment is exactly reproducible — which is why applications lock and libraries generally don't.

**References:**
- [Python Packaging User Guide — Dependency specifiers](https://packaging.python.org/en/latest/specifications/dependency-specifiers/)

---

### Q10. How should semantic versioning shape the version constraints you put on a dependency? {#q10}

**Question:**
How should semantic versioning shape the version constraints you put on a dependency?

**Good answer:**
Under semantic versioning (`MAJOR.MINOR.PATCH`), a MAJOR bump signals breaking changes, MINOR adds backward-compatible functionality, and PATCH is backward-compatible fixes — so a constraint like `requests>=2.28,<3` (equivalently, PEP 440's compatible-release operator `requests~=2.28`) says "any version that promises not to break my code," accepting new features and fixes but rejecting the next major version until you've verified compatibility. Pinning to an exact version (`requests==2.28.0`) forfeits that flexibility entirely — safest against surprise regressions in your own testing, but it means you get *zero* automatic security or bug fixes and are far more likely to collide with another package's constraints.

**Code example:**
```
requests~=2.28    # PEP 440 compatible release: >=2.28, ==2.*
requests>=2.28,<3 # equivalent, spelled out
requests==2.28.0  # exact pin — no automatic updates at all
```

**Follow-up question:**
Semantic versioning is a promise the package author makes, not something Python enforces — what goes wrong when a maintainer breaks it, and how does that interact with your constraint choice?

**Follow-up good answer:**
If a maintainer ships a breaking change in a MINOR or PATCH release (violating semver), a `~=`/range constraint that trusted that promise will happily upgrade you into the breaking change, and your code fails despite "correctly" following semver conventions — there's no tooling-level enforcement, only convention and, ideally, the maintainer's changelog and your CI catching it. This is exactly the argument for combining ranges (for security/bugfix reachability) with a lockfile and CI (so an unexpected resolve is caught by tests before it reaches production) rather than relying on version constraints alone as a safety net.

**Glossary:**
- **PEP 440** — the specification defining Python's version identifier and specifier syntax (`~=`, `>=`, etc.).
- **Compatible release operator (`~=`)** — a PEP 440 operator meaning "at least this version, but within the same leading version segment."

**Mental model:**
Checks whether the candidate treats semver as a real engineering trade-off (flexibility vs. risk) with a known failure mode, not a rule to cite without understanding its limits.

**TL;DR:**
Use range/compatible-release constraints (`~=`) to stay open to fixes and security patches without opting into breaking changes, but pair that with a lockfile and CI since semver is a convention maintainers can (and do) violate.

**References:**
- [PEP 440 – Version Identification and Dependency Specification](https://peps.python.org/pep-0440/)

---

### Q11. How does the "test pyramid" apply to a Python codebase using pytest? {#q11}

**Question:**
How does the "test pyramid" apply to a Python codebase using pytest?

**Good answer:**
The test pyramid argues for many fast, isolated unit tests at the base, fewer integration tests that exercise real collaborators (a real database, a real HTTP call to another service) in the middle, and very few slow, brittle end-to-end tests at the top — because cost and flakiness both increase up the pyramid while precision of failure localization decreases. In pytest terms, this typically maps to: unit tests using `function`-scoped fixtures and mocks for speed/isolation; integration tests using `session`-scoped fixtures (a real test database container, say) shared across a module to amortize setup cost; and end-to-end tests marked separately (e.g. `@pytest.mark.e2e`) and often excluded from the default fast local run via `-m "not e2e"`.

**Follow-up question:**
What goes wrong in practice when a team inverts the pyramid — lots of end-to-end tests, few unit tests?

**Follow-up good answer:**
End-to-end tests are slow and often flaky (network timing, shared test environments, order dependencies), so an inverted pyramid means CI takes much longer and fails intermittently for reasons unrelated to real bugs, which erodes trust in the suite (people start re-running failed CI instead of investigating) and makes it hard to pinpoint *which* component actually broke when a broad end-to-end test fails, since it exercises the whole system at once rather than isolating a unit of logic.

**Glossary:**
- **Test pyramid** — a testing strategy heuristic favoring many fast unit tests, fewer integration tests, and very few end-to-end tests.
- **Flaky test** — a test that intermittently passes or fails without a change in the code under test, usually due to timing, ordering, or shared external state.

**Mental model:**
Tests whether the candidate can connect a general SE testing-strategy principle to concrete pytest mechanics (fixture scope, markers) rather than reciting the pyramid as trivia.

**TL;DR:**
Favor many fast unit tests with function-scoped mocks, fewer integration tests with shared session-scoped real fixtures, and mark end-to-end tests separately so they can be excluded from the fast local loop — inverting this makes CI slow, flaky, and hard to debug.

**References:**
- [pytest docs — Marking test functions with attributes](https://docs.pytest.org/en/stable/how-to/mark.html)

---

### Q12. Why do most teams prefer pytest fixtures over `unittest.TestCase`'s `setUp`/`tearDown`? {#q12}

**Question:**
Why do most teams prefer pytest fixtures over `unittest.TestCase`'s `setUp`/`tearDown`?

**Good answer:**
`setUp`/`tearDown` run for *every* test in the class unconditionally and can't be composed or shared cleanly across classes without inheritance gymnastics; pytest fixtures are requested explicitly by name, are composable (a fixture can depend on other fixtures), independently scoped (function/class/module/session), and reusable across any test file via `conftest.py` without subclassing anything. This makes it far easier to express "this specific test needs a database connection and a fake clock, but not the mocked S3 bucket every other test in the file uses," which `setUp` can't express without extra conditional logic.

**Follow-up question:**
Can pytest run `unittest.TestCase`-based test suites, and is there a reason to migrate an existing `unittest` suite to pytest fixtures?

**Follow-up good answer:**
Yes — pytest natively discovers and runs `unittest.TestCase` subclasses, so an existing `unittest` suite works under `pytest` with zero changes, and teams often adopt pytest as the *runner* first for its better output/plugin ecosystem. Migrating the tests themselves to fixtures is worthwhile mainly when the suite would benefit from the composability/scoping pytest fixtures offer (e.g. expensive shared setup that's currently duplicated per-class), but it's an optional, incremental step rather than a requirement — `unittest`-style tests remain fully supported.

**Glossary:**
- **`unittest.TestCase`** — the standard library's class-based test-case base class, using `setUp`/`tearDown` hooks.

**Mental model:**
Probes whether the candidate understands *why* fixtures are considered better (composability, explicit scoping) rather than just "pytest is more modern," and whether they know migration is optional, not forced.

**TL;DR:**
Fixtures beat `setUp`/`tearDown` because they're explicitly requested, composable, and independently scoped instead of running unconditionally for a whole class — but pytest runs existing `unittest` suites as-is, so migration is optional.

**References:**
- [pytest docs — unittest.TestCase support](https://docs.pytest.org/en/stable/how-to/unittest.html)

---

### Q13. A test passes alone but fails when run as part of the full suite. What's the most likely cause, and how do you find it? {#q13}

**Question:**
A test passes alone but fails when run as part of the full suite. What's the most likely cause, and how do you find it?

**Good answer:**
The most common cause is shared mutable state leaking between tests — a module-level list/dict, a class attribute, a monkeypatched global that an earlier test modified and never reset, or a broad-scope fixture whose state one test mutates that a later test assumes is pristine. To find it, use `pytest -p no:randomly` / a plugin like `pytest-randomly` to *shuffle* test order deliberately (if the failure appears/disappears depending on order, that confirms state leakage) and bisect by running subsets of the suite (`pytest tests/test_a.py tests/test_b.py`) to narrow down which earlier test is the culprit, then inspect what module/class-level or broad-scope state it touches.

**Follow-up question:**
Why does `pytest-randomly` (or similar order-randomization) actually help catch this class of bug rather than just making failures harder to reproduce?

**Follow-up good answer:**
A fixed, always-the-same test order can hide order-dependence indefinitely — the suite happens to pass every time because the tests always run in the same sequence, even though a real dependency between them exists. Randomizing order turns a latent bug into a *visible, intermittent* failure during development/CI, which is unpleasant in isolation but is strictly better than never surfacing the bug at all; combined with each run printing the seed used, a specific failing order can still be reproduced exactly for debugging.

**Glossary:**
- **Order-dependent test** — a test whose pass/fail outcome depends on which other tests ran before it, indicating shared mutable state.

**Mental model:**
Tests real debugging instinct for one of the most common flaky-test root causes, and whether the candidate knows the specific tools (order randomization, bisection) rather than just "check for shared state" as vague advice.

**TL;DR:**
Order-dependent failures almost always mean shared mutable state (module/class globals, a broad-scope fixture) leaking across tests; randomize test order to surface it and bisect the suite to find the offending test.

**References:**
- [pytest-randomly documentation](https://github.com/pytest-dev/pytest-randomly)

---

### Q14. What is `pip install -e .` (an editable install), and why did it need its own PEP? {#q14}

**Question:**
What is `pip install -e .` (an editable install), and why did it need its own PEP?

**Good answer:**
An editable install lets you import a package straight from its source checkout — edits to the source take effect immediately without reinstalling — instead of copying a snapshot of the code into `site-packages` the way a normal install does. It needed standardizing (PEP 660) because editable installs originally only existed as `setup.py develop`, a setuptools-specific mechanism; once PEP 517 allowed alternative build backends (Hatchling, Flit, Poetry-core), those backends had no `setup.py` to hook into, so there was no way to get an editable install without falling back to a compatible `setup.py develop` shim. PEP 660 added standard `build_editable`/`get_requires_for_build_editable`/`prepare_metadata_for_build_editable` hooks so any PEP 517 backend can support editable installs uniformly.

**Code example:**
```bash
pip install -e .
# Now `import mypkg` loads directly from ./src/mypkg, no reinstall needed after edits
```

**Follow-up question:**
Editable installs sometimes behave subtly differently from a real install — what's a concrete way that can bite you?

**Follow-up good answer:**
Because an editable install often works by pointing import machinery back at your source tree (historically via a `.egg-link`/`.pth` file, or PEP 660's finder-based approach) rather than copying files, packaging bugs like a missing `package_data`/`MANIFEST.in` entry for a non-Python file can go completely unnoticed in editable mode (the file's right there on disk) but break a real install from a built wheel, where only explicitly-included files get packaged — so "works in editable mode" isn't proof a real release install will work, and it's worth periodically testing against an actual built wheel.

**Glossary:**
- **PEP 517** — the standard defining a build-backend interface independent of `setup.py`, enabling alternative backends.
- **PEP 660** — the standard adding editable-install hooks to the PEP 517 backend interface.

**Mental model:**
Tests awareness that "editable" is a real, standardized mechanism with its own semantics and edge cases, not just a convenience flag with no implications.

**TL;DR:**
Editable installs let you import from a source checkout without reinstalling on every change; PEP 660 standardized this for non-setuptools PEP 517 backends, but editable mode can mask real packaging bugs (missing data files) that only show up in a genuine wheel install.

**References:**
- [PEP 660 – Editable installs for pyproject.toml based builds](https://peps.python.org/pep-0660/)

---

### Q15. What's the classic `unittest.mock.patch` mistake, and how do you avoid it? {#q15}

**Question:**
What's the classic `unittest.mock.patch` mistake, and how do you avoid it?

**Good answer:**
Patching the object where it's *defined* instead of where it's *looked up*. If module `b.py` does `from a import SomeClass` and then calls `SomeClass()`, patching `a.SomeClass` has no effect on `b`'s test, because `b` already holds its own reference to the original class in its own namespace after the `from...import` — you must patch `b.SomeClass` instead, since that's the name the code under test actually resolves at call time. The rule is: always patch the name as it's imported/used in the module under test, not the module where the object was originally defined.

**Code example:**
```python
# a.py
class SomeClass: ...

# b.py
from a import SomeClass
def do_thing():
    return SomeClass()

# test_b.py — CORRECT
from unittest.mock import patch

@patch("b.SomeClass")   # patch where it's looked up (in b), not "a.SomeClass"
def test_do_thing(mock_cls):
    ...
```

**Follow-up question:**
If `b.py` instead did `import a` and called `a.SomeClass()`, would you patch `a.SomeClass` or `b.SomeClass`?

**Follow-up good answer:**
`a.SomeClass` — because with `import a`, `b` doesn't hold its own reference to the class; it looks the attribute up on module `a` at call time via `a.SomeClass()`, so patching `a`'s namespace *is* patching what `b` looks up. This is the same underlying rule applied to the opposite import style: identify which namespace the code under test actually performs the attribute lookup in, and patch that one.

**Glossary:**
- **`unittest.mock.patch`** — a context manager/decorator that temporarily replaces an attribute in a given namespace for the duration of a test.

**Mental model:**
This is one of the most common real-world mocking bugs; the question tests whether the candidate can reason about *namespaces and name binding* rather than having memorized "patch the string that matches the import," which breaks the moment the import style changes.

**TL;DR:**
Patch where an object is looked up (the module under test's own namespace after `from...import`), not where it's defined — the target string must match how the code under test actually resolves the name.

**References:**
- [Python docs — unittest.mock: Where to patch](https://docs.python.org/3/library/unittest.mock.html#where-to-patch)

---

### Q16. What does `@pytest.mark.parametrize` give you that a `for` loop of assertions inside one test function doesn't? {#q16}

**Question:**
What does `@pytest.mark.parametrize` give you that a `for` loop of assertions inside one test function doesn't?

**Good answer:**
`parametrize` turns each parameter set into its own independently-tracked test case (visible in output as e.g. `test_eval[6*9-42]`), so pytest reports pass/fail per case, a single failing case doesn't stop the others from running (unlike a loop, which stops at the first failed `assert`), and you get the specific failing input directly in the failure output instead of a generic assertion error you have to add print statements to diagnose. Individual cases can also be marked separately (e.g. `pytest.mark.xfail` on just one tuple) without affecting the rest.

**Code example:**
```python
import pytest

@pytest.mark.parametrize("test_input,expected", [
    ("3+5", 8),
    ("2+4", 6),
    ("6*9", 42),
])
def test_eval(test_input, expected):
    assert eval(test_input) == expected
```

**Follow-up question:**
When would a `for` loop inside a test still be the right choice over `parametrize`?

**Follow-up good answer:**
When the values being iterated aren't really independent test cases but are checking one cohesive property together — e.g. asserting a single collection's items are all non-negative, where you want one pass/fail outcome for "the invariant holds for this collection," not N separately-reported cases with their own identities. `parametrize` is for genuinely distinct scenarios you want tracked and reported individually; a loop is fine for a single assertion logically spanning multiple values.

**Glossary:**
- **`@pytest.mark.parametrize`** — a pytest decorator that runs a test function once per given set of parameters.

**Mental model:**
Checks whether the candidate has actually used parametrize for its reporting/isolation benefits, versus just knowing the decorator syntax.

**TL;DR:**
`parametrize` reports each case as its own test (so one failure doesn't hide the rest, and the failing input is visible directly), which a `for`-loop-of-asserts can't do — reserve the loop for asserting one property across values, not distinct scenarios.

**References:**
- [pytest docs — How to parametrize fixtures and test functions](https://docs.pytest.org/en/stable/how-to/parametrize.html)

---

### Q17. What problem does `tox` (or `nox`) solve that running `pytest` directly doesn't? {#q17}

**Question:**
What problem does `tox` (or `nox`) solve that running `pytest` directly doesn't?

**Good answer:**
Running `pytest` directly only tests against whatever interpreter/environment is currently active. `tox` automates creating a matrix of isolated virtual environments — one per Python version and/or dependency combination you declare — installing your package fresh into each, and running your test command in each, so you can verify "this actually installs and passes on 3.10, 3.11, and 3.12" (or against a min/max supported dependency version) in one command instead of manually switching interpreters. It's also commonly used as the single entry point CI calls, keeping the "how do I test this project" logic in the repo rather than duplicated in CI-specific YAML.

**Code example:**
```ini
# tox.ini
[tox]
envlist = py310, py311, py312

[testenv]
deps = pytest
commands = pytest
```

**Follow-up question:**
Your package installs and tests fine under `pytest` locally but `tox` reveals a failure only under a fresh environment — what class of bug does that usually indicate?

**Follow-up good answer:**
It usually means the package has an undeclared dependency, or relies on something present in your local dev environment by accident (a package you `pip install`ed once for a different project and forgot to add to `pyproject.toml`, or a stale `.pyc`/build artifact masking a real import error). Because `tox` builds and installs into a genuinely clean environment each time, it surfaces exactly the class of "works on my machine" bug that testing inside an already-populated dev environment can hide indefinitely.

**Glossary:**
- **`tox.ini`/`noxfile.py`** — the configuration file declaring the environment matrix and commands tox/nox should run.

**Mental model:**
Tests whether the candidate understands tox's value as *environment isolation for testing*, not just "a way to run pytest," and can connect that to a concrete failure mode (undeclared dependencies).

**TL;DR:**
`tox`/`nox` run your tests inside freshly-built, isolated environments across a declared matrix of Python versions/dependencies, catching "works on my machine" bugs — like undeclared dependencies — that testing in your existing dev environment can't reveal.

**References:**
- [tox documentation](https://tox.wiki/en/latest/)

---

### Q18. What is `uv`, and what does it change about the pip+venv workflow? {#q18}

**Question:**
What is `uv`, and what does it change about the pip+venv workflow?

**Good answer:**
`uv` is a Python package and project manager written in Rust, positioned as a drop-in-compatible, much faster replacement for pip, pip-tools, virtualenv, and parts of Poetry/pyenv — its maintainers report roughly 10-100x speedups over pip for installs with a warm cache. Functionally it unifies environment creation (`uv venv`), dependency resolution and locking (`uv lock`, producing `uv.lock`), and installing (`uv sync`/`uv pip install`) into one tool with one consistent, fast resolver, rather than stitching together `venv` + `pip` + `pip-tools` separately.

**Code example:**
```bash
uv venv
uv pip install requests   # pip-compatible interface
uv lock                   # generate uv.lock
uv sync                   # install exactly what's locked
```

**Follow-up question:**
`uv`'s resolver isn't just "the same algorithm as pip but faster" — what's one concrete behavioral difference documented between the two?

**Follow-up good answer:**
Package priority and ordering sensitivity differ: pip has additional internal priority rules that uv doesn't replicate, so `uv pip install foo bar` can resolve differently depending on argument order in ways plain `pip install foo bar` might not; and when a package is available across multiple configured indexes, uv stops at the first index with a match rather than pip's behavior of considering candidates from all configured indexes together — a deliberate choice that also mitigates dependency-confusion attacks, at the cost of resolutions not being guaranteed identical to pip's for the same input.

**Glossary:**
- **`uv.lock`** — uv's lockfile format recording an exact, reproducible resolution of a project's dependencies.

**Mental model:**
Tests whether the candidate is current on tooling that's rapidly displacing pip/Poetry in new projects, and whether they understand it as a different resolver/architecture rather than assuming byte-identical behavior to pip.

**TL;DR:**
`uv` is a Rust-based, unified replacement for pip/pip-tools/virtualenv claiming 10-100x install speedups, but its resolver has real behavioral differences from pip's (priority rules, multi-index handling) rather than being a drop-in identical algorithm.

**References:**
- [uv documentation](https://docs.astral.sh/uv/)
- [uv — pip-compatible interface: resolver differences](https://docs.astral.sh/uv/pip/compatibility/)

---

### Q19. You're starting a new Python project today — pip+venv, Poetry, or uv? What actually differs? {#q19}

**Question:**
You're starting a new Python project today — pip+venv, Poetry, or uv? What actually differs?

**Good answer:**
**pip+venv** (+ optionally `pip-tools` for locking) is the lowest common denominator — always available, no extra tool to install, but you're manually stitching together environment creation, dependency declaration, and locking, and pip's resolver is the slowest of the three for anything with a nontrivial dependency graph. **Poetry** bundles environment management, a `pyproject.toml`-native workflow, and its own lockfile/resolver into one opinionated tool with a mature plugin ecosystem, at the cost of a heavier, Python-implemented (slower) resolver and, historically, friction interoperating with PEP 621 metadata (Poetry predates and initially diverged from it). **uv** targets both speed and pip-compatibility — a Rust resolver claiming 10-100x pip's speed, PEP 621-native `pyproject.toml` support, and a pip-compatible CLI surface for easy adoption — trading some ecosystem maturity (it's newer) for raw performance and standards alignment.

**Follow-up question:**
Beyond raw speed, what's a concrete reason a team might still choose Poetry over uv today?

**Follow-up good answer:**
Poetry has a longer track record and a broader plugin ecosystem (custom publish targets, version-bumping plugins, etc.) that's had years to mature, plus documentation and community familiarity that many teams already have internalized; a team with an existing Poetry-based workflow, CI setup, and institutional knowledge may reasonably decide uv's speed advantage doesn't outweigh the migration cost and the (shrinking, but real) maturity gap in tooling and third-party integrations built specifically around Poetry.

**Glossary:**
- **Resolver** — the component of a package manager responsible for selecting a mutually-compatible set of dependency versions.

**Mental model:**
This is a trade-off/comparison question testing whether the candidate can reason about real engineering trade-offs (speed vs. maturity vs. standards-alignment) rather than reciting "uv is faster, use uv" without nuance.

**TL;DR:**
pip+venv is the manual, always-available baseline; Poetry is a mature, opinionated all-in-one with a slower resolver; uv is the fastest and most PEP 621-aligned option but is the newest of the three — the right choice depends on whether raw speed or ecosystem maturity matters more to the team.

**References:**
- [uv documentation — comparison with other tools](https://docs.astral.sh/uv/)

---

### Q20. Design question: how would you structure dependency management for a monorepo with three internal packages that depend on each other, deployed separately? {#q20}

**Question:**
Design question: how would you structure dependency management for a monorepo with three internal packages that depend on each other, deployed separately?

**Good answer:**
Give each package its own `pyproject.toml` with PEP 621 metadata declaring its dependencies (including on the other internal packages, as ordinary version-ranged dependencies), so each is independently installable and versionable. For local development, use editable installs (`pip install -e ./packages/core`, or a workspace-aware tool's equivalent — uv and Poetry both have workspace support for exactly this) so changes in one package are immediately visible to the others without republishing on every edit. For deployment, each package gets its own lockfile/build so its production install is reproducible and only pulls in a specific, tested version of its internal dependencies, rather than always building against whatever's currently on disk in the monorepo.

**Code example:**
```toml
# packages/api/pyproject.toml
[project]
name = "internal-api"
dependencies = ["internal-core>=1.2,<2"]
```

**Follow-up question:**
What's the risk of skipping internal versioning and just having every package always depend on "whatever's currently in the monorepo," and how does that show up in production?

**Follow-up good answer:**
Without real version boundaries, there's no way to deploy package B against a known-good, tested version of package A — a breaking change to A can silently propagate into B's next deployment with no explicit signal that a dependency changed, since "the dependency" is just whatever happened to be checked out at build time rather than a declared, reviewable version bump. In production this shows up as a deploy of B breaking due to an unrelated change to A that nobody realized was in scope, with no changelog or diff pointing at the actual cause — exactly the failure mode semantic versioning and explicit dependency declarations exist to prevent.

**Glossary:**
- **Workspace** — a package-manager feature for managing multiple interdependent local packages (with editable cross-links) under one root project.

**Mental model:**
A synthesis question combining packaging fundamentals (PEP 621, editable installs) with SE judgment about coupling and deployability in a multi-package codebase — testing whether the candidate can apply tooling knowledge to a realistic architecture problem, not just recite commands.

**TL;DR:**
Give each monorepo package real PEP 621 metadata and versioned dependencies on its siblings, use editable/workspace installs for fast local iteration, but lock and build each package independently for deployment so a breaking change in one doesn't silently ship inside another's release.

**References:**
- [PEP 621 – Storing project metadata in pyproject.toml](https://peps.python.org/pep-0621/)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=python&tags=packaging-testing-and-tooling&autostart=1" | relative_url }})
