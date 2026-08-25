---
layout: default
title: "React Interview — Server Components & SSR Hydration"
---

# React Interview — Server Components & SSR Hydration

This set covers how React renders on the server and comes alive in the
browser: server-side rendering (SSR) vs. client-side rendering (CSR) vs.
static generation, the mechanics of hydration, streaming SSR with Suspense,
and React Server Components (RSC) — the newer model where components run
only on the server and never ship to the client at all.

### Q1. What's the difference between CSR, SSR, and SSG, and what problem does each solve? {#q1}

**Question:**
Walk me through client-side rendering, server-side rendering, and static
site generation. What does each optimize for, and what does each cost you?

**Good answer:**
- **CSR (client-side rendering):** the server sends a near-empty HTML shell
  plus a JS bundle; the browser downloads, parses, and executes React to
  produce the UI. Fast to build, but the user stares at a blank page (or
  spinner) until JS loads and runs — bad Time-to-First-Byte-to-content, bad
  SEO for crawlers that don't execute JS well, bad on slow devices/networks.
- **SSR (server-side rendering):** the server renders the React tree to HTML
  *per request* and sends fully-formed markup immediately. The user sees
  content fast (good First Contentful Paint), and crawlers get real HTML.
  The trade-off: the server does rendering work on every request, and the
  page isn't interactive until React "hydrates" it client-side (attaches
  event handlers, reconciles state) — so there's a gap between visually
  complete and actually interactive.
- **SSG (static site generation):** the HTML is rendered *once*, at build
  time, and served as a static file (optionally from a CDN). Fastest
  possible delivery and cheapest to run (no per-request render cost), but
  the content is only as fresh as the last build — not suitable for
  per-request personalized data without extra machinery (ISR, on-demand
  revalidation, etc.).

The real-world choice is rarely all-or-nothing: most production apps mix
them per-route or per-component (e.g. a marketing page is SSG, a dashboard
is SSR, a settings modal is CSR-only).

**Code example:**
```jsx
// SSR: renders per request, on the server
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/main.js'],
    onShellReady() {
      res.setHeader('content-type', 'text/html');
      pipe(res);
    },
  });
});
```

**Follow-up question:**
If SSR gives you fast First Contentful Paint, why do users still sometimes
complain a server-rendered page "feels janky" right after it loads?

**Follow-up good answer:**
Because the page is visually complete but not yet interactive: the HTML
appeared without any JavaScript running, so clicking a button or typing in
a field does nothing until React finishes downloading, parsing, and
hydrating (attaching event listeners and reconciling state) on the client.
That gap between "looks done" and "is actually usable" is exactly what
Time to Interactive (and the INP web vital) measures, and it's the specific
problem that streaming SSR + selective/progressive hydration and, further,
React Server Components (which reduce how much JS has to hydrate at all)
are designed to shrink.

**Glossary:**
- **FCP (First Contentful Paint)** — time until the browser paints the
  first bit of content.
- **TTI (Time to Interactive)** — time until the page reliably responds to
  user input.
- **Hydration** — attaching React's event handlers and internal state to
  server-rendered HTML so it becomes interactive, without re-creating the DOM.

**Mental model:**
Tests whether the candidate thinks about rendering strategy as a spectrum
of trade-offs (freshness vs. speed vs. server cost vs. interactivity gap)
rather than a single "best" answer — and whether they connect the strategy
to actual user-perceived metrics, not just architecture buzzwords.

**TL;DR:**
CSR trades slow first paint for fast builds, SSR trades server cost for fast content + an interactivity gap, SSG trades freshness for the fastest possible delivery.

**References:**
- [Building Your Own React – Rendering strategies (react.dev conceptual overview via renderToPipeableStream)](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q2. What exactly does `hydrateRoot` do, and what must be true about the server and client output for it to work? {#q2}

**Question:**
Explain what `hydrateRoot` does under the hood. What's the contract between
the server-rendered HTML and the client render for hydration to succeed?

**Good answer:**
`hydrateRoot(container, reactNode)` tells React: "there's already HTML in
this container from the server — don't throw it away and re-render from
scratch, instead walk the existing DOM nodes and attach React's internal
fiber tree and event listeners to them." React's client-side render output
must be **identical** to what the server produced — same elements, same
text, same attributes — because React reuses the existing DOM nodes rather
than re-creating them; it does not diff-and-patch content on hydration the
way a normal re-render does. If the two don't match, that's a hydration
mismatch (see the follow-up).

**Code example:**
```jsx
// client entry point
import { hydrateRoot } from 'react-dom/client';
import App from './App';

hydrateRoot(document.getElementById('root'), <App />);
```

**Follow-up question:**
What actually happens when React detects a hydration mismatch — does it
patch just the mismatched node, or something more drastic?

**Follow-up good answer:**
In development, React detects the mismatch and logs a warning identifying
where server and client output diverged. But React does **not** patch just
that node — per the docs, there's no guarantee mismatched attributes get
patched at all, and if you subsequently call `root.render()` (which is what
happens once React decides it needs to take over), React clears the
server-rendered HTML in that subtree and switches to a full client-side
re-render for it. That's expensive: you paid the cost of SSR (server
render time + bytes over the wire) and then threw away its main benefit
(avoiding a client re-render) anyway. Common causes: branching on
`typeof window !== 'undefined'` inside render, using `Date.now()` or
`Math.random()` directly in JSX, browser-only APIs like
`window.matchMedia` during render, or locale/timezone differences between
server and client. For genuinely unavoidable differences (like a rendered
timestamp), `suppressHydrationWarning` on that one element is the
documented escape hatch — but it only works one level deep and isn't a fix
for the general problem.

**Glossary:**
- **Fiber** — React's internal unit of work / linked-list representation of
  the component tree used for reconciliation.
- **`suppressHydrationWarning`** — a prop that tells React to ignore a
  text/attribute mismatch on that specific element only.

**Mental model:**
Checks whether the candidate understands hydration is not "just re-render
and diff" — it's a distinct, more fragile operation with a strict
server/client output contract, and that violating it has a real
performance cost, not just a console warning.

**TL;DR:**
hydrateRoot reuses existing server-rendered DOM nodes instead of re-rendering, so server and client output must match exactly or React falls back to a full, costly client re-render.

**References:**
- [hydrateRoot – react.dev](https://react.dev/reference/react-dom/client/hydrateRoot)

---

### Q3. What is a React Server Component, and how is it fundamentally different from a component that's merely server-side rendered? {#q3}

**Question:**
People often conflate "my component runs during SSR" with "my component is
a Server Component." What's actually different about React Server
Components?

**Good answer:**
A traditionally-SSR'd component's **code still ships to the browser** — the
server renders it to HTML once for the initial paint, but the component's
JavaScript is also included in the client bundle so it can hydrate and
potentially re-render later. A **React Server Component** runs *only* on
the server (or at build time) and its code **never ships to the client at
all** — the client receives only the rendered output (serialized as a
special RSC payload, not plain HTML), not the component's source. That's
the "zero bundle size" property: a Server Component can import a large
server-only library (a markdown parser, an HTML sanitizer, a DB driver) and
none of those bytes reach the browser. Server Components can also be
`async` and directly `await` a database call or filesystem read — no API
route required — which collapses client → API → DB waterfalls into a
single server-side data fetch. The trade-off: Server Components can't use
`useState`, `useEffect`, or any browser API, because they never run on the
client and have no re-render lifecycle there.

**Code example:**
```jsx
// Server Component — runs only on the server, never bundled for the client
import { marked } from 'marked'; // stays server-side entirely

async function Page({ slug }) {
  const raw = await db.pages.findBySlug(slug); // direct data access, no API layer
  return <article dangerouslySetInnerHTML={{ __html: marked(raw.content) }} />;
}
```

**Follow-up question:**
If a Server Component can't hold state, how do you add a "like" button or
any interactive widget inside a page built mostly of Server Components?

**Follow-up good answer:**
You compose: the interactive piece is written as a **Client Component**
(marked with the `'use client'` directive at the top of its file), and the
Server Component simply renders it, typically passing serializable data as
props. The `'use client'` directive doesn't mean "this component only" — it
marks a **module boundary**: that file and everything it transitively
imports becomes part of the client bundle from that import point onward.
Server Components can still wrap Client Components as children (e.g. pass
server-rendered content through as `children`) without that content itself
becoming client code, because the boundary is about the *import*, not the
render-tree position. What you can't do is pass non-serializable values
(class instances, functions that aren't Server Functions, etc.) as props
across that boundary — only primitives, plain objects/arrays, Dates, and
JSX survive serialization.

**Glossary:**
- **RSC payload** — the serialized description of a Server Component's
  rendered output (and any Client Component references within it) sent to
  the browser, distinct from plain HTML or JSON.
- **`'use client'`** — directive marking a file (and its import subtree) as
  client code.

**Mental model:**
Separates candidates who've memorized "Server Components = SSR" from those
who understand the actual execution/bundling model — this is one of the
most commonly confused React concepts right now precisely because the
naming is close to SSR.

**TL;DR:**
A Server Component's code never ships to the client at all — only its rendered output does — unlike a normal SSR'd component whose JS still gets bundled for hydration.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)
- [use client – react.dev](https://react.dev/reference/rsc/use-client)

---

### Q4. How does streaming SSR with `<Suspense>` change what the user experiences compared to traditional (non-streaming) SSR? {#q4}

**Question:**
Explain how `renderToPipeableStream` combined with `<Suspense>` boundaries
changes the SSR experience versus the older `renderToString`-style approach.

**Good answer:**
Traditional SSR (`renderToString`) is all-or-nothing: the server can't send
*any* HTML until the *entire* tree — including any slow data-fetching
component — has finished rendering. One slow component blocks the whole
response. Streaming SSR splits the tree into a **shell** (everything
outside `<Suspense>` boundaries, which must be ready before anything is
sent) and **suspended content** (wrapped in `<Suspense>`, which can render
later). React sends the shell plus each Suspense boundary's `fallback`
immediately, then — as each boundary's real data resolves — streams down an
inline `<script>` that swaps the fallback for the real content in place,
in the browser, without a full page reload. Boundaries resolve and stream
**out of order**, in whatever order their data becomes ready, not
necessarily top-to-bottom DOM order.

**Code example:**
```jsx
function ProfilePage() {
  return (
    <ProfileLayout>          {/* part of the shell — sent immediately */}
      <ProfileCover />
      <Suspense fallback={<PostsGlimmer />}>
        <Posts />             {/* streamed in later, whenever its data is ready */}
      </Suspense>
    </ProfileLayout>
  );
}
```

**Follow-up question:**
What's the difference between `onShellReady` and `onAllReady`, and when
would you deliberately choose to wait for `onAllReady` instead of streaming
as fast as possible?

**Follow-up good answer:**
`onShellReady` fires as soon as the shell (non-suspended part) is done —
this is what you use for normal user-facing requests, because you want to
start streaming to the browser as early as possible and let Suspense
boundaries fill in progressively. `onAllReady` fires only once *everything*,
including all suspended content, has finished rendering — you deliberately
wait for this for **crawlers and static-generation use cases** (e.g.
detecting a bot user-agent, or prerendering for SSG) where you need one
complete HTML snapshot rather than a stream with placeholder fallbacks,
because a crawler that doesn't execute the follow-up `<script>` swaps would
otherwise index the fallback content instead of the real content.

**Glossary:**
- **Shell** — the part of the tree outside any `<Suspense>` boundary; must
  finish before any response is sent.
- **Out-of-order streaming** — Suspense boundaries resolve and are sent in
  the order their data becomes ready, not document order.

**Mental model:**
Probes whether the candidate understands Suspense-driven SSR as a real
scheduling/streaming mechanism (with concrete callback semantics) rather
than a vague "it shows a spinner" description.

**TL;DR:**
Streaming SSR sends a ready shell immediately and streams each Suspense boundary's real content in as it resolves, instead of blocking the whole response on the slowest piece.

**References:**
- [renderToPipeableStream – react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q5. How do you diagnose a hydration mismatch in production when you can't just read the dev-mode console warning? {#q5}

**Question:**
A user reports a page "flickers" or that clicks sometimes don't register
right after load, and you suspect a hydration mismatch — but this is a
production build where the detailed dev warnings are stripped. How do you
actually diagnose it?

**Good answer:**
Practical methodology:
1. **Reproduce in a dev build first.** Hydration mismatch warnings (with a
   diff of server vs. client output) only appear in development builds —
   so the fastest path is reproducing the same route/data locally or on a
   staging environment running an unminified build, where React will print
   exactly which text/attribute differed.
2. **Compare server output vs. client output directly.** curl the SSR'd
   HTML for the exact URL and compare it against what the client would
   render for the same props/data (e.g. render the component in isolation
   with the same server-fetched data) — look specifically for things that
   differ between environments: timezone/locale formatting, `Date.now()`
   or `Math.random()` in render, conditional rendering keyed on
   `typeof window`, and browser extensions injecting DOM before hydration
   runs (a very common false-positive source).
3. **Check for the visual symptom that correlates with a full client
   re-render**, not just a text warning: since React clears and re-renders
   the whole mismatched subtree, a mismatch can present as a "flash" where
   content briefly disappears and reappears right after load — that flash
   itself is a diagnostic signal.
4. **Use React DevTools' Components/Profiler tab** on the live page to
   confirm whether a suspiciously large commit happens immediately after
   hydration — a full-subtree re-render right after mount is consistent
   with a mismatch-triggered fallback to client rendering.

**Code example:**
```jsx
// A classic silent source of mismatches — different render paths per environment
function Banner() {
  const isClient = typeof window !== 'undefined'; // ❌ differs between server & client
  return isClient ? <ClientOnlyBanner /> : null;
}
```

**Follow-up question:**
Why is `typeof window !== 'undefined'` inside a component's render logic
specifically dangerous for hydration, and what's the correct pattern
instead?

**Follow-up good answer:**
It's dangerous because it changes the *render output itself* based on
environment: on the server, `window` is undefined, so the branch renders
one thing; on the client during hydration, `window` exists, so the same
component instantly renders something different — guaranteeing a mismatch
on the very first hydration pass, not just eventually. The documented
pattern is to render the **same output on both passes** and defer the
client-only behavior to *after* mount, using `useEffect` (which only runs
on the client, after hydration completes) to update local state and
trigger a *subsequent*, normal client-side re-render — that second render
is allowed to differ because it's not part of the hydration match
requirement, only the very first client render is.

**Glossary:**
- **Hydration mismatch** — divergence between server-rendered HTML and
  what the client's first render would produce for the same tree.

**Mental model:**
Tests real production debugging instinct, not just "know the rule" —
knowing *why* a mismatch happens is different from knowing how to track
one down when the helpful dev warning isn't available.

**TL;DR:**
Reproduce with a dev build to see the exact diff, then correlate a post-load content flash with a full client re-render of the mismatched subtree.

**References:**
- [hydrateRoot – react.dev](https://react.dev/reference/react-dom/client/hydrateRoot)

---

### Q6. Why does React Server Components' "zero bundle size" property matter for performance, beyond just download size? {#q6}

**Question:**
Server Components ship zero bytes of their own code to the client. Walk me
through why that matters for real-world performance, not just in theory.

**Good answer:**
It compounds across several performance dimensions, not just network
transfer:
- **Less JS to download** — obviously smaller bundle, faster on slow
  networks.
- **Less JS to parse and compile** — even cached/local JS still costs CPU
  time to parse and JIT-compile before it can run; this shows up
  disproportionately on low-end mobile devices.
- **Less JS to execute during hydration** — hydration walks the whole
  client component tree and attaches handlers; fewer client components
  means less hydration work, which directly reduces Time to Interactive
  and improves INP, because the main thread is blocked for less time.
- **Heavy dependencies never reach the client at all** — a markdown
  parser, syntax highlighter, or PDF library used only to produce static
  output can live entirely in a Server Component and contribute nothing to
  the client bundle, versus a traditional SSR setup where that dependency
  ships to the browser even though the browser never uses it after the
  initial paint.

The net effect: Server Components don't just make pages *load* faster,
they reduce the amount of main-thread work the browser has to do to make
the page *usable*.

**Follow-up question:**
Does this mean you should convert every component to a Server Component
for the performance win? What's the actual constraint that stops you?

**Follow-up good answer:**
No — the constraint is that Server Components fundamentally cannot be
interactive: no `useState`, no `useEffect`, no event handlers, no browser
APIs, because they run once on the server and are never re-rendered on the
client. Anything that needs to respond to user input, hold local UI state,
or use a browser-only API (canvas, `localStorage`, media queries) must be a
Client Component. The practical strategy is "Server Components by default,
Client Components at the leaves that actually need interactivity" — push
the `'use client'` boundary as far down the tree as possible so only the
genuinely interactive pieces (and their dependencies) end up in the client
bundle, rather than marking a large ancestor component as client and
dragging its whole subtree along with it.

**Glossary:**
- **INP (Interaction to Next Paint)** — a Core Web Vital measuring
  responsiveness to user interactions.

**Mental model:**
Checks that the candidate connects an architectural feature to concrete
Core Web Vitals impact, and understands the constraint isn't arbitrary —
it's a direct consequence of Server Components never running on the client.

**TL;DR:**
Zero bundle size means less JS to download, parse, and hydrate, which directly cuts Time to Interactive and improves INP — not just a smaller download.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q7. What is the "waterfall" problem in data fetching, and how do Server Components address it differently than `useEffect`-based fetching? {#q7}

**Question:**
Explain the classic client-side data-fetching waterfall problem, and how
Server Components change the picture.

**Good answer:**
In a typical CSR/SPA `useEffect`-based data-fetching pattern, a parent
component renders first, *then* its `useEffect` fires and kicks off a
fetch; a child component that needs data can't start its own fetch until
it mounts, which usually only happens after the parent has data and
renders the child. The result: fetches happen in a serial chain
(parent request → parent response → child request → child response) even
though the child's data doesn't actually depend on the parent's — each hop
adds a full round-trip of latency, and the waterfall compounds with tree
depth.

With **Server Components**, because rendering happens on the server (close
to or in the same network as the data source) and components can be
`async` and `await` directly, sibling/child Server Components can issue
their data requests independently and concurrently — the server can start
`Author`'s DB query without waiting for `Note`'s to resolve first, since
neither is gated behind a client mount lifecycle. This collapses many
network hops (client → API → DB per component) into direct server-side
data access, and removes the mount-order serialization that
`useEffect`-based fetching imposes.

**Code example:**
```jsx
// Server Components: independent data fetches, not serialized by mount order
async function Note({ id }) {
  const note = await db.notes.get(id);
  return (
    <div>
      <Author id={note.authorId} /> {/* fetches concurrently, not after Note resolves */}
      <p>{note.text}</p>
    </div>
  );
}

async function Author({ id }) {
  const author = await db.authors.get(id);
  return <span>By: {author.name}</span>;
}
```

**Follow-up question:**
Server Components still can't help once you're inside a Client Component
subtree that needs its own data. What's the recommended pattern there to
avoid reintroducing a waterfall?

**Follow-up good answer:**
The documented pattern is to **start the fetch on the server and pass the
unresolved Promise down as a prop** to the Client Component, rather than
having the Client Component kick off its own fetch after mount. The Client
Component then reads that promise with the `use()` API inside a
`<Suspense>` boundary, which suspends only that part of the tree until the
promise resolves — critical content (already-available data) can render
immediately while the slower promise streams in later, without forcing the
Client Component to wait until it mounts to even *start* its request.

**Glossary:**
- **Waterfall** — a chain of sequential network requests where each one
  only starts after the previous one resolves, even when they're logically
  independent.
- **`use()`** — a React API that lets components read the value of a
  Promise (or context) directly, integrating with Suspense.

**Mental model:**
Checks whether the candidate understands *why* Server Components help with
data fetching specifically (removing mount-order serialization and
network hops) rather than treating it as a vague "it's faster" claim.

**TL;DR:**
Server Components can fetch data concurrently during render (no mount-order dependency), collapsing the client-side useEffect waterfall into parallel server-side awaits.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q8. What is `useId` for, and why can't you just use an incrementing counter or `Math.random()` to generate unique DOM ids in a server-rendered app? {#q8}

**Question:**
You need unique `id` attributes to wire up `aria-describedby` between a
label and a hint. Why does React provide a dedicated `useId` hook for this
instead of you just incrementing a counter?

**Good answer:**
`useId` exists specifically to keep IDs **consistent between the server
render and the client hydration render**. A naive global incrementing
counter breaks under hydration because the *order* components are
processed on the client during hydration isn't guaranteed to exactly match
the server's render order in all cases (e.g. with streaming/out-of-order
Suspense resolution) — so `nextId++` on the client can produce a different
number than the server did for the "same" logical component, causing a
hydration mismatch on that attribute. `Math.random()` is even worse: it's
non-deterministic by definition, guaranteed to differ between server and
client. `useId` instead derives the ID from the component's **position in
the component tree** (its "parent path"), which is identical on server and
client because the tree structure itself is identical — so the generated
ID matches regardless of rendering/hydration order.

**Code example:**
```jsx
import { useId } from 'react';

function PasswordField() {
  const hintId = useId();
  return (
    <>
      <input type="password" aria-describedby={hintId} />
      <p id={hintId}>Must be at least 12 characters.</p>
    </>
  );
}
```

**Follow-up question:**
Should you use `useId` to generate `key` props when rendering a list?

**Follow-up good answer:**
No — the docs explicitly call this out. `key` needs to be **stable and
tied to the actual data item** across re-renders (so React can tell which
array item is which when the list is reordered, filtered, or has items
added/removed), whereas `useId` is derived from the component's position in
the tree. Using it for keys would tie the "identity" of a list item to
where it happens to render rather than to the underlying data, which
breaks exactly the kind of reordering/insertion cases `key` exists to
handle correctly. Keys should come from the data itself (a database ID, a
UUID already in the record) — `useId` is only for generating consistent
DOM-facing IDs like `aria-*` attribute targets, not for React's internal
list reconciliation.

**Glossary:**
- **`key`** — a special prop React uses to match array items across
  renders for reconciliation; unrelated to `useId`'s purpose.

**Mental model:**
A very common "gotcha" question — checks whether the candidate actually
read why `useId` exists (hydration-safe determinism) versus just knowing
"it makes unique ids," which leads directly to the classic mistake of
reaching for it as a key generator.

**TL;DR:**
useId derives ids from stable tree position so they match between server and client renders, unlike a counter or Math.random() which can diverge and break hydration.

**References:**
- [useId – react.dev](https://react.dev/reference/react/useId)

---

### Q9. What are Server Functions (formerly "Server Actions"), and how do they support progressive enhancement? {#q9}

**Question:**
Explain what a Server Function is, how a Client Component invokes one, and
what "progressive enhancement" means in this context.

**Good answer:**
A **Server Function** is an async function marked with the `'use server'`
directive that runs only on the server but can be called directly from
Client Component code (or passed as a Server Component prop) as if it were
a local function — React handles serializing the call and its arguments
into a request under the hood. The most common usage is passing one
directly to a `<form action={...}>` prop: React 19's form integration
calls the Server Function on submit and automatically resets the form on
success.

"Progressive enhancement" here means the form **works even before
JavaScript has loaded/hydrated**: because `action` on a `<form>` is a real
HTML attribute concept, a form wired to a Server Function can be submitted
as a normal HTML form POST if JS isn't ready yet, and React replays the
interaction once hydration completes — so the feature degrades gracefully
instead of being fully broken pre-hydration.

**Code example:**
```jsx
// actions.js
'use server';
export async function updateName(formData) {
  const name = formData.get('name');
  if (!name) return { error: 'Name is required' };
  await db.users.updateName(name);
}
```
```jsx
// client component
'use client';
import { updateName } from './actions';

function ProfileForm() {
  return (
    <form action={updateName}>
      <input type="text" name="name" />
      <button type="submit">Save</button>
    </form>
  );
}
```

**Follow-up question:**
How does `useActionState` improve on just passing the Server Function
directly to `action`?

**Follow-up good answer:**
`useActionState(action, initialState, permalink?)` wraps the Server
Function and gives you back `[state, submitAction, isPending]`: `state` is
the most recent value returned by the action (so you can show, e.g., a
validation error the server returned), `isPending` lets you disable the
form/show a spinner while the submission is in flight, and the optional
third argument (`permalink`) is a fallback URL the browser can navigate to
via a real form submission if JavaScript hasn't loaded yet at all — a
stronger form of progressive enhancement than the bare `action={fn}`
pattern. React also automatically replays form submissions that happened
before hydration finished, once the app becomes interactive, so an
impatient user who submits early doesn't lose their input.

**Glossary:**
- **`'use server'`** — directive marking a function as a Server Function,
  callable from the client but executed only on the server.
- **Progressive enhancement** — building a feature so its core function
  works with minimal capability (plain HTML) and gets enhanced when more
  capability (JS) is available.

**Mental model:**
Tests whether the candidate can explain a fairly new API (React 19 Server
Functions) in terms of the actual problem it solves (form usability during
the hydration gap), not just "it's how you do mutations now."

**TL;DR:**
Server Functions ('use server') let a form submit as a real HTML POST before JS loads, then get replayed once hydration completes — graceful degradation, not an all-or-nothing feature.

**References:**
- [Server Functions – react.dev](https://react.dev/reference/rsc/server-functions)

---

### Q10. When would you deliberately choose NOT to adopt React Server Components / an SSR framework, and stick with a plain client-rendered SPA? {#q10}

**Question:**
Given everything RSC and streaming SSR give you, when is a plain
client-side-rendered SPA still the right call?

**Good answer:**
Several legitimate cases:
- **No meaningful SEO/first-paint requirement** — an internal admin tool or
  authenticated dashboard behind a login wall gets little benefit from SSR
  (crawlers never see it, and users already accept a loading state after
  auth) while paying real complexity cost.
- **Team/infra unfamiliarity or hosting constraints** — SSR/RSC requires a
  Node (or compatible) server runtime and a framework that supports
  streaming responses; a team shipping to a purely static host, or without
  server-ops capacity, takes on real operational cost adopting it.
- **Highly interactive, app-like UIs with little static content** — e.g. a
  canvas-based design tool or a real-time collaborative editor is
  overwhelmingly client-state-driven; there's little "content" to
  server-render, so the SSR/hydration machinery adds complexity without a
  proportional benefit.
- **Server Components' serialization constraint becomes a real friction
  point** — apps that pass a lot of non-serializable data (class instances,
  functions, complex client-only state) across what would be server/client
  boundaries fight the model constantly.

The decision isn't "RSC is strictly better" — it's a genuine trade of added
architectural/infra complexity for faster initial paint, smaller client
bundles, and better SEO, and that trade isn't worth it for every app.

**Follow-up question:**
Suppose a team adopts RSC mainly for the bundle-size win, but their app is
almost entirely behind auth with no SEO need. Is that still a good reason?

**Follow-up good answer:**
It can be, but it's a narrower justification than the marketing pitch
suggests: bundle size and hydration cost benefits apply regardless of
SEO/auth status, since they're about client-side JS parse/execute cost
(directly affecting TTI/INP), not about crawlers. If the app has heavy
server-only dependencies (a large data-transformation or formatting
library used only to produce output) that a Server Component could keep
off the client entirely, that's a legitimate, SEO-independent performance
win. But the team should weigh it against the added complexity (a server
runtime, the client/server serialization boundary, new debugging surface)
versus simpler alternatives — like code-splitting/lazy-loading those heavy
dependencies in a plain CSR app — that might capture most of the bundle-
size benefit with less architectural change.

**Glossary:**
- **ISR (Incremental Static Regeneration)** — a middle ground between SSG
  and SSR where static pages are regenerated in the background after
  deploy, common in frameworks built on React's SSR/RSC primitives.

**Mental model:**
This is explicitly a trade-off/judgment question — it's testing whether
the candidate can argue against the technology they just explained
enthusiastically, which is a strong signal of real production experience
versus rehearsed advocacy.

**TL;DR:**
RSC/SSR complexity only pays off when SEO/first-paint or heavy server-only dependencies matter; behind-auth, highly interactive, or infra-constrained apps can stay plain CSR.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q11. Explain the RSC payload — what actually gets sent over the wire for a Server Component tree, and how is it different from HTML or JSON? {#q11}

**Question:**
When a Server Component renders, what format is the result sent to the
client in, and why isn't it just HTML?

**Good answer:**
The Server Component render output is serialized into a special-purpose
streaming format (commonly called the "RSC payload" or "Flight" format)
rather than plain HTML or generic JSON. It needs to represent more than
HTML can: it encodes not just the rendered tree structure, but also
**references to Client Components** that need to be hydrated at specific
points in that tree (so the client knows "mount this specific Client
Component here, with these serialized props"), and it can stream
incrementally as async Server Components resolve — similar in spirit to
how streaming SSR sends Suspense boundaries out of order, but carrying
component references instead of just HTML. Plain HTML can't express "here's
a placeholder for a Client Component with these props" — HTML has no
concept of component identity — and generic JSON has no built-in
streaming/incremental-resolution semantics for a tree with async holes in
it. When this payload is used for an *initial* page load, a framework
typically also renders it further down to actual HTML (via SSR) for first
paint, with the RSC payload used again on the client to reconcile/hydrate
and for subsequent navigations without a full page reload.

**Follow-up question:**
Why does this matter for a navigation *within* an already-loaded RSC app
(e.g. clicking a link), as opposed to the very first page load?

**Follow-up good answer:**
On a subsequent in-app navigation, the framework can fetch just the updated
RSC payload for the new route (server re-renders the changed Server
Components and streams back their payload) instead of doing a full HTML
document reload or re-fetching the entire client JS bundle. The client
already has the Client Component code it needs (from the initial load) and
just needs the new tree structure plus any new server-computed data —
this is conceptually similar to how a traditional SPA fetches JSON from an
API on navigation, except the "API response" here already knows how to
slot into specific points of the React tree, including which parts are
Server-rendered content versus references to already-loaded Client
Components, avoiding a redundant full JS payload.

**Glossary:**
- **Flight format** — the informal/internal name sometimes used for
  React's RSC serialization protocol.

**Mental model:**
A deeper internals question — checks whether the candidate has looked
past the "Server Components send output instead of code" one-liner and
understands *why* a new wire format was necessary rather than reusing
HTML or JSON.

**TL;DR:**
The RSC payload is a streaming format encoding both server-rendered content and references to Client Components to mount, which plain HTML or JSON can't express.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q12. What's a common pitfall when Server Components pass props to Client Components, and why does it happen? {#q12}

**Question:**
A developer tries to pass a class instance or a non-Server function as a
prop from a Server Component to a Client Component and gets a runtime
error. Why?

**Good answer:**
Because everything crossing the server → client boundary has to be
**serialized** — turned into a representation that can travel over the
RSC payload/network and be reconstructed on the other side. Only a
specific set of value types survive that: primitives (strings, numbers,
booleans, `null`/`undefined`), plain objects and arrays, `Date`s,
serializable iterables, JSX elements, and references to Server Functions
(marked `'use server'`) — not arbitrary class instances (their prototype
chain/methods can't be reconstructed), not regular closures/functions
(there's no way to "ship" arbitrary executable client code that wasn't
already part of the client bundle), and not things like `Map`/`Set` unless
the framework's serialization explicitly supports them. This is the same
fundamental constraint as `postMessage`/structured-clone in other
browser/worker contexts — you can't send "live" JS objects across a
process/environment boundary, only data.

**Code example:**
```jsx
// ❌ Server Component — throws, CustomClass instance can't be serialized
<ClientWidget handler={new CustomClass()} />

// ✅ Serializable data crosses the boundary fine
<ClientWidget data={{ id: 1, name: 'Ada' }} />
```

**Follow-up question:**
Given this constraint, how do you pass "behavior" (not just data) from a
Server Component into a Client Component, if you can't pass an arbitrary
function?

**Follow-up good answer:**
Two supported patterns: (1) pass a **Server Function** (`'use server'`) —
React specifically supports serializing a *reference* to a Server Function
across the boundary, so the Client Component can call it and React routes
the call back to the server, even though it's technically "a function" —
this works because it's not an arbitrary closure, it's a well-known,
server-registered reference; or (2) don't pass behavior across the
boundary at all — define the interactive logic *inside* the Client
Component itself (since it already has `'use client'` and can use
`useState`/event handlers locally), and only pass serializable data down
from the Server Component as input to that already-client-side behavior.

**Glossary:**
- **Serialization boundary** — the server/client edge where only
  serializable values (plus Server Function references) can cross.

**Mental model:**
Tests understanding of *why* the RSC model has this constraint (it's not
an arbitrary framework limitation, it follows directly from "Server
Component code never ships to the client") rather than just memorizing
"you can't pass functions."

**TL;DR:**
Only serializable values (primitives, plain objects/arrays, Dates, Server Function references) can cross the server→client boundary — arbitrary class instances or closures can't.

**References:**
- [use client – react.dev](https://react.dev/reference/rsc/use-client)
- [Server Functions – react.dev](https://react.dev/reference/rsc/server-functions)

---

### Q13. How does `<Suspense>` fallback content interact with SEO/crawlers during streaming SSR — is there a risk of a crawler indexing the loading skeleton instead of the real content? {#q13}

**Question:**
With streaming SSR, the browser initially receives Suspense fallbacks
(loading skeletons) before the real content streams in via injected
scripts. Could a search engine crawler end up indexing the skeleton
instead of the actual content?

**Good answer:**
It's a real risk if handled naively, which is exactly why
`renderToPipeableStream` exposes both `onShellReady` and `onAllReady` as
separate hooks. For a normal user request you want `onShellReady` (start
streaming immediately, let fallbacks fill in progressively) for the best
perceived performance. But for a crawler — detected via user-agent
sniffing, or more robustly via a known bot-detection mechanism — the
recommended pattern is to instead wait for `onAllReady` and only then send
the response, so the crawler receives one complete HTML document with the
*actual* content already resolved, never a fallback. Whether a given
crawler executes the follow-up streaming `<script>` swaps reliably enough
to see the real content is not something you want to depend on for
critical SEO content — waiting for `onAllReady` sidesteps that risk
entirely for bot traffic while regular users still get the faster
streaming experience.

**Code example:**
```js
const { pipe } = renderToPipeableStream(<App />, {
  bootstrapScripts: ['/main.js'],
  onShellReady() {
    if (!isBot(req)) {
      res.statusCode = 200;
      pipe(res); // stream progressively for real users
    }
  },
  onAllReady() {
    if (isBot(req)) {
      res.statusCode = 200;
      pipe(res); // send one complete document for crawlers
    }
  },
});
```

**Follow-up question:**
Is there a downside to always using `onAllReady` for everyone, to avoid
this complexity entirely?

**Follow-up good answer:**
Yes — you'd give up the core benefit of streaming SSR. `onAllReady` waits
for the *slowest* Suspense boundary's data to resolve before sending
anything, which means a user with a fast-loading shell but one slow
data-dependent widget would see nothing at all until that slow widget is
also ready — you're back to the traditional SSR problem where one slow
component blocks the entire response, just implemented with the streaming
API instead of `renderToString`. The whole point of the shell/fallback
split is to let fast content reach the user immediately while slow content
streams in later; routing everyone through `onAllReady` defeats that and
should be reserved specifically for the crawler/static-generation case
where a single complete snapshot is the actual requirement.

**Glossary:**
- **Bot detection** — identifying crawler/automated traffic (commonly via
  user-agent, though more robust methods exist) to serve it differently.

**Mental model:**
A practical, production-shaped question connecting a specific API
(`onShellReady`/`onAllReady`) to a real business concern (SEO), testing
whether the candidate can reason about when to trade the "fast" path for
the "complete" path.

**TL;DR:**
Waiting for onAllReady instead of onShellReady sends crawlers one complete document instead of a streaming shell + fallbacks, avoiding indexing loading skeletons.

**References:**
- [renderToPipeableStream – react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q14. What's "selective hydration," and how does it let a page become interactive faster than hydrating top-to-bottom? {#q14}

**Question:**
Explain selective hydration — how does React decide what to hydrate first
when a page has multiple Suspense boundaries?

**Good answer:**
Without selective hydration, a naive approach would hydrate the whole tree
in one synchronous pass, in document order, regardless of what the user
actually cares about. With Suspense boundaries in play, React can hydrate
**each boundary independently** as its content becomes available, and —
critically — React can **prioritize hydrating whichever boundary the user
is actually trying to interact with**, even if it's not the first one in
document order. If a user clicks inside a Suspense boundary that has
streamed in but not yet hydrated, React treats that as a signal to
prioritize hydrating *that* boundary next, ahead of others that streamed
in earlier but haven't been touched — so perceived interactivity tracks
actual user intent rather than a fixed top-to-bottom order. This directly
attacks the TTI problem: instead of "the whole page becomes interactive at
once, after everything hydrates," specific parts become interactive as
soon as their own hydration work is done, and the part the user is poking
at gets bumped to the front of the queue.

**Follow-up question:**
Does selective hydration mean you no longer need to think about how many
Suspense boundaries to use, or is there still a design trade-off?

**Follow-up good answer:**
There's still a real trade-off. Too few/coarse Suspense boundaries and
you lose the granularity that makes selective hydration valuable — a huge
single boundary hydrates (and streams) as one unit, so you're back to a
big blocking chunk of work. Too many/fine-grained boundaries add overhead
(each boundary has bookkeeping cost, and excessive fragmentation can hurt
layout stability/CLS as many small pieces pop in independently) without
a proportional interactivity benefit. The practical guidance is to draw
boundaries around genuinely independent, meaningfully-sized units of the
UI — a widget or section, not every individual element — so each boundary
represents a real, useful "streams and becomes interactive together" unit.

**Glossary:**
- **CLS (Cumulative Layout Shift)** — a Core Web Vital measuring unexpected
  layout movement, which can worsen with excessive independently-streamed
  fragments.

**Mental model:**
Checks understanding of Suspense-driven hydration as demand-aware, not
just "it splits work into chunks" — and whether the candidate can reason
about the granularity trade-off rather than treating "more Suspense
boundaries" as an unconditional win.

**TL;DR:**
Selective hydration lets React hydrate whichever Suspense boundary the user actually interacts with first, rather than a fixed top-to-bottom order.

**References:**
- [renderToPipeableStream – react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q15. What role does `suppressHydrationWarning` play, and why is it explicitly documented as "not a fix"? {#q15}

**Question:**
When and how should you use `suppressHydrationWarning`? Why does the
documentation warn against overusing it?

**Good answer:**
`suppressHydrationWarning={true}` on an element tells React to not warn
(and not attempt any mismatch handling) about a text/attribute difference
on **that specific element**, one level deep — it's meant for genuinely
unavoidable, intentional server/client differences, the canonical example
being a rendered timestamp (`new Date().toLocaleString()`) that will
legitimately differ by the milliseconds between server render time and
client hydration time. It's explicitly *not* a general-purpose fix for
mismatches, because: (1) it only suppresses the *warning* — React still
does not guarantee it patches the mismatched content correctly, so if the
difference is more than cosmetic (e.g. actually different data, not just a
timestamp), you can end up with incorrect content silently rendered; and
(2) it only works one level deep, so it can't paper over a structural
mismatch (different number/type of child elements) — only a leaf
text/attribute difference on the element it's applied to.

**Code example:**
```jsx
// Legitimate use: a genuinely-expected, cosmetic server/client difference
<span suppressHydrationWarning={true}>
  {new Date().toLocaleTimeString()}
</span>
```

**Follow-up question:**
If a team starts sprinkling `suppressHydrationWarning` across many
components to make console warnings go away, what's the actual risk
they're taking on?

**Follow-up good answer:**
They're masking the diagnostic signal for real bugs, not just cosmetic
ones. Since the prop suppresses the warning regardless of *why* the
mismatch exists, a genuine bug (e.g. conditional rendering based on
`typeof window`, or a data-fetching race that produces different content
server vs. client) would stop showing up in the console the same way a
harmless timestamp difference does — the team loses visibility into
whether hydration is actually clearing and re-rendering that subtree
client-side (with its associated performance cost and potential for
incorrect content) versus genuinely succeeding. It converts a loud, useful
signal into silence, which is the opposite of what you want when the whole
point of the warning is to catch bugs that degrade performance and
correctness.

**Glossary:**
- **`suppressHydrationWarning`** — a React prop suppressing the mismatch
  warning for one element's direct text/attribute content only.

**Mental model:**
Checks whether the candidate treats a "make the warning go away" API as a
targeted escape hatch versus a blanket fix — a distinction that matters a
lot in code review.

**TL;DR:**
suppressHydrationWarning only silences the warning on one element one level deep — it doesn't guarantee correct patching and can mask real bugs if overused.

**References:**
- [hydrateRoot – react.dev](https://react.dev/reference/react-dom/client/hydrateRoot)

---

### Q16. How would you use the browser's Performance/Network panel (or React DevTools) to confirm whether a slow-feeling SSR page is bottlenecked on server render time vs. client hydration time? {#q16}

**Question:**
A server-rendered page feels slow. Walk me through how you'd figure out
whether the bottleneck is the server producing HTML, network transfer, or
client-side hydration.

**Good answer:**
A layered approach:
1. **Network panel — Time to First Byte (TTFB).** A large gap between the
   request starting and the first byte of the HTML response arriving
   points at server-side render time (or an upstream data fetch the server
   render is blocked on) — not a client problem at all.
2. **Network panel — response streaming duration.** With streaming SSR, you
   can see the response connection stay open and bytes trickle in over
   time; if the *shell* arrives fast but the full document takes long,
   that's expected (Suspense boundaries resolving) rather than necessarily
   a problem — the useful metric is when the *shell* + first meaningful
   content arrives, not total document time.
3. **Performance panel — main thread activity after the HTML/shell
   arrives.** Look for a long task correlated with hydration: parsing and
   executing the JS bundle, then React's hydration pass walking the DOM.
   If there's a large gap between "HTML visible" and "page responds to
   clicks," and the Performance panel shows heavy scripting during that
   gap, that's hydration cost, not server or network.
4. **React DevTools Profiler** on the hydration commit specifically can
   show which components took the most time to hydrate, which is more
   actionable than a generic "hydration was slow" finding — it points at
   *which* subtree to optimize (e.g. moving a component to be a Server
   Component to remove it from the client bundle/hydration work entirely,
   or lazy-loading a Client Component that isn't needed immediately).

The key discipline is not guessing — TTFB, streaming duration, and
post-paint main-thread activity are three genuinely different phases with
different fixes (server/data-layer optimization, Suspense boundary
tuning, and bundle/hydration-scope reduction, respectively), and treating
them as one undifferentiated "SSR is slow" problem leads to fixing the
wrong thing.

**Follow-up question:**
Suppose the Profiler shows hydration itself is fast, but there's still a
multi-second gap before the page responds to input. What else could
explain that, given hydration finished quickly?

**Follow-up good answer:**
A few possibilities beyond hydration itself: (1) a large `useEffect` that
runs immediately after mount and does expensive synchronous work or
kicks off further client-side data fetching that the UI is effectively
waiting on, even though hydration technically "finished"; (2) the JS
bundle download/parse/compile happening *before* hydration can even start
— if hydration itself was fast once it started but there was a long delay
getting there, the bottleneck is upstream of hydration (network fetch of
the bundle, or main-thread contention from other scripts, like third-party
tags); (3) React 18+ concurrent rendering deliberately yielding to the
browser for other work (e.g. paint, other event handling) between chunks
of hydration work, which is usually a net win for responsiveness but can
look like "hydration finished, why is it still not responding" if you're
only measuring wall-clock time around the hydration call rather than
actual interactivity. The fix in each case is different, which is exactly
why isolating "which phase" before reaching for a fix matters.

**Glossary:**
- **TTFB (Time to First Byte)** — time from request start to the first
  byte of the response.
- **Long task** — a main-thread task exceeding 50ms, which can block input
  responsiveness.

**Mental model:**
A pure performance-diagnosis-methodology question — this is exactly the
"how do you actually find the bottleneck" style of question that's
trending, and it rewards a structured, tool-by-tool answer over a vague
"I'd profile it."

**TL;DR:**
Isolate the bottleneck by phase — TTFB for server/render time, streaming duration for Suspense resolution, and post-paint main-thread activity for hydration cost — each has a different fix.

**References:**
- [renderToPipeableStream – react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

### Q17. What's the theoretical requirement React's render function has always had, and why does that requirement become load-bearing (not just good practice) once SSR/hydration enters the picture? {#q17}

**Question:**
React has always said render functions should be "pure." In a purely
client-rendered app, an impure render might just cause a visual glitch or
extra re-render. Why does the same impurity become a correctness bug, not
just a style issue, once SSR and hydration are involved?

**Good answer:**
The purity requirement (same props/state in ⇒ same output out, no side
effects during render) is a general React rule, but in a client-only app,
if a render function is *slightly* impure (say, it reads `Date.now()`),
the consequence is usually just an extra render or a value that's a few
milliseconds stale — annoying, but the DOM produced is still internally
consistent because there's only ever one environment rendering it. Once
SSR is in play, that same component is rendered **twice, in two different
processes/environments, at two different points in time** — once on the
server, once on the client during hydration — and hydration's correctness
model depends on those two independent renders producing *identical*
output. An impure render (reading wall-clock time, `Math.random()`,
`window`, locale that could differ, non-deterministic iteration order)
is no longer just a style nit — it directly causes a hydration mismatch,
because the two invocations of the "same" render, given the "same" logical
input, produce different output through the side channel. This is the
concrete, practical payoff of the abstract "pure function" theory: it's
not a coding-style preference, it's what makes running the same component
twice, in two environments, actually reproducible.

**Follow-up question:**
Data that's inherently time-sensitive (like "posted 3 minutes ago") seems
to require some kind of impurity by nature. How do you reconcile that with
the purity requirement?

**Follow-up good answer:**
You don't render the non-deterministic part directly in the shared
server/client render path — you either: (1) compute the value *outside*
render, on the server, and pass it down as a fixed prop (e.g. compute
"3 minutes ago" once at request time and serialize that string, so both
server and client renders see the identical prop and produce identical
output — the underlying clock is still moving, but the *component* isn't
reading it directly); or (2) explicitly accept and scope the difference
with `suppressHydrationWarning` on just that element, if it's cosmetic and
truly can't be made deterministic (a live-updating "x seconds ago" that's
*meant* to tick client-side after hydration); or (3) defer the
time-sensitive part to a `useEffect`-driven update that only happens
*after* hydration completes, rendering a static/deterministic value on
first paint (matching between server and client) and then intentionally
updating it client-side on a subsequent, non-hydration render. All three
preserve purity for *the render that hydration compares* — the trick is
recognizing which render pass purity actually has to hold for.

**Glossary:**
- **Pure function (in React's sense)** — given the same inputs, always
  produces the same output, with no observable side effects during
  execution.

**Mental model:**
Directly connects abstract CS/SE theory (referential transparency / pure
functions) to a concrete, practical technology-specific consequence
(hydration correctness) — exactly the "theory mixed with practice" angle
the interview-prep contract asks for.

**TL;DR:**
Purity is cosmetic in pure CSR but load-bearing under SSR, because hydration's correctness depends on two independent renders (server and client) producing identical output.

**References:**
- [hydrateRoot – react.dev](https://react.dev/reference/react-dom/client/hydrateRoot)

---

### Q18. What's the real-world business/product problem that motivated Server Components' existence, beyond "it's a nice performance optimization"? {#q18}

**Question:**
Frame this differently: what production pain, before Server Components
existed, was bad enough that the React team built a whole new component
model to solve it?

**Good answer:**
Before RSC, teams building content-heavy, SEO-relevant apps with SSR faced
a persistent tension: to render rich content on the server (markdown
rendering, syntax highlighting, data transformation, sanitization), you
typically had to either (a) ship those libraries to the client too — even
though the client's *own* rendering of that content, post-hydration, would
just reproduce what the server already sent — bloating the bundle with
code whose output the user already has; or (b) build and maintain a
separate API layer purely to keep that logic server-only, which
reintroduces client → API → DB round-trips and the waterfall problem, plus
ongoing API surface to design and version. Neither option was free: (a)
directly hurt load performance and TTI on every user, permanently, to
support a case (re-rendering already-rendered content) that mostly never
happens; (b) traded that performance cost for architectural
complexity/maintenance cost and worse data-fetching latency. Server
Components collapse this tension: you write one component, it runs once on
the server with full access to server-only capabilities and dependencies,
and the client gets only the result — no bundle bloat *and* no separate API
layer required for that logic specifically.

**Follow-up question:**
Doesn't a traditional SSR framework (pre-RSC) already solve "render on the
server"? What was specifically still missing?

**Follow-up good answer:**
Traditional SSR renders the HTML on the server for the *first* request,
but the component's JS is still shipped and re-hydrated/re-rendered on the
client for every subsequent interaction and re-render — so any heavy
dependency used purely to produce that server-rendered output was still
part of the client bundle "just in case" the component re-renders
client-side. What was missing was a way to say "this component's logic and
dependencies should exist *only* on the server, permanently, not just for
the first render" — which requires the framework to track, at the module
level, which code is allowed to ever run on the client and exclude
everything else from the client bundle entirely. That's precisely the
`'use client'` boundary: it inverts the previous default (everything ships
unless you go out of your way to code-split it) to a new default
(nothing ships unless explicitly marked as needing to run on the client).

**Glossary:**
- **Bundle bloat** — client JavaScript bundle size inflated by code the
  browser doesn't actually need to execute.

**Mental model:**
Real-world-motivation question — checks whether the candidate can explain
*why* a technology was built (the pain it removes) rather than only *what*
it does, which is a strong signal of genuine understanding versus rote
API knowledge.

**TL;DR:**
RSC exists to stop shipping server-only-usable code to the client just to avoid building a separate API layer — collapsing both the bundle-bloat and waterfall problems at once.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q19. What common anti-pattern happens when teams migrate an existing CSR app to SSR/RSC without rethinking their data-fetching code, and why does it happen? {#q19}

**Question:**
A team migrates an existing client-rendered app to an SSR/RSC framework
but keeps most of their `useEffect`-based data fetching as-is. What goes
wrong, and why is it an easy trap to fall into?

**Good answer:**
The common anti-pattern is ending up with a page that's technically
"server-rendered" (the initial HTML shell is there) but still fetches most
of its actual data via `useEffect` inside Client Components after mount —
so the user sees a server-rendered *shell/skeleton*, then watches the real
content pop in via client-side fetches anyway, largely negating the SSR
benefit (fast, complete-content first paint) while still paying the SSR
infrastructure cost (server render time, streaming complexity). It's an
easy trap because `useEffect`-based fetching is the pattern most teams
already have muscle memory for from years of pure-CSR development, and
migrating the *rendering* strategy (adding SSR) doesn't automatically force
a rethink of the *data-fetching* strategy — the code still "works" (no
compile errors, no crashes), it just quietly loses most of the performance
benefit the migration was supposed to deliver.

**Follow-up question:**
What's the concrete fix — how should data fetching actually change during
this kind of migration, not just the rendering setup?

**Follow-up good answer:**
Data fetching needs to move from "fetch after mount, in a Client
Component's `useEffect`" to "fetch during render, in a Server Component,"
wherever the data doesn't specifically need to be re-fetched in response
to client-side interaction. Concretely: identify which fetches are for
*initial* page content (candidates to become `await`-based fetches
directly inside `async` Server Components, so they're part of the SSR
output and RSC payload from the start) versus which fetches genuinely
need to happen in response to user interaction after the page is already
interactive (those legitimately stay as client-side fetches, e.g.
triggered by a button click or a search-as-you-type input). Doing this
audit is the actual migration work — swapping the rendering entry point
to an SSR framework is necessary but not sufficient; without moving the
initial-load fetches into the server render path, the migration captures
little of the intended benefit.

**Glossary:**
- **Migration anti-pattern** — adopting a new architecture's surface
  (rendering setup) without adopting the practices that make it effective
  (data-fetching strategy), leaving most of the expected benefit unrealized.

**Mental model:**
A pitfalls/real-world-experience question — separates candidates who've
actually shipped an SSR/RSC migration (and hit this specific trap) from
those who only know the individual APIs in isolation.

**TL;DR:**
Migrating to SSR/RSC without moving initial-load fetches out of client useEffects leaves a server-rendered shell that still waits on client-side fetches, losing most of the benefit.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)

---

### Q20. Compare React Server Components to a traditional SSR-only framework approach (server renders HTML, full client bundle still ships) — what do you gain, and what new complexity do you take on? {#q20}

**Question:**
Give me a balanced comparison: React Server Components vs. "classic" SSR
(server renders HTML for first paint, but the whole app's JS still ships
to and hydrates on the client). What's the real trade-off?

**Good answer:**
**What RSC gains you over classic SSR:**
- Smaller client bundles (server-only code/dependencies never ship at all,
  not just "rendered first, shipped anyway").
- Reduced hydration scope/cost, since fewer components need to become
  interactive on the client — directly helping TTI/INP.
- Simpler data access for server-only logic (direct `await db.query(...)`
  in the component, no API layer needed just to keep that logic
  server-side).
- Fetches that used to be serialized behind client mount order can run
  concurrently on the server instead.

**What it costs you:**
- A new mental model to learn and get right: the server/client module
  boundary (`'use client'`/`'use server'`), what's serializable across it,
  and debugging when something unexpectedly ends up in (or is missing
  from) the client bundle.
- Framework/infra requirements — RSC isn't a drop-in library feature the
  way, say, a new hook is; it requires a framework (and bundler) that
  implements the RSC protocol/build pipeline, and a server runtime capable
  of streaming, which is a bigger infra commitment than classic SSR
  sometimes needs.
- Server Components genuinely cannot hold state or use browser APIs — that
  constraint is new complexity for developers used to a single, uniform
  component model where any component could become interactive if needed;
  now that decision has to be made deliberately, per component, up front.
- Debugging surface area grows: you now have server-only bugs, client-only
  bugs, *and* boundary-crossing/serialization bugs, versus classic SSR's
  simpler "it's basically the same component, running twice" model.

Neither is strictly better — RSC's wins compound most for
content/data-heavy apps with meaningful SEO or first-paint requirements
and non-trivial server-only dependencies; classic SSR (or plain CSR) can
still be the pragmatic choice for apps where that added architectural
complexity doesn't buy proportional benefit.

**Follow-up question:**
If a team is already running classic SSR successfully and considers
migrating to RSC purely for the bundle-size benefit, what would you want
to know before recommending it?

**Follow-up good answer:**
I'd want to know: (1) how much of their current client bundle is actually
attributable to server-only-usable code (heavy formatting/parsing/
sanitization libraries, DB clients accidentally bundled) versus code that
genuinely needs to be interactive — if it's mostly the latter, RSC's
bundle-size win is small and doesn't justify the migration cost; (2)
whether their current hosting/infra already supports the streaming server
runtime RSC needs, or whether adopting it means a real infrastructure
change, not just a code change; (3) how much `useEffect`-based data
fetching they have that would need to be restructured to actually realize
the benefit (per Q19) — if that refactor is large, the migration cost is
much bigger than "swap the framework"; and (4) whether their team has
bandwidth to absorb the new server/client boundary debugging model. If the
bundle-size win is modest and the infra/migration cost is high, a narrower
fix (code-splitting/lazy-loading the few heaviest dependencies in their
existing classic-SSR setup) might capture most of the benefit for a
fraction of the cost.

**Glossary:**
- **Classic SSR** — server renders HTML for first paint, but the full
  component tree's JS also ships to and hydrates on the client (as
  opposed to RSC, where server-only components never ship at all).

**Mental model:**
A closing trade-off/comparison question that requires synthesizing most of
the set — tests whether the candidate can give a genuinely balanced
recommendation (including talking a team out of a migration) rather than
one-sided technology advocacy.

**TL;DR:**
RSC shrinks bundles and hydration cost but adds a new server/client boundary mental model and infra requirement — worth it mainly for content-heavy, SEO-relevant, server-dependency-heavy apps.

**References:**
- [Server Components – react.dev](https://react.dev/reference/rsc/server-components)
- [renderToPipeableStream – react.dev](https://react.dev/reference/react-dom/server/renderToPipeableStream)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=react&tags=server-components-and-ssr-hydration&autostart=1" | relative_url }})
