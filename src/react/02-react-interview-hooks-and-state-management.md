---
layout: default
title: "React Interview — Hooks & State Management"
---

# React Interview — Hooks & State Management

Twenty questions on React's hooks system and state-management patterns: how
hooks are actually stored and ordered under the hood, the mental model
behind batching and Context, the pitfalls that trip up even experienced
developers (stale closures, dependency arrays, referential equality), and
the trade-offs between local state, Context, and external state libraries.

### Q1. Why can't you mutate state directly in React (e.g. `state.push(x)` on an array in state), and what should you do instead?

**Question:**
Why can't you mutate state directly in React (e.g. `state.push(x)` on an array in state), and what should you do instead?

**Good answer:**
React decides whether to re-render by comparing the previous state value to
the new one you pass to the setter, using `Object.is`-style reference
comparison for objects/arrays. If you mutate an object or array in place,
its reference never changes, so React can't tell anything happened and
won't schedule a re-render — even though the underlying data did change.
Beyond that, React (and React DevTools' time-travel, and features like
concurrent rendering) relies on being able to treat a past render's state as
an immutable snapshot; mutating it out from under a component that's mid
render can produce inconsistent UI. The fix is to always create a new
object/array with the same shape (`[...arr, x]`, `{...obj, key: value}`,
`arr.map(...)`) and pass that new reference to the state setter.

**Code example:**
```jsx
// Wrong: mutates in place, no re-render is scheduled
function addTodo(todos, setTodos, text) {
  todos.push({ id: Date.now(), text }); // same array reference
  setTodos(todos); // React sees the same reference, bails out
}

// Right: create a new array reference
function addTodo(todos, setTodos, text) {
  setTodos([...todos, { id: Date.now(), text }]);
}
```

**Follow-up question:**
What's the "you might not need an Effect" version of this mistake — mirroring a prop into state instead of just using the prop or deriving a value during render?

**Follow-up good answer:**
A common related mistake is copying a prop into `useState` (`const [value] = useState(props.value)`) so it can be "local." `useState`'s initial value is only used on the very first render — if `props.value` changes later, the state won't follow it, producing stale, duplicated state that drifts from its source of truth. React's own guidance is: if a value can be computed entirely from existing props/state during render, don't store it in state at all — just compute it inline (or memoize the computation with `useMemo` if it's expensive). Reach for `useState` to *initialize* from a prop only when you deliberately want an editable local copy that's allowed to diverge (e.g. an uncontrolled form field seeded from a default), and in that case reset it explicitly via a `key` change rather than a `useEffect` that re-syncs it.

**Glossary:**
- **Reference equality** — comparing whether two variables point to the same object in memory, not whether their contents are equal.
- **Derived state** — a value that can be fully computed from other state/props and therefore shouldn't be stored redundantly in its own state slot.

**Mental model:**
This tests whether the candidate understands that React's rendering model is
built on immutable snapshots and reference comparison, not deep equality or
imperative mutation — a foundational mismatch with how many developers
instinctively work with JS data structures.

**TL;DR:**
Mutating state in place leaves its reference unchanged, so React can't detect the change — always pass a new object/array to the setter.

**References:**
- [Updating Objects in State – react.dev](https://react.dev/learn/updating-objects-in-state)
- [You Might Not Need an Effect – react.dev](https://react.dev/learn/you-might-not-need-an-effect)

---

### Q2. React 18 introduced "automatic batching." What does that mean, and how is it different from React 17's batching behavior?

**Question:**
React 18 introduced "automatic batching." What does that mean, and how is it different from React 17's batching behavior?

**Good answer:**
Batching means React groups multiple `setState` calls that happen during
the same tick into a single re-render, rather than re-rendering after each
call. In React 17 and earlier, this batching only happened inside React's
own event handlers (e.g. `onClick`); state updates inside a `setTimeout`,
a native DOM event listener, a `Promise.then`, or an `async` function each
triggered their own separate synchronous re-render. React 18's automatic
batching (enabled by using `createRoot`) extends batching to *all* of those
contexts — timeouts, promises, native event handlers — so multiple
`setState` calls anywhere are batched into one re-render by default. You can
still force an update to flush synchronously outside batching using
`flushSync` from `react-dom` when you need the DOM to reflect a state change
immediately (e.g. before measuring layout).

**Code example:**
```jsx
function handleClick() {
  setTimeout(() => {
    setCount(c => c + 1);
    setFlag(f => !f);
    // React 18 (createRoot): batched into ONE re-render
    // React 17: TWO separate re-renders (setTimeout isn't a React event)
  }, 0);
}
```

**Follow-up question:**
If you need one specific state update inside that timeout to force an immediate, unbatched re-render, how do you do it?

**Follow-up good answer:**
Wrap that specific update in `flushSync` from `react-dom`: `flushSync(() => setFlag(f => !f))`. This forces React to apply that update and flush the DOM synchronously before continuing, opting that update out of automatic batching. It's an escape hatch meant for rare cases (e.g. you need to read a DOM measurement, like scroll position, immediately after a state change) — using it routinely defeats the performance benefit batching exists to provide, so it should be reached for deliberately, not by default.

**Glossary:**
- **Batching** — combining multiple state updates into a single re-render pass instead of one render per update.
- **`flushSync`** — a `react-dom` API that forces a state update (and its DOM commit) to happen synchronously, bypassing batching.

**Mental model:**
Tests whether the candidate has kept up with a genuine React 18 behavioral
change that silently alters re-render counts and can break code that
(incorrectly) relied on synchronous re-renders after every `setState`.

**TL;DR:**
React 18's automatic batching extends batching (many setStates → one re-render) to timeouts/promises/native handlers, not just React event handlers; `flushSync` opts a specific update out.

**References:**
- [Queueing a Series of State Updates – react.dev](https://react.dev/learn/queueing-a-series-of-state-updates)
- [`flushSync` – react.dev](https://react.dev/reference/react-dom/flushSync)

---

### Q3. Why must Hooks always be called in the same order on every render (the "Rules of Hooks"), and what breaks internally if that rule is violated?

**Question:**
Why must Hooks always be called in the same order on every render (the "Rules of Hooks"), and what breaks internally if that rule is violated?

**Good answer:**
React doesn't associate hook state with the variable name you assigned it
to — it associates it with the *position* in which the hook was called.
Internally, each component instance (fiber) keeps a linked list of "hook"
objects; on each render, React walks that list in order, handing each
`useState`/`useEffect`/etc. call the next node in the list. If a hook is
called conditionally (inside an `if`, a loop, or after an early return) and
the condition differs between renders, the number or order of hook calls
changes, so React ends up handing the wrong stored state/effect to the
wrong hook call — corrupting state silently or throwing an "Rendered fewer
hooks than expected" error. That's why Hooks must always be called
unconditionally at the top level of the component, in the same order every
render.

**Code example:**
```jsx
// Wrong: conditional hook call — order changes between renders
function Profile({ showBio }) {
  if (showBio) {
    const [bio, setBio] = useState(""); // sometimes called, sometimes not
  }
  const [name, setName] = useState(""); // now this can land on the wrong slot
}

// Right: always call hooks, branch on the value instead
function Profile({ showBio }) {
  const [bio, setBio] = useState("");
  const [name, setName] = useState("");
  if (!showBio) { /* just don't render bio */ }
}
```

**Follow-up question:**
Why don't the equivalent rules apply to plain function calls or `if` statements elsewhere in your component's render body?

**Follow-up good answer:**
Ordinary variables and function calls don't need to persist identity across renders — a local `const x = compute()` is just recomputed fresh every render, with no expectation that render N's `x` is "the same" as render N-1's. Hooks are different: they exist specifically to give a function component *persistent* state and behavior across renders (a `useState` call needs to keep returning the same stored value until it's updated; a `useEffect` needs to compare its dependencies to the previous render's). Because React has no other way to correlate "this hook call" across renders except position in the call sequence (there's no hook "name" it can key off), that positional list is the entire mechanism — and it only works if the sequence is stable.

**Glossary:**
- **Fiber** — React's internal per-component-instance data structure that (among other things) holds the linked list of that instance's hooks.
- **Rules of Hooks** — the constraint that hooks must be called unconditionally, in the same order, only from React function components or custom hooks.

**Mental model:**
This is a classic "explain the internals, not just the rule" question — it
separates candidates who've memorized "don't call hooks conditionally" from
those who understand *why*, which is what lets them debug subtle
order-related state bugs instead of just avoiding the obvious cases.

**TL;DR:**
Hooks are tracked by call position in a per-fiber linked list, not by name — calling one conditionally shifts that order and corrupts state, hence the Rules of Hooks.

**References:**
- [Rules of Hooks – react.dev](https://react.dev/reference/rules/rules-of-hooks)
- [Invalid Hook Call Warning – react.dev](https://react.dev/warnings/invalid-hook-call-warning)

---

### Q4. What is a "stale closure" bug in a `useEffect`, and how does an incomplete dependency array cause it?

**Question:**
What is a "stale closure" bug in a `useEffect`, and how does an incomplete dependency array cause it?

**Good answer:**
Every render of a function component creates a brand-new closure — the
functions defined inside it (including the callback passed to `useEffect`)
capture the props/state values as they were *during that specific render*.
If you omit a value from the dependency array, the effect only re-runs when
the values you *did* list change, but the function body itself still
references the props/state as captured in whichever render last set up the
effect — so it keeps reading an outdated ("stale") value instead of the
current one. This shows up classically as a `setInterval` callback inside
`useEffect(() => { setInterval(() => console.log(count), 1000) }, [])` that
always logs the initial `count`, because the closure over `count` was
captured once, on mount, and the effect never re-ran to recreate it with a
fresh closure.

**Code example:**
```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // stale: always logs 0, the value from mount
    }, 1000);
    return () => clearInterval(id);
  }, []); // missing `count` in deps — effect never re-runs to refresh the closure

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Follow-up question:**
What are two different fixes for this, and what's the trade-off between them?

**Follow-up good answer:**
1. **Add the missing dependency** (`[count]`): correct and simple, but it means the effect tears down and re-runs (clearing and re-creating the interval) on every `count` change, which is wasteful/unwanted if the interval itself shouldn't restart.
2. **Use a functional state update** (`setCount(c => c + 1)`, or read the latest value from a ref) so the effect doesn't need `count` in its closure at all — this lets the effect run once and stay stable, at the cost of only working for updates that don't need to *read* the current value elsewhere (just update it). For cases where you genuinely need to read fresh props/state inside a stable effect without re-running it, React added `useEffectEvent` (stable, React 19.2) specifically to extract that "always latest, never a dependency" logic out of the effect.

**Glossary:**
- **Closure** — a function bundled with references to the variables from the scope it was created in.
- **Stale closure** — a closure that keeps referencing an old, outdated value because it was captured in a render that hasn't been superseded by a fresh effect run.

**Mental model:**
Tests whether the candidate actually understands JS closures over React's
render cycle, not just "the linter told me to add a dependency" — this is
one of the most common real-world React bugs and a strong signal of
conceptual depth.

**TL;DR:**
A stale closure happens when an effect's callback captured an old value and the effect never re-runs to refresh it, because that value was left out of the dependency array.

**References:**
- [Removing Effect Dependencies – react.dev](https://react.dev/learn/removing-effect-dependencies)
- [`useEffectEvent` – react.dev](https://react.dev/reference/react/useEffectEvent)

---

### Q5. `useMemo` and `useCallback` both "cache" something between renders. When do they actually help performance, and when are they pointless or even harmful?

**Question:**
`useMemo` and `useCallback` both "cache" something between renders. When do they actually help performance, and when are they pointless or even harmful?

**Good answer:**
`useMemo` caches the *result* of an expensive computation, recomputing it
only when its dependencies change; `useCallback` caches a *function
reference* itself (it's really `useMemo` specialized for functions). Both
only pay off when: (1) the computation is genuinely expensive enough that
recomputing it every render matters, or (2) the *referential stability* of
the value matters downstream — e.g. it's passed as a prop to a
`React.memo`-wrapped child, or used as a dependency in another hook's
array, where a new reference every render would defeat memoization or
retrigger effects. Used everywhere reflexively, they're often net-negative:
each call still costs memory (storing the cached value/deps) and a
dependency-array comparison on every render, and if nothing downstream
actually benefits from stable references, you're paying that cost for
nothing. The React team's own guidance is to profile first and add
memoization where a profiler shows it matters, not as a default habit.

**Code example:**
```jsx
// Pointless: cheap computation, no memoized child consuming it
const label = useMemo(() => `Hello, ${name}`, [name]); // string concat isn't expensive

// Worth it: stable reference required by a memoized child + genuinely expensive derive
const sorted = useMemo(() => bigList.slice().sort(expensiveComparator), [bigList]);
const handleSelect = useCallback((id) => onSelect(id), [onSelect]);
return <ExpensiveMemoizedList items={sorted} onSelect={handleSelect} />;
```

**Follow-up question:**
How would you actually verify, in a real app, whether adding `useMemo`/`useCallback` to a component made things faster or not?

**Follow-up good answer:**
Use the React DevTools Profiler: record a session, trigger the interaction in question, and compare the flame graph / ranked chart before and after the change — look at both the render duration of the component itself and whether downstream children re-rendered when they shouldn't have (the Profiler highlights "why did this render" reasons in recent versions). For a more general performance signal, pair this with browser DevTools' Performance tab to see actual frame timing/long tasks, and in production, watch Core Web Vitals like INP (Interaction to Next Paint) for regressions. The key discipline is measuring before *and* after the memoization change on the same interaction — "it feels faster" isn't evidence, and speculative memoization is a common source of code complexity that doesn't pay for itself.

**Glossary:**
- **Referential stability** — a value keeping the same object/function reference across renders unless its inputs actually changed.
- **React DevTools Profiler** — a browser extension panel that records render timings per component and why each render happened.

**Mental model:**
This is the "performance methodology" question for React — it's testing
whether the candidate treats memoization as a measured optimization or a
cargo-culted default, which is exactly the distinction senior candidates are
expected to draw.

**TL;DR:**
`useMemo`/`useCallback` only pay off when the computation is genuinely expensive or referential stability matters downstream (memoized children, hook deps) — otherwise they're pure overhead.

**References:**
- [`useMemo` – react.dev](https://react.dev/reference/react/useMemo)
- [`useCallback` – react.dev](https://react.dev/reference/react/useCallback)

---

### Q6. Walk through your methodology for diagnosing "this component re-renders too often" in a real app.

**Question:**
Walk through your methodology for diagnosing "this component re-renders too often" in a real app.

**Good answer:**
First, confirm there's actually a problem — open React DevTools' Profiler,
enable "Record why each component rendered" (in the Profiler settings), and
record the interaction that feels slow. The flame graph shows every
component that rendered in that commit, its duration, and — with that
setting on — the reason (props changed, state changed, hooks changed,
parent re-rendered, context changed). From there: (1) check whether a
child re-rendered because its *own* state/props actually changed (expected,
not a bug) vs. because its parent re-rendered and passed new object/array/
function literals as props (a classic unstable-reference issue); (2) check
whether a Context value change is broadcasting re-renders to consumers that
only care about part of the value; (3) for deep/expensive subtrees,
consider `React.memo` on the child plus `useMemo`/`useCallback` on the
parent to stabilize the props it receives, or restructure state so it lives
closer to where it's used instead of high up triggering a broad re-render.
Fix one thing, re-profile the same interaction, and confirm the render
count/duration actually improved before moving to the next.

**Follow-up question:**
What's the difference between a re-render and an actual DOM update, and why does that distinction matter when you're optimizing?

**Follow-up good answer:**
A "re-render" is React calling your component function again to compute a new React-element tree; a "DOM update" (commit) only happens for the specific DOM nodes where React's diff between the new and previous element tree finds an actual difference. React re-rendering a component doesn't necessarily touch the real DOM at all if the resulting output is identical. This matters because the Profiler's render count/duration measures the (often cheap) JS work of re-running component functions and reconciling the virtual tree — it is not directly the cost of expensive DOM mutations or layout thrashing. A component can re-render frequently with negligible cost (cheap function, tiny output, no DOM change) while a component that renders rarely but produces large DOM/layout changes can be the real bottleneck — so profiling should look at both the React Profiler (render frequency/reasons) and the browser Performance tab (actual paint/layout cost) rather than treating "re-render count" alone as the metric to minimize.

**Glossary:**
- **Reconciliation** — React's process of diffing the new element tree against the previous one to compute the minimal set of real DOM changes.
- **Commit** — the phase where React actually applies the computed DOM changes.

**Mental model:**
This is the headline "how do you find and fix a performance problem"
question interviewers are increasingly asking for React specifically —
looking for a concrete tool + a structured measure→hypothesize→fix→re-measure
loop, not "I just add `useMemo` everywhere."

**TL;DR:**
Diagnose excess re-renders with the React DevTools Profiler's "why did this render" data, fix the actual cause (unstable references, broad Context), then re-profile to confirm.

**References:**
- [React Developer Tools Profiler – react.dev](https://legacy.reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html)
- [`memo` – react.dev](https://react.dev/reference/react/memo)

---

### Q7. What's wrong with initializing state from a prop and then never updating it — e.g. `const [items, setItems] = useState(props.items)`?

**Question:**
What's wrong with initializing state from a prop and then never updating it — e.g. `const [items, setItems] = useState(props.items)`?

**Good answer:**
The argument passed to `useState` is only used to compute the state's
*initial* value, on the component's first render — React ignores it on
every subsequent render. If `props.items` changes later (the parent
re-renders with a new array), the component's local `items` state does not
follow it; it silently keeps the value from mount, creating two disconnected
sources of truth for the same conceptual data. This is a specific case of a
broader anti-pattern: state that duplicates something derivable from props
should usually not be state at all — either use the prop directly during
render, or if you need an editable local copy that's allowed to diverge
from the prop (e.g. a draft the user can revert), that divergence should be
an explicit, intentional design, not an accident of `useState`'s
mount-only-initializer semantics.

**Follow-up question:**
If you do intentionally want local state seeded from a prop that resets whenever a specific identity changes (e.g. switching between editing different records), what's the React-idiomatic way to do that without a `useEffect`?

**Follow-up good answer:**
Pass a `key` prop derived from that identity (e.g. `record.id`) to the component. React treats a changed `key` as "this is conceptually a different component instance," so it unmounts the old instance (discarding all its state, including hooks) and mounts a fresh one, which re-runs `useState`'s initializer against the new prop value. This avoids a `useEffect` that manually re-syncs state (which runs an extra render and is easy to get wrong with dependency arrays) and matches React's own recommended pattern for "reset state when some prop changes" from the *You Might Not Need an Effect* guide.

**Glossary:**
- **Initializer argument** — the value (or lazy-initializer function) passed to `useState`, used only on the component's first render.
- **`key` prop** — a special prop React uses to determine component identity across renders; changing it forces remount.

**Mental model:**
Probes whether the candidate understands `useState`'s exact semantics (not
just "it holds state") and knows the idiomatic React alternative to reaching
for `useEffect` as a state-syncing hammer.

**TL;DR:**
`useState`'s initializer only runs on mount, so state seeded from a prop silently stops tracking that prop once it changes later.

**References:**
- [`useState` – react.dev](https://react.dev/reference/react/useState)
- [Resetting state with a key – react.dev](https://react.dev/learn/you-might-not-need-an-effect#adjusting-some-state-when-a-prop-changes)

---

### Q8. What real problem does "lifting state up" solve, and what pain does Context solve that lifting state up alone doesn't?

**Question:**
What real problem does "lifting state up" solve, and what pain does Context solve that lifting state up alone doesn't?

**Good answer:**
When two sibling components need to share or stay in sync with the same
piece of state, the fix is to move ("lift") that state to their closest
common ancestor and pass it down as props, plus a callback to update it —
this keeps a single source of truth instead of each component holding its
own disconnected copy. The pain Context solves is what happens once that
common ancestor is far above the components that need the value: passing it
down as props through every intermediate component that doesn't itself care
about the value ("prop drilling") clutters every layer in between with
props it only forwards, and makes refactoring the tree shape painful since
every intermediate component's signature has to change. Context lets the
ancestor provide a value once, and any descendant, at any depth, read it
directly via `useContext` without every layer in between needing to know
about it.

**Follow-up question:**
Given that Context solves prop drilling, why isn't "wrap everything in Context" the default recommendation for all shared state?

**Follow-up good answer:**
Two reasons: (1) performance — every component that calls `useContext` on a given context re-renders whenever that context's value changes, with no built-in way to subscribe to just part of it, so a single Context holding a large object can cause broad re-renders for changes the consumer doesn't even care about; (2) it doesn't help with fixing *unrelated* components' need to invoke updates outside of a rendering context, and it can obscure data flow — since any descendant can silently pull from a context, it's harder to trace where a value comes from compared to explicit props. React's own guidance is to lift state only as high as necessary, keep Context values narrow/split by concern (so unrelated updates don't force unrelated re-renders), and reach for it specifically for cross-cutting concerns (theme, auth, locale) rather than as a default replacement for prop passing.

**Glossary:**
- **Prop drilling** — passing a prop through several layers of components that don't use it themselves, just to reach a deeply nested consumer.
- **Single source of truth** — keeping one authoritative copy of a piece of state rather than multiple, independently-updated copies.

**Mental model:**
Tests whether the candidate can articulate the actual problem each pattern
solves (not just "how to use `useContext`"), which is what lets them choose
the right tool instead of reaching for Context by default.

**TL;DR:**
Lifting state up gives sibling components one shared source of truth; Context avoids threading that value through every intermediate layer as props ("prop drilling").

**References:**
- [Sharing State Between Components – react.dev](https://react.dev/learn/sharing-state-between-components)
- [Passing Data Deeply with Context – react.dev](https://react.dev/learn/passing-data-deeply-with-context)

---

### Q9. When would you reach for Context vs. an external state library (Redux, Zustand, Jotai) for global-ish state?

**Question:**
When would you reach for Context vs. an external state library (Redux, Zustand, Jotai) for global-ish state?

**Good answer:**
Context is built into React and is well suited to values that change
infrequently and are read broadly — theme, authenticated user, locale,
feature flags — where the "every consumer re-renders on any change" cost is
acceptable because changes are rare. It's not itself a state-management
library — it has no built-in way to update state from outside React, no
selectors/derived state, no devtools, and (without extra work) no partial
subscriptions, so as an app's shared state grows large and changes
frequently (e.g. a shopping cart, real-time collaborative data, complex
form state shared across a wizard), those gaps start to hurt. Libraries
like Redux, Zustand, or Jotai exist specifically to fill them: they provide
fine-grained subscriptions (a component only re-renders when the specific
slice it reads changes, not on every store update), time-travel debugging,
middleware for async logic, and often better performance characteristics at
scale by design. The trade-off is added dependency weight, a learning
curve, and (for Redux specifically) more boilerplate — Zustand/Jotai were
built partly in reaction to that.

**Follow-up question:**
Concretely, how does a library like Zustand avoid the "every consumer re-renders on any state change" problem that plain Context has?

**Follow-up good answer:**
Rather than storing the whole state as a single Context value that all `useContext` consumers subscribe to wholesale, these libraries implement their own subscription mechanism outside of React's Context (typically using `useSyncExternalStore` under the hood since React 18): each component calls a hook with a *selector* function (e.g. `useStore(state => state.cartTotal)`), and the library only re-renders that component when the specific selected slice actually changes (compared by reference or a custom equality function) — not on every update to the overall store. This gives fine-grained, per-field subscriptions that plain Context's all-or-nothing broadcast can't provide without manually splitting state into many small Context providers.

**Glossary:**
- **Selector** — a function that picks out a specific slice of a larger state object, used to scope a subscription to just that slice.
- **Fine-grained subscription** — re-rendering only the components that depend on the exact piece of state that changed, not all consumers of the store.

**Mental model:**
A classic trade-offs/comparison question — tests whether the candidate
reaches for tools based on the actual problem shape (change frequency,
subscription granularity needs) rather than habit or hype.

**TL;DR:**
Context suits infrequently-changing, broadly-read values; external stores (Redux/Zustand/Jotai) suit large, frequently-changing state that needs fine-grained subscriptions.

**References:**
- [Passing Data Deeply with Context – react.dev](https://react.dev/learn/passing-data-deeply-with-context)
- [`useSyncExternalStore` – react.dev](https://react.dev/reference/react/useSyncExternalStore)

---

### Q10. If a Context's value is an object like `{ user, theme, setTheme }`, and a component only reads `theme`, does it still re-render when `user` changes? Why?

**Question:**
If a Context's value is an object like `{ user, theme, setTheme }`, and a component only reads `theme` (destructured from the result of `useContext`), does it still re-render when `user` changes? Why?

**Good answer:**
Yes. `useContext` subscribes the whole component to the *entire* context
value, not to individual fields of it. When the Provider re-renders with a
new value object (even if only `user` actually changed and `theme` is the
same conceptual value), React sees a new reference for the whole value and
re-renders every component that calls `useContext` on that context —
regardless of which field they destructure out of it afterward. The
destructuring happens in your component's own code, after the re-render has
already been triggered; React's Context implementation has no visibility
into which fields you actually use.

**Code example:**
```jsx
// Every consumer of ThemeContext re-renders whenever ANY field changes,
// even a component that only ever reads `theme`.
const ThemeContext = createContext();

function Provider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState("light");
  const value = { user, theme, setTheme }; // new object every render
  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

**Follow-up question:**
What are two ways to avoid that unnecessary re-render for a component that only cares about `theme`?

**Follow-up good answer:**
1. **Split the Context** — use separate `UserContext` and `ThemeContext` (or `ThemeContext` + `ThemeSetterContext`) instead of one combined value, so a component only subscribes to (and re-renders for) the specific context it actually reads.
2. **Move to a selector-based store** — use `useSyncExternalStore`-backed state (either hand-rolled or via a library like Zustand) that supports subscribing to a derived slice with a custom equality check, so the component only re-renders when the *selected* value actually changes, not whenever the source object's reference changes. Splitting Context is usually the simpler, more idiomatically-React fix when the values genuinely change independently; a selector store is worth it once you have many independently-changing fields and splitting into that many separate Contexts becomes unwieldy.

**Glossary:**
- **Context Provider** — the component that supplies a value to `useContext` calls in its subtree.
- **Whole-value subscription** — Context's default behavior of re-rendering a consumer whenever the value reference changes, with no built-in per-field granularity.

**Mental model:**
Tests whether the candidate actually understands Context's re-render
semantics at the mechanism level rather than treating it as "just global
state that works like magic" — a very common source of real performance
bugs in production apps.

**TL;DR:**
`useContext` subscribes to the whole context value, so any field changing re-renders every consumer regardless of which field it actually destructures.

**References:**
- [Passing Data Deeply with Context – react.dev](https://react.dev/learn/passing-data-deeply-with-context)
- [`useContext` – react.dev](https://react.dev/reference/react/useContext)

---

### Q11. When would you reach for `useReducer` instead of several `useState` calls?

**Question:**
When would you reach for `useReducer` instead of several `useState` calls?

**Good answer:**
`useReducer` is a better fit once a component's state updates involve
several related pieces of state that change together in specific, well-
defined ways — e.g. a form with validation state, a multi-step wizard, or
any state machine with distinct named transitions. With several independent
`useState` calls, it's easy for related updates to fall out of sync (you
have to remember to call every relevant setter together, in the right
order, at every call site that triggers the transition), and the "what can
change this state and how" logic ends up scattered across every event
handler that calls a setter. `useReducer` centralizes that logic into one
pure `(state, action) => newState` function, so every possible transition
is defined in one place, is easy to test in isolation (it's a pure
function — no rendering needed), and event handlers just `dispatch` a
described action instead of directly poking at multiple state pieces. This
is directly the Flux/Redux "reducer" pattern from state-machine theory,
just scoped to a single component.

**Code example:**
```jsx
function formReducer(state, action) {
  switch (action.type) {
    case "field-changed":
      return { ...state, [action.field]: action.value, errors: {} };
    case "submit-failed":
      return { ...state, errors: action.errors };
    default:
      return state;
  }
}

const [state, dispatch] = useReducer(formReducer, { email: "", errors: {} });
dispatch({ type: "field-changed", field: "email", value: "a@b.com" });
```

**Follow-up question:**
`useReducer` and Redux both use the `(state, action) => newState` pattern. What's the actual difference in scope and capability between them?

**Follow-up good answer:**
`useReducer` manages state local to a single component tree (whatever subtree has access to the dispatch function, typically via props or Context) — it has no global store, no middleware system, and no built-in devtools or persistence. Redux is a standalone, app-wide store: a single global state tree, a middleware pipeline (for things like async action handling via `redux-thunk`/`redux-saga`, logging, or persistence), and mature devtools with time-travel debugging. In practice, `useReducer` is often used *inside* a component paired with Context to get a lightweight, component-scoped version of the Redux pattern, without pulling in a whole library — it's the same core mental model (pure reducer + dispatched actions) at a smaller, more local scope.

**Glossary:**
- **Reducer** — a pure function that takes the current state and an action, and returns the next state, with no side effects.
- **Action** — a plain object (conventionally with a `type` field) describing what happened, dispatched to trigger a state transition.

**Mental model:**
Tests whether the candidate understands `useReducer` as an application of
general state-machine/reducer theory (not a React-specific trick), and can
judge when centralizing transition logic is worth the extra structure over
several independent `useState` calls.

**TL;DR:**
`useReducer` centralizes related state transitions into one pure function, which scattered `useState` setters across handlers can't do as cleanly.

**References:**
- [`useReducer` – react.dev](https://react.dev/reference/react/useReducer)
- [Extracting State Logic into a Reducer – react.dev](https://react.dev/learn/extracting-state-logic-into-a-reducer)

---

### Q12. What happens if you pass an inline object or function as a `useEffect` dependency, e.g. `useEffect(() => {...}, [{ id }])` or `useEffect(() => {...}, [() => {}])`?

**Question:**
What happens if you pass an inline object or function as a `useEffect` dependency, e.g. `useEffect(() => {...}, [{ id }])`?

**Good answer:**
React compares each entry in the dependency array to the previous render's
entry using `Object.is` (reference equality), not deep/structural equality.
An object or array or function literal created inline in the render body is
a brand-new reference on every single render, even if its contents are
identical to last time — so React sees "the dependency changed" on every
render and re-runs the effect every time, defeating the entire purpose of
the dependency array (skip re-running when nothing meaningfully changed).
This is a very common source of "why is my effect running on every render"
bugs, and often cascades into infinite loops if the effect itself sets
state that causes a re-render.

**Code example:**
```jsx
// Bug: `filters` is a new object every render, so this effect
// (and any request it fires) re-runs on every single render.
function List({ status }) {
  const filters = { status }; // new reference every render
  useEffect(() => {
    fetchItems(filters);
  }, [filters]);
}

// Fix: depend on the primitive value(s) that actually matter
function List({ status }) {
  useEffect(() => {
    fetchItems({ status });
  }, [status]); // primitive comparison, stable unless status itself changes
}
```

**Follow-up question:**
If you genuinely need to pass a whole object as a dependency (not just a primitive derived from it), how do you avoid the identity-changes-every-render problem?

**Follow-up good answer:**
Memoize the object itself with `useMemo` so its reference only changes when its actual inputs change (`const filters = useMemo(() => ({ status }), [status])`), which then makes it safe to list as a dependency. Alternatively, and often better where possible, depend on the primitive fields you actually read inside the effect (as in the fix above) rather than the containing object at all — this sidesteps the identity problem entirely and also makes the effect's real dependencies more explicit/self-documenting, which is React's own general recommendation in its dependency-array guidance.

**Glossary:**
- **`Object.is`** — the equality algorithm React's dependency comparison uses; essentially reference equality for objects/functions, with special-cased handling for `NaN` and signed zero.
- **Dependency array** — the second argument to `useEffect`/`useMemo`/`useCallback` listing the values the effect/computation depends on.

**Mental model:**
Tests understanding of *how* the dependency comparison actually works
(reference equality), which is the root cause behind a large fraction of
real-world "my effect runs too often" bug reports.

**TL;DR:**
Inline object/array/function literals get a new reference every render, so as dependencies they make an effect re-run on every render regardless of content.

**References:**
- [`useEffect` – react.dev](https://react.dev/reference/react/useEffect)
- [Removing Effect Dependencies – react.dev](https://react.dev/learn/removing-effect-dependencies)

---

### Q13. What is "tearing" in the context of concurrent React, and how does `useSyncExternalStore` prevent it?

**Question:**
What is "tearing" in the context of concurrent React, and how does `useSyncExternalStore` prevent it?

**Good answer:**
Tearing is when different parts of the UI, within the same render, end up
showing inconsistent versions of the same underlying data — e.g. one
component renders with the old value of an external store and another
renders with the new value, because the store's data changed *while*
React was in the middle of rendering (something concurrent rendering
explicitly allows, since renders can be interrupted, restarted, or split
across time). Manually subscribing to an external, non-React data source
inside a `useEffect` + `useState` (the pre-18 pattern) doesn't protect
against this, because React has no visibility into that subscription and
can't guarantee every component finishes reading a fully consistent
snapshot. `useSyncExternalStore` is a hook designed specifically to let a
component safely read from an external store: React uses it to force a
synchronous consistency check right before committing a render, ensuring
every component subscribed to the store sees the same snapshot in that
commit, even under concurrent rendering, re-rendering synchronously to
correct any detected inconsistency rather than letting a torn UI be
painted.

**Code example:**
```jsx
function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener("online", callback);
      window.addEventListener("offline", callback);
      return () => {
        window.removeEventListener("online", callback);
        window.removeEventListener("offline", callback);
      };
    },
    () => navigator.onLine // getSnapshot
  );
}
```

**Follow-up question:**
Who is `useSyncExternalStore` actually meant to be used by directly — application developers, or library authors?

**Follow-up good answer:**
Primarily library authors building state-management or data-subscription libraries (Redux, Zustand, and similar all use it internally as of React 18) — React's own docs describe it as rarely needed in typical application code. Application developers subscribing to React's own state (`useState`/`useReducer`/Context) don't need it, since React already guarantees consistency for its own state under concurrent rendering; it's specifically for bridging *external*, non-React-owned mutable data sources (browser APIs, third-party stores, module-level mutable variables) into React's rendering guarantees. Reaching for it directly in application code is a signal the code is either building its own micro state-management layer or subscribing to something that should probably be React state instead.

**Glossary:**
- **Tearing** — rendering inconsistent snapshots of the same data within a single UI update, particularly a risk under concurrent/interruptible rendering.
- **`getSnapshot`** — the function passed to `useSyncExternalStore` that returns the store's current value; React calls it to detect changes.

**Mental model:**
Tests awareness of a subtle, genuinely new-in-concurrent-React problem class
and the API React added specifically for it — separates candidates with
surface-level hooks knowledge from those who understand what concurrent
rendering changes about correctness guarantees.

**TL;DR:**
Tearing is inconsistent data shown across a UI mid concurrent-render; `useSyncExternalStore` prevents it by forcing a synchronous consistency check before commit.

**References:**
- [`useSyncExternalStore` – react.dev](https://react.dev/reference/react/useSyncExternalStore)

---

### Q14. What's the difference between `useRef` and `useState`, and when would using `useRef` for something be a mistake?

**Question:**
What's the difference between `useRef` and `useState`, and when would using `useRef` for something be a mistake?

**Good answer:**
Both persist a value across renders, but `useState` updates trigger a
re-render (React schedules a new render because the component's *displayed*
output may need to change), while `useRef` returns a mutable object
(`{ current: value }`) that you can read and write directly without
triggering any re-render at all — React just doesn't watch it. `useRef` is
the right tool for values that need to persist between renders but never
directly drive what's rendered: a `setInterval` ID to clear later, a DOM
node reference, a "did this effect already run" flag, or a mutable
instance value that's read inside event handlers. Using `useRef` for
something that should actually cause the UI to update is a mistake — if you
mutate `ref.current` expecting the screen to reflect the new value, it
won't, because no render was scheduled; the component will only show the
new value the *next* time something else happens to trigger a render, which
produces confusing, hard-to-debug staleness in the UI.

**Code example:**
```jsx
function Timer() {
  const intervalRef = useRef(null); // fine: doesn't need to drive UI
  const [seconds, setSeconds] = useState(0); // must be state: drives UI

  useEffect(() => {
    intervalRef.current = setInterval(() => setSeconds(s => s + 1), 1000);
    return () => clearInterval(intervalRef.current);
  }, []);

  return <p>{seconds}s</p>;
}
```

**Follow-up question:**
Does updating `ref.current` during render (not inside an effect or event handler) cause any problems?

**Follow-up good answer:**
Yes — mutating a ref during the render phase itself is discouraged and can be unsafe, because render functions are supposed to be pure (same inputs → same output, no side effects), and React may call a component's render function more than once for the same commit (e.g. under `StrictMode`'s double-invoking in development, or when a render is thrown away and retried under concurrent features). Mutating `ref.current` in that path makes the render impure and can leave the ref holding a value from a render that was discarded rather than the one that actually committed. Refs should be read/written from event handlers or Effects (which run after commit, exactly once per actual commit), not from the render body — the one narrow exception React documents is lazily initializing a ref on first render (`if (ref.current === null) ref.current = new Thing()`), which is safe specifically because it's idempotent.

**Glossary:**
- **Mutable ref object** — the `{ current }` object returned by `useRef`, which can be mutated directly without triggering a re-render.
- **Render phase** — the (must-be-pure) phase where React calls component functions to compute the next output, as opposed to the commit phase where DOM changes are applied.

**Mental model:**
Tests whether the candidate understands the state-vs-ref distinction at the
"does it schedule a render" mechanism level, and knows the purity
constraints on where refs can safely be mutated.

**TL;DR:**
`useRef` persists a value across renders without triggering a re-render — using it for anything that should visibly update the UI is a bug.

**References:**
- [`useRef` – react.dev](https://react.dev/reference/react/useRef)
- [Referencing Values with Refs – react.dev](https://react.dev/learn/referencing-values-with-refs)

---

### Q15. A parent passes an `onClick` handler to a memoized child (`React.memo`). Even though the handler is defined as an arrow function inline in the parent, the child re-renders every time the parent does. Why, and how do you fix it?

**Question:**
A parent passes an `onClick` handler to a memoized child (`React.memo`). Even though the handler is defined as an arrow function inline in the parent, the child re-renders every time the parent does. Why, and how do you fix it?

**Good answer:**
`React.memo` skips re-rendering a component only if all of its props are
shallowly equal (`Object.is` per prop) to the previous render's props. An
inline arrow function (`onClick={() => doThing(id)}`) is a brand-new
function object created on every render of the parent — even though its
*behavior* is identical, its *reference* is different every time, so the
shallow-equality check on that prop fails and `React.memo` correctly
concludes "a prop changed" and re-renders the child. The fix is to make the
handler's reference stable across renders when its logic doesn't actually
need to change: wrap it in `useCallback` with the right dependency array in
the parent, so the same function reference is reused across renders unless
its dependencies change, which then lets `React.memo`'s shallow comparison
actually pass and skip the child's re-render.

**Code example:**
```jsx
// Child re-renders every time Parent does — new function reference each render
function Parent({ id }) {
  return <MemoChild onClick={() => doThing(id)} />;
}

// Stable reference — MemoChild only re-renders if `id` (or its own props) changes
function Parent({ id }) {
  const handleClick = useCallback(() => doThing(id), [id]);
  return <MemoChild onClick={handleClick} />;
}
```

**Follow-up question:**
If `MemoChild` also receives an object or array prop (not just the function), does `useCallback` on the handler alone fully solve the re-render problem?

**Follow-up good answer:**
No — `React.memo`'s shallow comparison checks *every* prop, so any other prop that's a new object/array/function reference each render (e.g. an inline `style={{ color: "red" }}` or `options={[1, 2, 3]}`) will independently fail the comparison and force a re-render regardless of the handler being stabilized. Each such prop needs its own stabilization — `useMemo` for objects/arrays, `useCallback` for functions — or those values need to be moved to module scope (if they're truly static and don't depend on props/state) so they're not recreated at all. This is why "add `useCallback`/`useMemo` everywhere a memoized child is involved" can spiral: `React.memo` only helps if *all* the props it receives are reference-stable, not just some of them.

**Glossary:**
- **Shallow comparison** — comparing each top-level prop by reference (`Object.is`), not recursively comparing nested contents.
- **`React.memo`** — a higher-order component that skips re-rendering its wrapped component if its props are shallowly equal to the previous render's.

**Mental model:**
Tests whether the candidate can reason precisely about *why* a specific
memoization didn't work, rather than just knowing the APIs exist — a very
common real interview scenario since this exact "I added memo but it's
still re-rendering" confusion is extremely common in practice.

**TL;DR:**
`React.memo`'s shallow-equality check fails on a fresh inline function reference each render; `useCallback` stabilizes that reference so the check can pass.

**References:**
- [`memo` – react.dev](https://react.dev/reference/react/memo)
- [`useCallback` – react.dev](https://react.dev/reference/react/useCallback)

---

### Q16. What does it mean that a custom hook doesn't share state between the components that use it — and why is that the case, given it's "just a function"?

**Question:**
What does it mean that a custom hook doesn't share state between the components that use it — and why is that the case, given it's "just a function"?

**Good answer:**
A custom hook (any function whose name starts with `use` and that itself
calls other hooks) shares *logic*, not *state*. Every component that calls
a custom hook gets its own, completely independent copy of whatever
`useState`/`useReducer`/`useRef` state the hook sets up internally — the
hook function's body runs fresh, tied to the calling component's own fiber
and its own slot in that fiber's hook linked list, exactly as if you'd
written that `useState` call directly inside each component. So if two
components both call `const [value, setValue] = useCounter()`, calling
`setValue` in one component's instance has zero effect on the other's —
they just happen to run identical logic, not share a value. This
sometimes surprises developers coming from a mental model of "hooks are
like singletons/services" — a custom hook is closer to a reusable
"template" for stateful logic than a shared store.

**Code example:**
```jsx
function useCounter() {
  const [count, setCount] = useState(0);
  return { count, increment: () => setCount(c => c + 1) };
}

function App() {
  const a = useCounter(); // independent state
  const b = useCounter(); // independent state, NOT linked to `a`
  return <>{a.count} {b.count}</>; // clicking a's increment never changes b
}
```

**Follow-up question:**
If you actually do want two components to share the same live state, what would you reach for instead of (or in addition to) a custom hook?

**Follow-up good answer:**
The state itself needs to live in one place both components can read from — typically that means lifting the state up to a shared ancestor and passing it down (directly or via Context), or backing it with an external store (Context + `useReducer`, or a library like Zustand/Jotai/Redux) that both components subscribe to. A custom hook can still be a useful wrapper *around* that shared store — e.g. `useCartStore()` that internally reads from a Zustand store — the key distinction is that the shared state has to live in something external to the individual component instances (a store, or a common ancestor's state), not inside the custom hook's own `useState` calls, which are always per-caller.

**Glossary:**
- **Custom hook** — a function starting with `use` that calls other hooks to package up reusable stateful logic.
- **Per-instance state** — state that belongs to one specific component instance (fiber) and isn't automatically shared with other instances, even of the same hook or component.

**Mental model:**
Tests whether the candidate has an accurate mental model of what a hook
actually is (a function tied to a specific fiber's hook list on each call
site) versus a common but wrong assumption that hooks behave like shared
singleton services.

**TL;DR:**
A custom hook shares reusable logic, not state — each caller gets its own independent `useState`/`useRef` instance tied to its own fiber.

**References:**
- [Reusing Logic with Custom Hooks – react.dev](https://react.dev/learn/reusing-logic-with-custom-hooks)

---

### Q17. React's docs describe function components (and reducers) as needing to be "pure." What does that mean concretely, and what's an example of an impure component that still often "seems to work"?

**Question:**
React's docs describe function components (and reducers) as needing to be "pure." What does that mean concretely, and what's an example of an impure component that still often "seems to work"?

**Good answer:**
A pure function's output depends only on its inputs, and it produces no
observable side effects outside itself — call it twice with the same
inputs and you get the same result, with nothing external changed in
between. React relies on this for function components: it needs to be free
to call a component's function zero, one, or multiple times for a given
render (it does this deliberately — under `StrictMode` in development to
surface bugs, and potentially internally for concurrent features), so if a
component isn't pure, calling it extra times produces observably different
behavior instead of just wasted work. A classic impure example is mutating
a variable declared *outside* the component (e.g. a module-level counter or
an array from props) directly during render — it often "seems to work" in
simple cases because the component happens to only render once in practice,
but it silently breaks under `StrictMode`'s double-invocation, or produces
subtly wrong results once the app grows enough that re-renders (or
concurrent rendering) start actually exercising the double-call path.

**Code example:**
```jsx
// Impure: mutates state outside the render, and reads a module-level
// variable whose value depends on how many times this ran before.
let renderCount = 0;
function Impure() {
  renderCount++; // side effect during render
  return <p>Rendered {renderCount} times</p>; // output not purely a function of props
}
```

**Follow-up question:**
Does this purity requirement also apply to reducer functions passed to `useReducer`, and why would that matter?

**Follow-up good answer:**
Yes — reducer functions must also be pure: given the same `(state, action)` pair, a reducer must always return the same result, with no side effects (no API calls, no mutating the existing state object, no `Math.random()`/`Date.now()` inside it). React (and `StrictMode` specifically) may call a reducer twice in development for the same dispatched action to help surface impurity bugs, and tools like Redux DevTools' time-travel debugging rely on being able to replay a sequence of actions through the reducer and get deterministic, reproducible results — an impure reducer breaks that replayability, since re-running it against the same history wouldn't reliably reconstruct the same state.

**Glossary:**
- **Pure function** — a function whose output depends only on its inputs and that causes no observable side effects.
- **`StrictMode` double-invocation** — React deliberately calling certain functions (component bodies, reducers, state initializers) twice in development to help surface impurity bugs.

**Mental model:**
Connects a React-specific rule to the general software-engineering theory of
purity/referential transparency, and tests whether the candidate can
recognize impurity that "accidentally works" rather than only reciting the
rule.

**TL;DR:**
A pure component's output depends only on its inputs with no side effects, since React may call it more than once per render (e.g. `StrictMode`); mutating outside state during render breaks that.

**References:**
- [Keeping Components Pure – react.dev](https://react.dev/learn/keeping-components-pure)
- [`StrictMode` – react.dev](https://react.dev/reference/react/StrictMode)

---

### Q18. Why does React's `StrictMode` intentionally call your component function, state initializers, and reducers twice in development?

**Question:**
Why does React's `StrictMode` intentionally call your component function, state initializers, and reducers twice in development?

**Good answer:**
`StrictMode` double-invokes certain functions specifically because React
assumes they *should* be pure, and running a pure function twice with the
same inputs is, by definition, harmless — it produces the same result both
times with no observable difference. So double-invoking is a deliberate
diagnostic: if double-calling a function produces a visible difference
(duplicated side effects, a counter that's off by double, an API call
firing twice), that's proof the function wasn't actually pure, surfaced
immediately in development rather than manifesting later as a subtle,
hard-to-reproduce production bug under concurrent rendering or `StrictMode`
being toggled on for a subtree. It only runs in development — production
builds call these functions once — so it costs nothing at runtime; it's
purely a development-time safety net.

**Follow-up question:**
`useEffect` setup/cleanup functions are also double-invoked (mount → cleanup → mount again) under `StrictMode` in React 18+. What real bug class is that specifically designed to catch?

**Follow-up good answer:**
It's designed to catch Effects that don't properly clean up after themselves — specifically, Effects whose cleanup function is missing or incomplete, such that mounting, unmounting, and remounting a component leaves behind duplicated subscriptions, event listeners, timers, or connections. This scenario (mount → unmount → remount) is exactly what happens in real apps under things like React's `<Suspense>`-driven remounting, fast-refresh during development, or navigating away and back in an SPA — if an Effect's cleanup doesn't fully undo what its setup did, those duplicated side effects accumulate silently. By deliberately forcing that exact sequence in development for every component with an Effect, `StrictMode` catches missing/incomplete cleanup logic immediately instead of it surfacing as a hard-to-diagnose resource leak or duplicated-behavior bug much later.

**Glossary:**
- **Double-invocation** — `StrictMode`'s development-only behavior of calling certain functions twice to detect impurity or missing cleanup.
- **Effect cleanup function** — the function optionally returned from a `useEffect` callback, run before the next Effect execution and on unmount, meant to undo the setup's side effects.

**Mental model:**
Tests whether the candidate understands `StrictMode` as a designed
diagnostic tool tied directly to React's purity/cleanup contracts, rather
than dismissing the "double render/effect" behavior as a quirky annoyance
to work around.

**TL;DR:**
`StrictMode` double-invokes functions that are supposed to be pure so any observable difference between the two calls immediately exposes an impurity bug.

**References:**
- [`StrictMode` – react.dev](https://react.dev/reference/react/StrictMode)
- [You Might Not Need an Effect – react.dev](https://react.dev/learn/you-might-not-need-an-effect)

---

### Q19. React 19.2 introduced `useEffectEvent`. What problem does it solve that plain `useEffect` + dependency arrays couldn't solve cleanly?

**Question:**
React 19.2 introduced `useEffectEvent`. What problem does it solve that plain `useEffect` + dependency arrays couldn't solve cleanly?

**Good answer:**
Some logic inside an Effect is genuinely "reactive" (the Effect should
re-run when it changes — e.g. reconnecting a chat room when `roomId`
changes) while other logic referenced in the same Effect is not meant to be
reactive at all (e.g. reading the latest `theme` to log an analytics event
on connect, where you always want the *current* theme but don't want a
theme change to tear down and reconnect the chat room). Before
`useEffectEvent`, there was no clean way to express "read the latest value
of this, but don't treat it as a reason to re-run the Effect" — you either
included it in the dependency array (causing unwanted re-runs) or omitted it
(causing a stale closure, since the omitted value would be frozen at
whatever it was when the Effect last ran). `useEffectEvent` lets you wrap
that non-reactive logic in an "Effect Event": a function that always sees
the latest props/state when called, without needing to be listed as an
Effect dependency and without causing the Effect to re-run when those
values change.

**Code example:**
```jsx
function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    showNotification("Connected!", theme); // always reads latest theme
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on("connected", () => onConnected());
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // only roomId is a reactive dependency; theme isn't needed here
}
```

**Follow-up question:**
Before `useEffectEvent` existed, what was the common workaround for this exact problem, and what was wrong with it?

**Follow-up good answer:**
The common workaround was to stash the "always latest" value in a `ref`, updated via its own separate Effect on every render (`useEffect(() => { themeRef.current = theme; })`), and then read `themeRef.current` inside the main Effect instead of `theme` directly — avoiding the need to list `theme` as a dependency since refs aren't reactive. It worked, but it was verbose, easy to get subtly wrong (forgetting the ref-sync effect, or reading the ref during render instead of inside an event/Effect), and didn't read as intentional — it looked like a workaround rather than an idiom, so it was hard to recognize at a glance in review. `useEffectEvent` formalizes that exact pattern as a first-class API with the correct semantics enforced by React itself.

**Glossary:**
- **Effect Event** — a function created by `useEffectEvent` that always reads the latest props/state when called, but is never itself a reactive dependency.
- **Reactive dependency** — a value that, when changed, should cause an Effect to re-run; the values legitimately listed in an Effect's dependency array.

**Mental model:**
Tests whether the candidate follows React's evolving APIs and understands
the underlying reactive-vs-non-reactive distinction well enough to explain
*why* a new API was needed, not just that it exists.

**TL;DR:**
`useEffectEvent` lets an effect read the latest props/state for non-reactive logic without listing them as dependencies or triggering unwanted re-runs.

**References:**
- [`useEffectEvent` – react.dev](https://react.dev/reference/react/useEffectEvent)
- [Separating Events from Effects – react.dev](https://react.dev/learn/separating-events-from-effects)

---

### Q20. You're handed a component where a `useEffect` fires an API call, and the team suspects it's causing duplicate requests in production (not just the known `StrictMode` dev double-invoke). What's your process for confirming and fixing this?

**Question:**
You're handed a component where a `useEffect` fires an API call, and the team suspects it's causing duplicate requests in production (not just the known `StrictMode` dev double-invoke). What's your process for confirming and fixing this?

**Good answer:**
First, reproduce and measure in a production-like build (`StrictMode`'s
double-invoke is dev-only and shouldn't be the cause in production) — check
the Network tab for duplicate requests and correlate their timing with
component mount/re-render/unmount events, e.g. via React DevTools'
"highlight updates" or by temporarily logging inside the Effect. Common
real causes: (1) the Effect's dependency array includes a value whose
reference changes every render (an inline object/array/function), so the
Effect re-runs — and re-fires the request — on every render, not just when
the "real" dependency changes; (2) the component mounts, unmounts, and
remounts rapidly (e.g. due to a `key` change, route transition, or a
parent conditionally rendering it) without the Effect's cleanup function
properly aborting the in-flight request from the previous mount; (3) two
sibling instances of the same component are each independently firing the
same request without any shared caching/deduplication layer. The fix
follows from which cause it is: stabilize the dependency (primitive value or
`useMemo`), add an `AbortController` in the cleanup function to cancel
stale in-flight requests, or introduce request deduplication/caching (a
library like React Query/SWR, or a shared cache) if multiple independent
components legitimately need the same data.

**Follow-up question:**
Why does adding an `AbortController` cleanup specifically matter here, beyond just "cancelling a request that's no longer needed"?

**Follow-up good answer:**
Beyond saving the wasted network/server work, it prevents a real correctness bug: without cancellation, if a component fetches, then unmounts (or its dependency changes and the Effect re-runs, firing a new request) before the first request resolves, the first request's `.then` callback can still fire later and call `setState` on a component that's already unmounted, or overwrite newer state with a stale response that happened to resolve out of order (a "race condition" where an older request finishes after a newer one). `AbortController` cancels the stale request in the cleanup function, and checking `signal.aborted` (or letting the fetch's own abort rejection short-circuit the `.then`) ensures a stale response is discarded instead of applied — the Effect cleanup pattern exists precisely to make this kind of "cancel what's no longer relevant" logic straightforward.

**Glossary:**
- **Race condition** — a bug where the outcome depends on the relative timing of two async operations, here an earlier request resolving after a later one.
- **`AbortController`** — a Web API for cancelling in-flight `fetch` requests (or other abortable async work) via an associated `AbortSignal`.

**Mental model:**
The other core "performance/correctness diagnosis" question for React —
tests a structured measure → isolate cause → targeted fix process for a
very common real production bug class (duplicate/stale network requests
from Effects), not just abstract hooks trivia.

**TL;DR:**
Duplicate production requests from an effect usually trace to an unstable dependency, an unmounted-but-uncancelled fetch, or duplicate sibling fetches — diagnose via the Network tab + Profiler, fix with stable deps and an `AbortController` cleanup.

**References:**
- [`useEffect` – react.dev](https://react.dev/reference/react/useEffect)
- [`AbortController` – MDN](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=react&tags=hooks-and-state-management&autostart=1" | relative_url }})
