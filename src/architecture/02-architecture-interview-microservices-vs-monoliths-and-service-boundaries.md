---
layout: default
title: "Architecture Interview — Microservices vs Monoliths and Service Boundaries"
---

# Architecture Interview — Microservices vs Monoliths and Service Boundaries

This set covers the monolith-vs-microservices decision itself: how to draw service
boundaries with Domain-Driven Design, the mechanics of inter-service communication
(sync vs async, sagas, discovery, gateways), how to diagnose distributed-system
performance problems, the classic anti-patterns (distributed monolith, shared
database), and the migration patterns (monolith-first, strangler fig) that let you
get there incrementally instead of by rewrite.

### Q1. What is a microservice, and how does that actually differ from a well-factored module inside a monolith? {#q1}

**Question:**
What is a microservice, and how does that actually differ from a well-factored module inside a monolith?

**Good answer:**
A microservice is a small, independently deployable service that runs in its own process and owns its own data store, communicating with other services over the network (HTTP/gRPC or messaging) rather than via in-process function calls. A well-factored module in a monolith can have the same clean logical boundary — its own package, its own interface — but it still compiles, deploys, and runs inside the same process as everything else, sharing the same database and the same deploy cycle. The difference that actually matters isn't code organization, it's the *deployment and failure boundary*: a microservice can be deployed, scaled, and can fail independently of the rest of the system; a module cannot. That's why "we already have modules" isn't a rebuttal to "we should extract a service" — the module gives you compile-time decoupling, not runtime/operational decoupling.

**Follow-up question:**
If a module boundary already gives you clean separation of concerns, why would a team pay the operational cost of turning it into a service?

**Follow-up good answer:**
Mainly for independent deployability and independent scaling/fault isolation — reasons that only matter once a module's release cadence, team ownership, or resource profile diverges enough from the rest of the monolith to justify the network hop. If a payments module needs to deploy multiple times a day while the rest of the app deploys weekly, or it needs 10x the CPU under load, or a different team owns it end-to-end and is blocked by everyone else's release train, extracting it into a service turns those organizational/operational needs into something the architecture actually supports. If none of that is true yet, the module boundary alone is enough, and extracting it just adds network latency, partial-failure modes, and operational surface area for no benefit.

**Glossary:**
- **Microservice** — an independently deployable service with its own process and data store, communicating over the network.
- **Monolith** — a single deployable unit where all logic for handling a request runs in one process.

**Mental model:**
Tests whether the candidate conflates *logical* decoupling (modules, packages, interfaces) with *operational* decoupling (independent deploy/scale/failure). A senior candidate should be able to say "you can get most of the benefit of clean boundaries without paying the microservices tax" and know exactly which specific pressure (deploy cadence, team ownership, scaling profile) justifies actually paying it.

**TL;DR:**
A microservice differs from a monolith's module in deployment/failure boundary, not code structure — it can be deployed, scaled, and fail independently; a module cannot.

**References:**
- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)

---

### Q2. What is a "bounded context" in Domain-Driven Design, and why does it matter for drawing microservice boundaries? {#q2}

**Question:**
What is a "bounded context" in Domain-Driven Design, and why does it matter for drawing microservice boundaries?

**Good answer:**
A bounded context is the boundary within which a particular domain model — its terms, rules, and the meaning of its entities — is internally consistent and unambiguous. Domain-Driven Design's premise is that a single unified model for an entire large system isn't feasible: the same word ("customer," "order," "meter") means different things to different parts of the business, and forcing one model to serve all of them produces a model that's technically correct nowhere. A bounded context draws the line around where one specific model and its "ubiquitous language" holds. This maps almost directly onto good microservice boundaries: a service should own one bounded context, so its API and data model reflect one coherent, unambiguous meaning of its entities, instead of being an arbitrary technical slice (like "the database layer" or "the API layer") that has no business meaning of its own.

**Follow-up question:**
Two teams both have an entity called "Product" — one team's Product has pricing and inventory fields, the other's has marketing copy and images. Is that duplication a problem to fix?

**Follow-up good answer:**
No — that's exactly what bounded contexts predict and endorse. "Product" in the Pricing/Inventory context and "Product" in the Catalog/Marketing context are different models serving different purposes, and trying to unify them into one shared "Product" entity is what actually causes pain: you'd end up with a bloated model full of fields only one context cares about, and both teams coupled to a schema neither fully owns. The right move is to let each bounded context define its own Product shape, and handle the overlap explicitly at the boundary (e.g. each service publishes just the fields relevant to integration, referenced by a shared identifier) via an anti-corruption layer or a lightweight shared-kernel/context map relationship, rather than one god-model.

**Glossary:**
- **Bounded context** — the boundary within which a domain model is internally consistent and its terms have one unambiguous meaning.
- **Ubiquitous language** — the shared vocabulary between developers and domain experts, valid within one bounded context.
- **Context map** — a description of how multiple bounded contexts relate to and integrate with each other.

**Mental model:**
Tests whether the candidate can use DDD as a boundary-drawing tool, not just cite the term. The follow-up specifically probes whether they'll reflexively "fix" duplication that's actually healthy — a common mistake from engineers used to normalizing everything, applied wrongly at the service-boundary level.

**TL;DR:**
A bounded context is where one domain model is internally consistent — drawing service boundaries along bounded contexts, instead of technical layers, is what makes each service's API and data model actually coherent.

**References:**
- [Martin Fowler — Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)

---

### Q3. What's the practical difference between synchronous and asynchronous inter-service communication, and when do you reach for each? {#q3}

**Question:**
What's the practical difference between synchronous and asynchronous inter-service communication, and when do you reach for each?

**Good answer:**
Synchronous communication (REST over HTTP, or gRPC) means the caller blocks and waits for a response before continuing — it's simple to reason about and gives you an immediate result or error, but it couples the caller's availability to the callee's: if the downstream service is slow or down, the caller is too, and that failure can cascade across a chain of services. Asynchronous communication (publishing an event or message to a broker like Kafka/RabbitMQ/SQS) decouples the two in time: the publisher fires the message and moves on, and one or more consumers process it whenever they're able to. That buys resilience to downstream slowness/outages and lets you fan a single event out to multiple independent consumers, at the cost of giving up immediate consistency (the caller doesn't know the outcome right away) and adding operational complexity (message ordering, delivery guarantees, debugging a request across an async chain). In practice: use synchronous calls for reads/queries where the caller genuinely needs the answer now; use async messaging for writes that trigger side effects in other services, especially where those services shouldn't block the original request's latency.

**Follow-up question:**
A checkout flow calls Inventory, then Payment, then Shipping synchronously, one after another. What's wrong with that design, and how would you fix it?

**Follow-up good answer:**
The request's total latency is the sum of all three calls, and its availability is the *product* of all three services' availability — any one of them being slow or down takes down checkout entirely, and a failure partway through (say, Payment succeeds but Shipping fails) leaves you needing to figure out compensation manually mid-flow. The fix is usually to make only the truly synchronous, must-succeed-now steps synchronous (e.g. reserving inventory and charging payment, since the customer is waiting to know if the order is accepted) and push everything else — shipping label creation, notification emails, warehouse routing — into asynchronous events published once the order is confirmed. If even the payment/inventory sequence needs multi-service consistency, that's exactly the distributed-transaction problem a Saga is for, coordinating the steps with compensating actions instead of a blocking chain.

**Code example:**
```text
Synchronous chain (fragile):
  Checkout -> [wait] Inventory -> [wait] Payment -> [wait] Shipping -> response

Split into sync "accept" path + async side effects:
  Checkout -> Inventory (reserve) -> Payment (charge) -> respond "order accepted"
                                                    |
                                                    v
                                     publish OrderConfirmed event
                                          /              \
                                 Shipping service    Notification service
                                 (async consumer)    (async consumer)
```

**Glossary:**
- **Synchronous communication** — the caller blocks until it receives a response (e.g. REST, gRPC).
- **Asynchronous communication** — the caller publishes a message/event and continues without waiting for processing (e.g. Kafka, SQS, RabbitMQ).

**Mental model:**
Tests whether the candidate understands that this choice isn't stylistic — it directly determines your latency budget, your availability math (product of dependencies vs decoupled), and your failure-handling story. The follow-up checks whether they can actually redesign a naive synchronous chain rather than just recite the trade-off in the abstract.

**TL;DR:**
Sync calls couple caller and callee's availability/latency together; async messaging decouples them in time at the cost of immediate consistency — use sync for reads the caller needs now, async for side effects that shouldn't block the request.

**References:**
- [Martin Fowler — Microservices ("Smart endpoints and dumb pipes")](https://martinfowler.com/articles/microservices.html)

---

### Q4. What is the distributed transaction problem in a microservices architecture, and how does the Saga pattern address it? {#q4}

**Question:**
What is the distributed transaction problem in a microservices architecture, and how does the Saga pattern address it?

**Good answer:**
When each service owns its own database (the standard microservices data pattern), a single business operation that spans multiple services — like placing an order, which touches Inventory, Payment, and Shipping — can no longer be wrapped in one ACID database transaction, because there's no single database to commit or roll back atomically. Two-phase commit across services is theoretically possible but rarely used in practice because it requires all participants to be available and blocks resources for the duration, which is a poor fit for the availability goals microservices are usually built for. The Saga pattern solves this by breaking the operation into a sequence of local transactions, each in one service, where each step publishes an event or receives a command that triggers the next step. If a step fails partway through, the saga runs compensating transactions for every step that already succeeded, undoing their effects (e.g. refunding a payment that was already charged) rather than relying on an atomic rollback that isn't available across service boundaries.

**Follow-up question:**
Sagas give up traditional ACID isolation between steps — what concrete problem can that cause, and how do you mitigate it?

**Follow-up good answer:**
Because each local transaction commits immediately and is visible to other transactions before the whole saga finishes, another process can read or act on partially-completed state — e.g. see "reserved" inventory before payment is confirmed, or double-book the same inventory if two sagas run concurrently against it. Common mitigations include: using a semantic lock (marking the record as "pending" so other operations know not to treat it as fully available), reordering steps so the riskiest/least-reversible one happens last, and designing idempotent, commutative compensating actions so retries and out-of-order completion don't corrupt state. This isolation gap is a real cost of sagas, not a minor footnote — it has to be designed for explicitly, not assumed away.

**Glossary:**
- **Saga** — a sequence of local transactions across services, coordinated via events or commands, with compensating transactions for rollback.
- **Compensating transaction** — an operation that semantically undoes the effect of an already-committed local transaction.
- **Two-phase commit (2PC)** — a distributed transaction protocol requiring all participants to agree before committing; rarely used across microservices due to blocking and availability costs.

**Mental model:**
Tests whether the candidate actually understands *why* distributed transactions are hard (no shared database, so no atomic commit/rollback) rather than just having memorized "use the Saga pattern." The follow-up on isolation separates candidates who've only read the pattern name from those who've actually had to reason about the failure modes.

**TL;DR:**
Sagas replace one atomic cross-service transaction with a sequence of local transactions plus compensating actions for rollback, trading ACID isolation for something workable when there's no shared database to commit atomically.

**References:**
- [microservices.io — Saga pattern](https://microservices.io/patterns/data/saga.html)

---

### Q5. What's the difference between choreography-based and orchestration-based sagas? {#q5}

**Question:**
What's the difference between choreography-based and orchestration-based sagas?

**Good answer:**
In a choreography-based saga, there's no central coordinator — each service publishes an event when it completes its local transaction, and the next service(s) in the flow subscribe to that event and react on their own. For example, an Order service publishes "OrderCreated," which the Payment service listens for to charge the customer, which in turn publishes "PaymentCompleted" for Shipping to pick up. In an orchestration-based saga, a dedicated orchestrator component explicitly sends commands to each participant in sequence and tracks the overall state of the saga, deciding what happens next (including which compensating actions to trigger on failure) based on each participant's response. Choreography keeps services fully decoupled and is simple for short flows, but the overall business process ends up implicit, scattered across every service's event handlers, which gets hard to trace and reason about as the number of steps grows. Orchestration makes the process explicit and centrally visible/testable, at the cost of the orchestrator becoming a component every participant is coupled to (and a potential single point of complexity, if not of failure).

**Follow-up question:**
At what point does a choreography-based saga usually become painful enough that a team switches to orchestration?

**Follow-up good answer:**
Usually once the number of steps grows past a handful, or once conditional branching enters the picture — "if payment fails, skip straight to X; if inventory partially reserves, do Y instead of Z" — because that logic has to live somewhere, and in choreography it ends up smeared across multiple services' event handlers with no single place that shows the whole flow. Debugging in production ("why did this order get stuck between confirmed and shipped?") also gets much harder in choreography, since there's no single component tracking saga state — you have to reconstruct it from events scattered across services' logs. Orchestration re-centralizes that complexity into one place (the orchestrator's state machine), trading service decoupling for operational visibility, which most teams find worth it once the flow is non-trivial.

**Glossary:**
- **Choreography** — saga coordination via services reacting to each other's published events, with no central coordinator.
- **Orchestration** — saga coordination via a central component that issues commands and tracks the overall flow state.

**Mental model:**
Tests whether the candidate has actually operated one of these in practice, since the trade-off (implicit-but-decoupled vs explicit-but-coupled) only really bites once you've had to debug a stuck multi-step business process in production.

**TL;DR:**
Choreography coordinates a saga via services reacting to each other's events (decoupled but implicit); orchestration uses a central coordinator issuing commands and tracking state (coupled but explicit and easier to debug).

**References:**
- [microservices.io — Saga pattern](https://microservices.io/patterns/data/saga.html)

---

### Q6. How does client-side service discovery work, and how does it differ from server-side discovery? {#q6}

**Question:**
How does client-side service discovery work, and how does it differ from server-side discovery?

**Good answer:**
In client-side discovery, the calling service queries a service registry directly to find out which network locations currently host healthy instances of the service it wants to call, then picks one itself (often with client-side load balancing) and calls it. A classic example is Netflix's Eureka as the registry paired with a client library (like Ribbon) that resolves a logical service name into an actual instance address before the HTTP call goes out. Server-side discovery moves that lookup out of the calling application entirely: the client just calls a fixed, well-known address (a load balancer or router), and that intermediary is the one that queries the registry and routes the request to a healthy instance — the calling code never talks to the registry itself. Client-side discovery has fewer network hops and moving parts, but couples every calling service to the registry and requires a discovery-aware client library in every language you use; server-side discovery centralizes that complexity into the router/gateway, at the cost of an extra network hop and another piece of infrastructure to run.

**Follow-up question:**
In a Kubernetes environment, which of these two approaches is closest to what actually happens under the hood?

**Follow-up good answer:**
Kubernetes' built-in Service abstraction is architecturally closer to server-side discovery: calling code just resolves a stable DNS name (the Service name) and connects to a stable virtual IP, and kube-proxy (or the CNI's equivalent) transparently routes that connection to one of the healthy backing Pod IPs — the calling application code never queries the Kubernetes API or a registry directly. The registry-and-lookup work that a client-side pattern like Eureka+Ribbon does explicitly in application code is instead done for you by cluster infrastructure, which is a big part of why Kubernetes made patterns like Eureka far less necessary for services that already run on it.

**Glossary:**
- **Service registry** — a database of currently-healthy service instances and their network locations.
- **Client-side discovery** — the calling service queries the registry and picks an instance itself.
- **Server-side discovery** — an intermediary (load balancer/router) queries the registry on the client's behalf.

**Mental model:**
Tests whether the candidate understands discovery as a genuine architectural choice with trade-offs, not just "Eureka is what you use." The Kubernetes follow-up checks whether they can map the abstract pattern onto infrastructure they've actually likely used.

**TL;DR:**
Client-side discovery has the calling service query the registry and pick an instance itself (fewer hops, more client coupling); server-side discovery hides that behind a router/load balancer the client just calls directly.

**References:**
- [microservices.io — Client-side discovery pattern](https://microservices.io/patterns/client-side-discovery.html)

---

### Q7. What problem does an API Gateway solve for microservice clients, and what is the Backends-for-Frontends (BFF) variation? {#q7}

**Question:**
What problem does an API Gateway solve for microservice clients, and what is the Backends-for-Frontends (BFF) variation?

**Good answer:**
Without a gateway, a client (especially a mobile app on a slow, high-latency network) that needs data spread across several microservices would have to call each one directly and stitch the results together itself, which means multiple round trips, exposing internal service topology to the client, and every service change potentially breaking clients. An API Gateway sits as a single entry point in front of the services: some requests it simply proxies through to the right service, and others it fans out to multiple services and composes the results into one response, so the client makes one call instead of many. The Backends-for-Frontends variation takes this further by giving each type of client (web, mobile, third-party partner) its *own* gateway, tailored to exactly what that client needs, instead of forcing one generic gateway to serve every client's different shape of request — which otherwise tends to accumulate client-specific special cases until the single gateway becomes an unmaintainable dumping ground.

**Follow-up question:**
What's the downside of putting business logic (like data aggregation or composition) into the gateway itself?

**Follow-up good answer:**
The gateway starts as "just routing" but composition logic tends to accumulate there over time, and because it sits in front of every request, it becomes a shared bottleneck — both operationally (it now has to scale with all traffic, and a bug there can break every client) and organizationally (one team ends up owning logic that really belongs to individual domain services, so other teams have to coordinate through them to ship a change). The BFF pattern is partly a response to this: by splitting the gateway per client type, each BFF can be owned by the team that owns that client, and composition logic that's genuinely specific to how one client wants its data can live there without becoming everyone's shared dependency — but it doesn't fully solve the problem for logic that's inherently cross-cutting, which is often a sign that logic actually belongs in a domain service instead.

**Glossary:**
- **API Gateway** — a single entry point that routes and/or composes requests across multiple backend services.
- **Backends for Frontends (BFF)** — a variant with a separate, client-tailored gateway per client type (web/mobile/partner).

**Mental model:**
Tests whether the candidate sees the gateway as a genuine architectural component with its own failure modes and ownership costs, not just "the load balancer we put in front."

**TL;DR:**
An API Gateway gives clients one entry point instead of many direct service calls, proxying or composing as needed; BFF splits that gateway per client type so each client gets a tailored API instead of one generic gateway accumulating every client's special cases.

**References:**
- [microservices.io — API Gateway pattern](https://microservices.io/patterns/apigateway.html)

---

### Q8. How do you diagnose where latency is coming from when a request crosses five different microservices? {#q8}

**Question:**
How do you diagnose where latency is coming from when a request crosses five different microservices?

**Good answer:**
You need distributed tracing: a mechanism that follows a single logical request as it propagates across all five services and records it as one trace made up of spans, where each span is one unit of work (a service handling the request, a downstream call it makes, a DB query) with its own start/end time and metadata. Tools built on OpenTelemetry (feeding into a backend like Jaeger, Tempo, or a vendor APM) let you visualize that trace as a waterfall: you can see exactly which span is the slowest, whether spans ran sequentially when they could have been parallelized, and whether the time was spent in a specific service's own processing versus waiting on a downstream dependency. Without this, you're left correlating timestamps across five services' separate logs by hand, which doesn't scale and rarely finds the actual bottleneck.

**Follow-up question:**
The trace shows the total request took 800ms, but every individual span only adds up to 500ms. What does that gap usually mean, and what would you check next?

**Follow-up good answer:**
That gap is time that isn't accounted for by any instrumented span — usually time spent in un-instrumented code (serialization/deserialization, middleware, connection pool acquisition), queueing delay before a service even picks up the request (e.g. waiting for a thread-pool slot or a message to be dequeued), or clock skew between hosts producing an inaccurate picture of ordering/overlap. The next step is to check whether the tracing instrumentation covers the full request path (including client libraries, middleware, and any queueing/thread-pool layer) — a common gap is a service that trace-instruments its handler but not the wait time before that handler starts running, which hides exactly the kind of resource-contention bottleneck (undersized connection pool, thread starvation) that's most useful to find.

**Glossary:**
- **Trace** — the record of a single logical request's path through all the services it touched.
- **Span** — one unit of work within a trace, with its own timing and metadata.
- **OpenTelemetry** — a vendor-neutral standard/toolkit for producing traces, metrics, and logs.

**Mental model:**
Tests methodology, not tool trivia: can the candidate reason from "distributed system, unexplained latency" to "I need causally-linked timing across services" rather than reaching for single-service profiling tools that can't see cross-service time. The follow-up specifically checks whether they know unaccounted-for time is itself a diagnostic signal, not a rounding error to ignore.

**TL;DR:**
Diagnosing cross-service latency requires distributed tracing (linked spans across all services in one trace) to see which hop is actually slow — gaps between span totals and total latency point to un-instrumented time like queueing or connection-pool waits.

**References:**
- [OpenTelemetry — Observability primer (traces and spans)](https://opentelemetry.io/docs/concepts/observability-primer/)

---

### Q9. What causes an N+1-style problem in service-to-service calls, and how do you fix it? {#q9}

**Question:**
What causes an N+1-style problem in service-to-service calls, and how do you fix it?

**Good answer:**
It's the same shape of problem as the classic ORM N+1 query, just one network hop up: a service (or a client) fetches a list of N items from one service, then makes a separate call to another service *for each item* to fetch related data — one query to list orders, then N calls to a Product service to fetch each order line's product details. Each of those calls carries full network round-trip latency, so what should be roughly constant-time turns into O(N) sequential network calls, which is often far more expensive proportionally than an N+1 in-process DB query was. The fix is to avoid the per-item round trip: either add a batch/bulk endpoint on the downstream service (fetch all N products in one call by ID list) so the fan-out becomes one request instead of N, or use the API Composition pattern at an aggregation layer that issues the batched calls and joins the results in memory before returning to the original caller.

**Follow-up question:**
A batch endpoint fixes N calls down to 1, but the batch itself is now slow for very large N. What would you do?

**Follow-up good answer:**
Cap the batch size and paginate — issue multiple smaller batched calls (e.g. chunks of 100 IDs) potentially in parallel, rather than one unbounded batch that risks timing out the downstream service or blowing past a request-size limit. It's also worth checking whether the design even needs per-request freshness for all that data: if the related data (like product names/prices for a list of orders) changes infrequently relative to how often it's read, caching it locally or in a fast shared cache avoids most of the batch calls entirely, which is usually a bigger win than tuning batch size alone.

**Glossary:**
- **API Composition** — an aggregation layer that calls multiple services and joins results in memory, since cross-service database joins aren't possible.

**Mental model:**
Tests whether the candidate can transfer a pattern they likely already know (ORM N+1) to the distributed-systems version of the same shape of bug, which is a good signal for general pattern-recognition rather than memorized facts.

**TL;DR:**
Service-level N+1 is per-item round trips to a downstream service instead of one batched call — fix it with a bulk endpoint or an API-composition aggregation layer, and paginate/cache if the batch itself gets too large.

**References:**
- [microservices.io — API Composition pattern](https://microservices.io/patterns/data/api-composition.html)

---

### Q10. What does it mean for services to be "chatty," and what's your strategy to reduce it? {#q10}

**Question:**
What does it mean for services to be "chatty," and what's your strategy to reduce it?

**Good answer:**
"Chatty" describes an architecture where completing one logical operation requires many small back-and-forth calls between services, rather than a small number of coarser calls — each individual call might be fast, but the cumulative network round-trip latency (and the cumulative chance of one of those calls failing) dominates the total request time and reliability. It's usually a sign that service boundaries were drawn along a technical seam rather than a business-capability seam, so operations that are conceptually "one thing" end up needing several hops to complete. The fix has two levers: redesign the API surface to be coarser-grained (return everything a client typically needs in one call rather than requiring a follow-up call per field or sub-resource), and reconsider the service boundary itself — if two services are always called together for almost every operation, that's a signal they may belong in the same bounded context and should potentially be merged, rather than kept apart and continuously talking to each other over the network.

**Follow-up question:**
Merging two chatty services back together fixes the network overhead, but why might a team resist doing that?

**Follow-up good answer:**
Because the original split may have been driven by non-technical reasons that are still valid — different teams own each service, they need independent deploy cadences, or they have very different scaling/resource profiles — in which case merging trades away those organizational benefits just to fix a performance symptom that could be addressed another way (coarser APIs, caching, or an aggregation layer). Merging is the right call only when the chattiness reveals that the split never had a real bounded-context justification in the first place; if it did, the better fix is reducing the number/cost of the calls between them, not undoing the boundary.

**Glossary:**
- **Chatty services** — services that require many small round-trip calls between each other to complete one logical operation.

**Mental model:**
Tests judgment, not just diagnosis: does the candidate reach for "merge the services" reflexively, or do they check whether the boundary had a real organizational justification before recommending undoing it.

**TL;DR:**
Chattiness usually means service boundaries were drawn along a technical seam rather than a business one — fix it with coarser-grained APIs/aggregation first, and only consider merging services if the boundary never had a real bounded-context justification.

**References:**
- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)

---

### Q11. What is Conway's Law, and how does it show up — often unintentionally — in service boundaries? {#q11}

**Question:**
What is Conway's Law, and how does it show up — often unintentionally — in service boundaries?

**Good answer:**
Conway's Law, from Melvin Conway's 1968 paper, states that "organizations which design systems... are constrained to produce designs which are copies of the communication structures of these organizations" — in short, your system's architecture will end up mirroring how your teams are organized and communicate, whether or not that's the architecture anyone deliberately chose. In microservices terms, this shows up when service boundaries end up matching team boundaries not because that's the ideal bounded-context split, but simply because each team builds and owns whatever falls within its own communication reach, and cross-team coordination overhead pushes work toward staying inside one team's existing service rather than crossing into someone else's. The practical implication (sometimes called the "Inverse Conway Maneuver") is that if you want a particular service architecture, it often works better to first shape the team structure to match it, rather than trying to impose an architecture that fights against how your organization actually communicates.

**Follow-up question:**
A company reorganizes into cross-functional "stream-aligned" teams around each service. What is this trying to achieve, in Conway's Law terms?

**Follow-up good answer:**
It's a deliberate application of the Inverse Conway Maneuver: instead of letting the org chart passively produce whatever architecture falls out of existing communication patterns, the company reshapes team boundaries and ownership *first*, so that Conway's Law then works in their favor and produces the service architecture they actually want — each stream-aligned team owning one or a few services end-to-end reduces the cross-team coordination that otherwise pressures architecture toward either a shared monolith (when teams are organized by technical layer) or an accidental distributed monolith (when service splits don't match who actually needs to coordinate on changes).

**Glossary:**
- **Conway's Law** — system designs tend to mirror the communication structure of the organization that produces them.
- **Inverse Conway Maneuver** — deliberately shaping team structure first, to steer the architecture Conway's Law will produce.

**Mental model:**
Tests whether the candidate treats architecture as purely a technical decision, or understands that organizational structure is itself a architectural input/constraint — a distinction that separates candidates who've only designed systems from those who've also had to operate them inside a real organization.

**TL;DR:**
Conway's Law says system architecture mirrors organizational communication structure — service boundaries often end up matching team boundaries by default, which is why deliberately shaping team structure (the Inverse Conway Maneuver) is a real architectural lever.

**References:**
- [Melvin Conway — "How Do Committees Invent?"](https://www.melconway.com/Home/Committees_Paper.html)

---

### Q12. How does the Single Responsibility Principle apply at the service level, and how is that different from applying it to a class? {#q12}

**Question:**
How does the Single Responsibility Principle apply at the service level, and how is that different from applying it to a class?

**Good answer:**
At the class level, SRP is usually phrased as "a class should have one reason to change," meaning one cohesive piece of logic, typically judged by code-level cohesion — does this method belong with these fields conceptually. At the service level, the same principle applies to a much coarser unit: a service should own one bounded context / business capability, so it changes for reasons tied to that one capability, not because it's a grab-bag of unrelated responsibilities that happen to share a deployable. The key difference is the cost of getting it wrong: a class with too many responsibilities is a refactor within one codebase; a service with too many responsibilities means multiple teams or multiple unrelated concerns are now coupled to the same deploy cycle, the same scaling profile, and the same on-call rotation — the same principle, but a mistake here is a much more expensive one to unwind later (a service split is a distributed-systems change, not an in-IDE refactor).

**Follow-up question:**
How would you tell, in practice, that a service has actually violated SRP rather than just being "a bit big"?

**Follow-up good answer:**
Look for signals like: the service changes for multiple unrelated business reasons (e.g. both pricing logic changes and shipping-label logic changes require touching the same service), different parts of the service are owned/modified almost exclusively by different teams, or different parts of it have wildly different scaling/reliability needs (one part is read-heavy and latency-sensitive, another is a rarely-called batch job) forcing the whole service to be provisioned for its most demanding part. Size alone isn't the signal — a service can be large but cohesive if everything in it genuinely belongs to the same bounded context; the violation is about *reasons to change* and *ownership*, not lines of code.

**Glossary:**
- **Single Responsibility Principle (SRP)** — a component should have one, and only one, reason to change.

**Mental model:**
Tests whether the candidate can apply a familiar class-level principle at a different altitude and correctly identify that the *cost of a violation* — not just the principle's wording — is what actually changes between the two levels.

**TL;DR:**
SRP applies to services the same way it does to classes — one reason to change — but a violation at the service level is far costlier to fix, since undoing it means a distributed-systems change instead of an in-codebase refactor.

**References:**
- [Martin Fowler — Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)

---

### Q13. What organizational/operational problems does splitting into microservices actually solve? {#q13}

**Question:**
What organizational/operational problems does splitting into microservices actually solve?

**Good answer:**
The core wins are independent deployability and independent scaling. When services are truly independent, one team can ship a change to their service without coordinating a release with every other team, which removes the release-train bottleneck that large monoliths tend to develop as headcount grows. Independent scaling means a service under heavy load can be given more resources (or scaled horizontally) without over-provisioning the entire system to cover that one hot path. It also gives you fault isolation — a bug or resource exhaustion in one service doesn't necessarily take down the ability to serve any other functionality — and technology flexibility, letting a team choose the language/database/runtime that best fits their specific service's problem instead of everyone being locked to the monolith's stack (Fowler's "polyglot persistence" and, by extension, polyglot everything). None of these are free — they're bought with real distributed-systems complexity — but they're the actual problems the pattern solves, as opposed to "microservices are just better," which isn't true in general.

**Follow-up question:**
A 5-person startup adopts microservices from day one, citing "we want to scale like Netflix." What's wrong with that reasoning?

**Follow-up good answer:**
The benefits above only pay off once you actually have the organizational pressure that creates the problems they solve — multiple teams needing independent release cadences, genuinely different scaling profiles across parts of the system, or a real need for fault isolation at scale. A 5-person team has none of that yet: there's no release-train bottleneck to relieve with 5 people on one team, no scaling hot-spot that's been observed under real load, and no organizational boundary to preserve. What they get instead is all of the cost — network latency, partial failure handling, operational overhead of running and monitoring N services instead of one — with none of the benefit realized, which is precisely the case Fowler's "monolith first" argument is built on.

**Glossary:**
- **Fault isolation** — a failure in one component doesn't necessarily propagate to or take down other components.
- **Polyglot persistence** — different services using different data storage technologies best suited to each one's needs.

**Mental model:**
Tests whether the candidate can articulate microservices' benefits as *conditional on real pressures*, not as a universal good — and whether they'll push back on a bad real-world justification for adopting them rather than agreeing with whatever the interviewer's hypothetical company decided.

**TL;DR:**
Microservices solve independent deployability, independent scaling, fault isolation, and technology flexibility — but only pay off once you actually have the organizational/scaling pressure those solve; without it, you pay the cost for no benefit.

**References:**
- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)

---

### Q14. What new problems does a microservices architecture introduce that a monolith doesn't have? {#q14}

**Question:**
What new problems does a microservices architecture introduce that a monolith doesn't have?

**Good answer:**
The biggest one is that you lose the ability to use ACID transactions across what used to be one operation, because each service owns its own database — you now need patterns like Saga to handle multi-step operations and accept eventual consistency in places that used to be atomic and immediately consistent. You also inherit the full set of distributed-systems failure modes: partial failure (some services up, some down, mid-request), network partitions and timeouts, and the need for resilience patterns (retries, circuit breakers, timeouts) that a single-process monolith never had to think about, because in-process function calls don't fail the way network calls do. Operationally, you're now running, deploying, monitoring, and debugging N services instead of one — which means N sets of logs to correlate (requiring distributed tracing), N deployment pipelines, and a much larger surface area for configuration and infrastructure to get wrong. None of this is a reason to avoid microservices when the organizational benefits justify it, but it's real cost, not a footnote.

**Follow-up question:**
Which of these new problems tends to surprise teams the most, in your experience, and why?

**Follow-up good answer:**
Eventual consistency tends to be the most surprising, because it's not just an infrastructure problem, it's a product/UX problem: a customer can see "order confirmed" and then, seconds later, see an inventory-related failure surface as a separate notification, which never happened when the whole flow was one atomic transaction that either fully succeeded or fully failed before responding. Teams often don't design the product experience (or the customer support/ops runbooks) for that reality until it happens in production, whereas the purely technical problems (retries, circuit breakers, tracing) at least show up early during load testing and are more likely to already be on an engineering team's radar.

**Glossary:**
- **Eventual consistency** — a system reaches a consistent state over time rather than immediately, common when data changes are propagated asynchronously across services.
- **Partial failure** — some components of a distributed operation succeed while others fail, unlike an atomic monolith operation which succeeds or fails as a whole.

**Mental model:**
Tests balance: a candidate who can only list microservices' benefits (from Q13) without being able to list the costs just as fluently hasn't actually weighed the trade-off, they've memorized a sales pitch.

**TL;DR:**
Microservices trade away cross-service ACID transactions and in-process reliability for distributed-systems failure modes (partial failure, network issues) and N-times the operational surface area to run, monitor, and debug.

**References:**
- [microservices.io — Saga pattern](https://microservices.io/patterns/data/saga.html)

---

### Q15. What is the "distributed monolith" anti-pattern, and how does a team end up there? {#q15}

**Question:**
What is the "distributed monolith" anti-pattern, and how does a team end up there?

**Good answer:**
A distributed monolith is a system that's been physically split into separate deployable services, but is still tightly coupled in ways that force those services to be deployed together (or at least in careful lockstep), meaning the team pays every cost of running a distributed system — network latency, partial failure, operational overhead — while getting almost none of the independent-deployability benefit that was the point of splitting in the first place. A common cause, described by Ben Christensen, is heavy reliance on shared internal libraries that every service depends on for core behavior ("the platform") — when a shared library changes, every service that depends on it may need to be rebuilt and redeployed together, which is functionally the same coordination cost as a monolith's single deploy, just spread across a network. It also happens when service boundaries were drawn along technical layers (all UI-tier services, all business-logic-tier services) instead of business capabilities, so any single business change still ripples across several services that all have to change and deploy together.

**Follow-up question:**
Two services need to deploy together because Service A calls a new endpoint on Service B that doesn't exist in production yet. Is that a distributed monolith?

**Follow-up good answer:**
Not necessarily on its own — a single instance of temporary deployment ordering (deploy B first, then A) is a normal, manageable part of rolling out a backward-compatible API change, and doesn't by itself mean the architecture is a distributed monolith. It becomes the anti-pattern when this is the *default state* of the system rather than an occasional careful sequencing: if most changes require coordinating deploys across multiple services, or if backward-incompatible changes are common because there's no discipline around versioning/compatibility, that's the signal that the services are architecturally coupled, not just occasionally coordinated. The fix for the coupling is designing APIs to be backward-compatible by default (additive changes, versioned breaking changes) so that "which order do we deploy these in" stops being a routine question.

**Glossary:**
- **Distributed monolith** — services that are physically separate but must be deployed together (or in tightly coordinated lockstep) due to underlying coupling.

**Mental model:**
Tests whether the candidate can distinguish an anti-pattern from a normal, occasional operational reality — over-eager labeling of anything with a deploy dependency as "a distributed monolith" is itself a red flag of shallow understanding.

**TL;DR:**
A distributed monolith is services split across the network that still must deploy together due to coupling (often via shared internal libraries or technical-layer boundaries) — you pay for a distributed system without getting independent deployability.

**References:**
- [InfoQ — Microservices and the "distributed monolith"](https://www.infoq.com/news/2016/02/services-distributed-monolith)

---

### Q16. What is the shared-database anti-pattern, and why does it undermine microservices' benefits? {#q16}

**Question:**
What is the shared-database anti-pattern, and why does it undermine microservices' benefits?

**Good answer:**
It's when multiple services read from and write to the same physical database (or the same tables) directly, instead of each service owning its own private data store and only exposing data through its API. The standard microservices data pattern is "database per service" — private tables, a private schema, or an entirely separate database server per service — specifically so that a service's internal data model can change freely without breaking anyone else, and so a service is the only thing that can enforce its own invariants on its own data. A shared database defeats both of those: any service can read or write another service's data directly, bypassing whatever business rules that owning service enforces in its own code, and a schema change to support one service's needs can silently break every other service reading that same table. It also reintroduces exactly the tight coupling microservices were meant to remove — services can no longer be deployed independently if a shared schema change requires every consumer to update in lockstep.

**Follow-up question:**
A team argues that sharing one database is actually simpler and avoids the "eventual consistency" problems of Saga-based coordination. What's the response?

**Follow-up good answer:**
It is simpler in the short term, and if the team's actual requirement is strong, immediate cross-entity consistency with simple joins, that's a legitimate argument for *not* splitting those entities into separate services in the first place — a shared-database "microservices" split that isn't really decoupled is arguably worse than an honest monolith, because it has the monolith's coupling plus the distributed system's operational overhead. The response isn't "always avoid shared databases at any cost," it's "if you need a shared database to make this work, that's evidence these aren't actually separate bounded contexts yet, and you should either keep them as one service, or invest in proper per-service ownership and accept eventual consistency where it's genuinely acceptable."

**Glossary:**
- **Database per service** — each service owns a private data store, accessible only through that service's own API.
- **Shared database anti-pattern** — multiple services reading/writing the same database or tables directly, bypassing service ownership of data.

**Mental model:**
Tests whether the candidate treats "database per service" as dogma or as a means to an end (independent deployability, enforced invariants) — the best answers recognize that a shared-database need is diagnostic information about whether the split was right, not just a rule being broken.

**TL;DR:**
Sharing a database across services lets them bypass each other's business rules and couples their schemas together, undoing the independent deployability and data ownership that "database per service" is meant to provide.

**References:**
- [microservices.io — Database per Service pattern](https://microservices.io/patterns/data/database-per-service.html)

---

### Q17. Why does Martin Fowler's "MonolithFirst" advice argue against starting a brand-new project with microservices? {#q17}

**Question:**
Why does Martin Fowler's "MonolithFirst" advice argue against starting a brand-new project with microservices?

**Good answer:**
Two reasons. First, YAGNI: a new application needs to prioritize fast iteration and cheap feedback loops while the team is still figuring out what the product even is, and microservices add real overhead (network calls, deployment infrastructure, cross-service debugging) to every one of those iteration cycles — overhead that's a poor trade when you don't yet know if you'll need the benefits it buys. Second, and more fundamentally: drawing good service boundaries requires already understanding the domain well enough to know where the natural bounded contexts are, and that understanding is exactly what's missing at the start of a new project — boundaries drawn too early are usually wrong, and unlike refactoring a class boundary inside a monolith, refactoring a *service* boundary means changing a network contract, migrating data across two datastores, and coordinating a deploy, which is drastically more expensive. Fowler's own observation is that almost every successful microservices system he'd seen had started as a monolith that later got broken up, while systems built as microservices from scratch tended to run into serious trouble.

**Follow-up question:**
If a team follows this advice and builds a monolith first, what should they actually be doing along the way to make a later split feasible?

**Follow-up good answer:**
Keep the monolith modular internally — clear, well-enforced boundaries between packages/modules along what look like natural business-capability lines, even though they all still run in one process and share one database. That internal discipline is what makes an eventual extraction tractable: if a module already has a clean interface and doesn't reach into other modules' internals or tables, pulling it out into its own service later is a comparatively contained exercise; if the whole codebase is a tangle with no internal boundaries, "start with a monolith" quietly turns into "build a big ball of mud that's now even harder to split than if you'd never tried to keep it clean."

**Glossary:**
- **YAGNI** — "You Aren't Gonna Need It," the principle of not building for a capability until you actually need it.

**Mental model:**
Tests whether the candidate can articulate a nuanced, non-dogmatic position (monolith first, but a *modular* one) rather than a simplistic "monoliths bad" or "microservices bad" stance — and whether they know the advice comes with a specific caveat about discipline, not permission to skip architecture entirely.

**TL;DR:**
MonolithFirst argues that new projects don't yet know their real domain boundaries or need the deploy/scale benefits microservices buy, so paying that overhead early is a bad trade — but the monolith should stay internally modular so a later split is tractable.

**References:**
- [Martin Fowler — MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)

---

### Q18. What is the Strangler Fig pattern, and how is it used to migrate a legacy monolith incrementally? {#q18}

**Question:**
What is the Strangler Fig pattern, and how is it used to migrate a legacy monolith incrementally?

**Good answer:**
Named after a vine that grows around a host tree and gradually takes over its structure until the tree beneath can die away, the Strangler Fig pattern replaces a legacy system incrementally: new functionality (or migrated pieces of old functionality) is built as new services, while a routing layer in front of the old and new systems directs each request to whichever one currently owns that piece of functionality. Over time, more and more traffic is routed to the new services and less to the legacy system, until eventually the legacy system handles nothing and can be decommissioned — at no point does the team attempt a single big-bang rewrite-and-cutover, which is the higher-risk alternative this pattern is specifically designed to avoid. Fowler frames it as four activities: clarify what outcome you actually want, decompose the legacy system to find seams where it can be split, replace pieces incrementally behind the router, and change how the organization builds software along the way so the new system doesn't just become an equally-brittle replacement.

**Follow-up question:**
What's the hardest part of Strangler Fig migrations in practice, and why does it usually have nothing to do with writing the new services?

**Follow-up good answer:**
The hardest part is usually the routing/interception layer and data synchronization between old and new during the transition — figuring out how to intercept and redirect specific requests without the legacy system noticing, and how to keep data consistent when the same logical entity might be read by the legacy system and written by a new service (or vice versa) during the migration window. Writing a new, clean service in isolation is comparatively easy; making it coexist correctly with a legacy system it's gradually replacing, for however long the migration takes, is where most of the real engineering risk and effort lives — which is exactly why "just rewrite it" often underestimates the actual work by focusing only on the new code.

**Glossary:**
- **Strangler Fig pattern** — incremental system replacement via a routing layer that shifts traffic from legacy to new functionality piece by piece.

**Mental model:**
Tests whether the candidate understands migration as a first-class engineering problem in its own right (routing, dual-write/dual-read consistency, rollback safety) rather than just "write the new thing and switch over," which is the naive framing that leads teams into risky big-bang rewrites.

**TL;DR:**
Strangler Fig migrates a legacy system incrementally by routing more and more traffic to new services over time, avoiding a risky big-bang rewrite — the hard part is usually the routing/data-sync layer during the transition, not the new services themselves.

**References:**
- [Martin Fowler — StranglerFigApplication](https://martinfowler.com/bliki/StranglerFigApplication.html)

---

### Q19. What is a service mesh, and what problem does the sidecar proxy pattern solve? {#q19}

**Question:**
What is a service mesh, and what problem does the sidecar proxy pattern solve?

**Good answer:**
A service mesh is an infrastructure layer that manages service-to-service communication for you — handling things like load balancing, retries, timeouts, mutual TLS, and telemetry collection — separately from the application code that's actually doing the business logic. The sidecar proxy pattern is how most service meshes (like Istio) implement this: a small proxy process is deployed alongside every service instance (in Kubernetes, as a second container in the same Pod), and all network traffic in and out of that service is routed through its sidecar rather than going directly to/from the application. Because the sidecar intercepts every call, it can transparently apply traffic policies (routing rules, circuit breaking), collect uniform telemetry across every service regardless of what language it's written in, and enforce service identity/mTLS — all without the application code needing to implement any of that itself or even know it's happening.

**Follow-up question:**
If a service mesh can add retries, timeouts, and circuit breaking transparently, why would an application ever still implement that logic itself?

**Follow-up good answer:**
The mesh operates at the network layer and doesn't understand your business logic, so it's good at generic, protocol-level resilience (retry a failed HTTP call, time out a slow one, trip a circuit breaker on error rate) but can't make business-aware decisions — like whether it's safe to retry a specific non-idempotent operation, what a sensible application-specific fallback value is when a dependency is down, or how to handle a partial failure inside a multi-step business transaction (the Saga territory from earlier). Teams typically let the mesh handle the generic transport-level resilience uniformly across every service, and keep business-logic-aware resilience decisions (idempotency keys, fallback behavior, compensating actions) in the application code where the necessary context actually lives.

**Glossary:**
- **Service mesh** — an infrastructure layer that manages inter-service traffic, security, and observability outside application code.
- **Sidecar proxy** — a proxy process deployed alongside each service instance that intercepts and manages its network traffic.

**Mental model:**
Tests whether the candidate understands the mesh's actual scope (transport-layer concerns) versus its limits (no business-logic awareness), rather than treating it as a magic box that eliminates the need to think about resilience in application code at all.

**TL;DR:**
A service mesh manages inter-service traffic/security/telemetry via a sidecar proxy next to each service instance, transparently and language-agnostically — but it only handles generic transport-level resilience, not business-aware decisions like safe retries or compensating actions.

**References:**
- [Istio — Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)

---

### Q20. Given a mid-sized product built by a single team as a monolith, what specific signals would tell you it's time to consider splitting out a service? {#q20}

**Question:**
Given a mid-sized product built by a single team as a monolith, what specific signals would tell you it's time to consider splitting out a service?

**Good answer:**
Look for concrete, observed pressure rather than a general feeling that "microservices are the modern way." Signals worth acting on: a specific part of the system has a genuinely different scaling profile under real load (e.g. an image-processing feature needs 10x the CPU of everything else, and scaling the whole monolith to match wastes resources on the other 90%); a specific capability needs a much faster or more independent release cadence than the rest of the product (a pricing engine that needs to ship rule changes several times a day while the rest of the app releases weekly); a distinct team is forming around a specific capability and is being slowed down by needing to coordinate every release with everyone else; or a piece of functionality has fundamentally different reliability/availability requirements (must stay up during a deploy of everything else) that the shared deploy cycle can't provide. Absent these, "it would be more scalable" or "it's cleaner architecture" alone aren't strong enough justifications — they're the reasoning MonolithFirst specifically warns against.

**Follow-up question:**
The team decides to extract the pricing engine into its own service based on release-cadence pressure. What should they check *before* extracting, to make sure it'll actually work?

**Follow-up good answer:**
Check whether pricing already has a clean logical boundary inside the monolith — its own module/package with a narrow, well-defined interface and no other code reaching into its internals or its share of the database directly. If it does, extraction is comparatively low-risk: the interface becomes an API contract, and the internal implementation doesn't need to change much. If pricing logic and data are tangled throughout the codebase (other modules query pricing's tables directly, or pricing calculations are duplicated/scattered rather than centralized), extracting it first requires doing that internal cleanup — turning it into a real bounded context — before the service split will actually deliver the independence the team is hoping for; skipping that step is how you end up with a distributed monolith on day one of the very first extraction.

**Glossary:**
- **Release cadence** — how frequently a component can be deployed independently of the rest of the system.

**Mental model:**
This is the capstone trade-off question for the set: tests whether the candidate has internalized every prior answer as one coherent decision framework (organizational pressure justifies the cost; bounded-context cleanliness determines whether the extraction will actually succeed) rather than a list of disconnected facts about patterns.

**TL;DR:**
Split out a service when there's real, observed pressure (scaling profile, release cadence, team ownership, or reliability requirements) that a shared monolith can't satisfy — and check the candidate module already has a clean bounded-context boundary before extracting it, or the split will start out as a distributed monolith.

**References:**
- [Martin Fowler — MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=architecture&tags=microservices-vs-monoliths-and-service-boundaries&autostart=1" | relative_url }})
