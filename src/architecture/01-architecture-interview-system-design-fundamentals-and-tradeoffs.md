---
layout: default
title: "Architecture Interview — System Design Fundamentals & Trade-offs"
---

# Architecture Interview — System Design Fundamentals & Trade-offs

This set covers the foundational vocabulary and theory behind system design interviews: latency vs throughput, scaling strategies, consistency models, the CAP/PACELC theorems, quorum replication, capacity estimation, and the core building blocks (caches, queues, sharding, rate limiters) that almost every design question eventually touches.

### Q1. What's the difference between latency and throughput, and why can optimizing for one hurt the other? {#q1}

**Question:**
What's the difference between latency and throughput, and why can optimizing for one hurt the other?

**Good answer:**
Latency is how long a single request takes end-to-end; throughput is how many requests the system completes per unit time. They're related but not the same axis: a system can have low latency and low throughput (a single fast worker), or high latency and high throughput (a batching pipeline that groups requests to amortize overhead). The conflict shows up directly in batching and buffering: increasing batch size or queue depth improves throughput (more work processed per fixed overhead) but increases the latency of any individual request sitting in that batch/queue. Networking makes this concrete too — TCP throughput over a link is capped by the relationship between bandwidth and round-trip latency (the bandwidth-delay product), so a high-latency path needs a bigger window to hit the same throughput as a low-latency one.

**Follow-up question:**
You're asked to design a logging pipeline. Would you optimize primarily for latency or throughput, and how does that choice change the design?

**Follow-up good answer:**
Logging is almost always a throughput-optimized, latency-tolerant workload — nobody needs a log line durably indexed within a millisecond, but the system does need to sustain a high sustained write rate without falling behind. That pushes the design toward batching writes client-side, buffering into a queue (e.g. Kafka), and writing to storage in bulk rather than one round trip per log line. The trade-off is explicit: an individual log line might take seconds to become queryable, but the pipeline can absorb orders of magnitude more events per second than a naive per-event synchronous write would allow.

**Glossary:**
- **Latency** — time for a single operation to complete, end-to-end.
- **Throughput** — number of operations completed per unit of time.
- **Bandwidth-delay product** — the amount of in-flight, unacknowledged data a connection needs to fully utilize available bandwidth given its round-trip latency.

**Mental model:**
This checks whether the candidate treats "performance" as a single number or understands it as (at least) two independent axes that often trade off against each other — a distinction that shapes almost every subsequent design decision (batch size, queue depth, consistency window).

**TL;DR:**
Latency is per-request time, throughput is requests-per-second — batching/buffering trades one for the other, so the right choice depends on which axis the workload actually needs.

**References:**
- [AWS Well-Architected Framework — Network architecture selection (Performance Efficiency Pillar)](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/network-architecture-selection.html)

---

### Q2. What's the difference between vertical and horizontal scaling, and when would you reach for each? {#q2}

**Question:**
What's the difference between vertical and horizontal scaling, and when would you reach for each?

**Good answer:**
Vertical scaling ("scaling up") means adding more capacity — CPU, RAM, faster disks — to a single existing node, e.g. moving an EC2 instance from a t3.medium to an m5.large. Horizontal scaling ("scaling out") means adding more nodes and distributing load across them, e.g. adding more instances behind a load balancer. Vertical scaling is simpler (no distributed-systems complexity, no data-partitioning problem) but is bounded by the largest machine you can buy, usually requires downtime to resize, and keeps a single point of failure. Horizontal scaling has effectively unlimited headroom and gives you fault tolerance for free (lose one node, the others keep serving), but it requires the workload to actually be distributable — stateless application tiers scale out trivially; a single-writer relational database does not, without sharding or a fundamentally different architecture.

**Follow-up question:**
Why is it common in practice to scale application servers horizontally but scale the primary database vertically first?

**Follow-up good answer:**
Application servers behind a load balancer are typically stateless (or push state to a shared cache/database), so adding another identical instance is nearly free and immediately increases capacity. A primary relational database is a single writer with strong consistency guarantees across all its data; splitting writes across multiple nodes (sharding) means giving up cross-shard transactions and joins, or building application-level logic to route and stitch queries — real engineering cost. So teams usually get more mileage per unit of complexity by buying a bigger database instance first (more CPU/RAM/IOPS) and only shard once vertical scaling hits a real ceiling (cost, or a single machine's absolute limits).

**Glossary:**
- **Vertical scaling (scale up)** — increasing the resources of a single node.
- **Horizontal scaling (scale out)** — adding more nodes and distributing load across them.
- **Sharding** — splitting a dataset across multiple database instances by some partition key.

**Mental model:**
Tests whether the candidate can match a scaling strategy to a component's actual constraints (stateless vs. stateful, single-writer vs. distributable) rather than reflexively saying "just add more servers" for every tier.

**TL;DR:**
Vertical scaling is simpler but capped by one machine's limits and keeps a single point of failure; horizontal scaling has near-unlimited headroom but requires the workload to be distributable — which is why stateless app tiers scale out and single-writer databases usually scale up first.

**References:**
- [Scaling Your Amazon RDS Instance Vertically and Horizontally — AWS Database Blog](https://aws.amazon.com/blogs/database/scaling-your-amazon-rds-instance-vertically-and-horizontally/)

---

### Q3. How do you approach back-of-envelope capacity estimation in a system design interview? {#q3}

**Question:**
How do you approach back-of-envelope capacity estimation in a system design interview?

**Good answer:**
Start from the given (or reasonably assumed) scale — daily/monthly active users, or a requests-per-day figure — and convert it into requests-per-second, accounting for peak-to-average ratios (traffic is rarely uniform across a day; a 2-5x peak multiplier over the daily average is a common rule of thumb). From QPS, derive downstream numbers: storage growth per day/year given an average payload size, bandwidth needed given payload size × QPS, and how many application/database instances are needed given a known single-instance capacity. The point isn't decimal precision — it's sanity-checking that the design's components (cache size, database IOPS, network links) are in the right order of magnitude for the load, and knowing which resource becomes the bottleneck first. Grounding these estimates requires internalizing rough orders of magnitude for common operations (memory access vs. disk seek vs. network round-trip), since those differences of several orders of magnitude are what actually drive architecture decisions like "you need a cache here."

**Follow-up question:**
Why does an order-of-magnitude memory-vs-disk-vs-network latency table matter for a design decision, not just as trivia?

**Follow-up good answer:**
Because those numbers directly justify architectural choices. A main-memory reference (~100 ns) versus a disk seek (several milliseconds) is a difference of roughly four to five orders of magnitude — that gap alone is the entire justification for putting a cache in front of a database, since a cache hit can be 10,000x+ faster than falling through to disk. Similarly, a same-datacenter round trip (roughly hundreds of microseconds) versus a cross-continent round trip (~150 ms) is why globally distributed systems use regional replicas or CDNs instead of routing every request to one origin region — the physical latency floor set by the speed of light cannot be engineered away, only avoided by moving data/compute closer to the user.

**Glossary:**
- **QPS** — queries per second, a standard capacity unit.
- **Peak-to-average ratio** — how much higher peak traffic is compared to the daily average; used to size for worst-case, not average, load.

**Mental model:**
Tests whether the candidate can turn a vague scale requirement into concrete numbers that expose the actual bottleneck, and whether they have internalized the latency hierarchy (memory/disk/network) that justifies caching and geographic distribution — not just parroting a memorized "cache in front of DB" pattern.

**TL;DR:**
Convert user/traffic scale into QPS (with a peak multiplier), then propagate that into storage/bandwidth/instance-count estimates — the goal is catching order-of-magnitude bottlenecks, and the memory/disk/network latency hierarchy is what justifies caching and regional distribution.

**References:**
- [Peter Norvig — "Teach Yourself Programming in Ten Years" (latency numbers)](https://norvig.com/21-days.html#answers)

---

### Q4. Walk through what happens to a request as it passes through a load balancer, a cache, and a database — what does each layer actually buy you? {#q4}

**Question:**
Walk through what happens to a request as it passes through a load balancer, a cache, and a database — what does each layer actually buy you?

**Good answer:**
A load balancer sits in front of a pool of application servers and distributes incoming requests across them (round-robin, least-connections, or latency-based), so no single instance is a bottleneck and the fleet can scale horizontally; it also removes unhealthy instances from rotation. The application server, on receiving a request, typically checks a cache (e.g. Redis/Memcached) before hitting the database — a cache hit returns in-memory speed (sub-millisecond) instead of paying a disk-backed database round trip. On a cache miss, the request falls through to the database, which is the authoritative, durable store but is also the slowest and least horizontally scalable link in the chain. Each layer exists to protect the one behind it from load it can't handle: the load balancer protects app servers from being overwhelmed by uneven traffic, and the cache protects the database from being hit by every single read.

**Follow-up question:**
If the cache and database disagree after a write, which one is "right," and how do you keep them from staying wrong for long?

**Follow-up good answer:**
The database is the source of truth — the cache is a lossy, disposable performance optimization, never the authoritative record. Divergence ("stale cache") is kept short-lived via a write strategy: write-through updates the cache synchronously on every database write, so it's never stale but pays two round trips per write; lazy loading (cache-aside) only populates the cache on a read miss, so a stale entry persists until it's evicted or expires — which is why lazy loading is almost always paired with a TTL, bounding the maximum staleness window even if no explicit invalidation happens.

**Glossary:**
- **Load balancer** — distributes incoming requests across a pool of servers.
- **Cache hit/miss** — whether requested data was found in the cache without querying the backing store.
- **Write-through / lazy loading (cache-aside)** — strategies for keeping cache and database in sync.

**Mental model:**
Probes whether the candidate understands each component's purpose as "protecting the next slower/more expensive layer from load," not just as a checklist of buzzwords to draw in a design diagram.

**TL;DR:**
Each layer exists to shield the next, slower one from load: the load balancer protects app servers, the cache protects the database — and the database, not the cache, is always the source of truth, with TTLs bounding how stale a cache entry can get.

**References:**
- [Caching strategies for Memcached — Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html)

---

### Q5. State the CAP theorem precisely. Is "pick 2 of 3" actually correct? {#q5}

**Question:**
State the CAP theorem precisely. Is "pick 2 of 3" actually correct?

**Good answer:**
No — Eric Brewer himself has said the popular "pick 2 of 3" framing is misleading. Partition tolerance isn't optional in a real distributed system: network partitions (dropped or arbitrarily delayed messages between nodes) will happen, so a system must be built to tolerate them. The actual, meaningful choice is between Consistency and Availability, and only during an active partition: when nodes can't communicate, you either refuse to serve some requests to keep every node's view consistent (favor C), or you keep serving from whichever nodes are reachable and accept that they may return divergent/stale data (favor A). Outside of a partition — the normal, common case — there is no need to sacrifice either consistency or availability; good distributed systems detect a partition, explicitly enter a degraded "partition mode" with a defined trade-off, and reconcile once connectivity is restored.

**Follow-up question:**
Given that, is it accurate to describe a system as simply "a CP system" or "an AP system"?

**Follow-up good answer:**
It's a common shorthand, but strictly it's imprecise, because the CAP trade-off only bites during a partition — a system's behavior outside of a partition isn't described by CAP at all (that's what PACELC extends to cover). "CP" or "AP" really describes what the system chooses to do specifically when a partition is detected, and even that choice is often not global: many real systems (Brewer's own recommendation) partition their operations, treating some as safe to serve during a partition (AP) and others as requiring consistency (CP) at a finer granularity than "the whole system."

**Glossary:**
- **Partition** — a communication failure between nodes such that messages are dropped or arbitrarily delayed.
- **Consistency (in CAP)** — every read receives the most recent write or an error.
- **Availability (in CAP)** — every request receives a (non-error) response, without guarantee it's the most recent write.

**Mental model:**
Distinguishes candidates who've memorized "CAP: pick 2 of 3" as trivia from those who understand it's a conditional trade-off that only activates under partition — a much more defensible, interview-differentiating answer.

**TL;DR:**
Partition tolerance isn't optional in the real world, so the real CAP trade-off is consistency vs. availability, and only while a partition is actually happening — not an always-on "pick 2 of 3."

**References:**
- [CAP Twelve Years Later: How the "Rules" Have Changed — Eric Brewer, InfoQ](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)

---

### Q6. What does PACELC add on top of CAP? {#q6}

**Question:**
What does PACELC add on top of CAP?

**Good answer:**
PACELC, proposed by Daniel Abadi, points out that CAP only describes a trade-off during a partition (P), but partitions are rare — the trade-off that matters essentially all the time is the "else" (E) case: even when the system is running normally with no partition, you still have to choose between Latency (L) and Consistency (C). Enforcing strong consistency (e.g. synchronously replicating a write to a quorum of replicas before acknowledging it) adds latency compared to acknowledging as soon as one node has the write and replicating asynchronously. So the full formulation is: if Partitioned, choose Availability or Consistency; Else, choose Latency or Consistency. This reframes system classification from a single CAP letter pair into four combinations (e.g. PC/EC — always prioritizes consistency, PA/EL — always prioritizes availability/low latency, and the mixed PC/EL and PA/EC).

**Follow-up question:**
Where does a system like DynamoDB's default read behavior fall on the PACELC spectrum, and why?

**Follow-up good answer:**
DynamoDB's default read is eventually consistent, and it's explicitly cheaper (half the cost of a strongly consistent read) and lower-latency, because it can be served by any replica without coordinating to find the most recent write. That's an EL choice in the non-partition case — trading consistency for lower latency and cost during normal operation. DynamoDB also offers an opt-in strongly consistent read for cases where staleness is unacceptable, which trades that latency/cost benefit back for consistency — making DynamoDB a system that lets the caller choose its point on the PACELC spectrum per request rather than being locked into one.

**Glossary:**
- **PACELC** — "if Partitioned: Availability or Consistency; Else: Latency or Consistency."
- **Eventually consistent read** — a read that may not reflect the most recent write, but converges over time.

**Mental model:**
Tests whether the candidate sees CAP as a special case (the rare partition scenario) rather than the whole picture, and whether they can name the far more common latency/consistency trade-off that shapes system design decisions made every day, not just during outages.

**TL;DR:**
PACELC says CAP only covers the rare partition case — the trade-off you make constantly, partition or not, is latency vs. consistency, and DynamoDB's cheaper eventually-consistent-by-default read is a concrete example of choosing latency.

**References:**
- [PACELC theorem — Wikipedia](https://en.wikipedia.org/wiki/PACELC_theorem)
- [DynamoDB read consistency — Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)

---

### Q7. Explain strong, eventual, and causal consistency — what does each actually guarantee an application author? {#q7}

**Question:**
Explain strong, eventual, and causal consistency — what does each actually guarantee an application author?

**Good answer:**
Strong consistency guarantees that any read after a write returns that write (or a later one) — every client sees the same, most-up-to-date value, as if there were only one copy of the data, though achieving this in a replicated system requires coordination (like a quorum read/write or routing all reads to a single primary) that costs latency. Eventual consistency only guarantees that, absent new writes, all replicas will *eventually* converge to the same value — a read shortly after a write may return stale data on some replica, with no bound on how stale beyond "eventually." Causal consistency sits between the two: it guarantees that operations which are causally related (e.g. a comment written after reading a post) are seen by every observer in that same order, while operations with no causal relationship (two independent users posting at the same time) may be seen in different orders by different observers. It's a meaningfully weaker guarantee than strong consistency but stronger and more intuitive than plain eventual consistency, at a lower coordination cost than full strong consistency.

**Follow-up question:**
DynamoDB's default read is eventually consistent. What could an application observe as a result, and how would you avoid it for a specific request that needs freshness?

**Follow-up good answer:**
An application could write an item and then immediately read it back (e.g. from a different replica or a global secondary index) and get the pre-write value, or a `null`/not-found for a very recent write, because eventually consistent reads "might not reflect the results of a recently completed write operation." To avoid this for a specific request, DynamoDB exposes an explicit `ConsistentRead` parameter on `GetItem`/`Query`/`Scan` — setting it to true forces a strongly consistent read that reflects all prior successful writes, at roughly double the read cost and (implicitly) at the latency cost of not being servable from just any replica. Note this option isn't available on global secondary indexes or streams, which only support eventually consistent reads.

**Glossary:**
- **Strong consistency** — every read reflects the most recent write.
- **Eventual consistency** — replicas converge to the same value over time, with no fixed staleness bound.
- **Causal consistency** — causally related operations are observed in the same order by everyone; unrelated ones may not be.

**Mental model:**
Checks whether the candidate can reason precisely about what a consistency model actually promises an application (not just recite the three names), since picking the wrong one is a common source of subtle production bugs (e.g. "I just saved it and it's not there").

**TL;DR:**
Strong consistency guarantees every read sees the latest write; eventual consistency only guarantees convergence with no staleness bound; causal consistency guarantees order for related operations only — and DynamoDB lets you opt into strong consistency per-read via `ConsistentRead` when you need it.

**References:**
- [DynamoDB read consistency — Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)

---

### Q8. What does it mean for a quorum system to satisfy W + R > N, and why does that guarantee consistency? {#q8}

**Question:**
What does it mean for a quorum system to satisfy W + R > N, and why does that guarantee consistency?

**Good answer:**
N is the number of replicas a piece of data is stored on; W is the minimum number of replicas that must acknowledge a write for it to be considered successful; R is the minimum number of replicas a read must contact before returning a result. If W + R > N, any read quorum and any write quorum are mathematically guaranteed to overlap in at least one replica — so any read is guaranteed to contact at least one node that has the most recent write, and can return the freshest value (e.g. by comparing versions/timestamps across the replicas it read from). Amazon's Dynamo popularized this configurable trade-off: with N=3, a common configuration is W=2, R=2 (2+2=4 > 3), which tolerates one replica being unavailable for either a read or a write while still guaranteeing overlap, without requiring all N replicas to participate in every operation (which would hurt both latency and availability).

**Follow-up question:**
What happens if a system instead chooses W + R ≤ N — what does it gain and what does it give up?

**Follow-up good answer:**
It gains latency and availability: fewer replicas need to respond for an operation to succeed, so a request can complete faster and tolerate more simultaneous replica failures/slowness before failing outright. What it gives up is the read-your-writes guarantee that quorum overlap provided — because there's no guaranteed overlap between the specific replicas a given read and a given write touched, a read can return a value older than the most recent successful write. This is exactly the availability/consistency trade-off from CAP, expressed as a tunable knob rather than a fixed system-wide choice — Dynamo-style systems let each operation pick its own point on that spectrum.

**Glossary:**
- **N** — number of replicas holding a piece of data.
- **W (write quorum)** — minimum replicas that must ack a write.
- **R (read quorum)** — minimum replicas a read must contact.

**Mental model:**
Tests understanding of quorum-based replication as a concrete, tunable mechanism for the CAP trade-off, not just an abstract theorem — this is the kind of detail that separates candidates who've built or deeply studied a distributed datastore from those who've only heard the buzzwords.

**TL;DR:**
W + R > N guarantees every read quorum and write quorum share at least one replica, so a read is guaranteed to see the latest write; dropping below that threshold trades that guarantee away for lower latency and higher availability.

**References:**
- [Dynamo: Amazon's Highly Available Key-value Store (DeCandia et al., SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

---

### Q9. A service's p99 latency has doubled. How do you figure out whether it's CPU-, memory-, disk-, or network-bound? {#q9}

**Question:**
A service's p99 latency has doubled. How do you figure out whether it's CPU-, memory-, disk-, or network-bound?

**Good answer:**
Use a systematic checklist rather than guessing — Brendan Gregg's USE method (Utilization, Saturation, Errors) applied per resource is a good structure: for each candidate resource (CPU, memory, disk, network), check its utilization (percentage of time busy), saturation (how much work is queued waiting because the resource can't keep up — e.g. run-queue length for CPU, swap activity for memory), and errors (which degrade performance and are often overlooked, e.g. retransmits on the network, ENOSPC on disk). A resource near 100% utilization with significant queued/waiting work is a strong bottleneck signal; conversely, low utilization across all four resources despite high latency usually points to something outside this framework — e.g. lock contention or a slow downstream dependency — rather than a hardware resource limit.

**Follow-up question:**
Your CPU utilization is only 40% but requests are still slow. What does the USE method suggest you check next, and why might low utilization be misleading?

**Follow-up good answer:**
Check saturation and errors on the *other* resources, and also consider that utilization can be misleadingly low even when a resource is actually the bottleneck — e.g. a single-threaded hot path can pin one core at 100% while overall CPU utilization (averaged across all cores) looks moderate. The USE method's "saturation" check exists precisely for this: a resource can be underutilized on average while still having a growing queue of waiting work at specific moments or on specific cores/queues, which average utilization numbers hide. If all four resources show low utilization and low saturation, the bottleneck is likely non-hardware — lock contention, a slow synchronous call to another service, or an inefficient algorithm — which the USE method flags by elimination rather than by itself diagnosing.

**Glossary:**
- **Utilization** — percentage of time a resource is busy servicing work.
- **Saturation** — the degree to which work is queued because a resource can't service it immediately.
- **USE method** — Utilization, Saturation, Errors — a systematic per-resource performance-triage checklist.

**Mental model:**
Tests whether the candidate has an actual, repeatable methodology for performance triage rather than an ad hoc "check CPU, then guess" approach, and whether they understand utilization alone can be a misleading single metric.

**TL;DR:**
Apply the USE method (Utilization, Saturation, Errors) to each resource in turn — a resource with high utilization and growing saturation is your bottleneck; low utilization everywhere points away from hardware and toward contention or a slow dependency.

**References:**
- [The USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html)

---

### Q10. How do you validate, before writing any code, that a proposed design will actually meet a stated latency/throughput target? {#q10}

**Question:**
How do you validate, before writing any code, that a proposed design will actually meet a stated latency/throughput target?

**Good answer:**
Decompose the target into per-component budgets and check each against known or estimated capacity: given a required end-to-end p99 latency, allocate a latency budget across the network hop, load balancer, application logic, cache lookup, and database query, and sanity-check each slice against realistic numbers for that component (e.g. a cross-region round trip alone can consume most of a tight budget). For throughput, take the required QPS and divide by a single instance's tested/estimated capacity to get a minimum fleet size, then check that fits within cost and infrastructure constraints (e.g. database connection limits, cache memory size for the expected working set). Where real numbers aren't available, look for public benchmarks or documented service limits (e.g. a managed cache or queue's published throughput ceiling) rather than guessing, and flag explicitly which numbers are assumptions to be validated with a load test once a prototype exists.

**Follow-up question:**
Why is a load test still necessary even after this back-of-envelope validation?

**Follow-up good answer:**
Back-of-envelope math necessarily uses simplified, average-case assumptions (uniform request cost, independent bottlenecks, no contention) that real systems violate — actual traffic has hot keys and non-uniform payload sizes, components interact under load in ways that are hard to model analytically (e.g. cache eviction pressure, connection pool exhaustion, GC pauses under memory pressure), and tail latencies (p99, p999) are driven by exactly the rare, hard-to-estimate interactions that an average-case calculation smooths over. A load test exercises the actual system under realistic (or synthetic worst-case) conditions and surfaces those emergent effects, which is why capacity estimation and load testing are complementary steps, not substitutes for each other.

**Glossary:**
- **Latency budget** — the portion of an end-to-end latency target allocated to one component in the request path.
- **p99 / p999 latency** — the latency below which 99% / 99.9% of requests complete; captures tail behavior that averages hide.

**Mental model:**
Checks whether the candidate treats capacity estimation as a real validation step with a clear method (budget decomposition), and whether they understand its limits — a senior answer explicitly distinguishes "sanity check" math from "proof," and knows a load test is still required.

**TL;DR:**
Break the target latency/throughput into per-component budgets and check each against known capacity numbers to catch order-of-magnitude problems early — then load-test, since back-of-envelope math can't capture real contention, hot keys, or tail-latency effects.

**References:**
- [AWS Well-Architected Framework — Performance Efficiency Pillar](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/welcome.html)

---

### Q11. What does Amdahl's Law say, and why does it mean "just add more servers" has diminishing returns? {#q11}

**Question:**
What does Amdahl's Law say, and why does it mean "just add more servers" has diminishing returns?

**Good answer:**
Amdahl's Law, from Gene Amdahl's 1967 paper, states that the maximum speedup achievable by parallelizing a workload is fundamentally limited by the fraction of that workload that must remain sequential — even with an infinite number of parallel processors driving the parallelizable portion's time toward zero, the total time can never drop below the time required for the sequential portion alone. Concretely: if 10% of a task is inherently sequential, the maximum possible speedup from parallelization is capped at 10x, no matter how many processors you throw at the other 90%. This generalizes directly to distributed systems: any part of a request path that can't be parallelized — a single-writer database, a global lock, a serialized coordination step — becomes the hard ceiling on how much horizontal scaling can improve throughput, and each additional processor/node added beyond that point yields progressively less benefit.

**Follow-up question:**
You have a request pipeline where 20% of the time is spent in a step that must run on a single node. What's the maximum theoretical speedup from adding unlimited parallelism to the other 80%?

**Follow-up good answer:**
By Amdahl's Law, maximum speedup = 1 / (sequential fraction) as parallel processors approach infinity, so with a 20% sequential fraction the ceiling is 1 / 0.2 = 5x — no amount of additional parallel capacity on the remaining 80% can push the total speedup past 5x, because that sequential 20% represents a fixed floor on total time. This is exactly why identifying and eliminating (or further parallelizing) the sequential bottleneck is usually a higher-leverage investment than adding more nodes once a design already has significant parallelism — the returns on more hardware shrink fast as you approach that ceiling.

**Glossary:**
- **Sequential fraction** — the portion of a workload that cannot be parallelized, forming a hard floor on total execution time.
- **Speedup** — the ratio of time-without-parallelism to time-with-parallelism.

**Mental model:**
Tests whether the candidate can reason quantitatively about *why* horizontal scaling has limits, rather than treating "add more instances" as an unconditionally valid answer to every throughput problem.

**TL;DR:**
Amdahl's Law caps the maximum speedup from parallelism at 1/(sequential fraction) — so any serialized part of a pipeline (a single-writer DB, a global lock) sets a hard ceiling that more servers alone can't push past.

**References:**
- [Amdahl's law — Wikipedia](https://en.wikipedia.org/wiki/Amdahl%27s_law)

---

### Q12. Why do caching layers exist at all — what breaks without one? {#q12}

**Question:**
Why do caching layers exist at all — what breaks without one?

**Good answer:**
Without a cache, every read for frequently-requested data hits the database directly, and databases are the most expensive, least horizontally-scalable, and slowest (disk-backed) component in most architectures. As read traffic grows, the database becomes the bottleneck long before the application tier does, because scaling a database (especially a single-writer relational one) is fundamentally harder than scaling stateless application servers. A cache absorbs the vast majority of reads for "hot" data in-memory, at latencies orders of magnitude lower than a database round trip, which both reduces user-facing latency and — just as importantly — reduces load on the database so it has headroom for the writes and complex queries that can't be cached away. The trade-off it introduces is staleness risk, managed via TTLs and explicit invalidation strategies.

**Follow-up question:**
If a cache node fails and is replaced with an empty one, what happens to the system, and is that acceptable?

**Follow-up good answer:**
With a lazy-loading (cache-aside) strategy, an empty cache node is not fatal: every request that would have been a cache hit becomes a cache miss, falls through to the database, and repopulates the cache as it goes — the application keeps functioning, just with temporarily elevated latency and increased database load until the cache warms back up. Whether that's acceptable depends on the database's spare capacity: if the database was already near its own limits, a sudden flood of cache misses (a "thundering herd") from an empty cache can itself cause an outage, which is why some systems pre-warm a replacement cache node or throttle/queue requests during cache recovery rather than exposing the database to the full miss rate at once.

**Glossary:**
- **Cache-aside (lazy loading)** — populate the cache only on a read miss.
- **Thundering herd** — a surge of simultaneous requests (e.g. after a cache is cleared) overwhelming a backing store.

**Mental model:**
Probes whether the candidate understands caching as solving a specific, concrete scaling problem (protecting a hard-to-scale database) rather than treating "add a cache" as a reflexive answer with no grounding in what failure mode it prevents.

**TL;DR:**
Caches exist because databases are the hardest component to scale — absorbing hot reads in-memory protects the database from becoming the bottleneck, at the cost of managed staleness.

**References:**
- [Caching strategies for Memcached — Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html)

---

### Q13. Why introduce a message queue between a producer and a consumer instead of having the producer call the consumer directly? {#q13}

**Question:**
Why introduce a message queue between a producer and a consumer instead of having the producer call the consumer directly?

**Good answer:**
A direct call couples the producer's success to the consumer's availability and speed in real time: if the consumer is down, slow, or overwhelmed, the producer either blocks, fails, or has to implement its own retry/backoff logic. A queue decouples them — the producer enqueues a message and moves on regardless of the consumer's current state; the consumer dequeues and processes messages at its own pace. This buys several concrete properties: the producer doesn't need to know how many consumers exist or their addresses (new consumers can be added with no producer changes), temporary consumer downtime doesn't lose data (messages wait durably in the queue), and traffic spikes can be absorbed by letting the queue grow temporarily rather than requiring the consumer to instantaneously scale to match.

**Follow-up question:**
What failure mode does a queue introduce that a direct synchronous call doesn't have?

**Follow-up good answer:**
Unbounded queue growth if the consumer falls permanently behind the producer's rate: unlike a direct call (which would simply fail fast or push back visibly), a queue can silently absorb backlog until it runs out of storage or until the delay between enqueue and processing becomes unacceptable for the use case — turning a capacity problem into a slowly-growing latency problem that's easy to miss without explicit monitoring on queue depth and consumer lag. This is why production queue-based systems need backpressure or alerting on queue depth, and why "decoupled" doesn't mean "the capacity mismatch problem went away" — it just moved from being immediately visible to being deferred.

**Glossary:**
- **Producer / consumer** — the component adding messages to a queue vs. the component reading and processing them.
- **Backpressure** — a mechanism that slows or rejects producers when downstream consumers can't keep up.

**Mental model:**
Tests whether the candidate can articulate the concrete decoupling benefits of a queue (not just "it's a best practice") and is aware that decoupling shifts a failure mode rather than eliminating it — a senior-level nuance.

**TL;DR:**
A queue decouples producer and consumer lifecycles and pace, giving durability and independent scaling — but it converts a visible synchronous failure into an easy-to-miss silent backlog if the consumer falls behind.

**References:**
- [What is Amazon Simple Queue Service? — Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)

---

### Q14. Why does sharding exist — what happens to a dataset that outgrows a single database node? {#q14}

**Question:**
Why does sharding exist — what happens to a dataset that outgrows a single database node?

**Good answer:**
A single database node has a hard ceiling on storage capacity, memory for caching hot data/indexes, and I/O throughput — once a dataset or its write rate exceeds what one node (even a vertically-scaled, maximally-sized one) can hold or serve, the only way forward is to split the data across multiple nodes. Sharding does this by partitioning the dataset by some key (a partition/hash key) so each shard holds a subset of the data and serves a subset of the traffic, letting total capacity and throughput scale roughly linearly with the number of shards. The cost is that operations spanning multiple shards (joins, multi-key transactions, global sorts/aggregations) become significantly harder — they either require querying every shard and merging results in the application, or are restricted/disallowed, which is a real architectural constraint that has to be designed around, not an afterthought.

**Follow-up question:**
What happens if your partition key is poorly chosen — say, most writes go to a small range of keys?

**Follow-up good answer:**
You get a "hot partition" — one shard absorbs disproportionate load while others sit idle, meaning the system's *effective* capacity is limited by that one overloaded shard even though aggregate capacity across all shards looks fine on paper. A common concrete cause is a monotonically increasing key (like a timestamp or auto-incrementing ID) that concentrates all new writes on whichever shard currently owns the newest key range. The standard fix is to add randomness or a more evenly-distributed attribute into the partition key (e.g. appending a random suffix or hashing a more uniformly distributed field) so writes spread evenly across shards — trading away easy range queries on that key for even load distribution.

**Glossary:**
- **Shard / partition** — a subset of a dataset stored on one node, split by a partition key.
- **Hot partition/key** — a shard or key receiving disproportionately high traffic relative to others.

**Mental model:**
Checks whether the candidate understands sharding as a response to a specific, hard physical ceiling (not a default "for scale" pattern to apply everywhere) and is aware that a poor partition key choice can defeat the entire point of sharding.

**TL;DR:**
Sharding splits data across nodes once one node's storage/throughput ceiling is exceeded — but it sacrifices easy cross-shard queries, and a poorly chosen partition key can create a hot shard that becomes the new bottleneck anyway.

**References:**
- [Using write sharding to distribute workloads evenly in your DynamoDB table — Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html)

---

### Q15. What does "premature optimization" mean in a system design context, and how do you avoid it without under-designing? {#q15}

**Question:**
What does "premature optimization" mean in a system design context, and how do you avoid it without under-designing?

**Good answer:**
The phrase traces to Donald Knuth's 1974 paper, with the fuller (and more useful) quote being: "We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%." In a system design context this means adding complexity — extra caching layers, sharding, microservices, exotic data stores — to solve a scaling problem that hasn't been shown to exist yet, at the cost of real engineering time and ongoing operational burden. Avoiding it without under-designing means distinguishing decisions that are cheap to defer (add a cache later, once you have real hit-rate data) from decisions that are expensive to retrofit (a partition key or a fundamental data model, which are painful to change once data exists) — the latter deserve real upfront thought even if the system doesn't need to be fully built out for that scale on day one.

**Follow-up question:**
In an interview, how do you signal you're not over-engineering the initial design while still showing awareness of what happens at scale?

**Follow-up good answer:**
Present a simple version first (a single database, no cache, a monolithic service) and explicitly name the assumption under which it's sufficient (e.g. "this handles up to roughly N QPS on a single reasonably-sized instance"), then walk through what breaks first as load grows and what you'd add at that specific point and why — this demonstrates the judgment to right-size a design for stated requirements while proving you know the scaling path exists, rather than either over-building unnecessary complexity from the start or being unable to describe how the system would evolve under real growth.

**Glossary:**
- **Premature optimization** — investing effort optimizing a part of a system before establishing it's actually a bottleneck.

**Mental model:**
This question, more than most, is testing engineering judgment and communication rather than raw knowledge — can the candidate justify *not* building something as confidently as they'd justify building it.

**TL;DR:**
Premature optimization is solving scaling problems you haven't confirmed exist — avoid it by starting simple and naming the specific threshold at which each piece of added complexity becomes justified, especially for decisions that are expensive to change later (data model, partition key).

**References:**
- [Donald Knuth — Wikiquote (citing "Structured Programming with Go To Statements," ACM Computing Surveys, 1974)](https://en.wikiquote.org/wiki/Donald_Knuth)

---

### Q16. How do you identify and eliminate single points of failure (SPOFs) in a design? {#q16}

**Question:**
How do you identify and eliminate single points of failure (SPOFs) in a design?

**Good answer:**
Walk the request path and, for every component, ask "if this one instance/node/service disappears, does the system keep working?" Any component where the answer is "no" is a SPOF. Common examples: a single application instance with no load balancer/failover, a single database with no replica, a single AZ or region deployment, or a component that's technically redundant but shares an underlying dependency (two app instances that both depend on one unreplicated database). The fix pattern is consistent — deploy redundant components across fault-isolation boundaries (multiple instances behind a load balancer, database replicas, multiple Availability Zones or regions) so that the failure of any one unit doesn't take down the whole system, and explicitly design and test the failover path rather than assuming redundancy alone is sufficient (a replica that's never been tested as a promotion target is a SPOF in practice, even if it exists).

**Follow-up question:**
You have two application instances behind a load balancer in the same Availability Zone. Is that sufficient redundancy? Why or why not?

**Follow-up good answer:**
No — the two instances are redundant against an individual instance failure, but the Availability Zone itself is now the SPOF: an AZ-level event (power, networking, or facility failure) takes down both instances simultaneously regardless of the instance-level redundancy. This is exactly the distinction AWS's Well-Architected Reliability Pillar draws with fault isolation zones — true redundancy means distributing across independent failure domains (multiple AZs, and for the highest reliability bar, multiple Regions), not just multiple instances within the same domain, because redundancy only helps against the specific failure modes it's actually isolated from.

**Glossary:**
- **Single point of failure (SPOF)** — any component whose failure alone can take down the system.
- **Fault isolation zone** — an independent failure domain (e.g. an Availability Zone or Region) such that a failure in one doesn't propagate to another.

**Mental model:**
Tests whether the candidate applies redundancy analysis recursively (checking the failure domain of the redundant components themselves), which is exactly where naive "I added a load balancer so it's fine" answers fall short.

**TL;DR:**
Find SPOFs by asking "does this survive the loss of any one component?" recursively — redundant instances in the same failure domain (e.g. one AZ) just move the SPOF up a level, so real resilience requires spreading across independent fault isolation zones.

**References:**
- [Reliability Pillar — AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)

---

### Q17. What problem does consistent hashing solve that naive `hash(key) % N` doesn't? {#q17}

**Question:**
What problem does consistent hashing solve that naive `hash(key) % N` doesn't?

**Good answer:**
With naive modulo hashing, changing N (adding or removing a node) changes the divisor for every single key's assignment — almost every key maps to a different node than before, meaning nearly the entire dataset has to be redistributed/re-cached just to add or remove one node. Consistent hashing, introduced by Karger et al. for relieving web cache hot spots, maps both nodes and keys onto a conceptual ring (via a hash function), and assigns each key to the next node clockwise on the ring. When a node is added or removed, only the keys that were mapped to the ring segment immediately affected by that node move — a small, bounded fraction of the total keyspace (proportional to 1/N) — rather than nearly everything. This is what makes it practical to scale a sharded cache or database cluster up or down incrementally without a massive, disruptive data-movement event each time.

**Follow-up question:**
Basic consistent hashing (one point per node on the ring) can still produce very uneven load if nodes land close together by chance. How is this usually fixed?

**Follow-up good answer:**
By using virtual nodes: instead of placing each physical node at a single point on the ring, each physical node is assigned many points (virtual nodes) spread across the ring — so its share of the keyspace is the sum of many smaller, independently-random segments rather than one large, luck-dependent segment. This averages out load much more evenly across physical nodes (the more virtual nodes per physical node, the more even the distribution tends to be) and also means that when a physical node is removed, its load is spread across many other nodes rather than dumped entirely onto whichever single node was next on the ring.

**Glossary:**
- **Consistent hashing** — a hashing scheme where changing the number of nodes only remaps a small, bounded fraction of keys.
- **Virtual nodes** — multiple ring positions assigned to one physical node, to smooth out load distribution.

**Mental model:**
Checks whether the candidate understands consistent hashing as solving a specific, quantifiable problem (minimizing remapped keys on resize) rather than just recognizing it as a system-design buzzword to drop into a sharding answer.

**TL;DR:**
Naive `hash(key) % N` remaps almost every key when N changes; consistent hashing bounds that to roughly 1/N of the keyspace by placing nodes and keys on a ring — and virtual nodes fix the uneven-load problem plain single-point consistent hashing can still have.

**References:**
- [Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web (Karger et al., STOC 1997) — Semantic Scholar](https://www.semanticscholar.org/paper/Consistent-hashing-and-random-trees:-distributed-on-Karger-Lehman/02bb762c3bd1b3d1ad788340d8e9cdc3d85f33e1)

---

### Q18. Compare the lazy-loading (cache-aside) and write-through caching strategies — what does each optimize for, and what does each risk? {#q18}

**Question:**
Compare the lazy-loading (cache-aside) and write-through caching strategies — what does each optimize for, and what does each risk?

**Good answer:**
Lazy loading (cache-aside) only writes to the cache in response to a read miss: the application checks the cache, and on a miss, queries the database and populates the cache with the result for next time. It optimizes for cache efficiency — only data that's actually requested ever occupies cache space — and node failures are non-fatal, since a fresh empty node just repopulates itself from subsequent misses. Its risk is staleness: because the cache is never updated on a write, data already cached can silently diverge from the database until it expires or is explicitly invalidated. Write-through instead updates the cache synchronously on every database write, so cached data is never stale, at the cost of extra latency on every write (two round trips: cache and database) and the risk of "cache churn" — caching data that's written but rarely if ever read, wasting space. In practice these are often combined: write-through for freshness, plus a TTL to bound staleness on any data that lazy-loading alone would leave unbounded.

**Follow-up question:**
Under a write-through strategy, a new (empty) cache node is added to the pool via consistent hashing. What data problem does this create, and how would you fix it?

**Follow-up good answer:**
The new node is empty, but write-through only populates the cache in response to *writes* — so any keys mapped to that new node that aren't written again soon will simply be missing (a permanent-feeling miss) rather than lazily backfilled, since the write-through strategy has no read-triggered population path by itself. The standard fix, as AWS's own ElastiCache guidance notes, is to combine write-through with lazy loading: reads that miss on the new node still fall through to the database and populate the cache, giving the node a self-healing path for data that write-through alone wouldn't restore.

**Glossary:**
- **Lazy loading (cache-aside)** — populate the cache only on a read miss.
- **Write-through** — populate/update the cache synchronously on every write.
- **Cache churn** — wasted cache space from caching data that's rarely read.

**Mental model:**
Probes whether the candidate can reason about the specific failure modes each caching strategy leaves open (staleness for lazy loading, missing-data-on-new-nodes for write-through-only), rather than treating "add a cache" as a single undifferentiated pattern.

**TL;DR:**
Lazy loading only caches what's actually read (efficient, but can go stale); write-through keeps the cache always fresh (safer, but adds write latency and can leave new/empty nodes with permanently missing data unless combined with lazy loading).

**References:**
- [Caching strategies for Memcached — Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html)

---

### Q19. Compare token bucket, leaky bucket, and sliding-window rate limiting — how does each behave under a burst of traffic? {#q19}

**Question:**
Compare token bucket, leaky bucket, and sliding-window rate limiting — how does each behave under a burst of traffic?

**Good answer:**
Token bucket maintains a bucket that refills with tokens at a fixed rate up to some maximum capacity; each request consumes one token, and requests are rejected once the bucket is empty. Because unused tokens accumulate up to the bucket's capacity while a client is idle, it explicitly allows bursts — a client that hasn't made requests in a while can fire off a burst up to the bucket's capacity all at once, then must wait for tokens to refill. Leaky bucket instead processes (or forwards) requests at a strictly constant output rate regardless of how bursty the input is — excess incoming requests queue up (or are dropped once the queue is full) rather than being let through in a burst, which protects a downstream system that genuinely cannot handle bursts even briefly. Sliding-window approaches count requests within a continuously moving time window (rather than a bucket of tokens), which avoids the "boundary burst" problem of naive fixed windows (where a client could send a full window's worth of requests right at the end of one window and again right at the start of the next, doubling the effective rate briefly) while still allowing some natural burstiness within the window.

**Follow-up question:**
AWS API Gateway's throttling is described as token-bucket-based, with both a steady-state rate and a burst limit. What do those two numbers each control?

**Follow-up good answer:**
The rate (requests per second) controls how quickly the bucket refills with tokens — this is the sustainable, steady-state throughput the API can handle indefinitely. The burst limit controls the token bucket's maximum capacity — the largest number of concurrent/near-simultaneous requests API Gateway will accept in a short window before it starts returning `429 Too Many Requests`, even if the sustained rate hasn't been exceeded on average. Together they let API Gateway absorb realistic short spikes (via the burst capacity) while still enforcing a hard ceiling on sustained load (via the refill rate) that protects backend integrations from being overwhelmed.

**Glossary:**
- **Token bucket** — refills at a fixed rate, allows bursts up to bucket capacity.
- **Leaky bucket** — processes at a strictly constant rate, smoothing out bursts.
- **Sliding window** — counts requests in a continuously moving time window, avoiding fixed-window boundary bursts.

**Mental model:**
Tests whether the candidate can match a rate-limiting algorithm's specific behavioral profile (does it allow bursts? does it smooth them?) to the actual requirement (protect a fragile downstream vs. give API consumers reasonable burst tolerance), rather than treating all three as interchangeable "rate limiter" trivia.

**TL;DR:**
Token bucket allows accumulated bursts up to a cap; leaky bucket forces a constant output rate regardless of input burstiness; sliding window avoids the fixed-window boundary-doubling problem — pick based on whether the downstream can tolerate bursts at all.

**References:**
- [Throttle requests to your REST APIs for better throughput in API Gateway — Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)

---

### Q20. What access-pattern characteristics should actually drive a choice between a SQL and a NoSQL database? {#q20}

**Question:**
What access-pattern characteristics should actually drive a choice between a SQL and a NoSQL database?

**Good answer:**
The decision should follow from how the data is queried and how it needs to scale, not from "SQL is old, NoSQL is modern" framing. A relational (SQL) database is the right default when data has complex relationships that need ad hoc, flexible querying — joins across many entities, multi-row transactions, and a schema that benefits from being enforced up front — and when the write/read volume fits comfortably on a scaled-up single writer (or a small number of read replicas). A NoSQL database fits better when access patterns are known in advance and narrow (e.g. "always fetch this item by this key"), when the schema needs to be flexible or evolve without migrations, and — most importantly for the scaling argument — when the required throughput or dataset size genuinely exceeds what a single-writer relational system can serve, since most NoSQL stores are designed from the ground up for horizontal partitioning across many nodes. In practice, many real systems use both: a relational store for the complex, transactional core and a NoSQL store for a specific high-scale, simple-access-pattern subset of the data.

**Follow-up question:**
A team picks a NoSQL database purely because "it will scale better later," without knowing their access patterns yet. What's the risk?

**Follow-up good answer:**
Most NoSQL databases require you to design the data model (especially the partition/sort key structure) around your known access patterns up front, precisely because they don't offer the flexible ad hoc querying (arbitrary joins, secondary-attribute filtering) that a relational database provides for free — if the access patterns aren't actually known yet, there's a real risk of modeling the data in a way that makes a common query expensive or impossible without a costly migration later. This is the inverse of the relational database's trade-off: SQL defers the "how will this be queried" decision to query time at some performance cost, while most NoSQL databases require committing to it at data-modeling time in exchange for scalability — choosing NoSQL before access patterns are known trades away exactly the flexibility that would have covered for that uncertainty.

**Glossary:**
- **Access pattern** — the specific, expected ways data will be queried (by what key, what filters, what joins).
- **Partition key** — the attribute a NoSQL store uses to distribute items across nodes, chosen based on expected access patterns.

**Mental model:**
Checks whether the candidate makes this decision based on concrete access-pattern and scale reasoning rather than trend-following, and whether they understand the up-front data-modeling cost NoSQL imposes in exchange for its scalability.

**TL;DR:**
Choose SQL when you need flexible ad hoc querying and transactions at a scale one writer can handle; choose NoSQL when access patterns are known and narrow and scale genuinely exceeds a single writer — picking NoSQL before access patterns are known just trades query flexibility for scalability you may not need yet.

**References:**
- [What is a NoSQL Database? — AWS](https://aws.amazon.com/nosql/)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=architecture&tags=system-design-fundamentals-and-tradeoffs&autostart=1" | relative_url }})
