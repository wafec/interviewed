---
layout: default
title: "React Interview — Rendering Performance & Reconciliation"
---

# React Interview — Rendering Performance & Reconciliation

Twenty questions on how React actually decides what to render and how fast:
the render/commit cycle, the fiber-based reconciliation algorithm, the hooks
that exist purely to control re-renders (`memo`, `useMemo`, `useCallback`),
the pitfalls that quietly cause extra work (unstable keys, unmemoized
context, stale closures), the concurrent-rendering features built to keep
apps responsive (`startTransition`, `useDeferredValue`, Suspense), and the
methodology/tools (React DevTools Profiler, the `<Profiler>` API, Core Web
Vitals' INP) for actually diagnosing a slow React app instead of guessing.

### Q1. Walk me through what happens, step by step, when you call `setState` in a React component.

**Question:**
Walk me through what happens, step by step, when you call `setState` in a React component.

**Good answer:**
React's update cycle has three steps: **trigger, render, commit**. `setState`
(or a `useState` setter) *triggers* a render — either the initial render via
`root.render()`, or a re-render because state changed. In the *render* step
React calls your component function again to figure out what the UI should
look like now; this must be a pure computation (no side effects, same input
→ same output — React even calls it twice in Strict Mode in development to
catch impurity). If a component returns other components, React recursively
renders those too. Finally, in the *commit* step, React applies only the
**minimal necessary changes** to the actual DOM to match the new render
output — e.g. if only a prop on an `<h1>` changed but a sibling `<input>` is
unchanged in the JSX, React leaves that `<input>` DOM node alone entirely,
which is why an uncontrolled input's typed text and focus survive a
re-render of its siblings.

**Code example:**
```jsx
function Clock({ time }) {
  return (
    <>
      <h1>{time}</h1>
      <input /> {/* never touched by React even though <h1> re-renders */}
    </>
  );
}
```

**Follow-up question:**
Does calling a state setter always cause a synchronous, immediate re-render?

**Follow-up good answer:**
No. State setters schedule a render; React doesn't necessarily run it
synchronously on the same tick. Since React 18, updates are **automatically
batched** almost everywhere — inside React event handlers (as before React
18), but now also inside promises, `setTimeout`, native event handlers, and
any other context. Multiple `setState` calls in the same batch produce a
single re-render, not one per call. If you genuinely need a DOM update to
happen synchronously before the next line of code runs (e.g. to measure a
DOM node right after a state change), you opt out with `flushSync` from
`react-dom`.

**Glossary:**
- **Trigger/Render/Commit** — React's three-step update cycle: something
  schedules a render, React calls component functions to compute output,
  then React applies minimal DOM changes.
- **Automatic batching** — React 18+ behavior of grouping multiple state
  updates from the same event/tick into a single re-render, regardless of
  where the update originates.
- **`flushSync`** — a `react-dom` API that forces a state update to be
  applied to the DOM synchronously, opting out of batching for that update.

**Mental model:**
This tests whether the candidate has an accurate mental model of *when*
React actually touches the DOM, versus assuming every `setState` call is an
independent, immediate re-render — a misconception that leads to writing
code that "over-renders" or relies on synchronous DOM timing that isn't
guaranteed.

**References:**
- [Render and Commit – react.dev](https://react.dev/learn/render-and-commit)
- [React v18.0 – automatic batching](https://react.dev/blog/2022/03/29/react-v18)

---

### Q2. How does React decide whether to preserve or reset a component's state between renders, and where does the `key` prop fit in?

**Question:**
How does React decide whether to preserve or reset a component's state between renders, and where does the `key` prop fit in?

**Good answer:**
React does **not** associate state with the component itself — it associates
state with the component's **position in the render tree**. As long as the
same component type renders at the same position across re-renders, its
state is preserved even if its props change. If a *different* component type
renders at that position (e.g. swapping a `<Counter />` for a `<p>`), React
destroys the old subtree — including its state — and mounts a fresh one. The
`key` prop lets you override this position-based identity: giving two
elements at the same position different `key`s tells React they are
different instances, so switching between them resets state (and switching
back to a key seen before does *not* restore old state — it's a fresh
mount). This is why changing a `key` is a common trick to force-remount a
component.

**Code example:**
```jsx
{/* Different key => different identity => state reset on switch */}
{isPlayerA
  ? <Counter key="Taylor" person="Taylor" />
  : <Counter key="Sarah" person="Sarah" />}
```

**Follow-up question:**
Why is using an array index as the `key` in a rendered list considered an anti-pattern?

**Follow-up good answer:**
An index is a proxy for *position*, not *identity* — exactly the thing keys
exist to override. If the list is reordered, or an item is inserted/deleted
anywhere but the end, every item after that point shifts to a different
index, so React matches the wrong previous state/DOM node to the wrong data.
This causes subtle bugs: a text input's typed value can end up attached to
the wrong list row, per-item local state can appear to "belong" to a
different item, and React does unnecessary DOM mutation/remounting work
instead of the cheap reorder it could have done with stable keys. The fix is
a key derived from the data itself (a database id, or a locally generated
stable id such as `crypto.randomUUID()`), never `Math.random()` (which
changes every render and defeats the purpose entirely).

**Glossary:**
- **Reconciliation** — the process by which React compares a new render
  output to the previous one to compute the minimal set of DOM changes.
- **`key`** — a prop that gives React an explicit, stable identity for an
  element, independent of its position among siblings.

**Mental model:**
Tests whether the candidate understands identity vs. position — a
foundational reconciliation concept that explains a large fraction of "my
component's state got weird after a list update" bug reports in real
codebases.

**References:**
- [Preserving and Resetting State – react.dev](https://react.dev/learn/preserving-and-resetting-state)
- [Rendering Lists – react.dev](https://react.dev/learn/rendering-lists)

---

### Q3. What is React Fiber, and what problem was it introduced to solve?

**Question:**
What is React Fiber, and what problem was it introduced to solve?

**Good answer:**
Fiber is the reconciliation engine React has used internally since React 16,
replacing the earlier "stack reconciler." The old reconciler walked the
component tree recursively using the JS call stack, which meant a large
update was a single, synchronous, uninterruptible unit of work — on a big
tree this could block the main thread long enough to make input handling and
animations visibly janky, with no way to pause. Fiber restructures each
component's unit of work as a node in a linked-list-like data structure (a
"fiber") that tracks its own pending work, so React can process the tree
incrementally: pause after a fiber, yield back to the browser to handle
higher-priority work (like a keystroke), and resume later, or abandon
in-progress work entirely if it's no longer needed. This incremental,
interruptible model is the foundation concurrent features like
`startTransition`, `useDeferredValue`, and Suspense are built on.

**Follow-up question:**
Concretely, what capability does Fiber give React that the old stack
reconciler didn't?

**Follow-up good answer:**
Prioritization and interruption of rendering work. Because each fiber
carries its own state about the work still to do, React can assign different
priorities to different updates (e.g. a text input keystroke is urgent; a
large list re-filter triggered by that keystroke can be a lower-priority
"transition"), start rendering the low-priority update, and if a
higher-priority update comes in mid-render, pause the in-progress work,
handle the urgent update first, then resume or restart the lower-priority
one. The old recursive stack reconciler had no such checkpointing — once it
started walking the tree, it ran to completion before yielding to the
browser at all.

**Glossary:**
- **Fiber** — a unit-of-work data structure React 16+ uses per component
  instance, enabling pausable/resumable/prioritizable rendering.
- **Stack reconciler** — React's pre-16 reconciliation engine, based on
  synchronous recursive calls with no ability to pause mid-tree.
- **Concurrent rendering** — React's ability to work on multiple versions of
  the UI (at different priorities) without blocking the main thread.

**Mental model:**
Separates candidates who've memorized "Fiber = React's reconciler" from
those who understand *why* it exists — the interruptibility/prioritization
story is the actual engineering answer, and it's the prerequisite for
understanding every concurrent-rendering feature asked about later in this
set.

**References:**
- [React Fiber Architecture (Andrew Clark, React core team) – GitHub](https://github.com/acdlite/react-fiber-architecture)
- [Render and Commit – react.dev](https://react.dev/learn/render-and-commit)

---

### Q4. What does `React.memo` actually do, and what's the most common way it fails to prevent a re-render even though "nothing changed"?

**Question:**
What does `React.memo` actually do, and what's the most common way it fails to prevent a re-render even though "nothing changed"?

**Good answer:**
`memo` wraps a component so that, on a parent re-render, React skips
re-rendering it if its props are shallowly equal to the previous render's
props — compared field-by-field using `Object.is`. It's purely a performance
optimization, not a correctness guarantee: React can still choose to
re-render a memoized component, and memoization does nothing to prevent
re-renders triggered by the component's *own* state or context changes. The
most common failure mode is passing a new object, array, or inline function
literal as a prop on every parent render — `Object.is({}, {})` is `false`
because they're different references, even if their contents are identical,
so `memo`'s shallow comparison sees "changed" every time and re-renders
anyway.

**Code example:**
```jsx
// ❌ New object every render defeats memo() on <Profile>
function Page({ name, age }) {
  return <Profile person={{ name, age }} />;
}

// ✅ Memoize the object, or pass primitives instead
function Page({ name, age }) {
  const person = useMemo(() => ({ name, age }), [name, age]);
  return <Profile person={person} />;
}
```

**Follow-up question:**
When would reaching for `memo` actually be the wrong fix, even if it "works"?

**Follow-up good answer:**
When the component wasn't the bottleneck in the first place, or when its
props genuinely change on almost every render — in that case `memo`'s
comparison overhead runs every time and buys nothing. React's own guidance
is to use `memo` only after profiling has shown a real, expensive re-render
problem, not as a default wrapper on every component; reaching for it
preemptively adds indirection and a comparison cost without a measured
benefit, and can mask the real fix (e.g. moving state down, or restructuring
so the expensive subtree isn't a child of the thing that changes often).

**Glossary:**
- **Shallow comparison** — comparing each prop with `Object.is` rather than
  deep-comparing nested structure; two different object references are
  never equal even with identical contents.
- **`Object.is`** — the equality function React uses for prop/dependency
  comparisons; behaves like `===` with edge-case differences for `NaN` and
  signed zero.

**Mental model:**
Tests whether the candidate treats `memo` as a magic "make it fast" button
or understands it as a narrow, reference-equality-dependent optimization
that has to be paired with stable prop references to work at all —
otherwise it's dead weight.

**References:**
- [memo – react.dev](https://react.dev/reference/react/memo)

---

### Q5. When should you reach for `useMemo`, and how do you decide if a calculation is "expensive enough" to justify it?

**Question:**
When should you reach for `useMemo`, and how do you decide if a calculation is "expensive enough" to justify it?

**Good answer:**
`useMemo` caches the *result* of a computation across renders, recomputing
only when one of its listed dependencies changes (compared with
`Object.is`). It's worth using when a calculation is measurably slow
(filtering/transforming a large array, heavy derived-data computation) *and*
its dependencies change infrequently relative to how often the component
re-renders — otherwise you're paying for cache bookkeeping without reaping
cache hits. React's own recommendation is to measure first: wrap the
calculation in `console.time`/`console.timeEnd` (or profile it) and only
reach for `useMemo` if it's actually slow in practice, rather than
memoizing speculatively. It's also commonly used not for the raw compute
savings but to keep a derived value's **reference stable** so it doesn't
break a downstream `memo`-wrapped child or an effect's dependency array.

**Code example:**
```jsx
function TodoList({ todos, tab }) {
  const visibleTodos = useMemo(
    () => filterTodos(todos, tab), // only reruns when todos or tab change
    [todos, tab]
  );
  return <List items={visibleTodos} />; // stable reference -> memo(List) can skip
}
```

**Follow-up question:**
What's the actual cost of using `useMemo`, and why isn't it a "free" performance win to sprinkle everywhere?

**Follow-up good answer:**
`useMemo` isn't free: React still has to store the cached value and its
dependency array, and on every render it has to run the `Object.is`
comparison across all dependencies to decide whether to recompute. For a
cheap calculation, that bookkeeping can cost more than just recomputing the
value directly — so blanket-wrapping every derived value in `useMemo` can
make code slower and harder to read for no benefit. React's guidance is
explicit: treat it as a targeted optimization applied after profiling shows
a real cost, not a default habit; a codebase where "nothing works without
useMemo everywhere" usually has a different underlying problem (e.g. broken
reference stability) that memoizing everywhere is only masking.

**Glossary:**
- **Memoization** — caching a computed value so a later call with the same
  inputs can reuse it instead of recomputing.
- **Dependency array** — the list of values `useMemo`/`useCallback`/`useEffect`
  compare (via `Object.is`) between renders to decide whether to recompute
  / rerun.

**Mental model:**
This is the classic "performance question" trap: candidates who reach for
`useMemo` reflexively, without a profiling methodology, reveal they're
optimizing by superstition rather than measurement — exactly the habit
interviewers are probing for when they ask performance-diagnosis questions.

**References:**
- [useMemo – react.dev](https://react.dev/reference/react/useMemo)

---

### Q6. What's the actual difference between `useMemo` and `useCallback`?

**Question:**
What's the actual difference between `useMemo` and `useCallback`?

**Good answer:**
They cache different things but work identically underneath: `useMemo`
caches the *return value* of a function you pass it, while `useCallback`
caches the *function reference itself* without calling it. In fact,
`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. The
practical use case for `useCallback` is almost always the same as for
`useMemo` — keeping a reference stable so it doesn't break a `memo`-wrapped
child's shallow prop comparison, or doesn't retrigger a `useEffect` that
lists the function as a dependency.

**Code example:**
```jsx
// useMemo caches a computed VALUE
const requirements = useMemo(() => computeRequirements(product), [product]);

// useCallback caches a FUNCTION REFERENCE
const handleSubmit = useCallback((orderDetails) => {
  post('/product/' + productId + '/buy', { referrer, orderDetails });
}, [productId, referrer]);
```

**Follow-up question:**
If you pass a non-memoized callback to a `memo`-wrapped child, what actually happens — does it break, or just under-optimize?

**Follow-up good answer:**
It doesn't break correctness — the child still receives a working function
and behaves correctly. What breaks is the optimization: since a new function
is created (a new reference) on every parent render, the memoized child's
shallow prop comparison sees the `onClick`-style prop as "changed" every
time, so `memo` re-renders the child on every parent render regardless —
functionally identical to not having wrapped it in `memo` at all for that
prop. This is a common reason `memo` "doesn't seem to work" in practice:
the memoization is technically correct, but a sibling prop's unstable
reference defeats it.

**Glossary:**
- **Reference equality** — whether two values point to the same object in
  memory; functions and objects are only reference-equal to themselves, not
  to a newly-created "equivalent" one.

**Mental model:**
Checks whether the candidate can articulate the mechanical relationship
between the two hooks precisely (rather than "useCallback is for functions,
useMemo is for values" as a memorized rule with no underlying model) and
reason about the downstream consequence of skipping one.

**References:**
- [useCallback – react.dev](https://react.dev/reference/react/useCallback)
- [memo – react.dev](https://react.dev/reference/react/memo)

---

### Q7. What is a "stale closure" bug in a `useEffect`, and why does the dependency array exist?

**Question:**
What is a "stale closure" bug in a `useEffect`, and why does the dependency array exist?

**Good answer:**
Every render of a function component creates a new closure over that
render's props and state. An Effect function created during render "closes
over" the values from *that specific render*. If the Effect only runs once
(e.g. `[]` dependency array) but reads a prop or state value that can
change, subsequent changes to that value are invisible to the Effect — it
keeps using the value from the render where it was created. That's a stale
closure. The dependency array exists precisely to prevent this class of bug:
by listing every reactive value the Effect reads, React knows to re-run the
Effect (tearing down the old closure via cleanup, then creating a fresh one)
whenever any of them changes, so the Effect always operates on current
values.

**Code example:**
```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [serverUrl]); // 🔴 missing roomId — Effect keeps using the OLD roomId
}
```

**Follow-up question:**
When dependencies change, in what order do the cleanup and the new setup run — and why does that ordering matter?

**Follow-up good answer:**
React always runs the *previous* render's cleanup function (using the old,
still-closed-over values) before running the *new* setup with the new
values — this happens on every dependency change, not just on unmount:
cleanup(old) → setup(new). Ordering matters because it guarantees symmetry:
if the setup subscribed to something, opened a connection, or registered a
listener using the old value, the matching teardown must happen with that
same old value before a new one is opened, or you leak the old
subscription/connection while a second one silently piles up alongside it.
On final unmount, only the last cleanup runs, with no subsequent setup.

**Glossary:**
- **Closure** — a function bundled with references to the variables in
  scope where it was created; in React, a fresh closure is created every
  render.
- **Stale closure** — a closure that keeps referencing outdated values
  because it wasn't recreated (e.g. an Effect that never re-ran).
- **Cleanup function** — the function an Effect optionally returns, run
  before the next Effect invocation and on unmount, to undo what setup did.

**Mental model:**
Stale closures are one of the most common real-world React bugs reported by
teams, and this question tests whether the candidate has an accurate mental
model of "a new function per render" rather than a vague "just add
everything the linter complains about" heuristic.

**References:**
- [useEffect – react.dev](https://react.dev/reference/react/useEffect)

---

### Q8. What changed about state-update batching in React 18, and how do you opt out when you need a synchronous DOM update?

**Question:**
What changed about state-update batching in React 18, and how do you opt out when you need a synchronous DOM update?

**Good answer:**
Before React 18, React only batched multiple `setState` calls into a single
re-render when they happened inside a React event handler; updates inside
`setTimeout`, native DOM event listeners, promise callbacks, or other async
contexts each triggered their own separate re-render. React 18 introduced
**automatic batching**: updates are batched everywhere by default,
regardless of where they originate, which reduces unnecessary intermediate
renders across the whole app without any code changes. To opt out for a
specific update — e.g. you need the DOM to reflect a state change
immediately before running the next line of synchronous code, such as
measuring a scroll position right after — you wrap that update in
`flushSync` from `react-dom`, which forces it to be applied synchronously
outside the normal batch.

**Code example:**
```js
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 18: one re-render for both, even inside setTimeout
}, 1000);

import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // forces an immediate, synchronous render
```

**Follow-up question:**
Why can over-relying on `flushSync` hurt performance rather than help it?

**Follow-up good answer:**
`flushSync` deliberately breaks batching for that update, forcing a full
synchronous render and DOM commit before returning control. If it's called
repeatedly (e.g. inside a loop, or on every keystroke) it reintroduces the
exact problem automatic batching exists to avoid: React re-renders and
touches the DOM once per call instead of coalescing the work, which can
visibly hurt input responsiveness and layout thrashing on a fast-updating
UI. It should be reserved for the rare cases that genuinely need synchronous
DOM consistency (like a measurement immediately after a state change), not
used as a general-purpose "make my update happen now" habit.

**Glossary:**
- **Automatic batching** — React 18+'s default behavior of grouping state
  updates from the same tick/event into one re-render, everywhere, not just
  inside React event handlers.
- **`flushSync`** — a `react-dom` escape hatch that forces a given update to
  render and commit synchronously, opting out of batching.

**Mental model:**
Probes for accurate, version-aware knowledge (a common trap: candidates who
learned React on 17 assume `setTimeout` updates are never batched) and for
judgment about when an escape hatch is appropriate versus a performance
foot-gun.

**References:**
- [React v18.0 – automatic batching](https://react.dev/blog/2022/03/29/react-v18)

---

### Q9. What problem does `startTransition` solve, and how does it relate to "urgent" vs. "non-urgent" updates?

**Question:**
What problem does `startTransition` solve, and how does it relate to "urgent" vs. "non-urgent" updates?

**Good answer:**
Without it, every state update is treated as equally urgent: if a state
change triggers an expensive re-render (e.g. re-filtering a huge list), that
render work can block the main thread long enough that the UI feels frozen
— even something as simple as typing into an unrelated text box lags.
`startTransition` lets you mark a state update as a **non-urgent
transition**: React still applies it, but it renders it in the background at
lower priority, and if a genuinely urgent update (like a keystroke) comes in
while the transition is mid-render, React can interrupt/restart the
transition to handle the urgent one first, then resume. The result is that
low-priority UI work no longer blocks high-priority interactivity — the app
stays responsive under expensive re-render workloads instead of janking.

**Code example:**
```jsx
function TabContainer() {
  const [tab, setTab] = useState('about');
  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab); // marked non-urgent; can be interrupted by e.g. typing
    });
  }
}
```

**Follow-up question:**
What can't `startTransition` do, and when would you reach for `useTransition` or `useDeferredValue` instead?

**Follow-up good answer:**
`startTransition` itself gives no way to know whether the transition is
still pending — no loading indicator hook. `useTransition` wraps the same
capability but also returns an `isPending` boolean, which is what you'd use
to show a spinner or dim stale content while the transition renders.
`useDeferredValue` solves a related but different problem: instead of
marking a *state update* as low priority, you defer a *value* you've
already received (often a prop you don't control, like `searchQuery` from a
parent) so a render depending on it can lag behind the urgent render by one
or more frames. Also, transitions can't be used to defer synchronous DOM
mutations inside `setTimeout`, and any state update after an `await` inside
an async transition function needs to be wrapped in its own
`startTransition` call — the wrapper doesn't propagate across the await.

**Glossary:**
- **Transition** — a state update marked as interruptible/low-priority via
  `startTransition` or `useTransition`.
- **Concurrent rendering** — React's ability to prepare multiple versions of
  the UI (at different priorities) and interrupt lower-priority work for
  higher-priority updates, enabled by the Fiber architecture.

**Mental model:**
Tests whether the candidate understands transitions as a *scheduling*
mechanism (priority, interruptibility) rather than a vague "makes things
faster" black box, and knows the boundary between the three related APIs
instead of treating them as interchangeable.

**References:**
- [startTransition – react.dev](https://react.dev/reference/react/startTransition)

---

### Q10. How does `<Suspense>` decide when to show its fallback, and what's a common mistake that makes people think "Suspense isn't working"?

**Question:**
How does `<Suspense>` decide when to show its fallback, and what's a common mistake that makes people think "Suspense isn't working"?

**Good answer:**
`<Suspense>` shows its `fallback` while any descendant inside its boundary
is "suspended" — meaning it threw a Promise that hasn't resolved yet. This
happens for a specific set of mechanisms React recognizes: lazy-loaded
component code via `lazy()`, reading a Promise with the `use()` hook
(including data from Server Components or a Suspense-enabled framework),
streaming SSR waiting for HTML to arrive, and a few newer/experimental cases
(stylesheets with `precedence`, canary View Transition font/image loading).
By default all children inside one boundary are treated as a single unit —
they all reveal together once ready, not incrementally — though you can
nest boundaries for progressive reveal. The most common "Suspense isn't
working" mistake is fetching data inside a `useEffect` or an event handler:
Suspense has no visibility into that at all, since it isn't one of the
recognized suspension mechanisms — only `use()` (or a framework's built-in
data layer) integrates with it.

**Code example:**
```jsx
// ❌ Suspense can't see this — it's invisible to the boundary
useEffect(() => { fetchData().then(setAlbums); }, []);

// ✅ Suspense sees this because it reads a Promise via use()
function Albums({ artistId }) {
  const albums = use(fetchData(`/${artistId}/albums`));
  return <ul>{albums.map(a => <li key={a.id}>{a.title}</li>)}</ul>;
}
```

**Follow-up question:**
Why does nesting multiple `<Suspense>` boundaries matter for perceived performance, rather than wrapping the whole page in one?

**Follow-up good answer:**
A single top-level boundary means the *entire* wrapped subtree waits for
its *slowest* suspending descendant before anything appears — a fast
above-the-fold section is held hostage by one slow widget lower down.
Nesting boundaries around independently-loading sections lets React reveal
each piece as soon as its own data/code is ready, so the page renders
progressively (fast parts appear immediately, slower parts show their own
localized fallback) instead of an all-or-nothing wait gated by the single
slowest dependency.

**Glossary:**
- **Suspend** — a component "suspends" when it (or something it reads via
  `use()`) throws a Promise that hasn't resolved, pausing its rendering
  until the Promise settles.
- **`use()`** — a React API for reading the value of a Promise (or Context)
  during render, integrated with Suspense.

**Mental model:**
Checks whether the candidate understands Suspense's actual integration
surface (a specific set of recognized suspension triggers) rather than
treating it as generic "loading state" magic that works with any async
code, which is the single most common misconception.

**References:**
- [Suspense – react.dev](https://react.dev/reference/react/Suspense)

---

### Q11. What are React Server Components, and how do they differ from the client components most React developers are used to?

**Question:**
What are React Server Components, and how do they differ from the client components most React developers are used to?

**Good answer:**
Server Components render ahead of time, in a separate server environment —
either once at build time or per request on a web server — and their code
never ships to the browser at all, which is the key difference from
ordinary ("client") components. They can use `async`/`await` directly in
their render body, and can access a database or filesystem directly, without
building an API layer. What they *can't* do is use interactive/stateful
client APIs — no `useState`, no `useEffect`, no event handlers — because
they don't run on the client and never re-render there. Anything interactive
has to live in a component explicitly marked `"use client"`, which draws a
hard boundary: everything above/around it can be a Server Component (server
data access, zero client bundle cost), and everything inside it behaves like
traditional React.

**Code example:**
```jsx
// Server Component: runs on the server only, code never sent to the client
async function Note({ id }) {
  const note = await db.notes.get(id); // direct DB access, no API round-trip
  return <div>{note.text}</div>;
}
```

**Follow-up question:**
What's the actual bundle-size/performance benefit Server Components give you, concretely — not just "less JS"?

**Follow-up good answer:**
Concretely: any library used only for server-side work (e.g. a markdown
parser plus an HTML sanitizer) never appears in the client bundle at all,
because the component using it never ships — the react.dev example cites
~75KB gzipped of markdown/sanitization libraries avoided entirely this way.
Beyond bundle size, static/data-dependent content can render at build time
or on first request and be visible on first paint without a client-side
data-fetch waterfall (fetch note → then fetch author, etc.), and promises
started on the server can stream to the client and resolve inside a
`<Suspense>` boundary, so slow data doesn't block the whole page's initial
HTML. The trade-off is real complexity: Server Component APIs are explicitly
called out as unstable across React 19.x minor versions, and using them
requires a framework/bundler with support for the server/client split.

**Glossary:**
- **`"use client"`** — a directive marking a module's components as client
  components, defining the boundary where interactivity/browser APIs start.
- **Server Component** — a component that renders only on the server; its
  code is never included in the client JS bundle.

**Mental model:**
Tests whether the candidate can articulate the actual mechanism behind
"smaller bundles" (code never shipping vs. code shipped-then-tree-shaken)
and is aware this is genuinely new architecture with real trade-offs, not
just SSR rebranded.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q12. What do error boundaries actually catch, and what's a mistake teams commonly make assuming an error boundary will handle it?

**Question:**
What do error boundaries actually catch, and what's a mistake teams commonly make assuming an error boundary will handle it?

**Good answer:**
An error boundary is a class component implementing
`static getDerivedStateFromError` (to compute fallback UI state — must be
pure) and/or `componentDidCatch` (for side effects like logging to an
error-tracking service). It catches errors thrown **during rendering**, in
lifecycle methods, and in constructors, anywhere in its child tree,
including distant descendants — replacing the crashed subtree with fallback
UI instead of unmounting the whole app. The common mistake is expecting it
to catch errors thrown in **event handlers** (a `throw` inside an `onClick`)
or in plain **asynchronous code** like a `setTimeout` callback or an
unhandled promise rejection — those are ordinary JS exceptions outside
React's render cycle, and an error boundary simply never sees them; they
need a regular `try/catch` (or a promise `.catch`) instead. One exception:
errors thrown inside a `startTransition` callback are caught, because
transitions are part of React's render/commit machinery.

**Code example:**
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error, info) { logErrorToService(error, info.componentStack); }
  render() {
    return this.state.hasError ? <p>Something went wrong</p> : this.props.children;
  }
}
```

**Follow-up question:**
Why does React require two separate methods (`getDerivedStateFromError` and `componentDidCatch`) instead of one?

**Follow-up good answer:**
They're split by purity requirement, mirroring React's general render/commit
separation. `getDerivedStateFromError` runs during the render phase and must
be a pure function with no side effects — its only job is to compute the
next state (e.g. "show fallback UI") from the error, and React may call
render-phase methods more than once (e.g. in Strict Mode), so it can't be
trusted to safely log to a server exactly once. `componentDidCatch` runs
during the commit phase, after the tree has actually updated, so it's the
safe place for side effects like network calls to an error-logging service —
guaranteed to run once per real error, in a phase where side effects are
allowed.

**Glossary:**
- **Error boundary** — a class component that catches rendering errors in
  its child tree and shows fallback UI instead of unmounting.
- **`componentDidCatch`** — commit-phase lifecycle method for side-effecting
  on a caught error (e.g. logging).

**Mental model:**
Real-world question: teams routinely ship an error boundary and assume
"we're covered" without realizing entire categories of runtime errors
(handlers, async) silently bypass it — this checks whether the candidate has
actually hit that gap in production, not just read the docs once.

**References:**
- [Catching rendering errors with an error boundary – react.dev](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)

---

### Q13. Why must Hooks always be called in the same order on every render, never inside a condition or loop?

**Question:**
Why must Hooks always be called in the same order on every render, never inside a condition or loop?

**Good answer:**
React doesn't track hook state by name or variable — it tracks it by **call
order**, storing each component's hooks as an ordered list tied to that
component's fiber. The first `useState` call always reads/writes the first
slot in that list, the second call the second slot, and so on, for every
render of that component. If a hook call is conditionally skipped on some
render (e.g. inside `if (cond) { useContext(...) }`), every hook call after
it shifts by one position relative to the stored list, so React ends up
handing a later hook the *wrong* stored state — silent, hard-to-debug
corruption rather than a clean crash. The rule ("only call Hooks at the top
level, never in conditions/loops/nested functions") exists purely to
guarantee the call sequence is identical every render, which is the only
thing that makes the position-based storage scheme work at all.

**Code example:**
```jsx
// 🔴 Broken: when cond is false, useState reads the WRONG slot
function Bad({ cond }) {
  if (cond) { useContext(ThemeContext); }
  const [count, setCount] = useState(0);
}

// ✅ Correct: unconditional top-level calls, identical order every render
function Good({ cond }) {
  const theme = useContext(ThemeContext);
  const [count, setCount] = useState(0);
}
```

**Follow-up question:**
Is it fine to have a variable number of hook calls if the branch condition itself never changes after mount?

**Follow-up good answer:**
No — the rule is about call order being identical on *every* render of that
component instance, full stop, regardless of whether a particular condition
happens to be stable in practice. Relying on "this flag never changes after
mount" is fragile and unenforceable by the linter or by React itself; a
future code change (or an unexpected prop) can flip it and silently corrupt
hook state with no error message pointing at the real cause. The
`eslint-plugin-react-hooks` rule flags any conditional hook call
unconditionally, precisely because there's no safe version of "usually
fine."

**Glossary:**
- **Fiber** — React's per-component-instance data structure; each fiber
  holds, among other things, the ordered list of that component's hook
  state.
- **Rules of Hooks** — React's constraint that hooks are called
  unconditionally, in the same order, at the top level of a component or
  custom hook on every render.

**Mental model:**
Tests whether "don't call hooks conditionally" is a memorized linter rule or
an understood consequence of how hook state storage actually works — the
latter lets a candidate reason correctly about edge cases the rule doesn't
explicitly spell out.

**References:**
- [Invalid Hook Call Warning – react.dev](https://react.dev/warnings/invalid-hook-call-warning)

---

### Q14. A teammate says "the app feels laggy when I type in this filter box." What's your actual methodology for diagnosing this, step by step?

**Question:**
A teammate says "the app feels laggy when I type in this filter box." What's your actual methodology for diagnosing this, step by step?

**Good answer:**
First, reproduce and quantify rather than guess: open the browser's
Performance tab (or React DevTools' **Profiler** panel) and record a
session while typing, so you're looking at real timings instead of
intuition. In React DevTools Profiler specifically, start a recording,
interact, stop, and inspect the flame graph/ranked chart per commit — it
shows which components rendered on each commit and how long each took,
immediately surfacing "why did *this* component re-render on *every*
keystroke" candidates. Cross-reference against the `<Profiler>` API's
`actualDuration` vs. `baseDuration` if you want production-safe
instrumentation instead of the extension. Common root causes to check, in
order of likelihood: (1) an expensive computation re-running every render
without `useMemo` because its dependency is unstable; (2) a `memo`-wrapped
component still re-rendering because a parent passes a new
object/array/function reference every render, defeating the shallow prop
comparison; (3) a Context value changing on every render, force-re-rendering
every consumer regardless of `memo`; (4) simply too much work happening
synchronously on every keystroke that should instead be deferred with
`useDeferredValue` or debounced. Only after profiling identifies the actual
expensive component/render do you reach for `memo`/`useMemo`/`useCallback`
— applying them blind is how you get code that's harder to read without
being measurably faster.

**Follow-up question:**
Beyond React DevTools, what browser-level signal would tell you this input-lag problem is actually a **Core Web Vitals** issue worth prioritizing?

**Follow-up good answer:**
Interaction to Next Paint (INP) — the Core Web Vital that replaced First
Input Delay, measuring end-to-end responsiveness across *all* user
interactions on the page (not just the first one), by observing the latency
between an interaction and the next paint via the Event Timing API. Google's
guidance targets an INP of 200ms or less for a "good" experience; a laggy
filter box that keeps triggering slow re-renders on every keystroke will
show up directly as inflated INP in field data (e.g. via Chrome UX Report or
`web-vitals` JS library instrumentation) — which is the signal that
justifies spending engineering time on this over some other perceived
sluggishness, and the metric you'd re-check after a fix to confirm it
actually helped real users, not just your local recording.

**Glossary:**
- **Flame graph / ranked chart** — React DevTools Profiler visualizations
  showing per-component render time within a commit, letting you spot the
  most expensive or most frequently re-rendering components.
- **INP (Interaction to Next Paint)** — a Core Web Vital measuring the
  latency of user interactions across a page's lifetime; ≤200ms is
  considered good.

**Mental model:**
This is the archetypal "performance methodology" interview question —
graders are listening for reproduce → measure → hypothesize by likelihood →
fix → re-measure, using named tools at each step, versus a candidate who
jumps straight to "I'd add `useMemo`" without ever profiling first.

**References:**
- [React Developer Tools – react.dev](https://react.dev/learn/react-developer-tools)
- [Profiler – react.dev](https://react.dev/reference/react/Profiler)
- [Interaction to Next Paint (INP) – web.dev](https://web.dev/articles/inp)

---

### Q15. How does the `<Profiler>` component's `onRender` callback let you distinguish "this render was slow" from "this render was unnecessary but cheap"?

**Question:**
How does the `<Profiler>` component's `onRender` callback let you distinguish "this render was slow" from "this render was unnecessary but cheap"?

**Good answer:**
`<Profiler id="App" onRender={onRender}>` wraps a subtree and calls
`onRender(id, phase, actualDuration, baseDuration, startTime, commitTime)`
on every commit within it. `actualDuration` is how long this specific update
actually took to render the subtree; `baseDuration` is an estimate of how
long it would take to render that same subtree from scratch *without* any
memoization optimizations. Comparing the two tells you which problem you
have: if `actualDuration` is much lower than `baseDuration`, memoization is
doing real work and skipping most of the subtree efficiently. If they're
close together, either memoization isn't helping (nothing's actually being
skipped) or the render genuinely has to redo the full subtree's work every
time — in which case the fix isn't "add more memoization," it's reducing
what actually needs to recompute. `phase` (`"mount"` / `"update"` /
`"nested-update"`) tells you whether this is the expensive initial mount
(often fine to be slower) versus a re-render (where high duration is more
often the actual problem).

**Follow-up question:**
Why is `<Profiler>`'s data disabled in production builds by default, and how do you get real production numbers if you need them?

**Follow-up good answer:**
Collecting per-render timing data has real runtime overhead — timing every
commit and computing `actualDuration`/`baseDuration` adds work on every
render, which is unacceptable to pay for on every user's machine by default.
React ships this instrumentation compiled out of the standard production
build for that reason. To get real production numbers, you build against a
special **profiling build** of `react-dom` that keeps the instrumentation
enabled in an otherwise production (minified) bundle, deploy that
selectively (e.g. to a percentage of traffic or an internal cohort), and
read the `<Profiler>` callback data from real user sessions instead of only
ever profiling your own dev machine, which frequently doesn't represent real
user hardware/network conditions.

**Glossary:**
- **`actualDuration`** — milliseconds spent rendering the profiled tree for
  the current update.
- **`baseDuration`** — estimated milliseconds to render the same tree from
  scratch with no memoization, used as a baseline for comparison.
- **Profiling build** — a special `react-dom` build that keeps Profiler
  instrumentation active inside an otherwise production-minified bundle.

**Mental model:**
Distinguishes candidates who know `<Profiler>` exists from candidates who
can actually read its output to form a diagnosis — the
`actualDuration`/`baseDuration` comparison is the whole point of the API,
and is exactly the kind of "which tool, which number, what conclusion"
detail a strong senior candidate should have internalized from real
profiling work.

**References:**
- [Profiler – react.dev](https://react.dev/reference/react/Profiler)

---

### Q16. If you put frequently-changing state into React Context so multiple components can read it, what performance problem are you likely to introduce?

**Question:**
If you put frequently-changing state into React Context so multiple components can read it, what performance problem are you likely to introduce?

**Good answer:**
When a Context's value changes, **every** component that calls
`useContext` on that Context re-renders — regardless of whether that
specific consumer actually uses the part of the value that changed, and
even if the consumer is wrapped in `memo` (Context reads bypass `memo`'s
prop-based shallow comparison entirely, since the trigger isn't a prop
change). If you put fast-changing state (e.g. mouse position, a text input
value, a reducer with frequent dispatches) into a single Context consumed
widely across the tree, every one of those consumers re-renders on every
change, even ones only interested in a completely unrelated piece of the
value. This is a common cause of "why is everything re-rendering" bugs in
apps that reach for Context as a general-purpose global store.

**Code example:**
```jsx
// Splitting state and dispatch into separate contexts limits blast radius:
// dispatch never changes, so consumers reading only TasksDispatchContext
// don't re-render when the tasks state itself updates.
<TasksContext value={tasks}>
  <TasksDispatchContext value={dispatch}>
    {children}
  </TasksDispatchContext>
</TasksContext>
```

**Follow-up question:**
What are two concrete mitigation strategies for this, beyond "just don't use Context"?

**Follow-up good answer:**
First, **split the Context** by what actually changes together: separating
frequently-changing state from rarely-changing dispatch/callback references
(as in the code example) means a component that only needs `dispatch`
doesn't re-render at all when the state value changes, since `dispatch`'s
reference stays stable. Second, **narrow the scope** of who's inside the
Provider and consuming it — pushing the Provider down closer to only the
subtree that actually needs the value limits the blast radius of a change,
versus wrapping the entire app in one Provider so every distant, unrelated
component pays the re-render cost. For genuinely high-frequency updates
shared across a large, unrelated part of the tree, an external state
library with selector-based subscriptions (which re-render only components
reading the specific slice that changed) is often a better fit than Context
at all — Context is not designed as a fine-grained reactive store.

**Glossary:**
- **Context** — React's mechanism for making a value available to a
  subtree without prop drilling; any consumer re-renders on value change.
- **Blast radius** — how much of the tree is forced to re-render as a
  consequence of one state change.

**Mental model:**
Tests whether the candidate treats Context as "free global state" (a common
junior/mid-level assumption) or understands its actual re-render contract
well enough to reach for it appropriately versus recognizing when it's the
wrong tool.

**References:**
- [Scaling Up with Reducer and Context – react.dev](https://react.dev/learn/scaling-up-with-reducer-and-context)

---

### Q17. "React uses a virtual DOM, so it's always faster than manipulating the real DOM directly." Is that a fair statement — how would you correct it?

**Question:**
"React uses a virtual DOM, so it's always faster than manipulating the real DOM directly." Is that a fair statement — how would you correct it?

**Good answer:**
Not as stated — it conflates "faster than the naive alternative" with
"faster than any possible alternative," and the latter is false: a real
DOM API call, executed exactly when and where you know it's needed, is
essentially always faster than diffing a virtual tree and then issuing that
same DOM call, because the diffing itself has a cost. The virtual DOM's
actual value proposition is different: it lets *you* write declarative code
("render this UI given this state") without hand-tracking every possible
transition between UI states yourself, while React does the bookkeeping to
translate "new declared output" into a *minimal, correct* set of real DOM
operations automatically. That's a productivity and correctness trade —
avoiding the class of bugs from manually written, ad-hoc imperative DOM
updates getting out of sync with app state — traded against some raw
per-update overhead versus perfectly hand-optimized imperative code, which
you'd almost never actually write correctly and consistently at scale by
hand.

**Follow-up question:**
Give a concrete scenario where hand-written imperative DOM manipulation genuinely outperforms React's approach — where would that trade-off flip?

**Follow-up good answer:**
A tight, highly localized update loop — e.g. dragging an element and
updating its `transform` style on every `pointermove` event, or a canvas/WebGL
animation loop — is a case where going straight to
`element.style.transform = ...` (or a canvas draw call) every frame will
outperform routing that update through React's render/reconcile/commit
cycle, because there's no benefit to diffing when you already know exactly
which single DOM property needs to change and exactly when. This is why
performance-sensitive libraries (drag interactions, animation libraries)
frequently escape-hatch to direct DOM manipulation via refs for the hot path
specifically, while still using React for everything else — it's not an
either/or, it's picking the right tool for each specific piece of UI
behavior.

**Glossary:**
- **Virtual DOM** — React's in-memory representation of the desired UI tree,
  diffed between renders to compute the minimal real-DOM update.
- **Declarative UI** — describing *what* the UI should look like for a given
  state, versus imperatively describing the steps to transform it.

**Mental model:**
This is an SE-theory-mixed-with-practice question: it checks whether the
candidate actually understands the trade-off React makes (developer
ergonomics/correctness vs. raw per-op speed) instead of repeating "virtual
DOM = fast" as an unexamined slogan, and whether they can identify the
narrow band of cases where the trade-off genuinely reverses.

**References:**
- [Render and Commit – react.dev](https://react.dev/learn/render-and-commit)

---

### Q18. What does `useDeferredValue` do, and how is it different from debouncing the same input with `setTimeout`?

**Question:**
What does `useDeferredValue` do, and how is it different from debouncing the same input with `setTimeout`?

**Good answer:**
`useDeferredValue(value)` gives you back a version of `value` that may
"lag behind" the real one during expensive renders — React renders with the
old deferred value first (keeping the UI responsive), then re-renders in the
background with the new value once it's ready, using the same interruptible
transition machinery as `startTransition`. The critical difference from
debouncing is *when* the update happens: a debounce with `setTimeout`
imposes a fixed artificial delay (e.g. "wait 300ms after the last
keystroke") regardless of whether the device could actually keep up faster
— on a fast machine you're leaving free responsiveness on the table, and on
a genuinely slow render you might still not have waited long enough.
`useDeferredValue` has no fixed delay at all: it responds to *actual*
render cost, deferring only for as long as the expensive render genuinely
takes, and can be interrupted immediately by a subsequent urgent update
(like the next keystroke) — so it's an adaptive scheduling primitive, not a
fixed-timer heuristic.

**Code example:**
```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // Expensive list renders against the possibly-lagging deferredQuery,
  // while the input itself (bound to `query` directly) stays instantly responsive.
  return <ExpensiveResultsList query={deferredQuery} />;
}
```

**Follow-up question:**
When would you reach for `useDeferredValue` instead of `startTransition`, given they're both built on the same concurrent-rendering mechanism?

**Follow-up good answer:**
`startTransition` wraps a *state update you're triggering* — you control
the `setState` call, so you mark it as a transition at the source. Use
`useDeferredValue` instead when you *don't* control the update that produces
the value — e.g. it's a prop passed down from a parent you don't own, or it
comes from a third-party hook/library — so you can't wrap the original
`setState` call in `startTransition` yourself. Deferring the *value* at the
point you consume it achieves the same "don't let this block urgent
interactivity" effect without needing access to where the state update
originates.

**Glossary:**
- **`useDeferredValue`** — a hook returning a version of a value that can
  lag behind during expensive renders, without imposing a fixed delay.
- **Debounce** — a fixed-delay technique (typically via `setTimeout`) that
  waits for a pause in events before acting, independent of actual render
  cost.

**Mental model:**
Checks whether the candidate understands concurrent-rendering scheduling as
fundamentally different from timer-based debouncing — a common wrong answer
is treating `useDeferredValue` as "just React's built-in debounce," which
misses the adaptive, interruptible nature that's the actual point of the
API.

**References:**
- [useDeferredValue – react.dev](https://react.dev/reference/react/useDeferredValue)
- [startTransition – react.dev](https://react.dev/reference/react/startTransition)

---

### Q19. You're asked to reduce re-renders across a mid-size React app with no profiling data yet. What's your prioritized approach, and why in that order?

**Question:**
You're asked to reduce re-renders across a mid-size React app with no profiling data yet. What's your prioritized approach, and why in that order?

**Good answer:**
Profile before touching any code — record a real user flow in React
DevTools Profiler (or `<Profiler>` in a canary/profiling build for
production-representative data) to get an actual list of expensive/frequent
re-renders ranked by cost, rather than guessing which components "feel"
slow. From there, prioritize by expected impact: (1) components that
re-render on *every* keystroke/scroll/frequent event and do meaningful work
— these dominate perceived responsiveness (and directly show up in INP);
(2) components high in the tree whose re-render cascades to a large
subtree, since fixing one node there (via `memo` + stable prop references,
or restructuring so state lives lower/closer to what needs it) prevents many
descendant re-renders at once, versus micro-optimizing a single leaf
component; (3) unstable references flowing through Context to many
consumers, since that's a single root cause with a wide blast radius; (4)
only then, targeted `useMemo`/`useCallback` on specific expensive
calculations the profiler actually flagged. This ordering matters because
structural fixes (state placement, Context splitting, memoizing a
high-fan-out component) typically eliminate whole categories of unnecessary
work, while reflexively memoizing individual components first tends to add
complexity for marginal, hard-to-measure gains and can miss the actual
bottleneck entirely.

**Follow-up question:**
After applying a fix, how do you validate it actually worked rather than just assuming it did?

**Follow-up good answer:**
Re-profile the *same* recorded interaction with the same tooling used to
diagnose it — compare the new flame graph/`actualDuration` numbers against
the "before" recording for the specific components you targeted, confirming
render count and duration actually dropped, not just that the code "looks
more optimized." For a user-facing responsiveness claim, also check the
real-world metric it should move — INP in field data (via a
`web-vitals`-instrumented RUM pipeline or Chrome UX Report), since a
component-level profiler improvement doesn't automatically guarantee it
moved the metric users actually experience; regressions elsewhere or
diminishing returns are easy to miss without checking the end-to-end number
you originally cared about.

**Glossary:**
- **Fan-out** — how many descendant components/subtrees a given component's
  re-render cascades into; high-fan-out components are high-leverage
  optimization targets.
- **RUM (Real User Monitoring)** — collecting performance metrics from
  actual users' sessions in production, as opposed to synthetic/local
  measurements.

**Mental model:**
This is the "how do you actually approach a performance investigation"
question interviewers increasingly lead with — it tests process and
prioritization judgment under ambiguity, not just hook trivia, and the
follow-up specifically checks whether the candidate closes the loop with
validation instead of treating a plausible-sounding fix as done.

**References:**
- [React Developer Tools – react.dev](https://react.dev/learn/react-developer-tools)
- [Interaction to Next Paint (INP) – web.dev](https://web.dev/articles/inp)

---

### Q20. Compare Context + `useReducer` against an external state-management library for cross-cutting app state. What's the actual trade-off, not just "Context is built-in"?

**Question:**
Compare Context + `useReducer` against an external state-management library for cross-cutting app state. What's the actual trade-off, not just "Context is built-in"?

**Good answer:**
Context + `useReducer` costs nothing extra to adopt (no dependency, fits
naturally with hooks) and is a fine default for state that changes
infrequently or is consumed by a small, well-defined part of the tree — the
`TasksContext`/`TasksDispatchContext` pattern of splitting state from
dispatch is the standard mitigation for its main weakness. That weakness is
the re-render model itself: Context has no concept of "subscribe to only
this slice of the value" — any consumer of a Context re-renders on *any*
change to that Context's value, full stop, regardless of which field
actually changed. External state libraries built for this problem (e.g.
selector-based stores) let a component subscribe to a derived slice of
global state and only re-render when that specific slice's value actually
changes, which scales much better to a large app with lots of independent
consumers reading different slices of one big state tree — exactly the
scenario where Context's all-or-nothing re-render behavior becomes a real
cost. The trade-off is added dependency weight and a new abstraction to
learn, against meaningfully finer-grained re-render control at scale.

**Follow-up question:**
Given that trade-off, what's a concrete signal in a real codebase that tells you it's time to move off Context for a piece of state, rather than just splitting it further?

**Follow-up good answer:**
When profiling shows a specific Context's consumers spread across many
unrelated parts of a large tree, each re-rendering on changes to fields they
don't read, *and* splitting the Context further into narrower contexts
would require creating a context-per-field (an unmanageable number of
providers/boundaries) rather than a clean two- or three-way split like
state/dispatch — that's the concrete signal the "split it more" mitigation
has hit its structural limit, not just that it hasn't been tried hard
enough. At that point, a selector-based store's fine-grained subscription
model solves the actual problem directly instead of fighting Context's
coarse-grained re-render contract with more and more provider nesting.

**Glossary:**
- **Selector-based subscription** — a state-management pattern where a
  component subscribes to a derived function of global state and re-renders
  only when that derived value changes, not on every store update.
- **Cross-cutting state** — state read/written by many, often unrelated,
  parts of the component tree (e.g. auth session, feature flags, a shopping
  cart).

**Mental model:**
Trade-off questions like this test whether a candidate defaults to tool
tribalism ("always use Context," "always use a store library") or can
articulate the actual mechanical difference (coarse vs. fine-grained
re-render subscription) that should drive the decision case by case.

**References:**
- [Scaling Up with Reducer and Context – react.dev](https://react.dev/learn/scaling-up-with-reducer-and-context)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=react&tags=rendering-performance-and-reconciliation&autostart=1" | relative_url }})
