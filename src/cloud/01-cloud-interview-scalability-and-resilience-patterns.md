---
layout: default
title: "Cloud Interview: Scalability & Resilience Patterns"
---

# Cloud Interview — Scalability & Resilience Patterns

Twenty questions on how cloud systems scale and stay available under failure:
horizontal vs. vertical scaling, load balancing internals, auto scaling,
rate limiting, retries, circuit breakers, bulkheads, multi-AZ/region design,
deployment strategies, and the performance-diagnosis methodology that's now
a staple of senior cloud interviews. Examples lean on AWS (the most commonly
interviewed cloud platform) but the underlying patterns are vendor-neutral.

### Q1. What's the difference between horizontal and vertical scaling, and why do cloud architectures default to horizontal?

**Question:**
Explain horizontal vs. vertical scaling. Why does AWS's Well-Architected Framework recommend horizontal scaling as the default strategy?

**Good answer:**
Vertical scaling ("scale up") means adding more CPU, memory, or IOPS to a single instance — you're constrained to the biggest machine available, and it typically requires downtime to resize. Horizontal scaling ("scale out") means adding more instances/nodes running the same workload behind a load balancer. Horizontal scaling is preferred in the cloud because: (1) it has no hard ceiling — you're not bounded by the largest available instance type, (2) it improves availability, since losing one of many nodes only removes a fraction of capacity instead of taking the whole system down, and (3) it maps naturally to elastic, pay-for-what-you-use billing (add/remove nodes with demand). The trade-off is that horizontal scaling requires the application to be stateless or externalize state (sessions, cache, files) so any node can serve any request — this is the real engineering cost, not the infrastructure change itself.

**Code example:**
```yaml
# Externalizing session state is the prerequisite for horizontal scaling —
# an ASG/ECS service can only be scaled out safely if any instance can
# serve any request. Example: storing sessions in ElastiCache instead of
# in-process memory.
SESSION_STORE=redis://interview-app-cache.xxxxx.cache.amazonaws.com:6379
```

**Follow-up question:**
What application-level changes are usually required before a monolith that scales vertically today can be scaled horizontally?

**Follow-up good answer:**
Externalize session state (a shared store like ElastiCache/Redis instead of in-process memory), move file uploads/writes off local disk onto shared storage (S3/EFS), remove reliance on in-process singleton caches that assume one instance sees all traffic, make scheduled/background jobs safe to run from any instance (leader election or a single dedicated worker) instead of relying on "the one server" to run them, switch to stateless auth (JWT) or a shared session store instead of sticky sessions, and add proper health-check endpoints plus graceful shutdown handling so the load balancer can safely route around any instance.

**Glossary:**
- **Scale up (vertical)** — increasing the resources of a single node.
- **Scale out (horizontal)** — increasing the number of nodes serving a workload.
- **Statelessness** — no request-critical state kept only in a single instance's memory/disk.

**Mental model:**
Tests whether the candidate reflexively reaches for "bigger machine" or understands that horizontal scaling is a design constraint (statelessness) as much as an infrastructure choice — a common gap between junior and senior thinking.

**References:**
- [Horizontal scaling — AWS Well-Architected Framework](https://wa.aws.amazon.com/wellarchitected/2020-07-02T19-33-23/wat.concept.horizontal-scaling.en.html)

---

### Q2. How does a load balancer decide which backend to send a request to, and what happens when a backend fails its health check?

**Question:**
Walk me through how an Application Load Balancer routes a request and how it reacts when a target becomes unhealthy.

**Good answer:**
An ALB routes based on a configurable algorithm at the target-group level: `round_robin` (default — sequential, even distribution, good when requests/targets are roughly uniform), `least_outstanding_requests` (routes to whichever target currently has the fewest in-flight requests — better when request cost or target capacity varies), or `weighted_random`. Independently, the ALB continuously runs health checks (HTTP/HTTPS/TCP) against each registered target on a configurable interval; a target must pass N consecutive checks to be marked healthy and fail M consecutive checks to be marked unhealthy (the thresholds are decoupled so flapping doesn't instantly kill or resurrect a target). Once a target is unhealthy, the load balancer stops routing new requests to it — in-flight requests aren't forcibly killed, but new ones are redirected to the remaining healthy targets — until it starts passing health checks again.

**Code example:**
```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/abc \
  --attributes Key=load_balancing.algorithm.type,Value=least_outstanding_requests
```

**Follow-up question:**
When would `least_outstanding_requests` cause worse behavior than `round_robin`?

**Follow-up good answer:**
`least_outstanding_requests` (LOR) can misbehave right after a new or recovered target joins the group: because it has zero in-flight requests, LOR floods it with a disproportionate share of new traffic immediately, before its actual capacity/warm-up state is known — a "thundering herd onto the new target" effect that `round_robin` doesn't have, since round robin doesn't react to instantaneous load signals. LOR is best suited when request cost/duration varies meaningfully across requests; for uniform, short-lived requests it adds bookkeeping overhead for essentially the same distribution round robin would already give you.

**Glossary:**
- **Target group** — the set of registered backends an ALB routes to.
- **Health check thresholds** — consecutive pass/fail counts before a status flips.

**Mental model:**
Checks whether the candidate can explain a "boring" piece of infrastructure at the mechanism level instead of "it just balances load" — this is the internals angle interviewers use to filter for real production experience.

**References:**
- [How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [Target groups for your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)

---

### Q3. A service's p99 latency suddenly doubled in production. Walk me through how you'd diagnose it.

**Question:**
Your API's p99 latency jumped from 200ms to 400ms after a deploy, with no error rate increase. What's your investigation process and what tools do you reach for?

**Good answer:**
First, correlate the timing precisely — is it tied to the deploy, a traffic pattern shift, or a downstream dependency? I'd check CloudWatch metrics (CPU, memory, target response time per instance) to see if it's evenly spread (systemic — e.g. a slower code path or GC change) or concentrated on a subset of instances (a bad host, an AZ issue, or uneven load balancing). Next, distributed tracing (AWS X-Ray or OpenTelemetry) to break the request down by segment — is the extra latency in application code, a downstream call (DB, cache, another service), or network? If it's in application code, I'd pull a profiler/flame graph for a hot path, or for JVM services, `jstack`/`jcmd Thread.print` to catch threads blocked on a lock or I/O. If it's a downstream DB call, I'd check `EXPLAIN ANALYZE` for a regressed query plan (e.g. a new deploy changed a query, or a missing index after a schema change). Only after isolating the layer would I look for a root cause: N+1 query introduced in the deploy, a lock contention issue, an autoscaling lag under load, or a noisy-neighbor effect. Finally, validate the fix by comparing p50/p95/p99 before/after in the same dashboard, not just "latency looks better."

**Follow-up question:**
The trace shows the extra latency is entirely inside your own service, not downstream calls — what's your next step?

**Follow-up good answer:**
Take repeated thread dumps (`jstack` / `jcmd <pid> Thread.print`) during the slow window to see whether threads are `BLOCKED` waiting on a lock/monitor (contention) versus busy `RUNNABLE` doing real work. If CPU-bound, pull a flame graph (async-profiler or AWS X-Ray/JFR) to see exactly which function is consuming the extra cycles. Also check GC metrics/logs — a deploy that changed allocation patterns can show up as more frequent or longer GC pauses. Comparing a flame graph from before vs. after the deploy (or canary vs. baseline) is usually what pinpoints the exact changed code path.

**Glossary:**
- **p99 latency** — the latency below which 99% of requests complete; sensitive to tail effects that averages hide.
- **Flame graph** — a visualization of stack samples showing where CPU time is spent.
- **Distributed trace** — an end-to-end record of a single request as it crosses service boundaries.

**Mental model:**
This is the performance-methodology question that's now standard across every technology track — the interviewer wants a systematic narrowing process (metrics → traces → profiler/EXPLAIN → root cause → validated fix), not a list of tool names.

**References:**
- [AWS X-Ray concepts](https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html)

---

### Q4. Explain the CAP theorem and how it shows up in real cloud database choices.

**Question:**
What does CAP theorem actually constrain, and how would you use it to justify choosing DynamoDB with eventual consistency over a strongly consistent relational setup?

**Good answer:**
CAP theorem (Brewer, 2000; formally proven by Gilbert & Lynch, 2002) states that during a network partition, a distributed system can only guarantee at most two of Consistency (every read sees the latest write), Availability (every request gets a non-error response), and Partition tolerance (the system keeps working despite dropped/delayed messages between nodes). Since partitions are unavoidable at scale (network is never perfectly reliable), the real-world trade-off is C vs. A when a partition happens. DynamoDB's default eventually-consistent reads favor availability and low latency — a read might return slightly stale data, but the system stays up and fast, which is right for things like a product catalog or a "likes" counter. DynamoDB also offers strongly consistent reads (still available, but the read routes to a leader and can have higher latency) for cases like a bank balance where staleness is unacceptable. A traditional single-primary RDBMS with synchronous replication favors consistency, at the cost of availability/latency during a partition or failover. The point isn't "eventual consistency is better" — it's picking the right point on that trade-off per use case, sometimes within the same application.

**Follow-up question:**
Is CAP theorem still relevant when there's no network partition happening — what governs the consistency/latency trade-off then?

**Follow-up good answer:**
Without an active partition, CAP doesn't force a choice — a system can, in principle, be both consistent and available. What governs behavior day-to-day is the PACELC extension to CAP (Abadi, 2010): **e**lse (no partition), a system still trades **L**atency against **C**onsistency — do you wait for a synchronous quorum/replica acknowledgment (stronger consistency, higher latency), or return from the nearest replica immediately (lower latency, possibly stale)? DynamoDB's default eventually-consistent read vs. its strongly-consistent read option is exactly this trade-off exercised during normal operation, independent of whether a partition is happening.

**Glossary:**
- **Partition tolerance** — the system continues operating despite arbitrary message loss/delay between nodes.
- **Eventual consistency** — replicas converge to the same value over time, but may diverge briefly after a write.

**Mental model:**
Probes whether the candidate treats CAP as a slogan ("pick two") or actually understands it constrains behavior only under partition — and can connect the abstract theorem to a concrete AWS service's consistency knobs, which is the "SE theory + practice" blend interviewers look for.

**References:**
- [Brewer's CAP Theorem — Baeldung on Computer Science](https://www.baeldung.com/cs/brewers-cap-theorem)

---

### Q5. What problem does the circuit breaker pattern solve, and what are its states?

**Question:**
Why would you put a circuit breaker in front of a downstream service call instead of just retrying on failure?

**Good answer:**
In a microservice call chain, if a downstream dependency becomes slow or unavailable, callers that keep retrying (or just keep waiting on a long timeout) tie up their own threads/connections waiting on a service that isn't going to respond — this can cascade the failure upstream and take down otherwise-healthy services (resource exhaustion, not the original bug, becomes the outage). A circuit breaker tracks the failure rate of calls to a dependency and trips to prevent further calls once a threshold is crossed. It has three states: **Closed** (normal — requests pass through, failures are counted), **Open** (failing fast — requests are rejected immediately without calling the dependency, giving it time to recover and freeing up caller resources), and **Half-Open** (after a cooldown, a limited number of test requests are let through; if they succeed the breaker closes, if they fail it reopens). This turns a slow cascading failure into a fast, contained one and gives the failing dependency breathing room to recover.

**Code example:**
```python
# Simplified state machine sketch
if breaker.state == "OPEN":
    if time.now() - breaker.opened_at > COOLDOWN:
        breaker.state = "HALF_OPEN"
    else:
        raise FastFailError("circuit open")
try:
    result = call_downstream()
    breaker.record_success()
except DownstreamError:
    breaker.record_failure()
    if breaker.failure_rate() > THRESHOLD:
        breaker.state = "OPEN"
    raise
```

**Follow-up question:**
How would you decide the failure-rate threshold and cooldown duration for a specific dependency?

**Follow-up good answer:**
Base the failure-rate threshold on the dependency's normal baseline error rate plus margin — high enough that ordinary noise (a baseline 0.1–1% error rate) doesn't trip it, low enough to catch real degradation quickly; a common approach is requiring a minimum sample size (e.g., 20+ requests) before evaluating a failure percentage, so a handful of unlucky failures on a quiet dependency doesn't falsely open the breaker. The cooldown should reflect how long the dependency realistically takes to recover from its typical failure modes (an autoscaling event, a failover) — too short causes flapping between open and half-open, too long keeps you degraded longer than necessary. Both numbers should come from historical incident data and be validated empirically (e.g., via chaos experiments), not set once and forgotten.

**Glossary:**
- **Fail fast** — rejecting a request immediately instead of waiting on a doomed call.
- **Cascading failure** — a failure in one component propagating to and taking down healthy components.

**Mental model:**
Tests understanding of resilience patterns as a response to a *real* production failure mode (cascading resource exhaustion), not as a checkbox pattern to name-drop.

**References:**
- [Circuit breaker pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)

---

### Q6. What is the "thundering herd" problem in the context of retries, and how does jitter fix it?

**Question:**
Your service and 999 other clients all get throttled by a downstream dependency at the same moment. If they all retry with the same exponential backoff, what goes wrong, and how does jitter help?

**Good answer:**
If every client uses the exact same deterministic backoff schedule (e.g. wait exactly 2^n seconds), they all retry at the same synchronized instants, producing repeated traffic spikes ("thundering herd") that can re-trigger the same throttling or overload that caused the failures in the first place — the retries themselves become the problem. Jitter fixes this by adding randomness to the backoff delay so retries spread out over a window instead of firing simultaneously. AWS's "full jitter" approach picks a random delay uniformly between 0 and the exponential backoff cap for each attempt, so 1,000 clients that failed together end up retrying at 1,000 different times spread across the window instead of one spike. AWS SDKs implement this as standard behavior in their retry strategies.

**Code example:**
```python
import random

def backoff_full_jitter(attempt, base=0.1, cap=20):
    max_delay = min(cap, base * (2 ** attempt))
    return random.uniform(0, max_delay)
```

**Follow-up question:**
Besides jitter, what else should a retry policy include to avoid making an outage worse (hint: think about total retry budget)?

**Follow-up good answer:**
A total retry budget: cap system-wide retries as a fraction of overall request volume (e.g., no more than 10% of requests may be retries) so a broad outage doesn't get amplified 2–3x by every caller retrying independently. This pairs with a hard cap on attempts per request, an overall deadline/timeout budget shared across all attempts of one logical request (not a fresh timeout per retry), and ideally a circuit breaker so retries stop entirely once failures cross a threshold instead of continuing to add load to an already-struggling dependency. AWS SDK retry strategies implement almost exactly this as a retry "token bucket" — drawn down by failures, replenished by successes — to prevent sustained retry storms.

**Glossary:**
- **Exponential backoff** — increasing the delay between retries exponentially with each attempt.
- **Full jitter** — randomizing the delay uniformly within `[0, backoff_cap]` rather than using a fixed value.

**Mental model:**
This is a classic "looks correct but isn't" pitfall question — most candidates know "add exponential backoff" but miss that naive backoff without jitter reproduces the same synchronized-spike problem at a different timescale.

**References:**
- [Exponential Backoff and Jitter — AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Retry behavior — AWS SDKs and Tools](https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html)

---

### Q7. What does the cooldown period do in an Auto Scaling group, and why does it apply asymmetrically to scale-out vs. scale-in?

**Question:**
Explain what the ASG cooldown period is for, and why scale-in is typically more conservative than scale-out.

**Good answer:**
The cooldown period pauses further scaling activity after a scaling action so the previous action has time to take effect (new instances need to boot and pass health checks) before the ASG evaluates whether to scale again — without it, the ASG could over-provision by reacting to a metric that hasn't stabilized yet, launching far more capacity than needed. The default is 300 seconds and it applies to simple/step scaling policies (target tracking has its own analogous cooldown settings). Scale-out is allowed to happen fairly aggressively because under-provisioning risks customer-facing errors/latency immediately — the cost of over-scaling is just money. Scale-in is intentionally more conservative (often a longer cooldown, or requiring the metric to stay low for longer) because prematurely terminating instances under a temporary dip in load risks having to scale back out right after, causing latency spikes and repeated cold-starts (thrashing) — the asymmetry is a deliberate bias toward availability over cost efficiency in the default behavior.

**Follow-up question:**
How does this interact with a load balancer's connection draining — what happens to in-flight requests on an instance that's about to be terminated during scale-in?

**Follow-up good answer:**
The deregistration delay (default 300s, configurable 0–3600s) puts a scaling-in instance into the `draining` state in the target group: the ALB immediately stops sending it *new* requests, but requests already in flight get up to that window to finish before the instance is actually terminated — if the delay expires with a request still in flight, it's cut off. ASG coordinates by deregistering the instance from the target group as part of scale-in before terminating it; a Lifecycle Hook can extend this further, pausing the actual termination until app-level cleanup (finishing in-flight work, flushing logs) completes, not just until the LB-side drain window elapses.

**Glossary:**
- **Cooldown period** — the wait time after a scaling activity before another one can be triggered.
- **Warm-up time** — how long a newly launched instance needs before it's counted toward capacity metrics.

**Mental model:**
Probes whether the candidate understands autoscaling as a control-loop stability problem (avoiding oscillation/thrashing), which is a general control-systems concept applied to cloud infra — not just "AWS scales instances up and down."

**References:**
- [Available warm-up and cooldown settings — Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/consolidated-view-of-warm-up-and-cooldown-settings.html)
- [Edit target group attributes for your Application Load Balancer — Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html)

---

### Q8. When would you choose predictive scaling over target-tracking (reactive) scaling?

**Question:**
Your traffic has a predictable daily spike at 9am as users log in. Reactive target-tracking scaling is causing a latency spike every morning before it catches up. What would you change?

**Good answer:**
Target-tracking (reactive) scaling only responds after a metric (e.g. CPU or request count) crosses a threshold, so there's inherent lag: metric breach → scaling decision → instance launch → warm-up, during which the existing fleet is under-provisioned and users see degraded latency. Predictive scaling addresses exactly this by analyzing historical load data (it needs at least 24 hours of history, more for weekly patterns) to forecast recurring daily/weekly traffic patterns and proactively scale out *ahead* of the anticipated spike, rather than waiting for it to be observed. It's designed to layer on top of an existing target-tracking or simple scaling policy, not replace it — predictive scaling handles the "I know this is coming" part of demand and reactive scaling still handles unexpected deviations. It's a good fit for cyclical business-hours traffic, batch workloads, or anything with recurring patterns; it's a poor fit for genuinely unpredictable or one-off traffic spikes, where there's no historical pattern to learn from.

**Follow-up question:**
What would you check in "Forecast Only" mode before turning predictive scaling on in production?

**Follow-up good answer:**
Compare the forecast against actual historical load for the same recurring windows to sanity-check accuracy (AWS surfaces predicted vs. actual capacity so this is visible before enabling it live). Check whether the forecast baked in a one-off anomaly (an incident-driven spike) as if it were a recurring pattern. Confirm you have enough history for the pattern being learned — ideally several occurrences of it (e.g., multiple Mondays for a weekly pattern), since a short window risks overfitting to an atypical day. And verify the forecasted proactive capacity doesn't conflict with, or fall below, any minimum floor set by an existing scaling/scheduled policy during low-traffic hours.

**Glossary:**
- **Target tracking** — a scaling policy that adjusts capacity to keep a chosen metric at a target value.
- **Predictive scaling** — a scaling policy that forecasts capacity needs from historical patterns and scales proactively.

**Mental model:**
Tests whether the candidate can match a scaling strategy to the actual shape of the traffic problem, rather than defaulting to "just add more reactive scaling" — a real trade-off/judgment question, not a definitions question.

**References:**
- [Predictive scaling for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html)

---

### Q9. What's the difference between Multi-AZ and Multi-Region deployment, and what failure modes does each protect against?

**Question:**
Why might a system be Multi-AZ but not Multi-Region, and what risk does that leave unaddressed?

**Good answer:**
An Availability Zone is one or more physically distinct data centers within a region, with independent power, cooling, and networking but low-latency links to other AZs in the same region. Multi-AZ deployment (e.g. an RDS Multi-AZ instance, or an ASG spanning 3 AZs behind an ALB) protects against a single data-center-level failure — a power outage, a hardware failure, a localized network issue — with minimal latency cost, since AZs are close together. It does *not* protect against a regional-level event: a region-wide service disruption, a natural disaster affecting the whole metro area, or a botched region-wide configuration change. Multi-Region deployment replicates the application/data across geographically separate regions, protecting against that class of failure, but at real cost: cross-region data replication has meaningfully higher latency (tens to hundreds of ms), consistency gets harder (you're back to CAP trade-offs across regions), and the operational complexity (routing, failover, data sync, testing failover regularly) is significantly higher. Most systems are Multi-AZ by default because that risk/cost trade-off is close to free; Multi-Region is reserved for workloads where regional outages are within the actual risk tolerance budget (regulatory requirements, extreme availability SLAs, disaster recovery mandates).

**Follow-up question:**
For a Multi-Region active-passive setup, what's your RTO/RPO and how do those numbers drive your replication and failover design?

**Follow-up good answer:**
RTO (max acceptable downtime) drives how "warm" the standby needs to be and how automated failover must be — a tight RTO (minutes) forces an active-passive or warm-standby design with infrastructure already provisioned and automated failover (e.g., Route 53 health-check-based routing), while a looser RTO tolerates a cold, restore-from-backup approach. RPO (max acceptable data loss) drives replication mode/frequency — a near-zero RPO forces synchronous or near-real-time replication (e.g., Aurora Global Database's typical <1s cross-region lag) despite the latency cost, while a looser RPO (hours) allows cheaper periodic snapshot-based replication. Together they size the actual dollar cost of the DR strategy, pushing you along the spectrum from backup/restore → pilot light → warm standby → multi-site active-active as both numbers tighten.

**Glossary:**
- **Availability Zone (AZ)** — one or more discrete data centers with independent infrastructure within a region.
- **RTO / RPO** — Recovery Time Objective / Recovery Point Objective — how long to recover, and how much data loss is acceptable.

**Mental model:**
Checks whether the candidate can reason about availability in terms of concrete failure domains and their real costs, instead of treating "more redundancy" as strictly better regardless of cost/complexity.

**References:**
- [AWS Fault Isolation Boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/abstract-and-introduction.html)
- [Disaster recovery options in the cloud — AWS Whitepapers](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)

---

### Q10. What is the bulkhead pattern and how does it prevent one failing component from taking down the whole system?

**Question:**
Explain the bulkhead pattern with a concrete example of resource exhaustion it prevents.

**Good answer:**
Named after a ship's watertight compartments, the bulkhead pattern isolates resources (thread pools, connection pools, or entire service instances) into separate pools per client, tenant, or downstream dependency, so that exhaustion or failure in one partition doesn't consume the resources needed by the others. A concrete example: a service calls three downstream APIs (A, B, C) using a single shared HTTP connection pool. If dependency C becomes slow, all its calls hold connections from the shared pool for longer, and eventually the pool is exhausted — starving calls to A and B even though they're healthy. With a bulkhead, each dependency gets its own dedicated connection pool (or thread pool) sized for its expected load, so C being slow only degrades calls to C; A and B keep working normally. AWS's Well-Architected Framework calls this "fault isolation" — bulkheads for data partitions are sometimes called "cells" (cell-based architecture), and the same idea applies at larger scale (isolating entire customer shards into separate infrastructure "cells" so one noisy tenant can't degrade others).

**Follow-up question:**
How do you size a bulkhead's resource pool — what happens if you make it too small vs. too large?

**Follow-up good answer:**
Size each pool from the dependency's realistic sustainable throughput and observed p99 latency for calls to it specifically — using Little's Law (concurrency ≈ throughput × latency) as a starting estimate, then load-testing to validate. Undersizing causes the caller to queue or reject requests to a dependency even when that dependency is healthy and fast, artificially capping throughput below what the system could actually sustain. Oversizing defeats the isolation purpose: if the sum of all bulkheads' capacities exceeds the host's real shared resource limits (total threads, memory, connections), a single slow dependency's oversized pool can still starve the others. In practice, monitor per-bulkhead rejection/queue-depth metrics and tune sizing from real traffic rather than a one-time guess.

**Glossary:**
- **Resource pool** — a bounded set of reusable resources (threads, connections) shared across callers.
- **Cell-based architecture** — partitioning an entire system into independent, isolated deployment units ("cells").

**Mental model:**
Wants a concrete failure scenario, not just the ship metaphor — separates candidates who've actually debugged a resource-exhaustion incident from those reciting a pattern name.

**References:**
- [REL10-BP03 Use bulkhead architectures to limit scope of impact — AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_fault_isolation_use_bulkhead.html)

---

### Q11. How does the token bucket algorithm work, and how does AWS API Gateway use it for throttling?

**Question:**
Explain the token bucket rate-limiting algorithm and the difference between its "rate" and "burst" parameters.

**Good answer:**
A token bucket holds up to a fixed capacity of tokens (the **burst limit**); tokens are added to the bucket continuously at a fixed **rate limit** (tokens/second), and every incoming request consumes one token. If a request arrives and a token is available, it's allowed and the token is removed; if the bucket is empty, the request is rejected (throttled) until tokens refill. This design allows short bursts up to the bucket's capacity while enforcing a steady-state average rate over time — which is more forgiving than a naive fixed-window counter (which allows a burst right at a window boundary, effectively doubling the momentary rate) and simpler than a sliding-window log. API Gateway applies this per-account and per-method: a default account-level quota is 10,000 RPS with a 5,000-request burst; when tokens run out, requests get a `429 Too Many Requests`. Throttles are enforced on a best-effort basis, not a hard guarantee, since enforcement is distributed.

**Code example:**
```python
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate, self.capacity = rate, capacity
        self.tokens, self.last = capacity, time.monotonic()

    def allow(self):
        now = time.monotonic()
        self.tokens = min(self.capacity, self.tokens + (now - self.last) * self.rate)
        self.last = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

**Follow-up question:**
How would you design rate limiting per-API-key instead of per-account, and what storage would you use to keep it consistent across multiple gateway nodes?

**Follow-up good answer:**
Give each API key its own token-bucket state (rate + burst) rather than one shared bucket per account. Using API Gateway usage plans and API keys, this is largely built in — a usage plan defines the throttle/quota target, and every API key attached to it is tracked and enforced independently by API Gateway's own distributed throttling, with no self-managed storage needed. If implementing it yourself across multiple gateway/app nodes, you need centralized, low-latency shared storage all nodes read/write consistently — typically Redis/ElastiCache, since `INCR` + `EXPIRE` (or a Lua script implementing the token-bucket refill atomically) gives sub-millisecond, race-free updates; DynamoDB with conditional updates is also viable but adds more per-request latency than an in-memory store.

**Glossary:**
- **Rate limit** — steady-state allowed throughput (tokens/sec).
- **Burst limit** — the bucket's capacity, i.e. how far above the steady rate a client can momentarily spike.

**Mental model:**
Tests fundamental algorithm knowledge tied to a real, nameable AWS mechanism — a "what happens internally" question dressed as a rate-limiting question.

**References:**
- [Throttle requests to your REST APIs — Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [Usage plans and API keys for REST APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)

---

### Q12. What does idempotency mean for a distributed operation, and how would you implement an idempotent Lambda handler that processes payments?

**Question:**
Why does a network-retriable API call need to be idempotent, and how would you enforce that for a "charge customer" Lambda function?

**Good answer:**
In a distributed system, a caller can't always tell whether a failed/timed-out request actually succeeded on the server before the response was lost — so retrying is necessary for resilience, but retrying a non-idempotent operation (like "charge $50") risks double-charging the customer. Idempotency means the operation produces the same result and side effects no matter how many times it's executed with the same input — so retries are safe. To implement it: the caller generates a unique idempotency key per logical operation (e.g. a client-generated UUID for "this specific charge attempt"), and the handler persists the key alongside the result (e.g. in DynamoDB) *before* performing the side effect, atomically enough that a concurrent or retried request with the same key finds the existing record and returns the cached result instead of re-executing the charge. AWS Lambda Powertools' idempotency utility implements exactly this: it does a conditional `PutItem` (fails if the key already exists) to claim the operation, executes the handler, then stores the result — later calls with the same key return the stored result without re-running the handler, and the record expires after a TTL window.

**Code example:**
```python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent, DynamoDBPersistenceLayer
)

persistence_layer = DynamoDBPersistenceLayer(table_name="IdempotencyTable")

@idempotent(persistence_store=persistence_layer)
def handler(event, context):
    charge_customer(event["customer_id"], event["amount"])
    return {"status": "charged"}
```

**Follow-up question:**
What happens if two retries with the same idempotency key arrive concurrently, before the first one has finished writing its result?

**Follow-up good answer:**
The second concurrent request's conditional `PutItem` fails, because the first request already claimed the idempotency key (the item exists in an `IN_PROGRESS` state) — so the second request never re-executes the handler. Powertools' idempotency utility raises an `IdempotencyAlreadyInProgressError` in this case, which should be mapped to a retryable response (e.g., `409`/`500` with a retry hint) rather than a false success, so the real client retries again after the first attempt finishes and its cached result becomes available. This is exactly the point of using an atomic conditional write instead of a naive "check, then insert" — it collapses the check-and-claim into one operation, closing the race window a two-step approach would leave open.

**Glossary:**
- **Idempotency key** — a client-supplied unique identifier for one logical operation, used to detect retries.
- **Conditional write** — a write that only succeeds if a condition (e.g. "item doesn't exist yet") holds, used to prevent races.

**Mental model:**
Tests whether the candidate connects the abstract distributed-systems concept (idempotency, at-least-once delivery) to a concrete implementation mechanism — a classic "theory mixed with practice" question, and a favorite for payments/e-commerce interviews specifically.

**References:**
- [Idempotency — Powertools for AWS Lambda (Python)](https://docs.aws.amazon.com/powertools/python/latest/utilities/idempotency/)

---

### Q13. What causes a Lambda "cold start," and when does provisioned concurrency not fully eliminate it?

**Question:**
Explain what actually happens during a Lambda cold start, and a scenario where provisioned concurrency doesn't save you from one.

**Good answer:**
A cold start happens when Lambda has to create a brand-new execution environment for an invocation — this involves downloading the code package, starting the runtime, and running any module-level initialization code, all before your handler even starts executing, adding latency on top of the actual function work. This happens on the first invocation after a deploy, when scaling out to handle more concurrent invocations than currently-warm environments can cover, or after an idle environment has been recycled. Provisioned concurrency mitigates this by keeping N execution environments pre-initialized and idle at all times, so invocations routed to them skip straight to the Invoke phase — but it must be configured against a specific published version or alias, not `$LATEST`, and it only guarantees no cold start up to the provisioned count; a burst of concurrent invocations beyond that number still causes cold starts for the overflow. It also doesn't eliminate the (rarer) case where Lambda has to fully reset/replace an execution environment for other reasons — it reduces cold-start *frequency*, not the mechanism itself.

**Follow-up question:**
For a Java or .NET Lambda with heavy JVM/CLR startup cost, what other techniques besides provisioned concurrency would you consider (hint: think about SnapStart)?

**Follow-up good answer:**
Lambda SnapStart (Java 11+, .NET 8+, and Python 3.12+) attacks cold starts differently than provisioned concurrency: instead of keeping environments continuously warm and billed, Lambda initializes the function once when a version is published, takes an encrypted Firecracker microVM snapshot of the initialized memory/disk state, and restores new execution environments from that snapshot on invocation — cutting init latency from potentially several seconds to sub-second in optimal cases without paying for idle capacity. It requires auditing init-time code for state uniqueness (cached random values, IDs, or connections created during init get baked into the snapshot and reused verbatim across environments unless explicitly refreshed after resume), and it can't be combined with provisioned concurrency, EFS, or >512MB ephemeral storage. Other options: trimming init-time work (lazy-loading heavy dependencies), a smaller deployment package, or a lighter runtime/native-image approach (e.g., GraalVM native image for Java) to reduce JVM/CLR startup cost directly.

**Glossary:**
- **Execution environment** — the sandboxed runtime instance Lambda creates to run your function code.
- **Provisioned concurrency** — a set number of execution environments kept initialized ahead of invocation.

**Mental model:**
A performance-internals question specific to serverless — tests whether the candidate understands cold start as a lifecycle event, not a vague "serverless is sometimes slow" impression, and can reason about the limits of the standard fix.

**References:**
- [Configuring provisioned concurrency for a function — AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [Improving startup performance with Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)

---

### Q14. Why does Amazon RDS Proxy exist, and what problem does connection pooling solve that connection limits alone don't?

**Question:**
Your Lambda-based API is overwhelming your RDS instance with connections during traffic spikes. Why does adding RDS Proxy fix this, mechanically?

**Good answer:**
Each Lambda concurrent execution can open its own DB connection, and because Lambda scales out very quickly under load, a traffic spike can spin up far more concurrent connections than the database's `max_connections` limit allows, causing connection errors and wasting DB memory/CPU on connection setup/teardown overhead (each new TCP + auth handshake is expensive relative to a pooled reuse). RDS Proxy sits between the application and the database and maintains a warm pool of actual database connections, then multiplexes many client connections onto a smaller number of pooled DB connections — by default reusing a pooled connection after each transaction completes ("multiplexing"). This means the database only ever sees a stable, bounded number of connections regardless of how many Lambda invocations are hitting the proxy concurrently; excess application-side connection requests are queued/throttled by the proxy rather than crashing the database. It's a managed, highly-available version of the connection-pooling pattern you'd otherwise have to build yourself (e.g. PgBouncer), with the added benefit of preserving application connections across a database failover (up to 66% faster failover) instead of every client having to reconnect and retry.

**Follow-up question:**
Multiplexing reuses a connection after each *transaction* — what class of application behavior (hint: session-level state) becomes unsafe once you introduce that, and how do you detect it?

**Follow-up good answer:**
Anything relying on session-scoped state — `SET`-configured session variables, temp tables, prepared statements, advisory locks, cursors, `LISTEN`/`NOTIFY` channels, MARS on SQL Server — becomes unsafe, because the next transaction on the same logical client connection can land on a *different* physical DB connection that never saw that state (a temp table created in one transaction could silently "vanish" in the next). RDS Proxy actually detects most of these automatically and "pins" that session to one physical connection for its lifetime rather than letting state corrupt silently — but pinning defeats the pooling benefit you added the proxy for in the first place. Detect it via the CloudWatch metric `DatabaseConnectionsCurrentlySessionPinned`: a high pinned ratio signals the application is issuing multiplexing-breaking statements that should be refactored or moved into the proxy's initialization query instead.

**Glossary:**
- **Connection multiplexing** — reusing one physical DB connection across multiple logical client sessions/transactions.
- **max_connections** — the hard cap on simultaneous connections a database instance will accept.

**Mental model:**
A very common "Lambda + RDS" gotcha in real production systems — tests whether the candidate has actually hit this failure mode or is naming the product without understanding the mechanism it fixes.

**References:**
- [Amazon RDS Proxy — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [Avoiding pinning an RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-pinning.html)

---

### Q15. What's the difference between a blue-green deployment and a canary deployment, and when would you pick one over the other?

**Question:**
Compare blue-green and canary deployment strategies. What determines which one you'd use for a high-traffic payment service?

**Good answer:**
Blue-green deployment runs the new version ("green") fully alongside the current live version ("blue") on entirely separate infrastructure, then cuts traffic over — typically all at once, or as a single fast switch — after the green environment has been validated; rollback is just switching traffic back to blue, which still exists. Canary deployment shifts a small percentage of live traffic (e.g. 10%) to the new version first, monitors real production metrics/alarms for a defined interval, then either proceeds to shift the rest of the traffic or automatically rolls back if a CloudWatch alarm fires — the new version is exposed to a *subset* of real traffic before full rollout, rather than validated separately and switched all at once. For a high-traffic payment service, canary is usually preferable: it limits the blast radius of a bad deploy to a small fraction of real transactions, catches issues that only show up under real production load/data patterns (which a separate green environment's testing might miss), and gives you an automated, metric-driven rollback rather than a manual "looks fine, cut over" decision. Blue-green is simpler and appropriate when you need a fast, all-or-nothing cutover (e.g. version compatibility constraints) and you trust pre-cutover validation to catch issues.

**Follow-up question:**
What CloudWatch alarm(s) would you wire up to auto-rollback a canary deployment for a payment service, and why those specifically?

**Follow-up good answer:**
At minimum: a 5xx/error-rate alarm scoped specifically to the canary target group (tight absolute threshold, since even low-volume payment errors are high severity), a p95/p99 latency alarm scoped to the same canary targets (degraded-but-not-erroring is still a failure mode for payments), and a business-metric alarm if one exists (e.g., a custom CloudWatch metric tracking failed charge/authorization attempts), since an HTTP 200 can still represent an incorrect business outcome an infra-level alarm won't catch. All alarms should read only the canary's slice of traffic, not the blended blue+green aggregate, and thresholds should sit well inside your normal SLO so a real regression triggers rollback before it burns meaningful error budget.

**Glossary:**
- **Blast radius** — the scope of impact if a deployed change is bad.
- **Traffic shifting** — gradually moving live request volume from one version/environment to another.

**Mental model:**
A trade-off/comparison question that also tests whether the candidate connects the strategy choice to risk tolerance for the specific domain (payments = minimize blast radius), not just definitions.

**References:**
- [CodeDeploy blue/green deployments for Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)

---

### Q16. How does consistent hashing help a distributed data store like DynamoDB scale, and what problem does it solve versus simple modulo hashing?

**Question:**
Why can't a distributed hash table just use `hash(key) % number_of_nodes` to decide which node owns a key?

**Good answer:**
With `hash(key) % N`, adding or removing even one node changes `N`, which changes the modulo result for nearly every key — meaning almost all keys have to be remapped and their data physically moved between nodes, which is prohibitively expensive at scale and makes elastic scaling impractical. Consistent hashing solves this by mapping both nodes and keys onto the same fixed hash ring (e.g. 0 to 2^128); a key is owned by the first node found walking clockwise from the key's hash position. Adding or removing a node then only affects the keys in the immediate neighboring range on the ring — the rest of the mapping is undisturbed, so only a small, bounded fraction of keys need to move. To avoid uneven load from an unlucky node placement, "virtual nodes" are used — each physical node is represented at many points on the ring, smoothing out the distribution. Conceptually, DynamoDB descends from this idea (from the original Amazon Dynamo paper); in practice, modern DynamoDB uses a centralized partition-metadata/routing service rather than a literal peer-to-peer hash ring, but the underlying goal — even key distribution with minimal data movement on scale events — is the same.

**Follow-up question:**
What's the "hot partition" problem, and how does DynamoDB's partition key design guidance try to prevent it?

**Follow-up good answer:**
A hot partition happens when reads/writes concentrate disproportionately on one partition — typically from a low-cardinality or predictably-clustered partition key (a raw timestamp, a status field with few values, a sequential ID) — so that one partition hits its fixed per-partition ceiling (3,000 RCU / 1,000 WCU) and throttles, even though the table's aggregate capacity has plenty of headroom elsewhere. DynamoDB's guidance is to pick high-cardinality, uniformly-accessed partition keys, and to shard an otherwise-hot logical key with a composite or randomized suffix (e.g., `userId-sessionId`, or `itemId#0`–`itemId#9` for a very hot single item) so writes spread across many physical partitions. DynamoDB also mitigates this automatically at runtime via adaptive capacity (temporarily reallocating throughput toward a hot partition) and split-for-heat (splitting a hot partition's keyspace), but those are safety nets, not a substitute for key design.

**Glossary:**
- **Hash ring** — a circular hash-value space onto which nodes and keys are both mapped.
- **Virtual nodes** — multiple ring positions assigned to one physical node to smooth load distribution.

**Mental model:**
A classic distributed-systems internals question — tests whether the candidate can reason from first principles about *why* a naive approach fails at scale, then connect it to a real system's design lineage.

**References:**
- [DynamoDB Deep Dive for System Design Interviews — Hello Interview](https://www.hellointerview.com/learn/system-design/deep-dives/dynamodb)
- [Best practices for designing and using partition keys effectively in DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)

---

### Q17. How does SQS provide backpressure, and what's the role of visibility timeout and a dead-letter queue in preventing a poison message from blocking a queue forever?

**Question:**
A malformed message keeps failing to process and keeps reappearing in your SQS queue, slowing down every other consumer. What mechanisms prevent this, and how would you configure them?

**Good answer:**
When a consumer receives a message from SQS, the message isn't deleted — it becomes invisible to other consumers for the **visibility timeout** duration (default 30s, configurable up to 12 hours), during which the consumer is expected to process it and explicitly delete it. If the consumer crashes or fails to delete it in time, the message becomes visible again and gets redelivered — this is what gives SQS its at-least-once, backpressure-friendly semantics: slow consumers naturally cause messages to queue up rather than being dropped, and failed processing gets retried automatically without extra code. The risk is a "poison message" — one that always fails processing (e.g. malformed payload) — which would otherwise cycle: deliver, fail, become visible, redeliver, forever, consuming consumer capacity on every cycle. A **dead-letter queue (DLQ)** solves this: you configure a `maxReceiveCount` on the source queue, and once a message has been received (and presumably failed) that many times, SQS automatically moves it to the DLQ instead of redelivering it — isolating the bad message so it stops consuming consumer throughput, while leaving it available for debugging.

**Code example:**
```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789012:my-queue-dlq",
  "maxReceiveCount": 5
}
```

**Follow-up question:**
How would you size the visibility timeout relative to your consumer's actual processing time, and what goes wrong if it's set too short?

**Follow-up good answer:**
Set the visibility timeout to at least the consumer's expected maximum processing time, with margin — AWS recommends at least 6x your function's timeout for a typical Lambda-SQS setup — and extend it dynamically via `ChangeMessageVisibility` for tasks whose duration varies a lot, rather than statically over-provisioning for the worst case. If it's set too short, a consumer still legitimately processing a message (just slower than the timeout assumed) sees that message become visible again and delivered to a second consumer before the first finishes — causing concurrent duplicate processing of the same message. That's a self-inflicted version of the exact failure mode a correctly sized timeout is meant to prevent, and it pushes you to lean on idempotency to paper over what's really a misconfiguration.

**Glossary:**
- **Visibility timeout** — the window during which a received-but-undeleted message is hidden from other consumers.
- **Poison message** — a message that reliably fails processing and would otherwise loop forever.

**Mental model:**
Tests understanding of queues as a backpressure/resilience mechanism (not just "a list of messages"), plus the specific real-world failure mode (poison messages) that every queue-based system eventually hits in production.

**References:**
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)

---

### Q18. What is chaos engineering, and how would you run a controlled experiment on AWS without risking an uncontrolled outage?

**Question:**
Explain chaos engineering as a discipline, and how AWS Fault Injection Simulator lets you run experiments safely in production.

**Good answer:**
Chaos engineering is the practice of deliberately injecting failure into a system to verify it actually handles the failure modes it was designed for, rather than assuming resilience mechanisms (retries, failover, autoscaling) work because they exist on paper. The standard method: (1) define the system's steady-state behavior via a measurable metric (e.g. error rate, latency), (2) form a hypothesis ("if we kill one AZ's instances, the ALB should route around it and error rate should stay flat"), (3) run a controlled experiment that introduces the fault, (4) observe whether the steady state held, and (5) fix whatever broke and repeat. AWS Fault Injection Simulator (FIS) formalizes this on AWS: experiment templates define **actions** (the fault to inject — e.g. terminate an instance, inject network latency, throttle an API), **targets** (which resources are affected, often a random subset), and — critically — **stop conditions**, which are CloudWatch alarms that automatically abort the experiment the instant a real metric breaches a safety threshold, bounding the blast radius even if the hypothesis was wrong. This is what makes it safe to run in production rather than only in staging: staging often can't reproduce real traffic patterns and data volume, so production experiments (with a tight blast radius and automatic stop conditions) give more trustworthy signal.

**Follow-up question:**
Why is testing resilience in staging alone often insufficient, and what would you need to trust a production chaos experiment enough to run it?

**Follow-up good answer:**
Staging usually can't reproduce production's real traffic volume/shape, real data skew (a hot customer, an outsized tenant, adversarial inputs), or actual fleet scale — a resilience mechanism that looks fine in staging (autoscaling catching up in time, a cache absorbing load) can fail in production purely because scale changes the failure dynamics. Trusting a production experiment enough to run it requires: tight, automated stop conditions (alarms that abort the instant real customer impact is detected), a small initial blast radius (a tiny random target subset, not "every instance in an AZ" on a first run), a tested and fast rollback/abort path, and starting during low-traffic windows before widening scope as confidence builds.

**Glossary:**
- **Steady state** — the measurable, expected normal behavior of the system before a fault is introduced.
- **Stop condition** — an automatic abort trigger (typically a CloudWatch alarm) that halts an experiment if it causes real harm.

**Mental model:**
An advanced/challenges question — separates candidates who've only read about resilience patterns from those who've validated resilience empirically, and tests whether they instinctively think about experiment safety (blast radius, stop conditions), not just "break things."

**References:**
- [Resources — Chaos engineering on AWS — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/chaos-engineering-on-aws/resources.html)

---

### Q19. How do EC2 Spot Instances achieve up to 90% cost savings, and what architecture decisions do you need to make to use them safely?

**Question:**
Explain how Spot Instance pricing and interruption work, and how you'd design a workload to tolerate interruptions.

**Good answer:**
Spot Instances let you bid for unused EC2 capacity at a price set by supply and demand, often up to 90% cheaper than On-Demand — the trade-off is that AWS can reclaim that capacity at any time when it's needed elsewhere, giving a **two-minute interruption notice** (and, earlier, an optional "rebalance recommendation" signal warning of elevated interruption risk before the hard notice) before terminating, stopping, or hibernating the instance. Using Spot safely requires designing for interruption, not just accepting cheaper instances: the workload should be stateless or checkpoint state externally (not on local/instance-store disk that disappears with the instance), work should be resumable or safely re-queued if interrupted mid-task (ties back to idempotency), and you should diversify across multiple instance types/AZs (via an EC2 Fleet or ASG with mixed instances policy using a "capacity-optimized" or "price-capacity-optimized" allocation strategy) so a capacity crunch in one pool doesn't take out your whole fleet at once. It's typically layered with an On-Demand baseline for the portion of capacity that must not be interrupted, with Spot covering the elastic/batch portion (e.g. CI workers, batch processing, stateless web tiers behind an ASG that just relaunches).

**Follow-up question:**
How would the two-minute interruption notice change how you handle an in-flight request on a Spot-backed web server behind an ALB?

**Follow-up good answer:**
On the two-minute interruption notice (surfaced via instance metadata, or an EventBridge event you can subscribe to), the instance should immediately deregister itself from the ALB target group (directly, or via a lifecycle hook) so the ALB stops routing *new* requests to it and the target enters `draining`, giving in-flight requests up to the target group's deregistration delay to finish. Because Spot's notice window (2 minutes) is shorter than the ALB's default 300s deregistration delay, that delay should be tuned down for Spot-backed target groups (or deregistration triggered immediately on notice, not deferred) so draining actually completes within the time available before AWS reclaims the instance — anything still in flight when the hard interruption lands is lost, which is why idempotent, retriable client behavior still matters here.

**Glossary:**
- **Spot price** — the current market price for Spot capacity, set by supply/demand.
- **Capacity-optimized allocation** — a Spot Fleet strategy that sources instances from the pools with the most available capacity, minimizing interruption risk.

**Mental model:**
A cost-vs-performance/reliability trade-off question — tests whether the candidate treats cost optimization as free money or understands it as a real architectural commitment (design-for-interruption) with engineering cost.

**References:**
- [Spot Instance interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)
- [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
- [Edit target group attributes for your Application Load Balancer — Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html)

---

### Q20. You need to add rate limiting, retries, circuit breaking, and bulkheads to a service mesh — which of these belong in application code vs. infrastructure, and why?

**Question:**
Given the resilience patterns discussed in this set (retries, circuit breakers, bulkheads, rate limiting), which would you implement in application code versus push down into infrastructure (e.g. a service mesh sidecar like Envoy, or API Gateway), and what's the trade-off?

**Good answer:**
Infrastructure-level implementation (a service mesh sidecar, load balancer, or API Gateway) centralizes these concerns so every service gets consistent behavior without each team re-implementing the pattern, and it can be tuned/observed uniformly across the fleet — this is generally preferable for retries, circuit breaking, rate limiting, and connection-level bulkheads, since they're largely mechanical and don't need business-logic awareness. The trade-off is less fine-grained control: a sidecar retrying a non-idempotent call it doesn't understand the semantics of can cause the exact double-execution problem idempotency is meant to prevent, so the *decision* of what's safe to retry still has to be informed by application-level guarantees (idempotency keys, etc.), even if the *mechanism* lives in infrastructure. Some patterns are harder to push down: bulkheads around business-meaningful resource pools (e.g. isolating a noisy tenant's workload) often need application/domain awareness of what constitutes a "partition." In practice, most production systems land on a hybrid: infra handles the generic transport-level resilience (retries, circuit breaking, rate limiting) while application code handles idempotency, business-level bulkheads, and anything that requires understanding the semantics of the operation, not just its shape as an HTTP/RPC call.

**Follow-up question:**
If a sidecar proxy is responsible for retries, how do you prevent it from retrying a request whose side effect already happened but whose response was lost?

**Follow-up good answer:**
The sidecar's retry mechanism must be scoped to what's actually safe to retry — in practice, auto-retrying only inherently safe/idempotent methods (GET, HEAD, idempotent PUT) by default, and for anything else requiring the application to attach an idempotency-key header that the sidecar forwards unchanged on every retry attempt, so the downstream service's own idempotency-key handling (as in Q12) can dedupe regardless of how many times the transport layer resent the call. A sidecar that blindly retries POST/RPC calls with no such contract reintroduces exactly the double-execution risk idempotency exists to prevent — which is why service meshes let you configure retryable methods/status codes per route rather than retrying everything, and why the retry-safety *decision* stays an application/API-contract concern even once the retry *mechanism* lives in infrastructure.

**Glossary:**
- **Service mesh sidecar** — a proxy deployed alongside each service instance that intercepts and manages network traffic (e.g. Envoy in Istio).
- **Transport-level vs. business-level concern** — a distinction between generic network resilience and logic that requires understanding what an operation actually does.

**Mental model:**
A synthesis question that ties the whole set together — tests architectural judgment about where resilience logic belongs, not just knowledge of individual patterns in isolation; this is the kind of question that surfaces at the senior/staff level.

**References:**
- [Circuit breaker pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
