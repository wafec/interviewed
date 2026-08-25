---
layout: default
title: "Java Interview — Collections & Streams Internals"
---

# Java Interview — Collections & Streams Internals

This set covers the Java Collections Framework and the Streams API from the
inside out: how `HashMap`, `ArrayList`, `TreeMap`, and `LinkedHashMap`
actually store and resize data, how the Streams pipeline achieves laziness
and short-circuiting, where these abstractions leak performance, and when to
reach for a loop instead of a stream.

### Q1. What is the general contract between `hashCode()` and `equals()`, and why does breaking it silently corrupt a `HashMap`?

**Question:**
Explain the contract between `hashCode()` and `equals()` in Java. What actually goes wrong if you override `equals()` but not `hashCode()`, and then use the object as a `HashMap` key?

**Good answer:**
The contract (from `Object`'s Javadoc) has three parts: (1) `hashCode()` must be consistent across repeated calls in the same run as long as the fields used in `equals()` don't change; (2) if two objects are equal per `equals()`, they *must* return the same `hashCode()`; (3) unequal objects are *not* required to have different hash codes, though distinct codes improve hash-table performance. If you override `equals()` without overriding `hashCode()`, two objects that are logically equal can end up with different hash codes (the default `Object.hashCode()` is identity-based). A `HashMap` uses `hashCode()` to pick the bucket and `equals()` only to compare within that bucket, so a "duplicate" key can land in a different bucket than an existing equal key — `get()` on a key that's logically present returns `null`, and `put()` silently creates a second entry instead of overwriting the first.

**Code example:**
```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        return o instanceof Point p && p.x == x && p.y == y;
    }
    // Missing hashCode() override — Point(1,1).equals(new Point(1,1)) is true,
    // but their default identity hashCodes almost certainly differ.
}

Map<Point, String> map = new HashMap<>();
map.put(new Point(1, 1), "origin-ish");
map.get(new Point(1, 1)); // returns null, not "origin-ish"
```

**Follow-up question:**
Why does the contract *not* require unequal objects to have different hash codes?

**Follow-up good answer:**
Because a perfect hash function (no collisions for any pair of unequal objects) is generally impossible for an unbounded domain of objects mapped into a fixed-size `int` — you'd need infinitely many distinct hash codes for infinitely many possible objects, or a hash range at least as large as the object space. Collisions are therefore expected and handled by the data structure (chaining in `HashMap`'s buckets). The contract only requires that collisions don't cause *correctness* problems (equal objects must collide, i.e., share a bucket), not that they never happen — good hash functions just make collisions rare enough that lookup stays close to O(1) on average.

**Glossary:**
- **Hash bucket** — the slot in a hash table's internal array that a key is routed to based on (a transformation of) its `hashCode()`.
- **Identity hash code** — the default `hashCode()` implementation, typically derived from the object's memory address/identity, unrelated to its field values.

**Mental model:**
Tests whether the candidate understands hashing as a *contract between cooperating methods*, not two independent overrides — a classic "looks fine, compiles fine, corrupts data silently" bug class that's hard to catch without knowing the rule.

**References:**
- [Object#hashCode() — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html#hashCode())

---

### Q2. Walk through what happens internally when a `HashMap` resizes.

**Question:**
`HashMap` starts with a default capacity of 16. What triggers a resize, and what actually happens to the existing entries when it resizes?

**Good answer:**
`HashMap` has a default initial capacity of 16 and a default load factor of 0.75. A resize ("rehash") is triggered once the number of entries exceeds `capacity × loadFactor` (12 entries at the default 16/0.75). On resize, the internal table roughly doubles in size, and every existing entry is redistributed across the new, larger bucket array based on `hash & (newCapacity - 1)` — since capacities are always powers of two, this is a cheap bitmask instead of a modulo. Because every entry's bucket index can change, rehashing is an O(n) operation over the whole map, done incrementally only in the sense that it's triggered lazily on the insert that crosses the threshold, not amortized across inserts.

**Code example:**
```java
// Avoid repeated resizes when the final size is known:
Map<String, Integer> m = new HashMap<>(expectedSize * 4 / 3 + 1);
```

**Follow-up question:**
Why does iterating a `HashMap` with a much larger capacity than its actual size hurt performance, and how would you avoid it?

**Follow-up good answer:**
Iteration over a `HashMap`'s views walks the entire internal bucket array, not just the occupied slots, so its cost is proportional to `capacity + size`, not just `size`. If a map was sized (or resized) far larger than the number of entries it actually holds — e.g. after many removals, or by over-provisioning the initial capacity — iteration wastes time skipping empty buckets. The fix is to size the map close to its expected entry count up front (as in the `Q2` example) rather than relying on default growth, and to avoid drastically over-allocating "just in case."

**Glossary:**
- **Load factor** — the fraction of `capacity` that, once exceeded by `size`, triggers a resize; default 0.75 trades memory for lookup speed.
- **Rehash** — recomputing bucket placement for every entry after the table grows.

**Mental model:**
Checks whether the candidate can reason about amortized cost and knows that a "cheap O(1) map lookup" has a real, sometimes-surprising cost model underneath — a common performance-diagnosis blind spot.

**References:**
- [HashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)

---

### Q3. What is HashMap treeification, and when does it kick in?

**Question:**
Since Java 8, `HashMap` can convert a bucket's linked list into a tree structure. When does this happen, and why?

**Good answer:**
When a single bucket accumulates 8 or more colliding entries (`TREEIFY_THRESHOLD = 8`) *and* the table's overall capacity is at least 64 (`MIN_TREEIFY_CAPACITY = 64`), that bucket is converted from a linked list into a red-black tree, giving worst-case O(log n) lookup within that bucket instead of O(n). This exists as a defense against pathological hash collisions (accidental or adversarial — e.g., an attacker crafting many keys with the same `hashCode()` to degrade a public-facing map into a linked list, a known denial-of-service vector). If entries are later removed and the bucket shrinks back to 6 or fewer entries (`UNTREEIFY_THRESHOLD = 6`), it's converted back to a linked list, since a tree has more per-node overhead and isn't worth it for a short chain. The 64-capacity gate exists so a small map with an accidentally lopsided (but harmless) distribution resizes first instead of paying tree overhead.

**Follow-up question:**
Why is `MIN_TREEIFY_CAPACITY` needed at all — why not treeify any bucket that hits 8 entries, regardless of table size?

**Follow-up good answer:**
In a small table, a bucket hitting 8 entries is more likely to be caused by too few buckets overall (high load) rather than a genuinely bad hash distribution — resizing the whole table (which redistributes entries across more buckets) is a cheaper and more effective fix than treeifying one bucket. `MIN_TREEIFY_CAPACITY = 64` — set to at least `4 × TREEIFY_THRESHOLD` — ensures the map prefers growing the table over treeifying while it's still small, and only treeifies once the table is large enough that a persistent 8-entry bucket really does indicate a collision problem rather than simple undersizing.

**Glossary:**
- **Treeification** — converting a hash bucket's linked-list chain into a red-black tree once it grows long enough to justify the overhead.
- **TREEIFY_THRESHOLD / UNTREEIFY_THRESHOLD / MIN_TREEIFY_CAPACITY** — the constants (8 / 6 / 64) controlling when a bucket becomes, and stops being, a tree.

**Mental model:**
Distinguishes candidates who've only used `HashMap` from those who understand it as an evolving, self-defending data structure — a strong signal of "read the source, not just the tutorial."

**References:**
- [HashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
- [HashMap.java — OpenJDK source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/HashMap.java)

---

### Q4. `ArrayList` claims "amortized constant time" for `add()`. What does that actually mean, and how does the list grow internally?

**Question:**
`ArrayList#add` is documented as amortized O(1). Explain what "amortized" means here and what happens under the hood when the backing array is full.

**Good answer:**
`ArrayList` is backed by a plain `Object[]`. Most `add()` calls just write into the next free slot — true O(1). But when the array is full, `add()` must allocate a new, larger array and copy every existing element into it — an O(n) operation. "Amortized constant time" means that if you average the cost of `add()` over a long sequence of n additions, the total work (n cheap writes plus the occasional O(n) copy) comes out to O(n) overall, i.e., O(1) per call on average — even though any *individual* call can spike to O(n). The Javadoc deliberately does not specify the exact growth policy, but the current OpenJDK implementation grows the array by roughly 50% (`oldCapacity >> 1`) each time it needs to expand, rather than growing by a fixed amount — geometric growth is what makes the amortized-O(1) argument work; growing by a constant increment each time would degrade to O(n) amortized per add.

**Code example:**
```java
List<String> list = new ArrayList<>(); // capacity 0 until first add
list.add("a"); // triggers allocation, e.g. capacity -> 10 (DEFAULT_CAPACITY)
// ... after 10 adds, next add() reallocates to ~15, copying 10 elements
```

**Follow-up question:**
If you know you're about to insert 100,000 elements, how do you avoid paying for repeated reallocations, and why does that matter?

**Follow-up good answer:**
Pre-size the list with `new ArrayList<>(100_000)`, which sets the initial backing-array capacity directly instead of relying on default growth from capacity 10. Without it, the list has to grow (and copy all existing elements) roughly a dozen times on the way to 100,000 elements at a 1.5x growth factor — each copy is O(current size), so the cumulative wasted copying, while still technically amortized O(1) per add, adds real constant-factor overhead and GC pressure from discarded intermediate arrays. Pre-sizing avoids that entirely when the target size is known ahead of time.

**Glossary:**
- **Amortized complexity** — average cost per operation over a worst-case sequence of operations, even if individual operations vary widely in cost.
- **Backing array** — the raw `Object[]` an `ArrayList` uses internally to store elements contiguously.

**Mental model:**
Tests whether "O(1)" is understood as a precise claim with conditions, not a magic label — and whether the candidate can connect that to a concrete performance-tuning action (pre-sizing).

**References:**
- [ArrayList — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html)
- [ArrayList.java — OpenJDK source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/ArrayList.java)

---

### Q5. When would you actually reach for `LinkedList` over `ArrayList`?

**Question:**
`ArrayList` and `LinkedList` both implement `List`. What's the real difference in practice, and is there a legitimate case for choosing `LinkedList`?

**Good answer:**
`ArrayList` stores elements in a contiguous array: O(1) random access (`get(i)`), but O(n) insertion/removal in the middle (everything after the index has to shift). `LinkedList` is a doubly-linked list: O(1) insertion/removal *once you're already at the right node* (e.g., via an `Iterator`, or at the head/tail — it also implements `Deque`), but O(n) random access since it has to walk from an end. In practice, `ArrayList` wins the overwhelming majority of the time: better cache locality (contiguous memory means fewer cache misses), lower per-element overhead (no prev/next node pointers), and most real workloads read far more than they insert-in-the-middle. `LinkedList` earns its keep specifically when you need frequent insert/remove at both ends combined with sequential (not random) access — e.g., implementing a work queue or an LRU eviction list via `Deque` — and even then, `ArrayDeque` usually outperforms `LinkedList` for pure queue/stack use since it avoids per-node allocation entirely.

**Follow-up question:**
Given that, why does `ArrayDeque`'s Javadoc recommend it over `LinkedList` for stack/queue use even though `LinkedList` also implements `Deque`?

**Follow-up good answer:**
`ArrayDeque` is backed by a resizable circular array, so like `ArrayList` it has better cache locality and no per-element node allocation/pointer overhead — pushing/popping at either end is O(1) amortized, same asymptotic complexity as `LinkedList`, but with a much better constant factor and less GC pressure (no boxed `Node` objects to allocate and collect per element). `LinkedList` only wins over `ArrayDeque` when you specifically need `null` elements (which `ArrayDeque` disallows) or need to hold onto and splice/remove from an arbitrary interior `ListIterator` position cheaply — narrow cases most queue/stack code doesn't hit.

**Glossary:**
- **Cache locality** — how close together in memory sequential data lives; contiguous arrays are far more cache-friendly than pointer-chased linked structures.
- **Deque** — a double-ended queue interface supporting insertion/removal at both ends.

**Mental model:**
Separates candidates who know Big-O from those who also understand constant factors and hardware-level cost (cache misses, allocation pressure) — the gap between "textbook correct" and "actually fast."

**References:**
- [LinkedList — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedList.html)
- [ArrayDeque — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayDeque.html)

---

### Q6. Compare `HashMap`, `LinkedHashMap`, and `TreeMap` — when do you reach for each?

**Question:**
All three implement `Map`. What's the actual difference in ordering guarantees and performance, and how would you choose between them?

**Good answer:**
`HashMap` gives no ordering guarantee at all — iteration order can even change across JVM runs or after resizes — but offers O(1) average-case `get`/`put`. `LinkedHashMap` extends `HashMap` and additionally maintains a doubly-linked list through all entries, preserving either insertion order (default) or access order (if constructed with `accessOrder = true`), at the cost of a bit more memory per entry (the extra prev/next pointers) — same O(1) average `get`/`put` as `HashMap`. `TreeMap` implements `NavigableMap` on top of a red-black tree, keeping keys in sorted order (natural ordering or a supplied `Comparator`) at the cost of O(log n) `get`/`put`/`remove` instead of O(1) — you pay for the ordering guarantee, but gain range queries (`headMap`, `tailMap`, `floorKey`, `ceilingKey`, etc.) that a hash-based map can't offer at all. Choose `HashMap` by default when order doesn't matter, `LinkedHashMap` when you need predictable iteration order (or an LRU cache via access-order mode), and `TreeMap` when you need sorted iteration or range queries.

**Follow-up question:**
How would you build a simple LRU cache using one of these three, without writing a custom eviction data structure?

**Follow-up good answer:**
Subclass `LinkedHashMap`, construct it with `accessOrder = true` so `get()`/`put()` move an entry to the end of the internal ordering (most-recently-used), and override `removeEldestEntry(Map.Entry eldest)` to return `true` once `size()` exceeds the desired capacity — `LinkedHashMap` calls this hook automatically after every `put()`/`putAll()`, and a `true` return evicts the eldest (least-recently-used) entry for you. This gives O(1) get/put with automatic LRU eviction, entirely from the standard library, no manual doubly-linked-list bookkeeping required.

**Code example:**
```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LRUCache(int capacity) {
        super(16, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

**Glossary:**
- **NavigableMap** — a `SortedMap` extension adding closest-match navigation methods (`floorKey`, `ceilingKey`, `headMap`, `tailMap`, etc.).
- **Access order** — a `LinkedHashMap` mode where reads (not just writes) move an entry to the most-recently-used end of iteration order.

**Mental model:**
Tests whether the candidate picks data structures by their actual guarantees (ordering, complexity) rather than habit, and whether they know the standard library already solves common problems like LRU caching.

**References:**
- [TreeMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html)
- [LinkedHashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html)

---

### Q7. What does it mean that `TreeMap` provides "guaranteed log(n)" time, and what's the underlying structure?

**Question:**
The `TreeMap` Javadoc says it provides guaranteed log(n) time for `containsKey`, `get`, `put`, and `remove`. What's actually backing that guarantee?

**Good answer:**
`TreeMap`'s Javadoc explicitly describes it as "A Red-Black tree based NavigableMap implementation." A red-black tree is a self-balancing binary search tree: it maintains a set of coloring/rotation invariants on every insert and delete that keep the tree's height bounded to O(log n) even in the worst case, unlike a naive unbalanced BST which can degrade to a linked list (O(n) per operation) if keys are inserted in sorted order. Because the height is always O(log n), every operation that walks from the root — search, insert, delete — costs O(log n) in the *worst* case, not just on average, which is what "guaranteed" (as opposed to `HashMap`'s "expected"/average-case O(1)) is contrasting.

**Follow-up question:**
Why does a plain (unbalanced) binary search tree not offer this same guarantee?

**Follow-up good answer:**
An unbalanced BST's height depends entirely on insertion order. If keys are inserted in already-sorted order, each new node just becomes the rightmost (or leftmost) child of the previous one, degenerating the tree into what is structurally a linked list — height O(n), so search/insert/delete also become O(n). A red-black tree avoids this by actively rebalancing (via rotations and recoloring) after every insert/delete to keep the longest root-to-leaf path from exceeding roughly `2 × log(n+1)`, so no insertion order can force it into a degenerate, list-like shape.

**Glossary:**
- **Red-black tree** — a self-balancing binary search tree that bounds height to O(log n) via coloring and rotation rules.
- **Worst-case vs. average-case complexity** — a guarantee that holds for *every* input (worst-case) versus one that holds "typically" but can degrade for adversarial/unlucky inputs (average-case).

**Mental model:**
Connects a library-level API guarantee back to the CS fundamentals (BST balancing) that make it true — checks whether "log(n)" is understood, not memorized.

**References:**
- [TreeMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html)

---

### Q8. What does "fail-fast" mean for `HashMap`/`ArrayList` iterators, and why is it explicitly *not* a reliability guarantee?

**Question:**
`ConcurrentModificationException` — what triggers it, and why does the Javadoc warn you not to rely on it?

**Good answer:**
Iterators on `HashMap`, `ArrayList`, and most non-concurrent collections are "fail-fast": each collection tracks a `modCount` field that increments on every structural modification (add/remove, but not `set`), and the iterator captures `modCount` when created. Each `next()`/`remove()` call checks the current `modCount` against the captured value — a mismatch (caused by modifying the collection some way other than through that same iterator) throws `ConcurrentModificationException`. The Javadoc is explicit that this is "best-effort" detection, not a guarantee: in a genuinely concurrent, unsynchronized-modification scenario, the check can race and simply *not* detect the conflict, or throw at an unpredictable point, or (in older/edge-case implementations) even cause other undefined behavior. It exists purely to help catch programming bugs during development/testing — "fail-fast" so a bug surfaces immediately near its cause instead of silently corrupting state — not to make concurrent access to these collections safe.

**Code example:**
```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) list.remove(s); // throws ConcurrentModificationException
}
// Correct: use an Iterator's own remove(), or removeIf()
list.removeIf(s -> s.equals("b"));
```

**Follow-up question:**
What's actually thread-safe for concurrent read/write access, and how does it avoid the same problem?

**Follow-up good answer:**
For maps, `java.util.concurrent.ConcurrentHashMap` supports safe concurrent reads and writes without external synchronization or `ConcurrentModificationException` — its iterators are "weakly consistent": they reflect the state of the map at (or since) iterator creation and are guaranteed not to throw `ConcurrentModificationException`, but may or may not reflect updates made during the iteration. For lists, `CopyOnWriteArrayList` takes a different approach — every mutation copies the entire backing array, so iterators (which snapshot the array reference at creation) never see concurrent modifications at all and never throw; it trades write cost (an O(n) copy per mutation) for read/iteration safety, making it suited to read-heavy, write-rare workloads.

**Glossary:**
- **Fail-fast** — detecting a probable bug (concurrent structural modification) and failing immediately and loudly, on a best-effort basis.
- **modCount** — an internal counter most `java.util.Collection` implementations bump on every structural change, used to detect concurrent modification.

**Mental model:**
Checks that the candidate distinguishes "throws an exception sometimes" from "is safe for concurrent use" — a common false sense of security around fail-fast collections.

**References:**
- [HashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
- [ConcurrentHashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

---

### Q9. Why is it dangerous to use a mutable object as a `HashMap` key?

**Question:**
What goes wrong if you use a mutable object (say, a `List<Integer>` or a POJO with setters) as a `HashMap` key and then mutate it after insertion?

**Good answer:**
A `HashMap` places a key into a bucket based on its `hashCode()` *at insertion time*. If the key object is later mutated in a way that changes what `hashCode()` returns (because `hashCode()` is derived from mutable fields), the entry is now sitting in the bucket for its *old* hash code, but any future lookup with an equal key computes the *new* hash code and looks in the wrong bucket — the entry becomes effectively unreachable via `get()`, even though `containsKey`/iteration might still show it's "in there" if you walk every bucket. This is a silent correctness bug, not an exception — nothing signals that the map is now broken for that entry.

**Follow-up question:**
Given this, what's the recommended practice for objects used as map keys?

**Follow-up good answer:**
Use immutable objects as keys — final fields set once in the constructor, with `hashCode()`/`equals()` derived only from those immutable fields (this is exactly why records, and classes like `String` and the boxed numeric types, are safe and common key choices). If you must use a mutable type, the discipline is to never mutate an instance after it's been inserted as a key anywhere, but that's fragile and hard to enforce across a codebase — preferring genuinely immutable key types removes the failure mode entirely rather than relying on convention.

**Glossary:**
- **Immutable object** — an object whose observable state cannot change after construction, making its `hashCode()` stable for its entire lifetime.

**Mental model:**
Tests whether the candidate connects "why prefer immutability" to a concrete, hard-to-debug failure mode rather than reciting it as a vague best practice.

**References:**
- [HashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)

---

### Q10. What does it mean that Java Streams are lazy, and why does that matter?

**Question:**
Explain the difference between intermediate and terminal stream operations, and what "laziness" actually buys you.

**Good answer:**
Intermediate operations (`filter`, `map`, `sorted`, `distinct`, etc.) return a new `Stream` and don't do any work themselves — they just describe a step in a pipeline. Nothing executes until a terminal operation (`collect`, `forEach`, `reduce`, `count`, etc.) is invoked; only then does the stream pull elements from the source and push each one through the whole chain of intermediate operations, one element at a time ("operator fusion" into effectively a single pass), rather than materializing a fully filtered list, then a fully mapped list, etc. Laziness matters for two reasons: it enables that single-pass fusion (better cache behavior, less intermediate allocation than chaining separate loops would), and it enables short-circuiting — operations like `limit(n)`, `findFirst()`, or `anyMatch()` can stop pulling from the source as soon as they have their answer, which is what makes operating on a (conceptually) infinite stream possible at all.

**Code example:**
```java
Stream<Integer> pipeline = Stream.iterate(1, n -> n + 1) // infinite
    .filter(n -> n % 7 == 0);
// Nothing has executed yet — filter() just built a pipeline description
int first = pipeline.findFirst().get(); // NOW it pulls elements, stops at 7
```

**Follow-up question:**
If you call `.map(...)` on a stream but never call a terminal operation, what happens?

**Follow-up good answer:**
Nothing — literally nothing executes. Because intermediate operations are lazy, calling `.map()` (or any chain of intermediate operations) with no terminal operation afterward just builds up an unused pipeline description that's eventually garbage collected without ever running the mapping function on a single element. This is a common source of "why didn't my side-effecting code run" bugs when someone expects `stream.map(x -> { doSomething(x); return x; })` to have an effect on its own — it won't, until something like `.forEach()` or `.collect()` triggers actual traversal.

**Glossary:**
- **Intermediate operation** — a lazy stream operation that returns another `Stream` (e.g., `filter`, `map`).
- **Terminal operation** — an operation that triggers pipeline execution and produces a result or side effect (e.g., `collect`, `forEach`).
- **Short-circuiting** — an operation that can complete without processing the entire (possibly infinite) source, e.g. `limit`, `findFirst`, `anyMatch`.

**Mental model:**
Verifies the candidate understands laziness as a mechanism with real consequences (both a performance benefit and a common gotcha), not just a buzzword from the Streams docs.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

---

### Q11. What are "non-interference" and "statelessness" in the context of stream behavioral parameters, and what breaks if you violate them?

**Question:**
The Streams documentation requires that lambdas passed to stream operations be "non-interfering" and (usually) "stateless." What do those terms mean, and what actually goes wrong if you ignore them?

**Good answer:**
"Non-interfering" means the lambda must not modify the stream's data source while the pipeline is executing — e.g., don't add/remove elements from the backing `List` inside a `.forEach()` that's iterating over `list.stream()`; this can throw `ConcurrentModificationException` or produce undefined results, the same class of problem as mutating a collection during iteration. "Stateless" means a lambda's result shouldn't depend on any mutable state that could change between invocations during the pipeline's execution — e.g., a `.map()` function that reads and updates a shared counter. The problem with stateful lambdas is that stream implementations make no guarantees about *order* or *thread* of execution for a given element beyond what the stream's inherent ordering/parallelism model requires — in a parallel stream in particular, a stateful lambda can be invoked concurrently from multiple threads with no visibility guarantees, producing data races and non-deterministic results.

**Code example:**
```java
// WRONG — stateful, unsafe, especially if parallel:
List<String> seen = new ArrayList<>();
stream.parallel().forEach(seen::add); // race condition, non-deterministic order/content

// CORRECT — stateless, let the API handle aggregation safely:
List<String> seen = stream.collect(Collectors.toList());
```

**Follow-up question:**
Why is `forEach` specifically called out as an exception to the "no side effects" guidance?

**Follow-up good answer:**
`forEach` (and `forEachOrdered`) are inherently terminal operations meant *for* side effects — there's no result value to collect, so their entire purpose is to do something with each element (print it, add it to an external structure, etc.). The "avoid side effects" guidance is really about the behavioral parameters of other operations (`filter`, `map`, `reduce`) that are meant to be pure functions describing a transformation — using `forEach` for its intended purpose is fine; the danger is using `map()` or `filter()` as a smuggled-in `forEach` (returning the input unchanged just to trigger a side effect), which breaks the laziness/ordering/parallelism assumptions those operations are allowed to make.

**Glossary:**
- **Non-interference** — a stream behavioral parameter must not modify the stream's data source during pipeline execution.
- **Behavioral parameter** — a lambda or method reference passed into a stream operation to define its behavior (e.g., the predicate in `filter`).

**Mental model:**
Probes whether the candidate treats streams as "fancy for-loops" (and writes unsafe, stateful lambdas out of habit) or actually understands the functional contract streams are built on.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

---

### Q12. What's actually happening when you call `.parallelStream()`, and why can it make things *slower*?

**Question:**
`.parallelStream()` looks like a one-line way to speed up processing. When does it actually help, and when does it backfire?

**Good answer:**
Parallel streams split the source into chunks (via the `Spliterator` abstraction) and process chunks concurrently using the common `ForkJoinPool` — the same shared thread pool used across the entire JVM for all parallel streams (and, unless configured otherwise, by any other code using `ForkJoinPool.commonPool()`). It helps when: the source is large, splits cheaply and evenly (arrays and `ArrayList` split well; `LinkedList` and I/O-backed sources split poorly), the per-element work is substantial (real CPU-bound computation, not a trivial operation), and the operations are stateless/associative so a parallel reduction is even valid. It backfires when: the dataset is small (thread coordination/splitting overhead exceeds any savings), the work per element is tiny (e.g., simple arithmetic — the parallelism overhead dwarfs the work), the source splits poorly, the lambda has hidden synchronization or shared mutable state (contention wipes out any parallel gain and can be *slower* than sequential due to lock contention plus thread overhead), or when other unrelated code sharing the same common `ForkJoinPool` gets starved of pool threads because your parallel stream is holding them.

**Follow-up question:**
How would you actually confirm whether a parallel stream helped, rather than assuming it did?

**Follow-up good answer:**
Benchmark it — with a proper microbenchmarking tool like JMH (a naive `System.currentTimeMillis()` wrapped around one run is unreliable due to JIT warm-up, and JMH exists specifically to control for that), comparing the sequential and parallel versions on realistic data sizes, ideally also under realistic concurrent load (since production code sharing the common pool behaves differently than an isolated benchmark). `jstack`/JFR thread dumps can also reveal `ForkJoinPool.commonPool-worker` threads contending or blocked if there's suspicion the parallel stream is starving other pool consumers. The general rule: never assume `.parallelStream()` is faster — measure on data and load representative of production before keeping it.

**Glossary:**
- **Spliterator** — the abstraction (splittable iterator) streams use to divide a source into sub-ranges for parallel processing.
- **Common ForkJoinPool** — the shared, JVM-wide thread pool (sized to `Runtime.availableProcessors() - 1` by default) that all parallel streams use unless explicitly run in a custom pool.

**Mental model:**
Distinguishes candidates who reach for `.parallelStream()` as a reflex from those who understand its real cost model and would measure before shipping it — directly maps to the "performance diagnosis methodology" interviewers are increasingly probing for.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

---

### Q13. What is `Collectors.groupingBy` doing internally, and what's a common mistake with it?

**Question:**
Explain what `Collectors.groupingBy` does and walk through a common mistake developers make when using it.

**Good answer:**
`Collectors.groupingBy(classifier)` performs a "GROUP BY"-style reduction: it applies the classifier function to each stream element to compute a key, then accumulates elements into a `Map<K, List<T>>`, appending each element to the list for its computed key (creating the list on first encounter of that key). A downstream collector can be supplied as a second argument to reduce each group further instead of just collecting to a list (e.g., `groupingBy(Person::getDept, counting())` to get counts per department, or `groupingBy(Person::getDept, summingInt(Person::getSalary))`). A common mistake is calling `groupingBy` and then trying to `.get()` a key that has zero matching elements — unlike a real SQL `GROUP BY`, if no elements map to a given key, that key simply doesn't appear in the resulting map at all (there's no entry with an empty list), so code that assumes every expected key is present (e.g., iterating a fixed list of categories and doing `map.get(category).size()`) throws a `NullPointerException` for empty groups instead of getting an empty list.

**Code example:**
```java
Map<Department, List<Employee>> byDept =
    employees.stream().collect(Collectors.groupingBy(Employee::getDepartment));

// Safe access for a department that might have zero employees:
List<Employee> eng = byDept.getOrDefault(Department.ENGINEERING, List.of());
```

**Follow-up question:**
How would you get a `Map<Department, Long>` of employee counts per department in one pass, without a manual loop?

**Follow-up good answer:**
Use `groupingBy` with a downstream collector: `employees.stream().collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()))`. This performs the grouping and the per-group reduction in a single stream traversal — the classifier computes the department key, and `Collectors.counting()` (itself implemented as a `reducing`/summing collector) accumulates a running count per group instead of building an intermediate `List` you'd then have to `.size()` yourself.

**Glossary:**
- **Downstream collector** — a second `Collector` passed to `groupingBy`/`partitioningBy` to further reduce each group instead of collecting it to a `List`.
- **Classifier function** — the function used by `groupingBy` to compute the grouping key for each element.

**Mental model:**
Checks familiarity with the `Collectors` toolkit beyond `toList()`, and whether the candidate has hit (and can explain) the "missing key" gotcha that trips up people expecting SQL-like `GROUP BY` semantics.

**References:**
- [Collectors — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html)

---

### Q14. What is `Optional` actually meant to be used for, according to its own documentation?

**Question:**
`Optional<T>` gets used all over the place in modern Java codebases — as return types, but also sometimes as fields or method parameters. What does the official guidance actually say, and why?

**Good answer:**
The `Optional` Javadoc is explicit: it's "primarily intended for use as a method return type where there is a clear need to represent 'no result,' and where using null is likely to cause errors" — i.e., it exists to make "this might not have a value" explicit and force callers to handle that case instead of silently risking a `NullPointerException`. It also explicitly warns that a variable of type `Optional` should never itself be `null` (defeating the purpose) — always point it at an actual `Optional` instance, using `Optional.empty()` for the absent case. `Optional` is documented as a *value-based class*: instances should be treated as interchangeable when equal, must not be used for synchronization (locking on one may fail unpredictably in future releases), and shouldn't be compared with `==`/`!=` against `Optional.empty()` (there's no singleton guarantee) — use `isPresent()`/`isEmpty()` instead. The Javadoc doesn't explicitly ban using it as a field type or parameter, but its "primarily a return type" framing plus its lack of `Serializable` support are why the common convention treats fields/parameters typed as `Optional` as a code smell.

**Follow-up question:**
Why is treating `Optional` as a field or constructor parameter generally considered bad practice, even though nothing stops you from doing it?

**Follow-up good answer:**
`Optional` isn't `Serializable`, so any class holding one as a field can't be serialized (a real, concrete problem beyond style) — that alone rules it out for many domain/entity classes. Beyond that, using it as a field or parameter adds an extra layer of wrapping/unwrapping (`.get()`/`.orElse()`) at every access site for a distinction — "this field might not be set" — that a well-designed domain model can usually express more directly (e.g., through separate constructors, a nullable field with clear documentation, or the Null Object pattern), while `Optional` as a *return type* has a narrower, well-defined job: giving the immediate caller of a method an explicit, compiler-enforced prompt to handle the "no result" case at the call site, which is exactly the scenario its design was optimized for.

**Glossary:**
- **Value-based class** — a class (like `Optional`) whose instances should be treated as interchangeable based on state equality, not identity; not meant for locking or identity comparisons.

**Mental model:**
Tests whether the candidate can cite the *actual* documented intent of a heavily-used API rather than repeating folklore, and can reason about the concrete cost (non-serializability, indirection) behind the "don't use Optional as a field" convention.

**References:**
- [Optional — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Optional.html)

---

### Q15. What real-world problem did the Streams API (Java 8) actually solve that plain loops didn't?

**Question:**
Java had `for` loops and the Collections Framework for over a decade before Streams arrived in Java 8. What problem were Streams actually introduced to solve?

**Good answer:**
Before Java 8, expressing a multi-step data transformation (filter, then transform, then aggregate) as an imperative loop mixed the *what* (the transformation logic) with the *how* (manual iteration, manual accumulator management, manual early-exit logic) — every such loop was hand-written and easy to get subtly wrong (off-by-one, wrong short-circuit condition, accidental mutation of the source). Streams let you *declare* a pipeline of operations and hand execution control to the library, which brought two concrete benefits beyond readability: it opened the door to transparent parallelism (the exact same pipeline can run sequentially or, by changing `.stream()` to `.parallelStream()`, be parallelized by the library without rewriting the algorithm as manually-partitioned, manually-synchronized code), and it gave the JIT/library a chance to fuse operations into a single pass (per Q10) rather than materializing intermediate collections at each step the way a naive chain of loops would. It also aligned Java with the functional-style, multicore-era direction the broader industry was moving toward — CPU clock speeds had plateaued, and expressing computation as composable, potentially-parallelizable operations was a more future-proof default than manual loops for many common data-processing tasks.

**Follow-up question:**
Given those benefits, why do experienced teams still sometimes deliberately choose a plain loop over a stream pipeline?

**Follow-up good answer:**
Debuggability and readability at the margins: a stepped-through `for` loop is trivial to inspect in a debugger (set a breakpoint, watch the accumulator), while a long chained stream pipeline can be awkward to step through and its stack traces on exceptions are often noisier (buried in internal `Stream`/`Collectors`/lambda-metafactory frames). Loops also handle checked exceptions naturally, while stream lambdas can't throw checked exceptions without wrapping. And for very simple, single-pass, non-parallel operations, a loop can be just as readable and marginally faster (no lambda invocation/boxing overhead, no pipeline-object allocation) — teams often set a house-style threshold, using streams for genuinely multi-step transformations and loops for simple, single-operation iteration, rather than treating streams as an unconditional replacement for loops.

**Glossary:**
- **Declarative vs. imperative** — describing *what* result you want (declarative, e.g. a stream pipeline) versus *how* to compute it step by step (imperative, e.g. a hand-written loop).

**Mental model:**
Checks whether the candidate can articulate the actual engineering motivation behind a language feature (not just "it's more modern"), and shows balanced judgment rather than dogmatic "always use streams" or "streams are slow, never use them" positions.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

---

### Q16. You're told a hot code path using streams is slower than expected. How do you actually diagnose whether the streams are the problem?

**Question:**
Walk through how you would investigate a performance complaint about a stream-heavy method — what tools, what methodology?

**Good answer:**
First, confirm the problem is real and reproducible with a proper benchmark — JMH (Java Microbenchmark Harness) rather than ad-hoc timing, since JIT warm-up, dead-code elimination, and constant-folding can make naive timing wildly misleading for hot-path micro-benchmarks. If a genuine regression is confirmed, profile rather than guess: async-profiler or JFR (Java Flight Recorder, low-overhead enough to run in production) to get CPU/allocation flame graphs, which will show whether time is actually going into the stream machinery itself (lambda invocation overhead, boxing/unboxing from primitive streams being avoided, `Collectors` allocation) versus into the business logic inside the lambdas (in which case streams aren't the culprit at all). Common stream-specific costs to look for in the profile: autoboxing (using `Stream<Integer>` instead of `IntStream` for numeric-heavy work forces boxing every element), unnecessary intermediate collection materialization (`.collect(toList())` followed immediately by `.stream()` again), or an inadvertent `.parallelStream()` on a small/poorly-splitting source (per Q12) actually adding coordination overhead. Once the profile points at a specific cause, fix that specific thing and re-run the JMH benchmark to confirm the fix actually helped, rather than assuming it did.

**Follow-up question:**
The flame graph shows a lot of time in `Integer.valueOf`/`intValue`. What does that tell you, and what's the fix?

**Follow-up good answer:**
That's the signature of autoboxing overhead — using a boxed `Stream<Integer>` (or a lambda mixing `int` and `Integer`) forces the JVM to box every primitive into an `Integer` object (and often hit the `Integer` cache lookup or allocate) and unbox it back, for every element, on a hot path. The fix is to use the primitive specializations — `IntStream`/`LongStream`/`DoubleStream` and their `mapToInt`/`mapToObj`/`sum`/`average` operations — which operate on unboxed primitives throughout the pipeline and only box at the boundary if a genuinely boxed result is needed (e.g., collecting to a `List<Integer>` at the very end). This typically eliminates most of that `Integer.valueOf`/`intValue` time in the flame graph entirely.

**Glossary:**
- **JMH (Java Microbenchmark Harness)** — the standard tool for writing reliable Java microbenchmarks, controlling for JIT warm-up and other measurement pitfalls that make naive timing unreliable.
- **Flame graph** — a visualization of profiler samples showing where CPU time (or allocations) is spent across the call stack.
- **Autoboxing** — the automatic conversion between a primitive type and its wrapper class (e.g., `int` ↔ `Integer`), which allocates and adds overhead when it happens implicitly in a hot loop.

**Mental model:**
This is the core "performance diagnosis" question for this file — checks for a concrete, tool-driven methodology (benchmark → profile → targeted fix → re-verify) rather than guessing-and-tweaking, which is exactly what's trending in current interview loops.

**References:**
- [Java Flight Recorder documentation — Oracle](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jfr.html)
- [IntStream — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/IntStream.html)

---

### Q17. What happens if you call `.stream()` on an unbounded source without a short-circuiting operation?

**Question:**
`Stream.iterate(0, n -> n + 1)` produces a conceptually infinite stream. What happens if you call `.collect(Collectors.toList())` directly on it, and how do you use it safely?

**Good answer:**
Calling a non-short-circuiting terminal operation (like `collect`, `forEach`, or `count` in general) on an infinite stream never terminates — the pipeline keeps pulling elements from the unbounded source forever, since nothing tells it to stop, and the program hangs (and, since `Collectors.toList()` keeps accumulating, will also eventually exhaust heap memory with an `OutOfMemoryError` if it runs long enough before you kill it). To use an unbounded source safely, you must apply a short-circuiting operation somewhere in the pipeline: an intermediate short-circuit like `.limit(n)` (turns the infinite stream into a finite one downstream) or a terminal short-circuit like `.findFirst()`/`.anyMatch()` (stops pulling as soon as the answer is determined) — either way, something in the pipeline must bound how many elements get pulled from the infinite source.

**Code example:**
```java
// Hangs forever, eventually OOMs:
List<Integer> bad = Stream.iterate(0, n -> n + 1).collect(Collectors.toList());

// Safe — limit() makes it finite before the terminal op runs:
List<Integer> firstTen = Stream.iterate(0, n -> n + 1).limit(10).collect(Collectors.toList());
```

**Follow-up question:**
Is there a way to generate a bounded stream from an unbounded generator *without* using `.limit()`?

**Follow-up good answer:**
Yes — `Stream.iterate` has an overload, `iterate(seed, hasNext, next)`, that takes a `Predicate` and stops generating elements once the predicate returns `false`, making the stream inherently finite without needing a separate `.limit()` call. There's also `Stream.generate(Supplier)` for a source with no dependency between elements, which is unbounded by construction and still requires `.limit()` (or another short-circuit) to bound it, since a plain `Supplier` has no natural stopping condition of its own.

**Glossary:**
- **Unbounded/infinite stream** — a stream whose source has no inherent end (e.g., `Stream.iterate` with no predicate, or `Stream.generate`); safe only when paired with a short-circuiting operation.

**Mental model:**
A concrete pitfall check — does the candidate recognize this specific hang/OOM failure mode on sight, or only understand laziness/short-circuiting in the abstract?

**References:**
- [Stream — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html)

---

### Q18. `Collectors.toList()` vs. manually building a `List` in a loop — is there really a meaningful difference for a simple filter-and-map?

**Question:**
For a simple "filter this list, then map to another type" operation, is there a real, measurable reason to prefer a stream over a plain loop, or is it purely a style preference?

**Good answer:**
For a simple, sequential, single-pass filter-and-map, the performance difference between a stream and an equivalent hand-written loop is usually negligible in practice — modern JIT compilation inlines and optimizes both forms well for straightforward cases, and the JMH-measurable overhead of lambda invocation and any pipeline-object allocation is typically small relative to real per-element work. So for that specific narrow case, it genuinely is largely a readability/style choice, not a performance one — the earlier "why Streams exist" reasoning (Q15) is about composability, parallelism potential, and avoiding hand-rolled iteration bugs for *non-trivial* pipelines, not about a simple loop being slow. Where streams do show a measurable edge is multi-step pipelines that a naive loop implementation would otherwise materialize as separate intermediate lists at each step (filter into list A, map list A into list B, etc.) — the stream's operator fusion (Q10) avoids those intermediate allocations that an unoptimized multi-loop version would incur.

**Follow-up question:**
So when would you actively recommend a teammate rewrite a stream pipeline back into a loop?

**Follow-up good answer:**
When the pipeline has grown long/nested enough to hurt readability more than a straightforward loop would (deeply chained `.flatMap`/`.collect` calls that are hard to read top-to-bottom), when it needs to throw or handle a checked exception (streams can't propagate checked exceptions from lambdas without wrapping, which usually makes the code uglier than the loop it replaced), when a profiler has specifically identified the stream pipeline as a measurable hot-path cost after the Q16 methodology (not a guess), or when the logic needs early-exit-with-side-effect patterns (e.g., accumulate into two different collections based on a condition, breaking out under some combined condition) that streams express awkwardly compared to a simple loop with an `if`/`break`.

**Glossary:**
- **Operator fusion** — the stream pipeline optimization where multiple intermediate operations execute in a single pass per element, rather than materializing a separate collection after each step.

**Mental model:**
Tests for engineering judgment and the ability to push back on "always use streams" dogma with a specific, defensible rationale — exactly the kind of trade-off reasoning senior interviews probe for.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

---

### Q19. What does `unordered()` do, and why would `Collectors.groupingByConcurrent` require it (or an unordered source) to run truly concurrently?

**Question:**
`Collectors.groupingByConcurrent` is documented as enabling concurrent reduction — but only under certain conditions. What are they, and why?

**Good answer:**
`groupingByConcurrent` can perform its reduction concurrently (multiple threads writing into a shared `ConcurrentHashMap`-backed result at once, rather than each thread building a partial result that gets merged afterward) only when the stream is parallel *and* either the stream is unordered (via `.unordered()`, or its source has no inherent order to begin with, like a `HashSet`) or the collector itself is marked with the `Collector.Characteristics.UNORDERED` characteristic. The reason is that preserving encounter order during a concurrent, multi-threaded reduction would require coordinating threads to merge partial results back together in the original element order — which reintroduces exactly the kind of synchronization/merging overhead concurrent reduction is meant to avoid. If order doesn't matter for the result (as is typically true for a `groupingBy` into a `Map`, where iteration order of the *values within each group* usually isn't semantically important), telling the stream that via `.unordered()` unlocks true concurrent writes into the shared result structure instead of falling back to a sequential merge step.

**Follow-up question:**
If you forget `.unordered()` on a parallel stream feeding `groupingByConcurrent`, does it throw an error, or just silently underperform?

**Follow-up good answer:**
It doesn't throw — it silently falls back to the ordered path (still using the `ConcurrentMap` result type it's documented to produce, but combining partial per-thread results sequentially to preserve order rather than writing concurrently), so you lose the concurrency benefit you were trying to get without any indication something's suboptimal. This is exactly the kind of subtle stream-performance gap that only shows up via profiling/benchmarking, not from reading a stack trace — reinforcing that "measure, don't assume" is the right default for anything performance-sensitive built on streams.

**Glossary:**
- **Collector.Characteristics.UNORDERED** — a flag a `Collector` can declare indicating it doesn't care about encounter order, allowing more efficient (e.g., concurrent) execution strategies.
- **Encounter order** — the order in which a stream's source would naturally present its elements (e.g., a `List`'s index order); some sources/operations have none.

**Mental model:**
A deeper-cut question testing whether the candidate has gone past "parallel streams use ForkJoinPool" into the specific conditions that actually unlock true concurrent (not just parallel) collection — separates strong from very strong candidates.

**References:**
- [java.util.stream package summary — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)
- [Collectors#groupingByConcurrent — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html)

---

### Q20. If you had to explain, end to end, why `HashMap.get()` is "usually O(1) but not guaranteed," what would you say?

**Question:**
Tie together everything about `HashMap` internals: why is `get()` typically described as O(1), and under what conditions does that stop being true?

**Good answer:**
`get(key)` computes `key.hashCode()` (spread through `HashMap`'s internal hash-spreading function to reduce the impact of poor hash implementations), uses it to compute a bucket index via a cheap bitmask (`hash & (capacity - 1)`), then walks that bucket looking for an entry whose key `equals()` the target. When keys are well-distributed across buckets, each bucket holds very few entries (close to `size / capacity`, kept low by the 0.75 load factor triggering resizes), so that walk is effectively O(1) — this is the "usually O(1)" case. It degrades when a bucket accumulates unusually many entries: below `MIN_TREEIFY_CAPACITY`/`TREEIFY_THRESHOLD`, that bucket is a plain linked list, so an adversarial or accidental hash-collision storm makes `get()` on colliding keys O(n) in the number of colliding entries; above those thresholds, `HashMap` treeifies the bucket into a red-black tree, capping the worst case at O(log n) instead of O(n) — better, but still not O(1). So the honest answer is: `get()` is O(1) average-case under a reasonable hash distribution, with a built-in fallback (as of Java 8) that bounds the *worst* case to O(log n) via treeification rather than letting a pathological collision pattern degrade all the way to O(n).

**Follow-up question:**
Why does `HashMap` apply an internal hash-spreading function on top of a key's own `hashCode()` instead of using it directly?

**Follow-up good answer:**
Because `HashMap` derives the bucket index from only the low-order bits of the hash (`hash & (capacity - 1)`, and capacities are typically small powers of two), a `hashCode()` implementation whose variation lives mostly in the high-order bits would produce lots of bucket collisions despite technically returning "different" hash codes for different keys. `HashMap`'s internal spread function XORs the high 16 bits of the hash into the low 16 bits before masking, so that high-order-bit variation also influences which bucket a key lands in — a defense against otherwise-reasonable `hashCode()` implementations that happen to vary mostly in bits the raw bucket-index calculation would ignore.

**Glossary:**
- **Hash spreading** — `HashMap`'s internal step of mixing a key's raw `hashCode()` bits before masking to a bucket index, to better use high-order bits that a simple mask would otherwise discard.

**Mental model:**
A synthesis question — pulls together resize/load-factor (Q2), treeification (Q3), and the hashCode contract (Q1) into one coherent story, checking whether the candidate has an integrated mental model of `HashMap` rather than a set of disconnected facts.

**References:**
- [HashMap — Java SE 21 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
- [HashMap.java — OpenJDK source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/HashMap.java)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=java&tags=collections-and-streams-internals&autostart=1" | relative_url }})
