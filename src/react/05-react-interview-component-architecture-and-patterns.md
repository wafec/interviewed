---
layout: default
title: "React Interview — Component Architecture & Patterns"
---

# React Interview — Component Architecture & Patterns

This set covers how to structure React component trees for maintainability
and performance: composition vs. inheritance, controlled/uncontrolled
components, Context internals, common composition patterns (compound
components, render props, HOCs), classic anti-patterns (prop drilling, god
components, key misuse), and the trade-offs behind larger architectural
choices like state colocation and micro-frontends.

### Q1. React's docs explicitly recommend composition over inheritance for reusing component logic. Why, and what's the idiomatic way to compose components?

**Question:**
Why does React recommend composition over inheritance, and what's the idiomatic pattern for building flexible, reusable components?

**Good answer:**
React's own team found that at Facebook's scale, they never encountered a use case where an inheritance hierarchy of components was a better solution than composition — "props and composition give you all the flexibility you need to customize a component's look and behavior in an explicit and safe way." Inheritance couples a subclass's behavior to its superclass's implementation details, which becomes brittle as a component tree grows; composition instead lets a component stay ignorant of what's inside it. The idiomatic pattern is **containment**: a component that doesn't know its children ahead of time (`Sidebar`, `Dialog`, `Card`) simply renders `{props.children}`, and the caller decides what goes inside. For components that need multiple distinct "slots" rather than one blob of children, you pass JSX via named props instead (e.g. `<SplitPane left={<Contacts />} right={<Chat />} />`). For sharing non-UI logic (not markup), the guidance is to extract it into a plain JS module or a custom hook rather than a base class.

**Code example:**
```jsx
function FancyBorder({ color, children }) {
  return (
    <div className={`FancyBorder FancyBorder-${color}`}>
      {children}
    </div>
  );
}

function WelcomeDialog() {
  return (
    <FancyBorder color="blue">
      <h1>Welcome</h1>
      <p>Thanks for visiting our spacecraft!</p>
    </FancyBorder>
  );
}
```

**Follow-up question:**
What's the equivalent of a "specialization" (e.g. `WelcomeDialog extends Dialog`) if you can't use class inheritance?

**Follow-up good answer:**
You express specialization by rendering the general component and configuring it via props/children, not by subclassing it. A generic `Dialog` component accepts `title` and `message` props (or a `children` slot); `WelcomeDialog` is then just a component that renders `<Dialog title="Welcome" message="Thanks for visiting!" />`. This is "composition as specialization" — the specialized component is a thin wrapper that supplies configuration to the general one, rather than a subclass overriding methods. It keeps the general component's internals fully encapsulated, since the specialized component only interacts with it through its public prop API.

**Glossary:**
- **Containment** — the pattern where a component doesn't know its children ahead of time and renders `props.children` (or a named JSX prop) to let the caller inject content.
- **Specialization** — building a more specific component by configuring a general one via props, instead of subclassing it.

**Mental model:**
This question checks whether a candidate defaults to OOP instincts (inheritance, base classes) or actually understands React's data-flow model, where UI reuse happens through composing function calls (components) and passing data/JSX down, not through a class hierarchy.

**References:**
- [Composition vs Inheritance — React docs](https://legacy.reactjs.org/docs/composition-vs-inheritance.html)

---

### Q2. What's the difference between a controlled and an uncontrolled component in React, and when would you pick one over the other?

**Question:**
Explain controlled vs. uncontrolled components, and describe a scenario where you'd deliberately choose an uncontrolled component.

**Good answer:**
In an **uncontrolled** component, the state that drives the UI (e.g. an input's current text) lives inside the component itself — the parent has no way to read or set it except through the DOM/refs. In a **controlled** component, that state is lifted up to a parent (or external store), which passes it down as props along with the handlers to change it; the component itself holds no independent source of truth for that value. React's official guidance is: for each unique piece of state, exactly one component should "own" it (the single source of truth) — controlled components exist so that ownership can live wherever coordination is needed. Controlled is the right default when multiple components need to stay in sync with that value (e.g. validating a form field against a sibling, or an accordion where opening one panel must close the others). Uncontrolled is reasonable for genuinely isolated, self-contained widgets — e.g. a simple search box with no external validation or synchronization needs — where lifting the state up just adds boilerplate and unnecessary re-renders of the parent for no coordination benefit.

**Code example:**
```jsx
// Uncontrolled: state lives in the child, parent can't see it
function Uncontrolled() {
  const [text, setText] = useState('');
  return <input value={text} onChange={e => setText(e.target.value)} />;
}

// Controlled: parent owns the state, passes it + a setter down
function Controlled({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}
```

**Follow-up question:**
You have two sibling `Panel` components in an accordion, and only one may be expanded at a time. Walk through the refactor from uncontrolled to controlled.

**Follow-up good answer:**
Follow React's "lifting state up" recipe: (1) remove the `isActive` state from each `Panel` — it currently manages its own open/closed flag independently, which is why two panels can be open simultaneously; (2) have the parent `Accordion` component hold a single piece of state, e.g. `activeIndex`, tracking which panel (if any) is open; (3) pass each `Panel` an `isActive={activeIndex === index}` prop and an `onShow={() => setActiveIndex(index)}` handler instead of letting it manage its own boolean. Now `Panel` is a controlled component — it renders based purely on props and reports user intent upward via `onShow` — and the parent is the single source of truth that enforces "only one open at a time" by construction, not by ad hoc coordination between siblings.

**Glossary:**
- **Lifting state up** — moving state from a child to its closest common parent so multiple children can share/coordinate it.
- **Single source of truth** — the principle that each piece of state should be owned by exactly one component.

**Mental model:**
This tests whether the candidate can recognize *when* local component state is actually a design smell — i.e. whether they reach for "lift state up" only when there's a real coordination requirement, rather than either never lifting state (leading to bugs) or reflexively lifting everything (leading to prop-drilling and unnecessary re-renders).

**References:**
- [Sharing State Between Components — React docs](https://react.dev/learn/sharing-state-between-components)

---

### Q3. What is "prop drilling," and what are React's built-in ways to avoid it?

**Question:**
Explain prop drilling as an anti-pattern, and name at least two ways React lets you avoid it.

**Good answer:**
Prop drilling is passing a prop through several layers of components that don't use it themselves, purely so a deeply nested descendant can receive it. It's not wrong in small trees, but as the tree grows it makes every intermediate component depend on data it doesn't care about, so any change to that data's shape forces edits across the whole chain. React offers two built-in ways around it: (1) **Context** (`createContext` + `useContext`), which lets a value be read directly by any descendant without every intermediate component declaring it as a prop; and (2) the **containment pattern** (passing JSX via `children` or a named prop), which restructures the tree so intermediate "layout" components never need to know about the data at all — they just render whatever they were handed. Context is the right tool when many components across different parts of the tree genuinely need the same value (e.g. current theme, logged-in user); containment is the right tool when the intermediate components are purely structural and shouldn't need to know about the data's existence in the first place.

**Follow-up question:**
Why might reaching for Context to "fix" prop drilling sometimes make things worse rather than better?

**Follow-up good answer:**
Context solves the *plumbing* problem but introduces a *coupling and re-render* problem: every consumer of a context re-renders whenever the context value changes, even if only part of that value matters to it — there's no built-in per-field subscription. If the value is broad (e.g. a big app-state object) and updates frequently, you can end up re-rendering a large swath of the tree, trading "verbose but explicit prop chains" for "implicit, hard-to-trace re-renders." It also makes components harder to reuse/test in isolation, since they now silently depend on a specific Provider being present in the tree above them instead of declaring their dependencies via props. The better fix, when the actual issue is "props pass through components that don't use them," is often just containment (children/JSX props) — Context is best reserved for genuinely cross-cutting values, not as a blanket prop-drilling fix.

**Glossary:**
- **Prop drilling** — passing a prop through intermediate components that don't use it, solely to reach a deep descendant.
- **Context** — React's API for making a value available to any descendant of a Provider without passing it through every intermediate component.

**Mental model:**
This probes whether the candidate treats Context as a reflexive cure-all or understands its real cost (re-render fan-out, implicit coupling), and can reach for the simpler containment pattern when that's actually the right fix.

**References:**
- [Passing Data Deeply with Context — React docs](https://react.dev/learn/passing-data-deeply-with-context)

---

### Q4. When a Context value changes, which components re-render? Walk through the internals.

**Question:**
If a Context Provider's value changes, exactly which components re-render — and does it matter how many intermediate components sit between the Provider and a consumer?

**Good answer:**
Every component that reads that context via `useContext` (or the legacy `Context.Consumer`) re-renders when the Provider's `value` changes — regardless of whether the component only cares about one field of that value. Crucially, intermediate components between the Provider and the consumer do **not** re-render on account of the context change alone — context "passes through" them; only components that actually call `useContext` on that context are forced to re-render (though their own children will then re-render too, as a normal consequence of the consumer re-rendering, not because of context propagation itself). This matters for two reasons: first, an unnecessary re-render only touches consumers-and-below, so structuring the tree so non-consumers sit as siblings (not descendants) of consumers limits blast radius; second, if the Provider re-renders for an unrelated reason (e.g. its own parent re-rendered) and creates a brand-new value object each time (`value={{ imageSize }}` inline), every consumer re-renders even though the *logical* value didn't change — because object identity, not deep equality, is what React compares.

**Code example:**
```jsx
// Value re-created every render -> every consumer re-renders on any App re-render
function App() {
  const [isLarge, setIsLarge] = useState(false);
  return (
    <ImageSizeContext value={{ imageSize: isLarge ? 150 : 100 }}>
      {children}
    </ImageSizeContext>
  );
}

// Memoized value -> consumers only re-render when imageSize actually changes
function App() {
  const [isLarge, setIsLarge] = useState(false);
  const imageSize = isLarge ? 150 : 100;
  const value = useMemo(() => ({ imageSize }), [imageSize]);
  return <ImageSizeContext value={value}>{children}</ImageSizeContext>;
}
```

**Follow-up question:**
Given that consumers re-render on any value change with no partial subscription, how would you structure Context to avoid re-rendering components that only need a slice of a large state object?

**Follow-up good answer:**
Split the state into multiple, independently-scoped contexts by how often each slice changes or by which components need it — e.g. a `ThemeContext` that changes rarely, separate from a `CurrentUserContext`, separate from a fast-changing `AnimationFrameContext` — since each `createContext()` call is fully independent and consumers only subscribe to the ones they actually read. For state that changes very frequently and is read by many scattered components, Context alone tends to become a bottleneck regardless of splitting, and the pragmatic move is a dedicated state-management library (Redux, Zustand, Jotai) that supports selector-based, fine-grained subscriptions — i.e. a consumer only re-renders when the specific slice it selected actually changes, not on every store update.

**Glossary:**
- **Provider** — the component (`<Context value={...}>`) that supplies a context's current value to its subtree.
- **Consumer** — any component that reads a context's value via `useContext`.

**Mental model:**
This targets whether the candidate actually understands Context's re-render semantics (all-consumers-or-nothing, no field-level granularity) rather than treating it as a magic global-state solution, and whether they know the standard mitigations (splitting contexts, memoizing the value, or moving to a selector-based store).

**References:**
- [Passing Data Deeply with Context — React docs](https://react.dev/learn/passing-data-deeply-with-context)
- [useMemo — React docs](https://react.dev/reference/react/useMemo)

---

### Q5. `React.memo` is often used to stop a component from re-rendering. What comparison does it actually perform, and when does a memoized component re-render anyway?

**Question:**
Explain exactly what `React.memo` compares by default, and list the cases where a memoized component still re-renders despite unchanged props.

**Good answer:**
By default, `memo` performs a **shallow comparison** of the previous and next props: for each prop, React checks `Object.is(oldProps[key], newProps[key])`. This is reference equality, not deep equality — two different object or array literals with identical contents are considered different (`Object.is({}, {}) === false`), so passing a freshly-created object/array/function as a prop every render defeats `memo` even if its contents never actually change. A memoized component still re-renders when: (1) any prop fails that reference check; (2) its own internal state changes — memoization only concerns props coming from the parent, not the component's own `useState`/`useReducer`; (3) a context it consumes via `useContext` changes, for the same reason. You can override the default comparison with a second argument, a custom `(oldProps, newProps) => boolean` function — but the docs are explicit that if you write one, you must compare *every* prop, including functions, or you risk silently skipping a real update.

**Code example:**
```jsx
const Chart = memo(function Chart({ dataPoints }) {
  // ...
}, arePropsEqual);

function arePropsEqual(oldProps, newProps) {
  return (
    oldProps.dataPoints.length === newProps.dataPoints.length &&
    oldProps.dataPoints.every((p, i) =>
      p.x === newProps.dataPoints[i].x && p.y === newProps.dataPoints[i].y
    )
  );
}
```

**Follow-up question:**
A parent passes an inline arrow function as an `onClick` prop to a `memo`-wrapped child. Does `memo` help here, and if not, what fixes it?

**Follow-up good answer:**
No — an inline arrow function (`onClick={() => doThing(id)}`) is a brand-new function object on every parent render, so `Object.is` on that prop always returns `false`, and the memoized child re-renders every time regardless of `memo`. The standard fix is to wrap the handler in `useCallback` in the parent with the correct dependency array, so the same function reference is reused across renders unless its dependencies actually change — then `memo`'s shallow comparison sees a stable reference and can actually skip the re-render. This is exactly why `useCallback` and `memo` are described as a pair in the docs: `useCallback` on its own does nothing for performance unless something downstream (usually `memo`) is actually relying on referential stability of that value.

**Glossary:**
- **Shallow comparison** — comparing each prop by reference (`Object.is`), not by deep-equality of contents.
- **Referential stability** — a value keeping the same object/function identity across renders when its logical contents haven't changed.

**Mental model:**
This is a classic trap question: many candidates "know" `memo` prevents re-renders but haven't internalized that it's a reference check, so they can't explain why `memo` "isn't working" in the extremely common inline-callback case — this question checks for that depth.

**References:**
- [memo — React docs](https://react.dev/reference/react/memo)
- [useCallback — React docs](https://react.dev/reference/react/useCallback)

---

### Q6. What does `createPortal` do, and how does event propagation work for content rendered through a portal?

**Question:**
Explain what a React portal is, a real use case for one, and how click events on portal content propagate — through the DOM tree or the React tree?

**Good answer:**
`createPortal(children, domNode)` renders `children` into a DOM node that lives outside the calling component's normal DOM position, while keeping that content in the same place in the *React* component tree. The classic use case is a modal/dialog: you want it to escape ancestor CSS like `overflow: hidden` or a low `z-index` stacking context by rendering into `document.body`, but you still want it to behave, logically, as a child of the component that opened it. The key subtlety is event propagation: **events from portal content bubble according to the React tree, not the DOM tree** — so if the portal is logically nested inside a component with an `onClick` handler, a click inside the portal fires that handler, even though the portal's DOM node isn't a descendant of that component's DOM node. This means state, context, and event handling all behave as if the portaled content rendered in its original location; only its physical DOM placement changes.

**Code example:**
```jsx
import { createPortal } from 'react-dom';

function Modal({ children, onClose }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      {children}
    </div>,
    document.body
  );
}
```

**Follow-up question:**
If a portal's content is logically nested inside a parent `<div onClick={handleParentClick}>` in the React tree, but is rendered into `document.body` in the DOM, does clicking inside the portal trigger `handleParentClick`?

**Follow-up good answer:**
Yes. Since portal events bubble through the React tree rather than the DOM tree, a click inside the portal's content is treated as if it originated inside that `<div>`, so `handleParentClick` fires — even though, in the actual DOM, the portal's node isn't nested under that `<div>` at all. This is explicitly called out in React's docs as a deliberate design choice so that portals remain transparent to component logic; it's a common source of confusion (and bugs, e.g. accidental `stopPropagation` needs) for engineers who reason about propagation purely in DOM terms.

**Glossary:**
- **Portal** — a React mechanism to render children into a DOM node outside the parent component's own DOM subtree, without changing where it sits in the React tree.
- **Stacking context** — a CSS concept controlling z-index layering; portals are often used to escape a restrictive one.

**Mental model:**
This checks whether the candidate understands that React's component tree and the browser's DOM tree are two separate structures with independent semantics — event bubbling is a React-tree concept here, which is easy to get backwards if you've only reasoned about DOM APIs.

**References:**
- [createPortal — React docs](https://react.dev/reference/react-dom/createPortal)

---

### Q7. How would you diagnose which component is causing a sluggish UI — what tools do you reach for and what's your process?

**Question:**
A page feels janky when typing into a search box. Walk through your methodology and tooling to find and fix the cause.

**Good answer:**
Start by reproducing and quantifying, not guessing: open the browser's Performance tab (or React DevTools' **Profiler** tab) and record while typing. React DevTools Profiler shows a flamegraph/ranked chart per commit — which components rendered, how long each took (`actualDuration`), and (via the "why did this render" info) why each rendered. Look for two distinct problems: (a) *too many components re-rendering* on each keystroke when they don't need the new state at all (usually a Context or state-lifting issue — the fix is usually splitting state/context or moving state down/local), or (b) *one component rendering expensively* — e.g. an unmemoized expensive computation or a huge unvirtualized list re-rendering completely on each keystroke. For (a), the fix is to isolate state closer to where it's used (e.g. keep the search box's own text in local state, not a context that also drives unrelated components) or split contexts. For (b), reach for `useMemo` around the expensive computation, `memo` on the expensive child so it doesn't re-render if its actual props haven't changed, or list virtualization if it's a long list. After the fix, re-profile the same interaction and confirm `actualDuration` actually dropped — don't just trust that the change "should" help.

**Follow-up question:**
The Profiler shows a component re-rendering with `actualDuration` roughly equal to `baseDuration` on every keystroke. What does that specific relationship tell you, and what would a healthy memoized component's numbers look like instead?

**Follow-up good answer:**
`baseDuration` is React's estimate of how long that subtree would take to render *without* any memoization (worst case); `actualDuration` is how long the last render actually took. If they're roughly equal every time, memoization for that component/subtree either isn't in place or isn't effective — React is doing the full render-cost work on every commit. A well-memoized component, once its props/state/context genuinely stop changing between renders, should show `actualDuration` far lower than `baseDuration` on those unaffected commits (ideally the component doesn't even show up as re-rendered, since `memo` bails out of rendering the subtree entirely) — the gap between the two numbers is effectively the "how much is memoization saving you" signal.

**Glossary:**
- **Flamegraph** — the Profiler's visualization of render time per component per commit.
- **actualDuration / baseDuration** — Profiler metrics: time actually spent rendering vs. the estimated no-memoization worst case, used to judge whether memoization is paying off.

**Mental model:**
This tests whether the candidate has an actual measurement-driven workflow (profile → hypothesize → fix → re-profile) versus guessing and applying `useMemo`/`memo` everywhere reflexively — a trending interview theme across all frontend/backend topics.

**References:**
- [Profiler API — React docs](https://legacy.reactjs.org/docs/profiler.html)

---

### Q8. What is list virtualization, and when is it worth the added complexity?

**Question:**
Explain what list/row virtualization does, why it improves performance, and a case where you would *not* bother with it.

**Good answer:**
List virtualization renders only the rows currently visible in (or near) the viewport, plus a small buffer, instead of mounting DOM nodes for every item in a large dataset. As the user scrolls, rows that leave the visible area are unmounted (or recycled) and new ones are mounted to replace them, so the number of live DOM nodes stays roughly constant regardless of total list length. This matters because DOM node count — not just React's render cost — has real cost: layout, paint, and memory scale with node count, so rendering 10,000 rows directly can make scrolling janky and blow up initial render time, even if each row is cheap individually. Libraries like `react-window` or `react-virtualized` (or `@tanstack/react-virtual`) implement this by measuring/estimating row heights and rendering an absolutely-positioned window into the list. It's not worth the complexity for short lists (dozens of items) — the overhead of virtualization's own bookkeeping and the loss of things like native browser find-in-page or accessibility tooling that expects all content present can outweigh the benefit; it earns its keep once lists are in the hundreds-to-thousands-of-rows range or rows are individually expensive to render.

**Follow-up question:**
Virtualization can break `Ctrl+F` find-in-page and some accessibility/SEO expectations, since off-screen rows aren't in the DOM at all. How would you address that if it's a real requirement?

**Follow-up good answer:**
There's no free fix — you're trading DOM size for content availability, so the mitigation depends on which requirement actually matters. For find-in-page, some teams accept the limitation for internal/authenticated tools where it's not a hard requirement; others fall back to non-virtualized rendering (or a "load more"/pagination pattern instead of true windowing) specifically for pages where search matters. For accessibility, virtualized-list libraries generally support ARIA roles like `role="listbox"`/`aria-rowcount`/`aria-posinset` so screen readers can announce a row's position even though only a subset is mounted — but this needs to be configured deliberately, it's not automatic. For SEO, virtualization is irrelevant for server-rendered/crawled content since crawlers typically don't execute the scroll interactions that would mount later rows — so a virtualized list on a page that needs to be indexed should be paired with server-side rendering of the full content (or a paginated URL structure) rather than relying on client-side windowing alone.

**Glossary:**
- **Virtualization / windowing** — rendering only the visible subset of a large list, recycling DOM nodes as the user scrolls.
- **react-window** — a common lightweight React library implementing list/grid virtualization.

**Mental model:**
This checks whether the candidate reaches for virtualization only when the data justifies it (measured, not assumed) and is aware of its real trade-offs, rather than treating it as a free performance win to slap on every list.

**References:**
- [react-window — GitHub](https://github.com/bvaughn/react-window)

---

### Q9. How does `React.lazy` + `Suspense` enable code-splitting, and what are its hard requirements?

**Question:**
Explain how `React.lazy` reduces initial bundle size, and list its constraints (export shape, where it must be declared, what must wrap it).

**Good answer:**
`React.lazy(loadFn)` takes a function returning a dynamic `import()` promise and returns a React component that, on first render, triggers that import and suspends rendering until it resolves — the bundler (Webpack/Vite/etc.) turns that dynamic `import()` into a separate chunk, so the code for that component isn't included in the initial bundle and is only fetched when the component is actually rendered. This is code-splitting: instead of one large bundle downloaded upfront, you defer the cost of rarely-used or below-the-fold parts of the UI (a settings modal, a rarely-visited admin page, a heavy chart library) until they're needed. Hard requirements: (1) the lazily-imported module must have the component as its **default export** — named exports aren't supported directly (you re-export as default from a small wrapper module if needed); (2) `lazy` components must be **wrapped in `<Suspense fallback={...}>`** somewhere in the tree, or React has no loading UI to show while the chunk downloads — an error is thrown otherwise; (3) `lazy(...)` must be called at **module scope**, not inside a component body, because it caches the underlying promise/component — calling it inside a component recreates the lazy wrapper (and re-triggers the import, resetting any state) on every render.

**Code example:**
```jsx
import { lazy, Suspense } from 'react';

const MarkdownPreview = lazy(() => import('./MarkdownPreview.js'));

function Editor() {
  return (
    <Suspense fallback={<Loading />}>
      <MarkdownPreview markdown={markdown} />
    </Suspense>
  );
}
```

**Follow-up question:**
If the dynamic import fails (e.g. network error, or a stale deployed chunk hash no longer exists on the server), what happens, and how do you handle it gracefully?

**Follow-up good answer:**
A failed `import()` promise rejection propagates as a thrown error during rendering, which is caught by the nearest **Error Boundary** above the `Suspense` boundary — not by `Suspense` itself, which only handles the pending/loading state, not failure. Without an Error Boundary in place, the failure crashes that part of the UI (or the whole app, depending on where the boundary would be). The practical fix is wrapping lazy-loaded sections in both an Error Boundary and a Suspense boundary, with the Error Boundary's fallback offering something actionable — commonly "reload the page," since a very common real-world cause is a stale client trying to fetch a chunk hash from a previous deployment that's since been replaced/removed from the CDN/server after a new release.

**Glossary:**
- **Code-splitting** — dividing a bundle into multiple chunks loaded on demand instead of all upfront.
- **Suspense** — a React component that shows a fallback while its descendants are "suspended" (e.g. waiting on a lazy import or async data).

**Mental model:**
This tests whether the candidate understands `lazy`/`Suspense` as a mechanical contract with real constraints (not just "wrap it and it works"), and specifically whether they know Suspense handles loading but not errors — a very common gap.

**References:**
- [lazy — React docs](https://react.dev/reference/react/lazy)
- [Catching Rendering Errors with an Error Boundary — React docs](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)

---

### Q10. How does the Single Responsibility Principle apply to React component design in practice?

**Question:**
Explain what "single responsibility" means for a React component, with a concrete example of a component that violates it and how you'd split it.

**Good answer:**
For a component, "single responsibility" generally means it should have one reason to change — typically split along the axis of *what it renders* vs. *how it gets/manages its data* vs. *business logic/formatting*. A common violation is a "god component" that fetches data, transforms/validates it, manages several pieces of unrelated UI state (a filter, a modal, pagination), and renders a large chunk of markup, all in one function — a change to the API shape, a change to the filter UI, and a change to the modal's copy all touch the same file and risk breaking each other. The typical split: extract data-fetching/derivation into a custom hook (`useUserList()`) so the component doesn't know *how* data arrives, only that it has it; break the render output into smaller presentational subcomponents (`UserTable`, `UserFilters`, `UserModal`) that each own their own slice of UI state where possible; and keep the parent as a thin composition point that wires the hook's data into the subcomponents. This isn't purity for its own sake — it directly reduces the re-render blast radius (state changes in `UserFilters` no longer force `UserTable` to re-evaluate) and makes each piece independently testable.

**Follow-up question:**
Doesn't decomposing one big component into five smaller ones just move the "god" problem into their shared parent, which now has to wire all of them together?

**Follow-up good answer:**
It can, if the parent ends up owning and threading through every child's state and callbacks directly — that's often the prop-drilling/parent-bloat trap. The fix is to keep as much state as possible local to the component that actually needs it (state colocation) rather than defaulting to lifting everything to the parent "just in case"; only lift state that genuinely needs to be shared or coordinated across the split-out pieces. Where several children legitimately need to share state or actions (e.g. filters affecting the table), a small dedicated hook or a scoped context for just that subtree is usually a better composition point than the top-level page component, which keeps the parent's own responsibility limited to "assemble these pieces" rather than "own everyone's state."

**Glossary:**
- **God component** — a component that has accumulated too many unrelated responsibilities (fetching, business logic, and multiple pieces of UI), making it hard to change safely.
- **State colocation** — keeping a piece of state as close as possible to the component(s) that actually use it, rather than lifting it further up than necessary.

**Mental model:**
This checks whether SOLID is something the candidate can only recite for OOP classes, or whether they can actually translate "single responsibility" into a concrete React refactor and reason about its real payoff (re-render scope, testability) rather than treating it as dogma.

**References:**
- [Thinking in React — React docs](https://react.dev/learn/thinking-in-react)

---

### Q11. What is the compound components pattern, and what problem does it solve that plain prop-based configuration doesn't?

**Question:**
Describe the compound components pattern (e.g. `<Select><Select.Option /></Select>`) and explain what it buys you over a single component with a big prop API.

**Good answer:**
Compound components are a set of components that work together to form one cohesive UI unit, sharing implicit state via Context, while exposing a JSX-based API to the consumer instead of one monolithic prop object. E.g. instead of `<Tabs items={[...]} activeIndex={i} onChange={...} renderTab={...} />` — where every tiny customization needs a new prop or render-prop escape hatch — you write `<Tabs><Tabs.List><Tabs.Tab>A</Tabs.Tab>...</Tabs.List><Tabs.Panels>...</Tabs.Panels></Tabs>`, and the parent `Tabs` component provides shared state (which tab is active) via Context that the child subcomponents (`Tab`, `Panel`) read. This buys flexibility: consumers can reorder, omit, wrap, or interleave arbitrary markup between the pieces, because the composition is expressed in JSX (which React and consumers already know how to manipulate) rather than in an ever-growing bespoke prop schema that the component author has to anticipate every combination of upfront.

**Code example:**
```jsx
const TabsContext = createContext(null);

function Tabs({ children }) {
  const [active, setActive] = useState(0);
  return <TabsContext value={{ active, setActive }}>{children}</TabsContext>;
}
Tabs.Tab = function Tab({ index, children }) {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button aria-selected={active === index} onClick={() => setActive(index)}>
      {children}
    </button>
  );
};
```

**Follow-up question:**
What's the main downside of compound components compared to a single prop-driven component?

**Follow-up good answer:**
The API is more implicit and more fragile to misuse: `Tabs.Tab` only works correctly when rendered somewhere inside a `Tabs` provider, and nothing at the type/compile level stops a consumer from rendering `<Tabs.Tab>` on its own outside that context (you have to guard against a missing context value at runtime, e.g. throwing a clear error). It also couples all the subcomponents to the specific Context/shared-state shape the parent defines, making the pieces less independently reusable than plain, self-contained components would be — and it's a heavier pattern to build and document than "just pass props," so it's overkill for a component that doesn't actually need this much compositional flexibility.

**Glossary:**
- **Compound components** — a set of components sharing implicit state (usually via Context) that together form one configurable unit, exposed as a JSX-composable API.

**Mental model:**
This checks whether the candidate can identify when a growing prop list (a "prop explosion" smell) signals that a compound-components refactor would actually help, versus reaching for it reflexively on components that don't need that level of composition flexibility.

**References:**
- [Passing Data Deeply with Context — React docs](https://react.dev/learn/passing-data-deeply-with-context)

---

### Q12. What is the "render props" pattern, and why has it largely been superseded by hooks?

**Question:**
Explain the render props pattern with an example, and why custom hooks are now generally preferred for the same use case.

**Good answer:**
Render props is a pattern where a component takes a function as a prop (often literally named `render`, or passed as `children`) and calls it with some internal state/data, letting the consumer decide what to render with that data — e.g. a `<Mouse render={({x, y}) => <Cursor x={x} y={y} />} />` that tracks pointer position internally but delegates the actual rendering. Before hooks existed, this (along with higher-order components) was the primary way to share *stateful, cross-cutting logic* between components without duplicating it, since plain function extraction couldn't carry React state/lifecycle. It fell out of favor once hooks arrived because a custom hook (`useMousePosition()`) achieves the same logic-sharing far more directly — the consuming component just calls the hook and gets values back as plain function-call results, without an extra wrapping component, without the "wrapper hell" of nesting several render-prop or HOC providers, and without the indirection of tracing what a `render`/`children`-as-function prop actually receives.

**Code example:**
```jsx
// Render props (older pattern)
<Mouse render={({ x, y }) => <Cursor x={x} y={y} />} />

// Equivalent with a custom hook (modern idiomatic React)
function Cursor() {
  const { x, y } = useMousePosition();
  return <CursorDot x={x} y={y} />;
}
```

**Follow-up question:**
Are there any cases today where render props are still the more natural choice over a custom hook?

**Follow-up good answer:**
Yes — when the logic genuinely needs to control *what gets rendered and where in the tree*, not just supply data/behavior. A custom hook can only return values a component then uses in its own render output; it can't itself decide to wrap children in extra DOM structure or conditionally render entirely different subtrees on the caller's behalf. Libraries providing layout/behavior primitives that need this kind of render-control (certain drag-and-drop libraries, some virtualization libraries, `<Suspense>` itself in spirit) still use a render-prop-like API for that reason. For "just give me some stateful data or a function to call," a hook is simpler and is the idiomatic default today.

**Glossary:**
- **Render props** — a component accepting a function prop that it calls with internal data, delegating the rendering decision to the caller.
- **Wrapper hell** — the nesting mess that results from composing many render-prop or HOC wrappers around a component.

**Mental model:**
This tests historical/architectural awareness — whether the candidate understands *why* the ecosystem moved to hooks (not just "hooks are newer"), and can still recognize the narrow cases where the older pattern remains genuinely useful.

**References:**
- [Reusing Logic with Custom Hooks — React docs](https://react.dev/learn/reusing-logic-with-custom-hooks)

---

### Q13. Compare higher-order components (HOCs) to custom hooks for sharing logic. What are the concrete downsides of HOCs that hooks fix?

**Question:**
What is a higher-order component, and what specific problems with HOCs motivated the move to hooks?

**Good answer:**
A higher-order component is a function that takes a component and returns a new component with extra props/behavior injected — e.g. `withAuth(Profile)` returns a component that checks auth state and renders `<Profile {...props} user={currentUser} />` (or a redirect) instead. HOCs have three concrete, well-documented downsides: (1) **prop name collisions** — if two HOCs both inject a prop called `data`, one silently overwrites the other, and there's no compile-time way to catch it; (2) **wrapper hell** — composing several HOCs (`withAuth(withTheme(withRouter(Component)))`) produces a deeply nested tree of wrapper components, which is painful to read in React DevTools and adds extra component instances with their own (small) overhead; (3) **indirect, hard-to-trace prop origins** — looking at `Profile`'s usage, you can't tell which props come from HOCs vs. its own parent without chasing definitions. Hooks solve all three: a custom hook's return values are named explicitly at the call site (`const { user } = useAuth()` — no collision, no indirection), there's no extra component layer in the tree, and you can call several hooks in one component without any nesting at all.

**Follow-up question:**
Are HOCs now effectively dead in modern React code, or is there still a legitimate reason to reach for one?

**Follow-up good answer:**
They're rare in new application code, but still show up for a specific case hooks can't cover: intercepting/wrapping the *rendering* of an existing component you don't control or don't want to modify — e.g. `React.memo` and `React.forwardRef` (before `ref` became a normal prop) are themselves HOCs, and third-party libraries sometimes still expose a `withStyles(Component)`-style API to inject rendering behavior around an arbitrary component without requiring it to call a specific hook internally. If you're the author of the component and just need to share stateful logic, a hook is almost always the better modern choice.

**Glossary:**
- **Higher-order component (HOC)** — a function that takes a component and returns a new, enhanced component.
- **Prop collision** — two independent enhancers (HOCs) injecting the same prop name, silently overwriting one another.

**Mental model:**
This checks whether the candidate can articulate the *specific, technical* reasons HOCs were superseded (not just "hooks are the new hotness"), which is a good signal for how deeply they actually understand the trade-offs rather than following trends.

**References:**
- [Higher-Order Components — React docs (legacy)](https://legacy.reactjs.org/docs/higher-order-components.html)

---

### Q14. What are error boundaries, what's their hard technical constraint, and what do they explicitly not catch?

**Question:**
Explain what an error boundary is, why it currently must be a class component, and list what kinds of errors it will *not* catch.

**Good answer:**
An error boundary is a component that catches JavaScript errors thrown anywhere in its child tree **during rendering** (also in lifecycle methods and constructors of descendants) and renders a fallback UI instead of letting the error unmount/crash that whole part of the tree. It must implement `static getDerivedStateFromError(error)` (to compute fallback state to render) and/or `componentDidCatch(error, info)` (to log the error, with `info.componentStack` telling you where it happened). As of today, there is no hook equivalent — you cannot write an error boundary as a function component, so teams commonly use the community `react-error-boundary` package to get an `<ErrorBoundary>` component with a function-component-friendly API without hand-writing the class themselves. Error boundaries explicitly do **not** catch: errors inside event handlers (use a plain `try/catch` there instead — event handlers aren't part of the render phase); errors in asynchronous code such as `setTimeout` callbacks or unhandled promise rejections; errors during server-side rendering; or errors thrown inside the error boundary's own code. One notable exception: errors thrown inside a `startTransition` callback (from `useTransition`) *are* caught by boundaries, since transitions are still part of React's render/commit machinery.

**Code example:**
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack);
  }
  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}
```

**Follow-up question:**
A `fetch` call inside a `useEffect` throws an unhandled rejection. Your `<ErrorBoundary>` wraps the component, but the app still crashes/logs an unhandled rejection instead of showing the fallback UI. Why, and how do you fix it?

**Follow-up good answer:**
Because that rejection happens in asynchronous code outside the render phase, which error boundaries are explicitly documented not to catch — the boundary only intercepts errors thrown while React is rendering/committing, not errors from callbacks that fire later on their own schedule. The fix is to handle the async error yourself: wrap the `fetch`/await in a `try/catch` inside the effect (or `.catch()` on the promise) and store the error in component state (e.g. `setError(err)`), then have the component's render output conditionally show error UI (or explicitly `throw error` during render so the boundary above it *can* catch it, since that re-throw now happens synchronously during rendering). Some data-fetching libraries (React Query, SWR) do exactly this internally so that async fetch errors can still surface through a normal error boundary.

**Glossary:**
- **Error boundary** — a class component that catches rendering-phase errors in its subtree via `getDerivedStateFromError`/`componentDidCatch`.
- **componentStack** — the string in `componentDidCatch`'s second argument showing which component tree branch threw.

**Mental model:**
This is a very common practical gotcha — many engineers assume error boundaries are a general try/catch for React apps, and this question checks whether the candidate knows the actual (narrower) scope and the standard workaround for the async case.

**References:**
- [Catching Rendering Errors with an Error Boundary — React docs](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [react-error-boundary — GitHub](https://github.com/bvaughn/react-error-boundary)

---

### Q15. What's the difference between using array index as a `key` vs. a stable ID, and when does index-as-key actually cause a visible bug?

**Question:**
Explain why React warns against (or discourages) using array index as `key`, and construct a concrete scenario where it produces an observable bug, not just a performance issue.

**Good answer:**
`key` tells React which array item a given rendered element corresponds to across renders, so it can match up old and new elements correctly — reuse the existing DOM node and component instance for the "same" logical item, rather than assuming the item at each position is the same as before. If you don't specify a key, React defaults to using the index, but the docs explicitly warn this "often leads to subtle and confusing bugs" once the list can be reordered, or have items inserted/deleted. Concretely: given a list of `<input>` fields, each with its own local text state, keyed by index — delete the first item — every remaining item shifts up one index, so React now believes index `0`'s component is the same one as before (just with new props), and it reuses that component instance, meaning its **internal state does not get reset**; the second item's typed text now visually appears under the first item's label, because the DOM node (and its state) stayed put while the *data* it represents shifted. This isn't just a performance/wasted-render issue — it's a correctness bug where the UI shows the wrong state attached to the wrong logical item.

**Code example:**
```jsx
// Buggy: index as key — deleting item 0 causes item 1's input state
// to appear to "belong" to item 0's row after the shift
{items.map((item, index) => (
  <TextInput key={index} label={item.label} />
))}

// Correct: stable id as key — React tracks each row by its actual identity
{items.map((item) => (
  <TextInput key={item.id} label={item.label} />
))}
```

**Follow-up question:**
If your data doesn't have a natural stable ID (e.g. it's a locally-generated, unsaved list), what should you use as a key instead of the index?

**Follow-up good answer:**
Generate a stable identifier once, at the time the item is created, and store it as part of the item's data — e.g. `crypto.randomUUID()` or an incrementing local counter — rather than deriving a key from the array position or generating a new random key on every render (`key={Math.random()}` is explicitly worse than index, since it forces React to treat every item as brand-new on every render, destroying all state and remounting all DOM nodes every time). The important property is that the same logical item must produce the same key across renders, for as long as that item logically exists; it does not need to be globally unique app-wide, only unique among its immediate siblings in that list.

**Glossary:**
- **key** — a prop that tells React which underlying data item a rendered element corresponds to, used to match elements across renders (add/remove/reorder).

**Mental model:**
This checks whether the candidate can go beyond "index-as-key is bad practice" (a memorized rule) to actually construct and explain the specific correctness bug it causes — showing they understand the reconciliation mechanism, not just the linting rule.

**References:**
- [Rendering Lists — React docs](https://react.dev/learn/rendering-lists)

---

### Q16. `useMemo`/`useCallback` are frequently overused. What's the actual cost of memoizing something that didn't need it, and how do you decide when it's worth it?

**Question:**
Explain the real cost of wrapping a value/function in `useMemo`/`useCallback` unnecessarily, and describe your rule of thumb for when it's actually worth doing.

**Good answer:**
`useMemo`/`useCallback` are not free: React still has to store the previous dependencies and value, compare the dependency array on every render, and the memoized function/value itself is a closure retained in memory as long as the component is mounted — so there's real (if usually small) CPU and memory cost, plus the code becomes harder to read and more error-prone (an incomplete or incorrect dependency array is a very common source of stale-closure bugs). The React docs are explicit that `useCallback` should only be used as a *deliberate* performance optimization, not a default reflex, and per their guidance it helps in specifically two situations: passing a callback/value to a component wrapped in `memo` (so referential stability actually matters to skip a re-render), or when the value/function is itself a dependency of another hook like `useEffect` (to keep that hook's dependency array stable and avoid re-running the effect every render). Outside of those, wrapping "just in case" adds cost and complexity without measurable benefit — especially for callbacks passed to non-memoized children, which re-render regardless of whether the callback reference is stable. The rule of thumb: don't add memoization speculatively; profile first, add it where the Profiler shows a real cost, and verify with the Profiler afterward that it actually helped.

**Follow-up question:**
The React team ships a "React Compiler" that can add memoization automatically. Does that make manual `useMemo`/`useCallback` obsolete?

**Follow-up good answer:**
Not entirely, but it substantially reduces the need for it in ordinary application code — the compiler analyzes component code at build time and inserts the equivalent of `useMemo`/`memo` automatically wherever it determines it's safe and beneficial, without the developer writing it by hand or maintaining dependency arrays. It doesn't eliminate every case: highly dynamic code the compiler can't statically reason about, or genuinely custom comparison logic (a custom `arePropsEqual` for `memo`), still needs to be written manually, and adopting the compiler requires following the "Rules of React" strictly enough for its static analysis to be sound. In practice it shifts manual memoization from "a routine performance habit" to "an escape hatch for cases the compiler can't handle," which is exactly what the docs are already nudging developers toward even without the compiler.

**Glossary:**
- **Stale closure** — a memoized function that continues referencing an old value from a prior render because it was missing from its dependency array.
- **React Compiler** — a build-time tool that automatically inserts memoization equivalent to manual `useMemo`/`memo` usage.

**Mental model:**
This checks whether the candidate treats memoization hooks as a cost/benefit engineering decision (backed by profiling) rather than a habitual "best practice" applied everywhere, which is one of the most common sources of over-engineered, harder-to-maintain React code in real codebases.

**References:**
- [useCallback — React docs](https://react.dev/reference/react/useCallback)
- [React Compiler — React docs](https://react.dev/learn/react-compiler)

---

### Q17. Compare a monolithic single-SPA component architecture to a micro-frontend architecture. What real problem does splitting into micro-frontends solve, and what does it cost you?

**Question:**
When would you actually recommend a micro-frontend architecture over a single React application, and what are the concrete downsides?

**Good answer:**
Micro-frontends split a large frontend into independently built, tested, and deployed pieces — often owned by separate teams — that are composed together at runtime (or build time) into one experience. The real problem this solves is **organizational**, not primarily technical: in a large single SPA with many teams contributing, everyone shares one build pipeline, one dependency-version set, and one release cadence, so a change (or a broken build) from any team can block or slow down every other team, and cross-team code review/ownership boundaries get blurry inside one shared codebase. Splitting lets each team own their slice end-to-end (their own repo, dependency versions, deploy schedule, even their own framework version) and ship independently. The cost is real and often underestimated: duplicated shared dependencies (multiple React copies, multiple copies of a design system) unless carefully de-duplicated, harder cross-app state sharing and navigation (routing, auth, shared UI state now cross a runtime boundary instead of being one in-memory tree), inconsistent UX/performance if teams don't coordinate a shared design system rigorously, and a meaningfully more complex build/deploy/observability setup than a single app. For a small-to-medium team on one codebase, this overhead almost never pays for itself — it's a scaling technique for organizational size, not a default architecture choice.

**Follow-up question:**
A team wants micro-frontends purely to "improve performance" by only loading the part of the app currently needed. Is that the right tool for that specific goal?

**Follow-up good answer:**
Usually not — that specific goal (loading only what's needed) is what **code-splitting** (`React.lazy` + `Suspense`, route-based chunking) already solves within a single application, without any of micro-frontends' organizational/deployment overhead. Micro-frontends address a *team-scaling and independent-deployability* problem; if the actual goal is smaller initial bundles and lazy-loaded routes/features, that's achievable with ordinary code-splitting inside one codebase, which is far simpler to build, test, and reason about. Reaching for micro-frontends for a performance goal alone is a common architecture mistake — it's worth explicitly asking "do we have an organizational/team problem, or a bundle-size problem?" before choosing.

**Glossary:**
- **Micro-frontend** — an architecture splitting a frontend into independently deployable pieces, typically owned by separate teams, composed at runtime.
- **Code-splitting** — breaking one app's bundle into on-demand chunks, an in-app technique distinct from micro-frontends.

**Mental model:**
This tests architectural judgment — whether the candidate can correctly attribute a given problem (org-scaling vs. bundle-size) to the tool that actually solves it, rather than reaching for a trendy, heavyweight architecture for a problem a much simpler technique already covers.

**References:**
- [Code-Splitting — React docs](https://react.dev/reference/react/lazy)

---

### Q18. What does "state colocation" mean, and what's a concrete example of moving state in the *wrong* direction (too high) that hurts performance?

**Question:**
Explain state colocation, and give a concrete example of a component whose state should be moved down (not up) — and what the concrete performance cost of leaving it high is.

**Good answer:**
State colocation means keeping a piece of state in the component closest to where it's actually used/read, rather than defaulting to lifting it to a distant common ancestor "in case something else needs it later." A common anti-pattern: a text input's "is this dropdown open" boolean, or a hover/focus flag for a single row in a large table, gets stored in the top-level page component (because that's where a lot of other state already lives, or because it was copy-pasted from a nearby pattern) instead of the component that actually renders that dropdown/row. The concrete cost: every time that state changes, React re-renders starting from the component that owns it — so if a single row's hover state lives in the page-level component, every keystroke/hover/toggle re-renders the *entire page*, including every other row and every unrelated section, even though only one small piece of UI actually needs to update. Moving that state down into the specific row component means a hover toggle only re-renders that one row; the rest of the page is untouched, without needing any `memo`/`useCallback` workarounds at all — because the re-render blast radius was never large to begin with.

**Follow-up question:**
If ten different rows in that table each need "is this row expanded" state, and the parent needs to know "how many rows are currently expanded" for a summary count, does colocation still apply, or do you have to lift the state after all?

**Follow-up good answer:**
You still colocate the *per-row* boolean in each row component — that doesn't change, since each row's own expand/collapse state doesn't need to live anywhere else for the row itself to work. What you lift is only the derived aggregate the parent actually needs: either (a) have each row still own its own boolean but *report* changes upward via a callback into a `Set`/count kept in the parent (so the parent only re-renders when the aggregate count changes, not on every row's internal render), or (b) if the parent genuinely needs the full per-row state to compute derived things, lift the state as a `Record<rowId, boolean>` in the parent but ensure each row only reads/writes its own entry, and wrap rows in `memo` so a change to one row's entry doesn't re-render sibling rows that didn't change. The principle is: lift only what's actually shared/aggregated, not the whole state, and use referential/structural tricks (memo, selecting a slice) so the parent-level state doesn't force unrelated children to re-render.

**Glossary:**
- **State colocation** — keeping state as close as possible to the component(s) that use it, to minimize re-render scope.
- **Re-render blast radius** — the set of components that re-render as a consequence of a given state update, determined by where that state lives in the tree.

**Mental model:**
This tests whether the candidate can reason about re-render cost purely from "where does this state live in the tree," which is a more fundamental performance lever than any memoization API, and whether they can handle the nuanced follow-up (partial lifting for aggregates) rather than treating colocation as all-or-nothing.

**References:**
- [You Might Not Need an Effect / Sharing State Between Components — React docs](https://react.dev/learn/sharing-state-between-components)

---

### Q19. How do you decide between building a bespoke internal component pattern (e.g. your own compound-components form library) vs. adopting an established library?

**Question:**
A team is debating writing their own compound-components-based form system versus adopting something like React Hook Form or Formik. How would you evaluate that decision?

**Good answer:**
This is a build-vs-buy trade-off, and the right framing is: what specifically does your product need that's *not* generic form behavior, and is that differentiator big enough to justify owning validation, accessibility, performance (uncontrolled-vs-controlled re-render behavior on every keystroke), and edge cases (async validation, field arrays, cross-field validation) yourself long-term? Established libraries have already solved the genuinely hard, easy-to-get-subtly-wrong parts — e.g. React Hook Form uses uncontrolled inputs internally specifically to avoid re-rendering the whole form on every keystroke, which is a non-trivial performance detail a hand-rolled version would need to reinvent. The case for building your own is narrow: you have requirements the ecosystem genuinely doesn't cover well (a very unusual field-composition UI, extremely tight bundle-size constraints where even a small library's size matters, or you're building a component library product where forms *are* the differentiator). For most product teams, the maintenance cost of a bespoke form system (bug fixes, accessibility gaps, keeping up with new field types) outweighs the flexibility gained, and adopting a well-maintained library is the pragmatic default.

**Follow-up question:**
The team already adopted React Hook Form, but now wants to wrap it in their own internal compound-components API (`<Form.Field>`, `<Form.Error>`) instead of exposing the library's API directly to feature teams. Is that a reasonable middle ground?

**Follow-up good answer:**
Yes, and it's a common, reasonable pattern — you get the library's underlying correctness/performance engineering "for free" while presenting a smaller, house-style API surface to your own teams, which can enforce consistency (every field automatically gets the right label/error/spacing markup) and insulate the rest of the codebase from a future library migration (only your wrapper needs to change, not every call site). The trade-off to watch is that your wrapper inevitably narrows what the underlying library exposes — if a feature team eventually needs something the wrapper doesn't surface (a specific validation mode, a less common field type), they either have to extend the wrapper (fine, if it stays generic) or bypass it entirely (a sign the abstraction has started fighting the use case) — so it's worth keeping the wrapper deliberately thin rather than trying to reproduce the entire underlying API.

**Glossary:**
- **Build vs. buy** — the decision to build a custom solution in-house versus adopting an existing library/tool.
- **Uncontrolled inputs (performance angle)** — React Hook Form's internal use of uncontrolled inputs specifically to avoid re-rendering the whole form on every keystroke.

**Mental model:**
This tests engineering judgment beyond pure React mechanics — whether the candidate can reason about total cost of ownership and correctly identify which parts of a "simple-looking" library are actually solving hard, easy-to-underestimate problems (like form re-render performance).

**References:**
- [React Hook Form — Advanced Usage (performance)](https://react-hook-form.com/advanced-usage)

---

### Q20. What's the difference between a "presentational" and a "container" component, and is that split still considered good practice today?

**Question:**
Explain the presentational/container component split, and whether you'd still recommend it in a modern (hooks-based) React codebase.

**Good answer:**
The pattern (popularized before hooks existed) splits components into **presentational** ones — concerned only with how things look, receiving all data and callbacks via props, with no data-fetching or business logic of their own — and **container** ones, which handle data-fetching/state/business logic and render presentational components, passing them the data they need. The goal was separation of concerns and reusability: a presentational `UserCard` could be reused anywhere, tested with plain props, and swapped in Storybook, independent of *how* its data was obtained. Since hooks arrived, this split is applied more loosely: instead of a dedicated "container component" whose only job is data-fetching, that logic usually lives in a custom hook (`useUser(id)`) that any component — including one that also renders markup — can call directly, without needing a wrapper component purely for that purpose. The underlying principle (keep data-fetching/business logic separable from rendering logic, so the rendering part stays easy to test/reuse) is still good practice; the specific mechanism (a dedicated container *component*) is largely superseded by custom hooks, which achieve the same separation with less structural overhead.

**Follow-up question:**
If a team is still writing dedicated container components today instead of custom hooks, is that necessarily wrong?

**Follow-up good answer:**
Not necessarily wrong, but usually more ceremony than needed for the same outcome — a container component adds an extra layer in the tree and an extra file/indirection purely to hold logic that a hook could hold without any extra component at all, and it's less composable (you can call two unrelated hooks in one component trivially; nesting two container components to get the same combined data is clumsier). It can still make sense when the "container" role also needs to control *what renders*, not just supply data — e.g. showing a loading skeleton vs. an error state vs. the real content is naturally expressed as a component making that render decision, even if the actual data-fetching itself is delegated to a hook underneath. In most modern codebases, the pragmatic default is: custom hooks for data/logic, kept as plain (though not dogmatically "pure") components for everything else, without insisting on a strict two-tier container/presentational file structure for every feature.

**Glossary:**
- **Presentational component** — a component concerned only with rendering, receiving data/behavior via props.
- **Container component** — a (largely legacy) pattern of a component whose job is fetching/managing data and delegating rendering to presentational children.

**Mental model:**
This checks whether the candidate can place an older, once-canonical pattern in context — recognizing which part of its intent is timeless (separation of concerns) versus which part was a workaround for a limitation (no hooks) that no longer exists.

**References:**
- [Reusing Logic with Custom Hooks — React docs](https://react.dev/learn/reusing-logic-with-custom-hooks)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=react&tags=component-architecture-and-patterns&autostart=1" | relative_url }})
