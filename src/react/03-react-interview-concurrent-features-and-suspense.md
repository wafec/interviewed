---
layout: default
title: "React Interview — Concurrent Features & Suspense"
---

# React Interview — Concurrent Features & Suspense

This set focuses on React 18+'s concurrent rendering model: what "concurrent" actually means under the hood, how `<Suspense>`, `startTransition`, and `useDeferredValue` work together, streaming SSR with selective hydration, React Server Components, and the pitfalls/performance-diagnosis skills that separate someone who has used these APIs from someone who understands why they exist.

### Q1. What is "concurrent rendering" in React 18, and what problem does it solve compared to the legacy synchronous renderer?

**Question:**
What is "concurrent rendering" in React 18, and what problem does it solve compared to the legacy synchronous renderer?

**Good answer:**
Before React 18, once React started rendering an update, it walked the whole component tree synchronously and couldn't stop until it was done — a large update (e.g. re-rendering a big list) would block the main thread and make the app feel unresponsive: typing lagged, animations stuttered. Concurrent rendering is a behind-the-scenes mechanism that lets React prepare multiple versions of the UI at the same time and makes rendering **interruptible**: React can start rendering an update, pause partway through to handle something more urgent (like a keystroke), and either resume or throw away the in-progress work. It's not a "mode" you turn on globally — concurrency is opt-in per update, only enabled for updates triggered by new APIs like `startTransition`, `useDeferredValue`, and Suspense-integrated data fetching. Without touching those APIs, an app upgraded to React 18 renders exactly as it did in React 17.

**Code example:**
```jsx
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('about');

  function selectTab(nextTab) {
    // Marks this update as low-priority/interruptible —
    // React can still respond to other input while it renders.
    startTransition(() => setTab(nextTab));
  }
  // ...
}
```

**Follow-up question:**
If concurrency is opt-in, what happens to an existing React 17 app after upgrading to React 18 with no code changes?

**Follow-up good answer:**
It renders synchronously, the same as before — React 18 doesn't change component behavior by default. The React team specifically designed the upgrade this way so teams could adopt React 18 with minimal or no code changes, then gradually opt individual updates into concurrent behavior using `startTransition`, `useTransition`, or `useDeferredValue` as needed, rather than facing a big-bang migration.

**Glossary:**
- **Concurrent rendering** — React's ability to work on a render without committing it, pausing/resuming/discarding it as priorities change.
- **Interruptible rendering** — rendering that can be paused mid-way to let a higher-priority update run first.

**Mental model:**
Tests whether the candidate understands that "concurrent" describes *how* React schedules work, not a new rendering output — and that it's additive/opt-in, which is a deliberate migration-safety design decision, not an accident.

**TL;DR:**
Concurrent rendering makes React's render work interruptible/pausable so urgent updates (like typing) aren't blocked, but it's opt-in per update via APIs like `startTransition`, not a global behavior change.

**References:**
- [React v18.0 blog post](https://react.dev/blog/2022/03/29/react-v18)
- [The Plan for React 18](https://react.dev/blog/2021/06/08/the-plan-for-react-18)

---

### Q2. What is a "Fiber" in React's internals, and how does it enable interruptible rendering?

**Question:**
What is a "Fiber" in React's internals, and how does it enable interruptible rendering?

**Good answer:**
A Fiber is React's internal representation of a unit of work — roughly one per component instance/element — that replaced the old stack-based reconciler. Instead of recursively walking the tree with the native JS call stack (which can't be paused once started), React builds a linked-list-like tree of Fiber nodes that it can traverse iteratively: process one fiber, decide whether to continue, yield back to the browser if there's higher-priority work or the frame budget is up, and resume later exactly where it left off. Each Fiber carries the information needed to resume: its type, pending props/state, and pointers to its parent/child/sibling fibers. This "virtual stack frame" design is what makes React's rendering pausable, resumable, and abortable — properties a plain recursive call stack doesn't have.

**Follow-up question:**
Why can't React "truly preempt" a running JavaScript function the way an OS preempts a thread?

**Follow-up good answer:**
JavaScript in the browser's main thread is single-threaded and run-to-completion — once a synchronous function starts executing, nothing (not even React) can interrupt it mid-statement the way an OS scheduler can suspend a thread at an arbitrary instruction. React's "interruption" is therefore cooperative, not preemptive: React only gets a chance to check "should I yield?" at the boundaries between processing individual fibers (units of work), not in the middle of one. That's why the fiber tree is broken into small, individually-resumable units — it's how React creates yield points inside what is otherwise non-preemptible JS execution, using the scheduler to voluntarily hand control back to the browser between units of work.

**Glossary:**
- **Unit of work** — the smallest chunk of rendering work React can pause after, roughly one fiber.
- **Reconciler** — the part of React that decides what changed and builds the fiber tree (`react-reconciler` package).

**Mental model:**
Probes whether the candidate has looked past the public hooks API into how React achieves "pausable rendering" in a single-threaded, run-to-completion language — a common follow-up trap for people who only know the marketing description of concurrent React.

**TL;DR:**
A Fiber is a resumable unit-of-work node replacing the old recursive call stack, letting React pause, resume, or abort rendering between units instead of being locked into one uninterruptible pass.

**References:**
- [facebook/react — react-reconciler source (ReactFiberWorkLoop.js and related)](https://github.com/facebook/react/tree/main/packages/react-reconciler/src)
- [React Fiber Architecture (community explainer, reviewed by React team members)](https://github.com/acdlite/react-fiber-architecture)

---

### Q3. What are "lanes" in React's scheduler, and how do they relate to update priority?

**Question:**
What are "lanes" in React's scheduler, and how do they relate to update priority?

**Good answer:**
Lanes are React's internal mechanism (implemented as a bitmask, in `packages/react-reconciler/src/ReactFiberLane.js`) for tracking and prioritizing pending updates. Each update gets assigned to one or more lanes based on how it was triggered — e.g. a direct user input like typing gets a high-priority "SyncLane", while an update wrapped in `startTransition` gets a low-priority "TransitionLane". Because it's a bitmask, React can efficiently represent and combine many simultaneous pending updates of different priorities on a fiber, and the scheduler can quickly determine which lanes have the most urgent pending work and process those first, batching or interrupting lower-priority lanes as needed. This is the actual mechanism behind the public "urgent vs. transition" distinction exposed by `useTransition`.

**Follow-up question:**
Why does React need a bitmask instead of a simple priority number for this?

**Follow-up good answer:**
A single priority number can only represent "what is the priority of this one update." A bitmask lets React represent *multiple, independently-tracked* pending updates on the same fiber simultaneously (e.g. a fiber might have both a pending sync update and a pending transition update at once), combine sets of lanes with cheap bitwise operations (union, intersection, "does this fiber have any lane in this set pending"), and efficiently batch updates that share a lane without having to compare many discrete priority values. It's a performance/data-structure choice suited to a scheduler that has to make these checks on every fiber, every render pass.

**Glossary:**
- **Lane** — a bit (or set of bits) in React's internal priority bitmask representing a class of update.
- **SyncLane** — the highest-priority lane, used for updates that must be reflected synchronously (e.g. direct user input in most cases).

**Mental model:**
This is a "have you read the source" question — it separates candidates who've internalized the public urgent/transition mental model from those who understand the actual data structure making it efficient at scale.

**TL;DR:**
Lanes are a bitmask React uses to track multiple simultaneous pending updates per fiber at different priorities, enabling cheap bitwise prioritization instead of comparing scalar priority numbers.

**References:**
- [facebook/react — ReactFiberLane.js source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberLane.js)

---

### Q4. What does it mean for a component to "suspend," and what actually happens under the hood?

**Question:**
What does it mean for a component to "suspend," and what actually happens under the hood?

**Good answer:**
A component suspends when, during render, it throws a Promise that hasn't resolved yet — either implicitly via a Suspense-integrated API (like `use()` reading a pending Promise, or `React.lazy()` loading a chunk) or, historically, by throwing one directly. When React catches that thrown Promise, it doesn't treat it as an error: it pauses rendering that subtree, walks up to find the nearest enclosing `<Suspense>` boundary, and renders that boundary's `fallback` instead. React then attaches a listener to the Promise; when it resolves, React schedules a retry of the suspended render. Rendering that produced a Suspense pause does **not** get committed to the DOM — React discards it, so a component doesn't keep partially-applied state changes across a suspension.

**Code example:**
```jsx
function Albums({ albumsPromise }) {
  // If albumsPromise is still pending, use() throws it,
  // and this component "suspends" — React shows the fallback instead.
  const albums = use(albumsPromise);
  return <ul>{albums.map(a => <li key={a.id}>{a.title}</li>)}</ul>;
}

<Suspense fallback={<Spinner />}>
  <Albums albumsPromise={fetchAlbums()} />
</Suspense>
```

**Follow-up question:**
Why does data fetched in a `useEffect` not activate a Suspense boundary, but data read with `use()` does?

**Follow-up good answer:**
Suspense only detects suspensions that happen **during render** — that's the point in the lifecycle where React is walking the fiber tree and can catch a thrown Promise before committing anything. `useEffect` runs *after* commit, as a side effect of an already-completed render, so by the time the effect's fetch kicks off, React has already finished (and committed) that render pass — there's nothing left for a Suspense boundary upstream to intercept. `use()`, by contrast, is called directly in the render function, so if its Promise is pending, the throw happens exactly at the point React can catch it and swap in the fallback before anything is painted.

**Glossary:**
- **Suspend** — a component throwing an unresolved Promise during render, causing React to show a fallback instead of committing.
- **`use()`** — the React API that can unwrap a Promise (or context) during render and integrates with Suspense.

**Mental model:**
Checks whether the candidate understands Suspense as a render-time mechanism, not a general "loading state" feature — a very common source of "why doesn't my spinner show" bugs when people fetch in effects and expect Suspense to catch it.

**TL;DR:**
A component suspends by throwing an unresolved Promise during render (e.g. via `use()`); React catches it, shows the nearest boundary's fallback, and retries once the Promise resolves — effects run too late to trigger this.

**References:**
- [Suspense reference — react.dev](https://react.dev/reference/react/Suspense)
- [use() reference — react.dev](https://react.dev/reference/react/use)

---

### Q5. What's the difference between `startTransition` and `useDeferredValue`, and when would you reach for each?

**Question:**
What's the difference between `startTransition` and `useDeferredValue`, and when would you reach for each?

**Good answer:**
Both mark work as low-priority/interruptible, but they operate on different things. `startTransition` (and its hook form `useTransition`) wraps a **state update you control** — you call `setState` inside the transition callback to tell React "this particular update is not urgent." `useDeferredValue` instead wraps a **value you receive**, typically a prop or a value derived from state you don't own the setter for — React returns a version of that value that may "lag behind" the latest one, re-rendering with the deferred (stale) value first and then re-rendering again in the background once it can catch up. Rule of thumb: if you're calling `setState` yourself, use `startTransition`; if you're consuming a value someone else is updating (e.g. a prop, or you don't want to touch how the underlying state is set), use `useDeferredValue`.

**Code example:**
```jsx
// startTransition: you own the setState call
startTransition(() => setSearchQuery(input));

// useDeferredValue: you only have the value, not its setter
const deferredQuery = useDeferredValue(query);
<SearchResults query={deferredQuery} />
```

**Follow-up question:**
Why can't you use `startTransition` to wrap the `setState` call that updates a controlled text input's value as the user types?

**Follow-up good answer:**
A controlled input must reflect the DOM's actual current value synchronously — if you marked that update as a transition, React could delay applying it, and the input's displayed value would visibly lag behind what the user typed (or even revert, since transitions can be interrupted/discarded), which breaks the fundamental controlled-input contract that the displayed value always matches state. React's docs explicitly call this out: transitions are for the resulting UI changes caused by input (e.g. filtering a list), not for the input's own committed value, which must stay urgent/synchronous.

**Glossary:**
- **Transition** — an update marked non-urgent via `startTransition`/`useTransition`.
- **Deferred value** — a value returned by `useDeferredValue` that may temporarily lag the source value during a low-priority re-render.

**Mental model:**
Tests precise API knowledge and whether the candidate has hit the "why did my input feel laggy" pitfall of misapplying transitions to input state.

**TL;DR:**
Use `startTransition` when you own the `setState` call; use `useDeferredValue` when you only have a value/prop you don't control the setter for — never wrap a controlled input's own value in either.

**References:**
- [useTransition reference — react.dev](https://react.dev/reference/react/useTransition)
- [useDeferredValue reference — react.dev](https://react.dev/reference/react/useDeferredValue)

---

### Q6. How do you diagnose jank caused by a large, low-priority update blocking urgent user input, and what's the fix?

**Question:**
A user reports that typing in a search box feels laggy while a large results list is rendering below it. How do you diagnose the cause, and how would you fix it?

**Good answer:**
Start with the React DevTools **Profiler** tab: record an interaction (type a character) and look at the flame graph/ranked chart for the commit — a long, uninterrupted render of the results list that dominates the frame is the smoking gun. Cross-reference with the browser's Performance panel to see the corresponding long task blocking the main thread. Once confirmed, the fix is to mark the state update that re-renders the expensive list as a transition: wrap the `setState` that drives the list in `startTransition` (if you own it), or wrap the value the list renders from in `useDeferredValue` (if you're just consuming a prop/query value). This lets React keep the input's own state update urgent/synchronous while rendering the expensive list in the background, interruptible by further keystrokes. After the fix, re-profile: the previously-long render should now be broken into smaller, interruptible chunks, and input latency in the Profiler's "Interactions" timeline should drop.

**Code example:**
```jsx
function Search() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {/* Renders with a possibly-stale query while typing continues smoothly */}
      <ResultsList query={deferredQuery} />
    </>
  );
}
```

**Follow-up question:**
After applying `useDeferredValue`, how do you show a visual indicator that the results are "stale" while the deferred render catches up?

**Follow-up good answer:**
Compare the live value to the deferred value — if they differ, a background render is still pending, so you can style the stale content (e.g. dim it) until they match again: `const isStale = query !== deferredQuery;` and apply something like `style={{ opacity: isStale ? 0.5 : 1 }}` to the results container. This is the pattern shown in React's own docs for `useDeferredValue` — it avoids needing separate loading state and keeps the "old content visible but dimmed" UX pattern.

**Glossary:**
- **React DevTools Profiler** — browser extension panel that records commits and renders a flame/ranked chart of render durations per component.
- **Long task** — any main-thread task over ~50ms, flagged in the browser Performance panel as a cause of input latency.

**Mental model:**
This is the "performance diagnosis methodology" question — it checks whether the candidate has an actual workflow (profiler → identify the long render → apply the right primitive → re-verify) rather than just knowing the API exists in the abstract.

**TL;DR:**
Diagnose with the DevTools Profiler + browser Performance panel to find the long blocking render, then fix by marking the expensive update as a transition (`startTransition`/`useDeferredValue`) so input stays responsive.

**References:**
- [useDeferredValue reference — react.dev (staleness pattern)](https://react.dev/reference/react/useDeferredValue)
- [React Developer Tools — Profiler documentation](https://react.dev/learn/react-developer-tools)

---

### Q7. Why does React only reveal newly-ready Suspense content "at most once every 300ms," and what problem does that solve?

**Question:**
Why does React only reveal newly-ready Suspense content "at most once every 300ms," and what problem does that solve?

**Good answer:**
If several Suspense boundaries on a page are fetching independently and each pops its fallback out the instant its own data resolves, users see a jarring cascade of content popping in one piece at a time (visual "flashing"). By batching reveals into a window (React commits ready boundaries together rather than one at a time, roughly every 300ms), React trades a small amount of extra latency for a visually calmer experience — several boundaries that become ready close together get revealed as one coherent update instead of several jarring ones.

**Follow-up question:**
Does this batching window apply to the very first time a Suspense boundary reveals its content, or only to subsequent re-suspensions?

**Follow-up good answer:**
It's about how React reveals *content becoming ready* in general during a render pass — the throttling governs how frequently React commits newly-revealed Suspense content to the screen, so that many boundaries resolving in a short window are shown together rather than trickling in individually. It's a general anti-flicker mechanism for Suspense reveals, not specifically tied to first-mount vs. re-suspension — the important distinction for re-suspension is instead about *whether the fallback shows again at all* (governed by whether the update was a transition), which is a separate rule from the reveal-batching window.

**Glossary:**
- **Suspense boundary** — the nearest `<Suspense>` ancestor that catches a suspension and shows its `fallback`.
- **Reveal** — the moment React swaps a Suspense boundary's fallback for its real content.

**Mental model:**
Tests whether the candidate has internalized Suspense as having deliberate UX-driven timing behavior, not just an on/off fallback switch — a detail most people miss unless they've read the reference docs closely.

**TL;DR:**
React throttles Suspense content reveals to roughly once per 300ms so multiple boundaries resolving close together appear together, avoiding a jarring one-at-a-time popping-in effect.

**References:**
- [Suspense reference — react.dev](https://react.dev/reference/react/Suspense)

---

### Q8. What real-world problem did "automatic batching" in React 18 solve?

**Question:**
What real-world problem did "automatic batching" in React 18 solve?

**Good answer:**
Before React 18, React only batched multiple `setState` calls into a single re-render when they happened inside a React event handler (e.g. `onClick`). If you called several `setState`s inside a `setTimeout`, a native DOM event handler, a Promise `.then()`, or any other non-React-managed callback, each call triggered its own separate re-render — wasteful, and it could also produce inconsistent intermediate UI states between the calls. React 18 made batching automatic everywhere by default, regardless of where the updates originate, so multiple `setState` calls inside a `setTimeout` or an `async` function's resolved Promise now produce a single re-render just like they would inside an `onClick` handler.

**Code example:**
```jsx
// React 17: two re-renders. React 18: one re-render (batched automatically).
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
}, 1000);
```

**Follow-up question:**
If you need to force a synchronous, unbatched update in React 18 (opting out of batching for a specific update), what API do you use?

**Follow-up good answer:**
`flushSync` from `react-dom`, which wraps a state update and forces React to flush it synchronously (and separately from any batch) before `flushSync` returns. It's an escape hatch meant for rare cases — like needing to read a DOM measurement immediately after a state-driven DOM change — and the docs discourage it as a first resort since it defeats the performance benefit batching provides.

**Glossary:**
- **Batching** — combining multiple state updates into a single re-render/commit.
- **`flushSync`** — a `react-dom` API that forces an update to be applied synchronously, opting out of batching for that update.

**Mental model:**
Probes whether the candidate understands *why* a change to internal scheduling behavior (batching) was significant enough to be a headline React 18 feature — connects a low-level implementation detail to a concrete, relatable bug class (inconsistent multi-setState UI).

**TL;DR:**
React 18 made batching automatic everywhere (timeouts, promises, native handlers), not just inside React event handlers, avoiding wasted extra re-renders; `flushSync` opts out when needed.

**References:**
- [React v18.0 blog post — Automatic Batching](https://react.dev/blog/2022/03/29/react-v18)

---

### Q9. What "waterfall" problem does Suspense for data fetching solve, and how?

**Question:**
What "waterfall" problem does Suspense for data fetching solve, and how?

**Good answer:**
A classic React data-fetching waterfall happens when a parent component fetches data in an effect, renders its children only once that data arrives, and *those* children then start their own fetches in their own effects — each fetch can only begin after the previous component has rendered, so requests that could have run in parallel end up serialized one after another, adding up their latencies instead of overlapping them. Suspense-based data fetching (via `use()` reading Promises, especially ones created by a Server Component or a framework's data layer) lets you *start* multiple fetches up front — before rendering, in parallel — and then let each component suspend independently while waiting for its own Promise, with nested `<Suspense>` boundaries controlling which parts of the UI wait for which data. Because the fetches were kicked off eagerly and in parallel rather than triggered one render-cycle at a time, the overall time-to-content is the slowest single fetch, not the sum of all of them.

**Follow-up question:**
Does simply switching from `useEffect`-based fetching to Suspense automatically fix a waterfall, or do you still have to change how the fetches are initiated?

**Follow-up good answer:**
Just switching to `use()` doesn't fix it by itself — if you still only create the Promise for a child's data inside the child component's render (or worse, only after some parent state resolves), the fetch still doesn't start until that point in the render, and you still get serialized waterfalls, just now expressed as suspensions instead of effects. The actual fix is to *initiate* the fetches as early and as parallel as possible — e.g. kick off all needed Promises at the top of the tree (or on the server) before rendering the components that will `use()` them — so that by the time each component's render reaches its `use()` call, the request is already in flight rather than about to start.

**Glossary:**
- **Waterfall** — a chain of sequential (rather than parallel) network requests caused by each one only starting after a prior render/fetch completes.
- **Render-as-you-fetch** — the pattern of starting data fetches before or during render rather than after, so components can suspend on already-in-flight requests.

**Mental model:**
This is a "do you actually understand the mechanism, not just the API" question — many candidates can define Suspense but don't realize it doesn't automatically parallelize fetches unless you also restructure *when* the fetches are kicked off.

**TL;DR:**
Suspense fixes fetch waterfalls only if fetches are also initiated early/in-parallel (render-as-you-fetch) — merely swapping `useEffect` for `use()` without restructuring when fetches start still serializes them.

**References:**
- [use() reference — react.dev (caching/Promise creation timing)](https://react.dev/reference/react/use)
- [Server Components reference — react.dev](https://react.dev/reference/rsc/server-components)

---

### Q10. What's a common mistake when placing `<Suspense>` boundaries, and what effect does it have?

**Question:**
What's a common mistake when placing `<Suspense>` boundaries, and what effect does it have?

**Good answer:**
A very common mistake is wrapping an entire page (or a large section with several independent pieces of content) in a single `<Suspense>` boundary. Since a boundary shows its fallback for *any* suspension anywhere inside it, and doesn't reveal anything until *everything* inside is ready, this means fast-loading content gets held hostage by the slowest child — a quick-to-render sidebar waits behind a slow comments section, even though they have nothing to do with each other. The fix is to nest boundaries around independently-loading pieces of content, so fast content can reveal immediately while only the genuinely slow piece shows its own, smaller fallback (e.g. wrap the whole page in an outer boundary for the overall shell, and put a nested boundary specifically around the slow `Posts` component).

**Code example:**
```jsx
// Too coarse: Posts being slow blocks Sidebar too
<Suspense fallback={<BigSpinner />}>
  <Sidebar />
  <Posts />
</Suspense>

// Better: Sidebar reveals immediately, Posts has its own fallback
<Suspense fallback={<BigSpinner />}>
  <Sidebar />
  <Suspense fallback={<PostsGlimmer />}>
    <Posts />
  </Suspense>
</Suspense>
```

**Follow-up question:**
Conversely, what's a downside of going too far the other way and wrapping every single small component in its own Suspense boundary?

**Follow-up good answer:**
Excessive granularity produces a UI that pops in piecemeal, one tiny fragment at a time, which can feel more jarring/inconsistent than a slightly coarser grouping that reveals a few related pieces together — and each boundary is also overhead (more fallback UI to design, more re-suspension edge cases to reason about, like whether a transition should keep old content visible). The practical approach is to group boundaries around content that's logically related and tends to load together, not around every individual component.

**Glossary:**
- **Suspense boundary granularity** — the choice of how much UI a single `<Suspense>` wraps, trading "reveal everything together" against "reveal each piece independently."

**Mental model:**
Tests real production judgment, not just API syntax — Suspense boundary placement is one of the most common "looks right, isn't" mistakes in real React codebases.

**TL;DR:**
One coarse Suspense boundary makes fast content wait behind the slowest sibling; nest boundaries around independently-loading pieces so fast content reveals immediately.

**References:**
- [Suspense reference — react.dev (nested boundaries example)](https://react.dev/reference/react/Suspense)

---

### Q11. What happens if you trigger a Suspense-causing update without wrapping it in `startTransition`?

**Question:**
What happens if you trigger a Suspense-causing update (e.g. navigating to a view that suspends) without wrapping it in `startTransition`?

**Good answer:**
By default, if a state update causes a component to suspend, React swaps the already-visible content for the nearest Suspense boundary's fallback right away — the user sees a jarring flash from real content back to a spinner, even if they were just looking at a fully-rendered previous view. Wrapping that same update in `startTransition` changes this behavior: React will *not* hide already-revealed content for suspensions triggered by a transition — instead it keeps showing the old content (and reflects the pending state via `isPending`) until the new content is ready, then swaps directly from old content to new content with no intermediate fallback flash.

**Code example:**
```jsx
function navigate(url) {
  // Without startTransition: currently-visible page flashes to fallback.
  // With startTransition: old page stays visible until new page is ready.
  startTransition(() => setPage(url));
}
```

**Follow-up question:**
If a transition-wrapped navigation takes a long time, does the user just see a frozen old page indefinitely with no feedback?

**Follow-up good answer:**
No — `useTransition` returns an `isPending` boolean specifically so you can show a non-blocking pending indicator (a small spinner, a progress bar, a disabled/dimmed state on a nav link) while the old content is still visible, giving the user feedback that something is happening without replacing the whole screen with a full-page fallback. That's the intended pattern: keep old UI interactive-looking, show a subtle pending signal, and swap only once the new content is fully ready.

**Glossary:**
- **`isPending`** — the boolean `useTransition` returns while its transition is in flight, for showing pending UI without unmounting current content.

**Mental model:**
Tests understanding of the actual UX payoff of transitions — many candidates know the API exists but can't articulate the specific "no fallback flash on suspend" behavior that's the whole point of pairing transitions with Suspense.

**TL;DR:**
Without `startTransition`, a suspending update immediately flashes visible content back to the fallback; wrapping it in a transition keeps old content visible (with `isPending`) until the new content is ready.

**References:**
- [Suspense reference — react.dev (revealing content section)](https://react.dev/reference/react/Suspense)
- [useTransition reference — react.dev](https://react.dev/reference/react/useTransition)

---

### Q12. What's a common misuse of `useTransition` that developers run into, related to async functions?

**Question:**
What's a common misuse of `useTransition` that developers run into, related to async functions?

**Good answer:**
Developers often assume that *any* state update inside a `startTransition(async () => { ... })` callback is automatically marked as a transition, including updates that happen after an `await`. In reality, only the synchronous portion of the callback — the part that runs before the first `await` — is tracked as part of the transition; state updates issued *after* an `await` have lost that tracked scope and are treated as regular (urgent) updates, which defeats the purpose (they'll show a fallback flash / block the UI just like an un-transitioned update would). The fix is to re-wrap the post-`await` state update in its own nested `startTransition` call.

**Code example:**
```jsx
// Bug: setPage() after await is NOT marked as a transition
startTransition(async () => {
  await someAsyncFunction();
  setPage('/about'); // treated as an urgent update
});

// Fix: re-wrap after the await
startTransition(async () => {
  await someAsyncFunction();
  startTransition(() => setPage('/about'));
});
```

**Follow-up question:**
Why does React lose track of the transition scope specifically at the `await` boundary?

**Follow-up good answer:**
React's `startTransition` marks updates as transitions by tracking scope synchronously as the callback executes — the function passed to `startTransition` runs immediately, and any `setState` calls made during that synchronous execution get tagged. Once execution hits an `await`, control returns to the JS event loop and resumes later as a fresh microtask/callback, outside of the synchronous call stack React was tracking; React has no built-in way (yet) to know that resumed continuation still "belongs" to the original transition. The docs note this is expected to improve once the TC39 `AsyncContext` proposal lands, which would let context (like "we're inside a transition") propagate across `await` boundaries.

**Glossary:**
- **Async context loss** — the loss of an implicit tracked scope (like "we're inside a transition") once execution crosses an `await`/microtask boundary.

**Mental model:**
A pitfall/edge-case question that filters for people who've actually hit this in a real codebase with async transitions (e.g. server actions) versus people who've only used transitions with purely synchronous `setState` calls.

**TL;DR:**
Only the synchronous part of a `startTransition` callback (before the first `await`) is tracked as the transition — `setState` calls after an `await` need their own nested `startTransition` or they run as urgent updates.

**References:**
- [useTransition reference — react.dev (async caveats section)](https://react.dev/reference/react/useTransition)

---

### Q13. What is "selective hydration," and what problem does it solve for streaming SSR?

**Question:**
What is "selective hydration," and what problem does it solve for streaming SSR?

**Good answer:**
In streaming SSR, React sends HTML to the client progressively, with Suspense boundaries producing fallback markup first and their real content streamed in later as it becomes ready on the server. Without selective hydration, the client would have to wait for *all* of that HTML (and its corresponding JS) to arrive before hydrating anything, which wastes the head start streaming gave you. Selective hydration lets React hydrate each Suspense boundary's content independently, as soon as its HTML and code have arrived — and, crucially, React prioritizes hydrating whichever part of the page the user is actually trying to interact with (e.g. if they click a button in a boundary that streamed in but hasn't hydrated yet, React will hydrate that boundary first, ahead of others that arrived earlier but the user isn't touching).

**Follow-up question:**
Why does streaming SSR without selective hydration risk making the page feel *more* broken than a traditional non-streaming SSR page, even though content appears faster?

**Follow-up good answer:**
Without selective hydration, content can be visible (streamed in, painted) but non-interactive for longer, because hydration still has to proceed in whatever fixed order the framework hydrates the tree — a user might see a fully-rendered button and click it, only to have nothing happen because that part of the tree hasn't hydrated yet. That's a worse experience than a traditional blocking SSR page where nothing is visible until hydration is basically ready, because at least there the "looks interactive" and "is interactive" states don't diverge as visibly. Selective hydration closes that gap by making the parts the user is actually trying to use hydrate first, regardless of streaming order.

**Glossary:**
- **Selective hydration** — hydrating individual Suspense boundaries independently and prioritizing whichever one the user is interacting with.
- **Streaming SSR** — sending server-rendered HTML to the client progressively rather than as one complete response.

**Mental model:**
Advanced-feature question that checks whether the candidate understands streaming SSR as more than "faster HTML delivery" — the hydration-ordering problem it also solves is the less obvious, more interesting half of the feature.

**TL;DR:**
Selective hydration hydrates each Suspense boundary independently and prioritizes whichever one the user is interacting with, instead of hydrating in a fixed order regardless of streaming arrival.

**References:**
- [renderToPipeableStream reference — react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q14. Walk through how `renderToPipeableStream` streams a page that has nested Suspense boundaries.

**Question:**
Walk through how `renderToPipeableStream` streams a page that has nested Suspense boundaries.

**Good answer:**
React first renders the "shell" — everything outside any Suspense boundary — synchronously on the server. Once the shell is ready, the `onShellReady` callback fires; that's the signal to start piping the response to the client, so the browser can start painting immediately (fast First Contentful Paint) even though nested content isn't ready yet. Suspense boundaries inside the shell are sent as their `fallback` markup initially. As each boundary's actual content finishes rendering on the server, React streams down the real HTML for that boundary along with an inline `<script>` that swaps the fallback out for the real content in the DOM once it arrives — this can happen out of order, whichever boundary's data resolves first streams first, regardless of its position in the tree. `onAllReady` fires once everything (including all boundaries) is fully rendered, which is useful for cases like crawlers where you want the complete HTML in one shot instead of a stream.

**Code example:**
```js
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    response.statusCode = 200;
    response.setHeader('content-type', 'text/html');
    pipe(response); // stream starts as soon as the shell is ready
  },
  onAllReady() {
    if (isCrawler) pipe(response); // wait for everything for crawlers
  },
  onShellError(error) {
    response.statusCode = 500;
    response.send('<h1>Something went wrong</h1>');
  }
});
```

**Follow-up question:**
What's the difference between handling an error via `onShellError` versus `onError`?

**Follow-up good answer:**
`onShellError` fires when the error happens *before* the shell itself finished rendering — nothing has been sent to the client yet, so there's no way to recover into a partial page; the correct response is to send a full fallback error page instead (e.g. a 500 status with static error HTML). `onError` fires for errors happening *outside* the shell — inside a Suspense-wrapped boundary after the shell has already started streaming — and those are recoverable: React sends that boundary's fallback (or triggers its nearest error boundary) while the rest of the already-streamed shell continues working normally; `onError` is mainly for logging/reporting those errors, since the streaming response has already begun and can't be aborted wholesale.

**Glossary:**
- **Shell** — the part of the tree outside any Suspense boundary, rendered and sent first.
- **`onShellReady` / `onAllReady`** — callbacks marking "safe to start streaming" vs. "everything is fully rendered."

**Mental model:**
Tests hands-on familiarity with the actual streaming SSR API surface, not just the conceptual pitch — a candidate who's actually wired this up in a framework (or by hand) will know these callback distinctions.

**TL;DR:**
`renderToPipeableStream` sends the shell first (`onShellReady`), streams each Suspense boundary's content as it resolves (out of order), and fires `onAllReady` once everything is done, e.g. for crawlers.

**References:**
- [renderToPipeableStream reference — react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q15. What are React Server Components, and how do they relate to Suspense?

**Question:**
What are React Server Components, and how do they relate to Suspense?

**Good answer:**
Server Components are a component type that renders ahead of time in a server-only environment — either once at build time or per-request on a server — and their JavaScript is never sent to the client; only their rendered output (a special serialized format, not plain HTML) ships to the browser. Because they run only on the server, they can `await` directly in the component body, hit a database or filesystem, and pull in server-only dependencies without any of that code bloating the client bundle. They can't use client-only hooks like `useState`/`useEffect`, so interactivity has to be delegated to Client Components (marked `"use client"`) that Server Components render as children. The Suspense connection: a Server Component can create a Promise (e.g. kick off a slower query) and pass it down without awaiting it itself, and a descendant Client Component can `use()` that Promise, suspending on the client while the data streams in from the server — letting higher-priority content (e.g. the page shell) render immediately while lower-priority data (e.g. comments) streams in later, all coordinated through the same Suspense mechanism used elsewhere.

**Follow-up question:**
Why do the React docs caution that "the underlying APIs for implementing Server Components don't follow semver"?

**Follow-up good answer:**
Server Components' implementation (the bundler/streaming protocol, the serialization format for the tree sent to the client, integration points a framework needs) is still evolving and considered lower-level than the stable public component APIs — these are primarily meant to be consumed through a framework (e.g. Next.js) that wires them up correctly, not built directly by most application developers. Because that plumbing can change in ways that would normally be considered breaking, React's guidance is that frameworks should pin to specific React versions (or track Canary releases) rather than application code depending directly on those low-level APIs and assuming normal semver stability guarantees.

**Glossary:**
- **Server Component** — a component that renders only on the server; its code never ships to the client.
- **Client Component** — a component marked `"use client"` that can use interactive hooks and runs (or hydrates) in the browser.

**Mental model:**
A trending, advanced-features question — checks whether the candidate can articulate the actual architectural split (what runs where, why) rather than just repeating "Server Components are faster."

**TL;DR:**
Server Components render server-side only and never ship JS to the client; they can pass unawaited Promises down to Client Components, which `use()` them and suspend, tying RSC data-fetching into the same Suspense mechanism.

**References:**
- [Server Components reference — react.dev](https://react.dev/reference/rsc/server-components)

---

### Q16. What's a real downside/trade-off of marking an update as a transition with `useTransition`?

**Question:**
What's a real downside/trade-off of marking an update as a transition with `useTransition`?

**Good answer:**
Marking an update as a transition means React can interrupt or even discard it in favor of newer, higher-priority work — that's the entire point for keeping the UI responsive, but it also means you can't rely on a transition-wrapped update happening at any predictable time, or even at all if it keeps getting superseded by newer transitions before it finishes (e.g. very rapid navigation clicks). It's also not free of overhead: React does the rendering work for a transition on top of whatever else is happening, so an app that's already CPU-constrained doesn't get more total throughput from transitions — it gets *better prioritized* throughput, meaning the urgent stuff stays snappy while the non-urgent stuff can still take just as long (or longer, since it keeps getting bumped) to finish. And because multiple transitions are currently batched together, if you have several independent transition-triggering interactions close together, they may end up coupled in ways you didn't intend.

**Follow-up question:**
Given that trade-off, what kind of update should you specifically avoid wrapping in a transition?

**Follow-up good answer:**
Anything where the user needs an immediate, guaranteed, and predictable reflection of their action — most obviously the value of a controlled input as covered earlier, but more generally any update where staleness or the possibility of the update being superseded/discarded would be confusing or wrong (e.g. a "form submitted" confirmation state, a toggle whose own visual state must match what was clicked). Transitions are for the *downstream consequences* of an interaction that can tolerate being deferred or superseded — not for the interaction's own direct, expected feedback.

**Glossary:**
- **Superseded transition** — a transition update that gets discarded because a newer transition interrupted it before it could complete.

**Mental model:**
A trade-offs question that pushes past "transitions are good, always use them for slow updates" into knowing where the technique actually has costs and doesn't apply.

**TL;DR:**
Transitions can be interrupted/discarded and add no extra CPU throughput — they only reprioritize work, so they're wrong for updates needing guaranteed, immediate, predictable feedback.

**References:**
- [useTransition reference — react.dev (caveats section)](https://react.dev/reference/react/useTransition)

---

### Q17. How would you use the React DevTools Profiler to figure out *which* component is causing dropped frames during an interaction?

**Question:**
How would you use the React DevTools Profiler to figure out *which* component is causing dropped frames during an interaction?

**Good answer:**
Open the Profiler tab, click record, perform the janky interaction, then stop recording. The Profiler shows a commit-by-commit timeline; select the commit(s) around the janky interaction and look at the flame graph (or ranked chart, which sorts components by self-time) — wide bars / high-ranked entries indicate components that took a long time to render in that commit. The flame graph also shows *why* a component re-rendered (props changed, state changed, hooks changed, or a parent re-rendered) when you hover/click a bar, which helps distinguish "this component is slow" from "this component didn't need to re-render at all." For frame-level detail (whether the render actually caused a dropped frame on the main thread), cross-reference with the browser's own Performance panel, which shows long tasks and frame timing alongside the same time window. The combination — Profiler for "what re-rendered and why," Performance panel for "did it actually block a frame" — is the standard workflow for isolating the culprit before reaching for a fix like memoization or a transition.

**Follow-up question:**
The Profiler shows a component re-rendered but its render time was tiny — could that component still be responsible for perceived jank?

**Follow-up good answer:**
Yes — a component can have a fast individual render but still contribute to jank if it's re-rendering unnecessarily and frequently (e.g. on every keystroke) as part of a large subtree, where the cumulative cost of many small re-renders across many components in the same commit adds up to a long total commit time even though no single component looks expensive in isolation. This is why the ranked/flame chart's *aggregate* commit duration matters as much as any single component's self-time — and it's a common reason to reach for `React.memo`/`useMemo` on a component that "isn't slow" per se, but is being re-rendered far more often than necessary and adding to the total.

**Glossary:**
- **Self-time** — time a component's own render took, excluding its children.
- **Ranked chart** — a Profiler view that lists components in a commit sorted by render duration.

**Mental model:**
The core performance-diagnosis-methodology question for this set — checks for an actual repeatable workflow, not just "I'd use the Profiler" without knowing what to look for once it's open.

**TL;DR:**
Record the interaction in the Profiler, read the flame/ranked chart for long/self-time-heavy renders and their re-render cause, cross-check with the browser Performance panel for actual blocked frames, then re-profile after the fix.

**References:**
- [React Developer Tools — react.dev](https://react.dev/learn/react-developer-tools)

---

### Q18. What problem does `useSyncExternalStore` solve, and what is "tearing" in the context of concurrent rendering?

**Question:**
What problem does `useSyncExternalStore` solve, and what is "tearing" in the context of concurrent rendering?

**Good answer:**
"Tearing" is when different components on screen end up displaying inconsistent versions of the same underlying data within a single render — it can happen when a component reads from a mutable external store (e.g. a state-management library that isn't React-aware) outside React's own state system, and that store's value changes *while* a concurrent render is in progress; because concurrent rendering can pause and resume, one part of the tree might read the store before the mutation and another part after, producing a UI where two components disagree about the current value in the same paint. `useSyncExternalStore` is React's official hook for safely subscribing to such external stores: it guarantees a **consistent snapshot** for the whole render by re-checking `getSnapshot` right before committing a transition update, and if it detects the snapshot changed mid-flight, React discards that render and redoes the update as a synchronous (blocking, non-interruptible) one instead — trading away some of the interruptibility benefit specifically to guarantee consistency for that update.

**Code example:**
```js
const value = useSyncExternalStore(
  store.subscribe,   // how React should know the store changed
  store.getSnapshot   // must return a cached/stable reference if unchanged
);
```

**Follow-up question:**
Why can't ordinary `useState`/`useReducer`-based state ever tear the way an external store can?

**Follow-up good answer:**
State managed via `useState`/`useReducer` lives inside React's own fiber tree and update/lane system — React fully controls when and how it changes as part of a render, so it can guarantee every component reading that state within the same render pass sees the same, consistent value, because the value literally *is* part of what React is coordinating. An external store, by definition, mutates outside React's knowledge and scheduling (e.g. a module-level variable, a third-party store's internal state) — React has no built-in way to know it changed except by being told via a subscription, which is exactly the gap `useSyncExternalStore` closes by giving React an explicit hook into "did this external thing change" so it can apply the same consistency guarantees it gives its own state.

**Glossary:**
- **Tearing** — different parts of the UI reflecting different, inconsistent versions of the same data within one render.
- **`getSnapshot`** — the function passed to `useSyncExternalStore` that must return a stable/cached value representing the store's current state.

**Mental model:**
An advanced internals question that connects concurrent rendering's interruptibility back to a concrete correctness risk it introduces for state outside React's control — checks whether the candidate understands concurrency has real trade-offs, not just benefits.

**TL;DR:**
Tearing is inconsistent UI reads of a mutable external store during a paused/resumed concurrent render; `useSyncExternalStore` guarantees a consistent snapshot by re-checking and falling back to a synchronous render if the store changed mid-flight.

**References:**
- [useSyncExternalStore reference — react.dev](https://react.dev/reference/react/useSyncExternalStore)

---

### Q19. Compare React's cooperative scheduling model to OS-level preemptive thread scheduling — what's the underlying CS concept, and where does React's approach fall short of true preemption?

**Question:**
Compare React's cooperative scheduling model to OS-level preemptive thread scheduling — what's the underlying CS concept, and where does React's approach fall short of true preemption?

**Good answer:**
In preemptive scheduling (how OS thread schedulers typically work), the scheduler can forcibly suspend a running thread at essentially any point — often via a hardware timer interrupt — regardless of what that thread is doing, and resume a different one. In cooperative scheduling, a running task must voluntarily yield control back to the scheduler at explicit checkpoints; if it never yields, the scheduler can't do anything about it. React's concurrent rendering is fundamentally cooperative: it can only "pause" between discrete units of work (fibers), which are the checkpoints where React's work loop asks "do I have time left in this frame, or is there higher-priority work waiting?" and decides whether to continue or yield back to the browser. The gap versus true preemption shows up when a single fiber's own render work is itself expensive (e.g. one component doing a huge synchronous computation in its render function) — React has no way to interrupt that single unit of work partway through, because JavaScript's run-to-completion semantics mean nothing can preempt a running synchronous function. So React's interruptibility is real between components, but not within one.

**Follow-up question:**
Given that gap, what's the practical guidance for a component whose own render logic is genuinely expensive (not just "there are many components")?

**Follow-up good answer:**
Since React can't chop up work *inside* a single component's render, the fix has to happen at the application level: break the expensive computation itself into smaller pieces that can be spread across multiple renders/units of work (e.g. windowing/virtualizing a huge list so each row is its own lightweight fiber instead of one component computing all rows at once), move genuinely heavy computation off the main thread entirely (a Web Worker), or memoize so the expensive work only reruns when its inputs actually change rather than on every render. The scheduler-level fixes (`startTransition`, `useDeferredValue`) help with *when* expensive work runs relative to other priorities, but they don't make one already-running synchronous computation interruptible — that requires actually restructuring the work.

**Glossary:**
- **Cooperative scheduling** — a scheduling model where tasks must voluntarily yield control at checkpoints, rather than being forcibly interrupted.
- **Preemptive scheduling** — a scheduling model where the scheduler can forcibly suspend running work at any point.

**Mental model:**
The "software engineering theory mixed with technology-specific practice" question for this set — checks whether the candidate can connect React's scheduler to the general CS vocabulary around scheduling, and correctly identify the real limit of the abstraction rather than overselling "React makes everything interruptible."

**TL;DR:**
React's scheduling is cooperative (yields only between fibers), not preemptive — it can't interrupt one component's own expensive synchronous render, which must be fixed by restructuring the work, not the scheduler.

**References:**
- [React Fiber Architecture — unit of work / cooperative yielding (community explainer, reviewed by React team members)](https://github.com/acdlite/react-fiber-architecture)
- [facebook/react — react-reconciler work loop source](https://github.com/facebook/react/tree/main/packages/react-reconciler/src)

---

### Q20. Suppose you suspend on a transition and the boundary was already showing content — does the user see the fallback again? What governs this?

**Question:**
Suppose an already-revealed Suspense boundary suspends again as the result of a transition-triggered update — does the fallback show again, and what determines the answer?

**Good answer:**
No — if the update that caused the re-suspension was wrapped in a transition (`startTransition`/`useTransition`), React will not hide the already-revealed content; it keeps showing the previous, still-valid content in place while the new render happens in the background, only swapping to the new content once it's ready (with `isPending` available to show a subtle in-progress indicator). This is specifically different from a re-suspension caused by a non-transition update, where React does swap back to the fallback immediately, producing the visible flash discussed earlier. This is the mechanism that makes patterns like "click a tab, old tab's content stays visible with a pending indicator until the new tab's data is ready" work smoothly instead of flashing a spinner on every tab switch.

**Follow-up question:**
What's the interaction between this "don't hide revealed content" rule and `useDeferredValue`'s staleness behavior — are they the same mechanism?

**Follow-up good answer:**
They're closely related but expressed through different APIs for different situations: `useDeferredValue` is the pattern for when you're consuming a *value* (not calling `setState` yourself) and want to keep rendering with the old value while a background render with the new value proceeds — you get both the current and deferred value back and can compare them to show staleness styling. The transition-and-Suspense "don't hide revealed content" behavior is the equivalent guarantee at the Suspense-boundary level, for state updates you do control via `startTransition`. Both ultimately rely on the same underlying idea — a low-priority update shouldn't blow away already-visible, still-useful content just because something downstream of it suspended — but `useDeferredValue` is scoped to a single value/prop, while the Suspense behavior is scoped to whatever's inside a boundary.

**Glossary:**
- **Re-suspension** — a Suspense boundary suspending again after it had already revealed real content.

**Mental model:**
Wraps up the set by testing whether the candidate can connect several previously-discussed pieces (transitions, Suspense reveal behavior, `useDeferredValue`) into one coherent mental model of "how does React avoid jarring UI during background updates," rather than knowing each API in isolation.

**TL;DR:**
A transition-triggered re-suspension keeps already-revealed content visible instead of flashing back to the fallback — the same "don't discard useful old content" idea `useDeferredValue` applies at the value level.

**References:**
- [Suspense reference — react.dev (re-suspension / transition interaction)](https://react.dev/reference/react/Suspense)
- [useDeferredValue reference — react.dev](https://react.dev/reference/react/useDeferredValue)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=react&tags=concurrent-features-and-suspense&autostart=1" | relative_url }})
