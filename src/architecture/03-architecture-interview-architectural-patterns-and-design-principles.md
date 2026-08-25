---
layout: default
title: "Architecture Interview — Architectural Patterns and Design Principles"
---

# Architecture Interview — Architectural Patterns and Design Principles

This set covers the design principles (SOLID) and architectural patterns (layered, hexagonal/ports-and-adapters, Clean Architecture, CQRS, Event Sourcing) that senior interviews probe for judgment, not just recall — knowing the definitions is table stakes; knowing when applying them is worth the complexity tax is what separates levels.

### Q1. What is the Dependency Inversion Principle, and how is it different from dependency injection? {#q1}

**Question:**
What is the Dependency Inversion Principle, and how is it different from dependency injection?

**Good answer:**
The Dependency Inversion Principle (the "D" in SOLID) states that high-level modules should not depend on low-level modules — both should depend on abstractions, and abstractions should not depend on details; details should depend on abstractions. Concretely: your business logic should define an interface it needs (e.g. `PaymentGateway`), and the low-level implementation (e.g. `StripeGateway`) depends on that interface, not the other way around. This inverts the "natural" dependency direction where high-level code imports and calls concrete low-level code directly.

Dependency injection is a *mechanism*, not the principle itself — it's one common way to satisfy DIP by supplying a dependency (usually via constructor or setter) from outside rather than having a class instantiate its own collaborators. You can follow DIP without a DI framework (manual wiring, factory functions); you can also use a DI framework while still violating DIP if the injected type is a concrete class rather than an abstraction.

**Follow-up question:**
Can you give an example of code that uses dependency injection but still violates DIP?

**Follow-up good answer:**
Yes — if a class's constructor takes a concrete `MySqlUserRepository` parameter instead of a `UserRepository` interface, that's dependency injection (the dependency is supplied externally) but not dependency inversion (the high-level class still depends on a low-level, concrete detail; swapping databases means changing the high-level class's type signature). True DIP requires the injected type to be an abstraction the high-level module owns or that's shared and stable, so the low-level module is the one depending on it, not vice versa.

**Glossary:**
- **High-level module** — code that encodes policy/business rules (what the system should do).
- **Low-level module** — code that encodes implementation detail (databases, frameworks, I/O).
- **Dependency injection** — supplying an object's dependencies from outside rather than having it construct them itself.

**Mental model:**
Tests whether a candidate can distinguish a *principle* (a rule about the direction of source-code dependencies) from a *mechanism* (constructor injection, a DI container) — a very common conflation that reveals whether someone learned DIP from a framework tutorial or actually understands the OOP design theory behind it.

**TL;DR:**
DIP is about dependency *direction* (both sides depend on an abstraction); dependency injection is just one mechanism for wiring that up, and you can use DI while still violating DIP if you inject concrete types.

**References:**
- [SOLID Principles of Object-Oriented and Agile Design, Robert C. Martin](https://gist.github.com/OddExtension5/5590a2a8197a31aa3bf1d4ca3ee20f83)

---

### Q2. What is the "Dependency Rule" in Clean Architecture, and why must dependencies always point inward? {#q2}

**Question:**
What is the "Dependency Rule" in Clean Architecture, and why must dependencies always point inward?

**Good answer:**
The Dependency Rule states that source code dependencies can only point inward, toward higher-level policy. Clean Architecture organizes code into concentric layers — Entities (enterprise-wide business rules) at the center, then Use Cases (application-specific logic), then Interface Adapters (controllers, presenters, gateways), then Frameworks & Drivers (web framework, database, UI) at the outer edge. Nothing in an inner circle may know anything about an outer circle — not a class name, not a function signature, not even that the outer thing exists.

The reason is stability and testability: business rules (the most valuable, most expensive-to-get-wrong part of the system) should not need to change when you swap a web framework or database. If the Entities layer imported a database driver type, every database migration would ripple into your core logic. Inward-only dependencies mean the core can be tested with no framework, no database, and no UI running at all.

**Follow-up question:**
Control flow often needs to go from an inner layer out to an outer layer (e.g. a use case needs to save data). How does Clean Architecture reconcile that with the Dependency Rule?

**Follow-up good answer:**
Via the Dependency Inversion Principle at the boundary: the inner layer defines an interface (e.g. a `UserRepository` port) that it needs, and the outer layer provides a concrete implementation (e.g. `PostgresUserRepository`) that implements that interface. At runtime, control flows outward when the use case calls the repository; but the *source code dependency* (the `import`/reference to the interface) still points inward, because the outer implementation depends on the inner interface, not the reverse. Only plain data structures cross the boundary — never framework-specific types like an ORM row.

**Glossary:**
- **Entities** — enterprise-wide business rules, the most stable layer.
- **Use cases** — application-specific orchestration logic.
- **Interface adapters** — converters between use-case-friendly data shapes and external formats.

**Mental model:**
Probes whether the candidate understands that "dependency" here means *source code reference direction*, not runtime call direction — a subtlety that trips up people who've only skimmed a diagram of the concentric circles.

**TL;DR:**
Source-code dependencies in Clean Architecture must point inward only, toward business rules — control can still flow outward at runtime because outer layers implement interfaces defined by inner layers (DIP at the boundary).

**References:**
- [The Clean Architecture, Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

### Q3. What problem does Hexagonal Architecture (Ports & Adapters) solve, and what exactly is a "port"? {#q3}

**Question:**
What problem does Hexagonal Architecture (Ports & Adapters) solve, and what exactly is a "port"?

**Good answer:**
Hexagonal Architecture, introduced by Alistair Cockburn, addresses business logic leaking into presentation and persistence code. When that happens, three things break: you can't test the logic without spinning up the UI or database, you can't run the same logic behind a different driver (e.g. batch job instead of a human at a screen), and program-to-program integration becomes awkward because the "real" logic is entangled with one specific delivery mechanism.

A **port** is a defined interface representing a purposeful conversation between the application core and the outside world — e.g. an `OrderRepository` port for persistence, or a `NotificationSender` port for outbound messages. It's technology-agnostic by design: the core only knows the port's method signatures, never who or what is on the other side. An **adapter** is the technology-specific implementation that plugs into a port — a Postgres adapter for `OrderRepository`, an SMTP adapter for `NotificationSender`, or a test double for either. The application sits "blissfully ignorant" of which adapter is actually connected.

**Follow-up question:**
How is a "driving" (primary) port different from a "driven" (secondary) port?

**Follow-up good answer:**
A driving/primary port is one the outside world uses to call *into* the application — e.g. a `PlaceOrderUseCase` interface that a REST controller or a CLI adapter invokes to trigger behavior. A driven/secondary port is one the application uses to call *out* to the world — e.g. `OrderRepository`, which a database adapter implements. The distinction matters because it clarifies who depends on whom: for driving ports, the adapter depends on (calls) the application's interface; for driven ports, the application depends on (calls) an interface that the adapter implements — in both cases the port lives with the application core, and adapters plug in from outside, but the direction of the "calling" relationship differs.

**Glossary:**
- **Port** — a technology-agnostic interface defining an interaction between the core and the outside.
- **Adapter** — a concrete, technology-specific implementation that connects to a port.
- **Driving/primary adapter** — initiates interaction with the application (e.g. a UI, a REST controller).
- **Driven/secondary adapter** — is called by the application (e.g. a database, an email provider).

**Mental model:**
Checks whether the candidate can explain hexagonal architecture in terms of *why* (testability, technology substitutability) rather than just reciting "ports and adapters" as jargon, and whether they grasp the driving/driven distinction that's easy to gloss over.

**TL;DR:**
Hexagonal architecture isolates the application core behind technology-agnostic ports so any adapter (UI, database, test harness) can plug in without the core ever knowing or caring which one is connected.

**References:**
- [Hexagonal Architecture, Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)

---

### Q4. Explain CQRS — what does it actually separate, and when does Martin Fowler say it's not worth using? {#q4}

**Question:**
Explain CQRS — what does it actually separate, and when does Martin Fowler say it's not worth using?

**Good answer:**
CQRS (Command Query Responsibility Segregation) splits the model used to update information (commands) from the model used to read information (queries), rather than using one conceptual model for both. In practice this often means a distinct write model (optimized for validating and applying business rules) and a separate read model (denormalized, optimized for the exact queries the UI needs) — sometimes on entirely different data stores or processes.

Fowler is explicit that CQRS should be applied narrowly, to a specific bounded context, never to a whole system, and only pays off in two cases: genuinely complex domains where separating read/write logic simplifies each side, or workloads with a large disparity between read and write volume/latency needs, where independent scaling matters. Outside those cases he calls it "risky complexity" — he's seen it cause serious difficulties on systems that would have been fine as plain CRUD, because you now have two models to keep consistent, more moving parts, and often eventual consistency to reason about.

**Follow-up question:**
If you introduce CQRS for one bounded context in an otherwise CRUD system, what new operational concern do you take on that a single-model CRUD system doesn't have?

**Follow-up good answer:**
Read/write consistency lag: if the read model is built asynchronously from the write model (e.g. via events or a projection process), a client can write data and then immediately query it and see stale results until the read side catches up. You now have to design for eventual consistency at the UI/API level (e.g. optimistic UI updates, "your change is being processed" messaging, or read-your-writes techniques like routing a user's own reads to the write store temporarily) — none of which exists in a single-model CRUD system where a write is immediately visible to the next read.

**Glossary:**
- **Command** — an operation that changes state and (typically) doesn't return data.
- **Query** — an operation that returns data without changing state.
- **Projection** — a read-optimized view built (often asynchronously) from write-side events or data.

**Mental model:**
Distinguishes candidates who've used CQRS as a buzzword from those who understand it's a narrow, bounded-context-scoped tool with a real consistency cost — interviewers want to hear the caveats, not just the definition.

**TL;DR:**
CQRS splits read and write models for one bounded context at a time, and is worth it only for genuinely complex domains or large read/write load asymmetry — everywhere else it's complexity without payoff.

**References:**
- [CQRS, Martin Fowler](https://martinfowler.com/bliki/CQRS.html)

---

### Q5. How does Event Sourcing reconstruct application state, and what problem do snapshots solve? {#q5}

**Question:**
How does Event Sourcing reconstruct application state, and what problem do snapshots solve?

**Good answer:**
In Event Sourcing, every state change is captured and persisted as an immutable event (e.g. `OrderPlaced`, `OrderShipped`), rather than only storing the current state. Current state is a derived value: to get it, you start from a blank/initial state and replay every event for that entity, in order, applying each one's effect — the state at any point in time can be reconstructed by replaying events up to that point. This gives you a complete audit trail for free and lets you answer "what did this look like last Tuesday" without a separate history table.

The problem snapshots solve is replay cost: an entity with years of history might have thousands of events, and replaying all of them on every load is wasteful. A snapshot periodically persists the fully-computed state at a point in time (e.g. end of day), so a later load only needs the snapshot plus events *since* that snapshot, not the entire history.

**Follow-up question:**
What complication does Event Sourcing introduce when an event handler needs to call an external system, like a payment gateway or a currency-exchange-rate API?

**Follow-up good answer:**
Two related problems. First, replaying events must not re-trigger side effects: if applying an `OrderPlaced` event during replay re-calls the payment gateway, you'd double-charge the customer — so external calls have to happen once, at the time the event was originally generated (e.g. in a separate command-handling step), and replay must only reapply the *resulting state change*, not the side effect itself. Second, for queries to external systems, the events must capture historically-accurate values at the time they occurred (e.g. the exchange rate used), because a system replayed today shouldn't ask "what's today's exchange rate" for an event from a year ago — the answer would be wrong and could even fail if the external system doesn't expose historical data at all.

**Glossary:**
- **Event** — an immutable record of something that happened, used as the unit of persistence.
- **Replay** — reconstructing state by reapplying a sequence of events from the beginning (or a snapshot).
- **Snapshot** — a periodically saved, fully-computed state used to shortcut replay.

**Mental model:**
Tests whether the candidate has actually operated an event-sourced system (they'll immediately bring up side effects and snapshots) versus having only read the one-paragraph pitch about "immutable audit log."

**TL;DR:**
State is a projection of replayed events, not a stored fact; snapshots cap replay cost by periodically checkpointing computed state, and side-effect-triggering logic must be kept out of the replay path.

**References:**
- [Event Sourcing, Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)

---

### Q6. What is an anemic domain model, and why does Martin Fowler consider it an anti-pattern? {#q6}

**Question:**
What is an anemic domain model, and why does Martin Fowler consider it an anti-pattern?

**Good answer:**
An anemic domain model looks like proper object-oriented design at a glance — it has classes with names like `Order` and `Customer` and relationships between them — but those classes are just data holders: getters, setters, and no behavior. All the actual logic (validation, calculations, business rules) lives in separate "service" classes that pull data out of the anemic objects, operate on it procedurally, and push it back.

Fowler calls this an anti-pattern for three reasons: it violates the basic idea of object orientation (bundling data with the behavior that operates on it); it pays the *cost* of a domain model — the mapping/persistence machinery (ORMs, etc.) needed to hydrate rich objects — while getting none of the *benefit*, since there's no real behavior to organize; and it tends to spiral, because once "logic goes in services" becomes the norm, all business rules end up scattered across services with duplicate validation and no single place that enforces an entity's invariants.

**Follow-up question:**
What's a concrete symptom in code review that suggests a domain model has become anemic?

**Follow-up good answer:**
A domain class exposes public setters for fields that should only ever change together or under specific business rules (e.g. `setStatus()` on an `Order` that lets any code jump straight to `SHIPPED` without going through `PLACED` → `PAID` → `SHIPPED`), and a separate `OrderService` class contains the actual state-transition logic and validation that should have been a method on `Order` itself (e.g. `order.ship()`, which internally checks preconditions). If you can construct or mutate a domain object into an invalid state from outside its own class, the model has likely gone anemic.

**Glossary:**
- **Domain model** — an object model that encapsulates both data and the business logic operating on it.
- **Invariant** — a condition that must always hold true for an object to be valid.
- **Transaction Script** — a simpler procedural pattern where each operation is a top-to-bottom script; fine on its own, but a mismatch when paired with domain-model-style persistence machinery.

**Mental model:**
Checks whether a candidate can recognize this extremely common real-world anti-pattern in code they've actually worked with, not just define it abstractly.

**TL;DR:**
An anemic model has the shape of OO (classes, relationships) but none of the substance (behavior); logic ends up scattered in services, so you pay for a rich-domain-model's persistence overhead while getting a procedural design's lack of encapsulation.

**References:**
- [AnemicDomainModel, Martin Fowler](https://martinfowler.com/bliki/AnemicDomainModel.html)

---

### Q7. State the Liskov Substitution Principle, and give a concrete example of a violation. {#q7}

**Question:**
State the Liskov Substitution Principle, and give a concrete example of a violation.

**Good answer:**
The Liskov Substitution Principle says objects of a superclass should be replaceable with objects of a subclass without altering the correctness of the program — i.e. any code written against the base type's contract must keep working, unmodified, when given any subtype. This is about *behavioral* compatibility, not just matching method signatures: a subclass must honor the base class's preconditions (not strengthen them), postconditions (not weaken them), and invariants.

The textbook violation is `Square extends Rectangle` where `Rectangle` has independent `setWidth()`/`setHeight()`. Code that does `rect.setWidth(5); rect.setHeight(10); assert rect.area() == 50` is correct for any `Rectangle`, but breaks for a `Square` (which must keep width == height, so setting one silently changes the other) — the subtype has silently strengthened a precondition/changed a postcondition the base type's clients relied on, even though `Square` is a `Rectangle` in ordinary geometric terms.

**Follow-up question:**
Is inheriting from `Rectangle` with `Square` always wrong, or does it depend on the interface `Rectangle` exposes?

**Follow-up good answer:**
It depends entirely on the contract. If `Rectangle` exposes mutable, independent `setWidth`/`setHeight`, then `Square` can't honor that contract and the inheritance violates LSP. But if `Rectangle` were immutable — constructed once with width and height, no setters — a `Square` subtype (constructed with one side length, reporting equal width/height) could satisfy that narrower contract without any client-visible surprise, because there's no operation whose behavior the subtype would need to weaken. LSP violations are a property of the specific interface/contract in use, not an inherent fact about the real-world "is-a" relationship between the concepts.

**Glossary:**
- **Precondition** — what must be true before a method is called for it to behave correctly.
- **Postcondition** — what the method guarantees is true after it returns.
- **Invariant** — a condition the type guarantees holds at all times, from a client's perspective.

**Mental model:**
Tests whether the candidate understands LSP as a statement about *behavioral contracts*, not about real-world taxonomy (a Square genuinely "is a" Rectangle geometrically, yet the code relationship can still violate LSP) — a distinction that separates rote SOLID recitation from actual understanding.

**TL;DR:**
LSP violations happen when a subtype breaks the behavioral contract clients rely on for the base type — the classic mutable Rectangle/Square example shows the violation is about the interface's contract, not the real-world "is-a" relationship.

**References:**
- [The Liskov Substitution Principle, Baeldung on Computer Science](https://www.baeldung.com/cs/liskov-substitution-principle)

---

### Q8. What does the Interface Segregation Principle solve that Single Responsibility doesn't already cover? {#q8}

**Question:**
What does the Interface Segregation Principle solve that Single Responsibility doesn't already cover?

**Good answer:**
The Interface Segregation Principle says no client should be forced to depend on methods it doesn't use — favor several small, client-specific interfaces over one large, general-purpose one. Single Responsibility is about why a *class* changes (it should have one reason to change); ISP is about the shape of the *interface* a class exposes to its consumers, independent of how the implementing class is organized internally.

They're complementary but distinct: a class could satisfy SRP (one clear responsibility) while still exposing a "fat" interface that bundles unrelated operations a given caller doesn't need — e.g. a `Worker` interface with both `work()` and `eat()`, forced on a `RobotWorker` that has no meaningful `eat()`. ISP would split that into `Workable` and `Eatable` so `RobotWorker` only implements what it actually supports, without being forced to provide (or stub out) a meaningless `eat()` method just because it's coupled to `work()` in one interface.

**Follow-up question:**
How does violating ISP cause a ripple effect during changes, even if the violating class itself never changes?

**Follow-up good answer:**
Any client that depends on the fat interface is coupled to *all* of its methods, even the ones it never calls — so if the interface changes (a new method added, a signature changed) to satisfy one particular consumer's needs, every other implementer must also update, and every client that merely holds a reference to the interface type may need to be recompiled/redeployed even though its actual usage didn't change. In languages with binary/ABI compatibility concerns, or in modular systems with independent deployment, this means unrelated modules end up coupled through a shared interface they only partially use — a much wider blast radius than the concrete class alone would suggest.

**Glossary:**
- **Fat interface** — an interface bundling more methods than any single client actually needs.
- **Role interface** — a small, client-specific interface named for the role it plays for that client.

**Mental model:**
Probes whether the candidate can articulate the difference between class-level cohesion (SRP) and interface-level client coupling (ISP) — many candidates can define both principles but blur the distinction when asked directly.

**TL;DR:**
SRP is about why a class changes; ISP is about not forcing a client to depend on interface methods it never uses — splitting fat interfaces into small, role-specific ones limits the blast radius of interface changes.

**References:**
- [SOLID Principles of Object-Oriented and Agile Design, Robert C. Martin](https://gist.github.com/OddExtension5/5590a2a8197a31aa3bf1d4ca3ee20f83)

---

### Q9. What is Conway's Law, and how does it show up in service/module boundaries? {#q9}

**Question:**
What is Conway's Law, and how does it show up in service/module boundaries?

**Good answer:**
Conway's Law states that organizations which design systems are constrained to produce designs that are copies of the communication structure of those organizations — each subsystem in the resulting design tends to correspond to a design group, and the interfaces between subsystems mirror the (often informally negotiated) communication paths between those groups. Where two teams don't talk to each other, the corresponding parts of the system tend not to have a clean, deliberate interface either — the seams in the org chart become the seams in the architecture, intentionally or not.

In service/module boundaries this shows up very literally: a company with three backend teams organized around "orders," "inventory," and "shipping" will very likely end up with three services or modules along those same lines — not necessarily because that's the best domain decomposition, but because that's how communication (and therefore negotiated interfaces) actually happened. Conway also notes the effect gets *more* pronounced as the organization gets larger and communication becomes more hierarchical/restricted.

**Follow-up question:**
What is the "Inverse Conway Maneuver," and why would a team deliberately restructure itself around a target architecture?

**Follow-up good answer:**
The Inverse Conway Maneuver is the practice of intentionally organizing teams to match the architecture you *want* to end up with, rather than accepting whatever architecture naturally falls out of your existing org structure. If you want loosely-coupled microservices with clean boundaries, you organize autonomous, cross-functional teams around those same boundaries (each owning one service end-to-end) — because Conway's Law predicts that whatever the team structure is, the system design will converge toward mirroring it anyway. It's cheaper to shape the org chart deliberately up front than to fight the natural pull of communication structure after the fact.

**Glossary:**
- **Conway's Law** — system design mirrors the communication structure of the organization that produced it.
- **Inverse Conway Maneuver** — deliberately restructuring teams to steer the system design toward a target architecture.

**Mental model:**
Checks whether the candidate connects architecture to organizational reality rather than treating architecture as a purely technical decision made in a vacuum — a very common blind spot for engineers who haven't yet worked across multiple teams.

**TL;DR:**
System boundaries tend to mirror team communication boundaries whether you plan it or not, so shaping team structure deliberately (the Inverse Conway Maneuver) is often more effective than trying to design an architecture that fights your org chart.

**References:**
- [How Do Committees Invent?, Melvin E. Conway](https://www.melconway.com/Home/Committees_Paper.html)

---

### Q10. How would you diagnose that a codebase has drifted from its intended architecture's dependency direction? {#q10}

**Question:**
How would you diagnose that a codebase has drifted from its intended architecture's dependency direction?

**Good answer:**
Start with automated dependency analysis rather than manual code reading at scale: tools that build a module/package dependency graph (e.g. language-specific tools like ArchUnit for Java, `import-linter` for Python, or general graph tools like `dependency-cruiser` for JS/TS) can enforce and report rule violations like "nothing in `domain` may import from `infrastructure`" as part of CI, turning architectural intent into an executable, continuously-checked assertion instead of a diagram nobody re-reads.

Beyond tooling, concrete smells to look for: business-logic classes importing framework or ORM types directly (e.g. a domain entity annotated with `@Entity` and JPA-specific types), a "clean" inner layer that can't actually be unit tested without spinning up a database or HTTP server, and circular dependencies between layers that are supposed to be strictly one-directional. If you can't write a fast, in-memory unit test for your core business logic, that's usually a direct symptom of a dependency-direction violation, not a testing-infrastructure problem.

**Follow-up question:**
Once you've found dependency-direction violations in a live codebase, what's a pragmatic strategy to fix them without a risky big-bang rewrite?

**Follow-up good answer:**
Introduce the missing abstraction at the boundary first, without moving code yet: define the interface the inner layer *should* depend on (e.g. `PaymentGateway`), have the existing concrete implementation implement it, and have the inner layer depend on the new interface instead of the concrete type — this is a small, low-risk, mechanically verifiable change (often supported by IDE "extract interface" refactoring). Then add the architecture-fitness-function test (e.g. ArchUnit rule) so the fix can't silently regress, and repeat boundary-by-boundary rather than attempting the whole layering in one pass. This "strangle the violation from the boundary inward" approach lets you ship the fix incrementally alongside normal feature work.

**Glossary:**
- **Architecture fitness function** — an automated, repeatable test that verifies an architectural characteristic (like dependency direction) holds.
- **Extract interface** — a refactoring that introduces an interface for an existing concrete class's public methods.

**Mental model:**
Distinguishes candidates who treat architecture as a one-time diagram from those who know it needs continuous, automated enforcement — and who have a concrete, low-risk remediation plan rather than "we'd need to rewrite it."

**TL;DR:**
Diagnose dependency-direction drift with automated fitness-function tooling (not just code review) and fix it incrementally by introducing the missing abstraction at each violated boundary, one at a time.

**References:**
- [The Clean Architecture, Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

### Q11. What's the difference between a design pattern and an architectural pattern? {#q11}

**Question:**
What's the difference between a design pattern and an architectural pattern?

**Good answer:**
A design pattern (in the Gang of Four sense — Strategy, Observer, Factory, Decorator, etc.) is a reusable solution to a recurring problem at the level of classes and objects within a single component — it affects how a handful of classes collaborate, and you can usually apply or remove one without touching the rest of the system. An architectural pattern (layered architecture, hexagonal architecture, microservices, event-driven architecture, CQRS) operates at the scale of the whole system or a major subsystem — it dictates how components, services, or layers are organized and allowed to communicate, and changing it is a project-level decision with wide blast radius, not a local refactor.

The two compose: you can implement a port in a hexagonal architecture using the Adapter design pattern, or implement a use case's extensibility using the Strategy pattern — the architectural pattern sets the large-scale skeleton, and design patterns fill in specific structural problems within it.

**Follow-up question:**
Why does this distinction matter when reviewing a proposed change to a system?

**Follow-up good answer:**
Because the appropriate level of scrutiny and reversibility differs enormously. Introducing a design pattern locally (swapping a conditional for a Strategy object) is a cheap, reversible, code-review-level decision — worst case you refactor it back in an afternoon. Introducing or changing an architectural pattern (moving from a monolith to microservices, adding CQRS to a bounded context) commits the team to new infrastructure, new operational complexity, and new failure modes that are expensive to reverse — it warrants the kind of scrutiny you'd give an ADR (Architecture Decision Record) and buy-in from more than one reviewer, not a routine PR approval.

**Glossary:**
- **Design pattern** — a reusable object/class-level solution to a recurring design problem (GoF catalog).
- **Architectural pattern** — a system- or subsystem-scale organizing structure governing component communication.
- **ADR (Architecture Decision Record)** — a short document capturing a significant, hard-to-reverse architectural decision and its rationale.

**Mental model:**
Tests whether a candidate treats "pattern" as one undifferentiated bucket of jargon, or understands that scale and reversibility should drive how much process and scrutiny a given change deserves.

**TL;DR:**
Design patterns solve local, class-level problems and are cheap to reverse; architectural patterns shape the whole system's structure and are expensive to reverse — the distinction should drive how much scrutiny a proposed change gets.

**References:**
- [Clean Code Episode 8: Foundations of the SOLID Principles, Robert C. Martin](https://cleancoders.com/episode/clean-code-episode-8)

---

### Q12. Explain the Open-Closed Principle with a concrete before/after example. {#q12}

**Question:**
Explain the Open-Closed Principle with a concrete before/after example.

**Good answer:**
The Open-Closed Principle says software entities should be open for extension but closed for modification — you should be able to add new behavior without editing existing, already-tested code. A common violation is a function with a growing `if/else` or `switch` on a type discriminator: `if (shape.type == "circle") { ... } else if (shape.type == "square") { ... }`. Every time a new shape is added, this function must be modified, risking regressions in the branches that already worked.

The fix is to define a `Shape` interface with an `area()` method, let each concrete shape (`Circle`, `Square`, ...) implement its own `area()`, and have the calling code just do `shape.area()` polymorphically. Adding a new shape means adding a new class that implements the interface — the calling code and every existing shape class are untouched, and closed to modification, while the system as a whole is open to extension.

**Follow-up question:**
Is it realistic — or even desirable — to design every part of a system to be Open-Closed from day one?

**Follow-up good answer:**
No — over-applying OCP prematurely (adding interfaces and abstraction layers for extension points that never actually get a second implementation) is a real cost: extra indirection, more files to navigate, and speculative generality that Martin himself warned against ("YAGNI" tension). The pragmatic approach is to write the straightforward `if/else` first, and only introduce the OCP-friendly abstraction once you have concrete evidence of variation — a second case that needs to be added, or a clear signal the axis of variation is real and recurring, not hypothetical. OCP is best applied where change is anticipated based on actual, observed churn, not preemptively everywhere.

**Glossary:**
- **Open for extension** — new behavior can be added.
- **Closed for modification** — existing, tested source code doesn't need to change to add that new behavior.
- **YAGNI** — "You Aren't Gonna Need It," a caution against speculative generality.

**Mental model:**
Checks whether the candidate can both explain OCP correctly *and* push back on over-applying it — senior candidates recognize that premature abstraction is its own anti-pattern, not just recite the principle uncritically.

**TL;DR:**
OCP replaces type-discriminator branching with polymorphism so new cases are added via new classes, not edits to existing code — but apply it only once real variation shows up, not speculatively.

**References:**
- [SOLID Principles of Object-Oriented and Agile Design, Robert C. Martin](https://gist.github.com/OddExtension5/5590a2a8197a31aa3bf1d4ca3ee20f83)

---

### Q13. What testability problem does classic layered architecture not solve that hexagonal/Clean Architecture does? {#q13}

**Question:**
What testability problem does classic layered architecture not solve that hexagonal/Clean Architecture does?

**Good answer:**
Classic layered architecture (presentation → business logic → data access, stacked strictly top-to-bottom) typically still allows — and in practice often ends up with — the business logic layer depending directly on concrete data-access classes below it, because "layered" only constrains which layers may call which, not the direction of the *compile-time dependency* between them. That means testing business logic in isolation still frequently requires a real (or heavily mocked) database layer wired up, because the business layer's code literally references data-access types.

Hexagonal/Clean Architecture fixes this by inverting that specific dependency: the business/use-case layer defines the interface (port) it needs, and the data-access layer implements it — so the dependency arrow at the source-code level points from data access toward business logic, not the other way around, even though the *runtime call* still flows from business logic to data access. That inversion is exactly what lets you substitute an in-memory fake for the real database in a unit test, with zero database involved, which plain layered architecture doesn't guarantee by itself.

**Follow-up question:**
If layered architecture doesn't enforce this by default, can a team retrofit the same guarantee onto an existing layered codebase without a full rewrite?

**Follow-up good answer:**
Yes — this is essentially "hexagonal architecture applied within a layered codebase." You don't need to change the physical folder structure or introduce new deployment units; you introduce interfaces at the specific seams that need to be testable (repository interfaces owned by the business layer, implemented by the data-access layer), and enforce with a fitness-function test (e.g. ArchUnit) that the business package never imports concrete data-access types directly, only the interfaces. It's an incremental, boundary-by-boundary change, not an architectural rewrite — the "hexagonal" property is really about dependency direction at specific seams, which can be introduced piecemeal.

**Glossary:**
- **Layered architecture** — organizing code into horizontal layers (presentation/business/data), typically without a strict rule on dependency *direction* beyond call order.
- **Fitness function** — an automated test enforcing an architectural property continuously.

**Mental model:**
Tests whether the candidate understands that "layered" and "hexagonal" aren't mutually exclusive alternatives on a list — hexagonal is really a *stricter dependency-direction rule* that can be layered on top of an existing layered structure.

**TL;DR:**
Layered architecture constrains call order but not dependency direction, so business logic can still depend on concrete data-access classes; hexagonal architecture inverts that specific dependency so business logic can be unit-tested with zero database involved.

**References:**
- [Hexagonal Architecture, Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)

---

### Q14. What consistency trade-off do you take on when combining CQRS with Event Sourcing? {#q14}

**Question:**
What consistency trade-off do you take on when combining CQRS with Event Sourcing?

**Good answer:**
When CQRS's read model is built by projecting events from an event-sourced write model, the read side is almost always eventually consistent with the write side: a command is validated and appended as an event to the write store, and only *afterward*, asynchronously, does a projector consume that event and update the read model a client actually queries. Between those two steps there's a window — often milliseconds, but unbounded in the worst case (a stalled projector, a backlog) — where a client that just wrote data will see stale results if it queries the read model immediately.

This differs sharply from a single-model CRUD system, where a write is immediately visible to the very next read because there's only one model and one consistency boundary. Adopting CQRS+ES means you've deliberately traded that immediate-consistency guarantee for the scalability and modeling benefits (independently scalable, differently-shaped read/write stores) — and you now need an explicit strategy (read-your-writes routing, optimistic client-side updates, or simply UX that tolerates a short lag) for cases where a user expects to see their own write reflected right away.

**Follow-up question:**
Name one concrete technique to give users a "read your own writes" experience despite the read model lagging.

**Follow-up good answer:**
Route reads for the entity a user just wrote to the write-side store (or a synchronously-updated cache) for a short window immediately after the write, falling back to the normal (lagging) read model afterward — sometimes called "read-your-writes consistency" or a sticky/session-based read override. An alternative is optimistic UI: apply the write's expected effect to the client's local view immediately, without waiting for the read model, and reconcile silently if the eventual projected state differs. Both avoid making every read pay the write-store's cost, while still giving the *specific* user who just wrote data an immediately-consistent view of their own change.

**Glossary:**
- **Eventual consistency** — a guarantee that all replicas/views will converge to the same state, but not necessarily immediately.
- **Projector** — a component that consumes events and updates a read-optimized view/projection.
- **Read-your-writes consistency** — a guarantee that a client always sees the effects of its own prior writes.

**Mental model:**
Probes whether the candidate can name the *specific* operational cost (consistency lag) rather than a vague "it's more complex," and whether they have a concrete mitigation technique rather than just acknowledging the problem exists.

**TL;DR:**
CQRS+Event Sourcing typically makes the read side eventually consistent with the write side, so you need an explicit read-your-writes or optimistic-UI strategy for users who expect to see their own change immediately.

**References:**
- [CQRS, Martin Fowler](https://martinfowler.com/bliki/CQRS.html)

---

### Q15. When is applying Clean/Hexagonal Architecture over-engineering for a simple CRUD app? {#q15}

**Question:**
When is applying Clean/Hexagonal Architecture over-engineering for a simple CRUD app?

**Good answer:**
When the application genuinely is thin CRUD — a handful of screens/endpoints that map close to 1:1 onto database tables, with little-to-no business logic beyond validation and simple lookups, a small number of collaborators, and no realistic expectation of swapping the framework or database — the full ceremony of entities/use-cases/interface-adapters layers, repository interfaces for every table, and DTOs at every boundary adds files, indirection, and cognitive overhead without a corresponding payoff. You end up writing an interface, an implementation, and a mapper for logic that will never have a second implementation and never needs to be tested independently of the database, because there's no meaningful business logic to isolate in the first place.

The signal to watch for is whether the "business logic" layer, if you introduced one, would contain anything beyond pass-through calls to the data layer — if a `UseCase` class's body is just `return repository.find(id)`, the abstraction isn't earning its cost. Clean/Hexagonal Architecture pays off in proportion to how much real, testable-in-isolation business logic exists and how likely infrastructure (database, external services) is to actually change.

**Follow-up question:**
What's a middle-ground approach for a codebase that starts as simple CRUD but might grow real business logic later?

**Follow-up good answer:**
Keep the codebase simple and direct initially (controllers calling a thin data-access layer, no speculative ports/adapters), but keep business logic that *does* exist encapsulated in the domain objects themselves (avoiding an anemic model) rather than smeared across controllers — that's a cheap, low-ceremony habit that pays off regardless of how much the app grows. Introduce the port/adapter abstraction at a specific boundary only once you have concrete evidence it's needed (a second implementation, a genuine need to unit-test business rules without a database) rather than architecting for hypothetical future flexibility across the whole app up front — this mirrors the same YAGNI reasoning that applies to premature Open-Closed abstraction.

**Glossary:**
- **CRUD** — Create, Read, Update, Delete; an application whose logic is mostly direct data manipulation.
- **DTO (Data Transfer Object)** — a plain data structure used to move data across a boundary without behavior.

**Mental model:**
Checks whether the candidate can push back on architectural dogma with judgment about cost/benefit — a strong signal of seniority, since junior engineers often apply patterns because they're "best practice" without weighing whether the specific context earns the cost.

**TL;DR:**
Full Clean/Hexagonal ceremony is over-engineering when there's no real business logic to isolate and no realistic infrastructure-swap scenario — introduce the abstraction at a specific boundary only once there's concrete evidence it's earning its cost.

**References:**
- [The Clean Architecture, Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

### Q16. How do plugin architectures apply the Dependency Inversion Principle at a whole-system level? {#q16}

**Question:**
How do plugin architectures apply the Dependency Inversion Principle at a whole-system level?

**Good answer:**
A plugin architecture is DIP scaled up from a single class boundary to the whole application: the core system defines a stable plugin interface/contract (e.g. `Plugin { activate(context) }`), and plugins — potentially built and deployed independently, by different teams or third parties — depend on and implement that interface. The core never references any concrete plugin type; it only knows the abstraction, discovers implementations at runtime (via a registry, service-loader mechanism, or directory scan), and invokes them polymorphically. This is exactly DIP's "both high-level and low-level modules depend on abstractions" applied at the granularity of entire deployable units instead of individual classes.

The payoff mirrors DIP's payoff at smaller scale: the core can ship and version independently of any given plugin, new plugins can be added without modifying or redeploying the core, and third parties can extend the system without access to its source code — they only need the published interface contract.

**Follow-up question:**
What's the biggest architectural risk in a plugin system that pure class-level DIP doesn't have to worry about?

**Follow-up good answer:**
Interface/contract versioning across independently-deployed, independently-versioned units. At the class level, a DIP violation gets caught immediately at compile time in the same codebase — but a plugin built against interface version 1.0 that's loaded by a core now running version 2.0 fails at runtime, potentially in production, if the interface changed in a breaking way. Plugin architectures need an explicit compatibility strategy (semantic versioning of the plugin API, capability negotiation, or strict backward-compatibility discipline on the interface) that a same-codebase DIP boundary gets almost for free from the compiler and a shared test suite.

**Glossary:**
- **Plugin architecture** — a system design where the core defines stable extension points and independently-built plugins implement them.
- **Service loader / registry** — a runtime mechanism for discovering and loading implementations of an interface without the core knowing their concrete types at compile time.

**Mental model:**
Tests whether the candidate can generalize a principle they learned at class-level scope (DIP) to system-level architecture, which is a strong signal of transferable understanding rather than memorized definitions.

**TL;DR:**
Plugin architectures are DIP applied at the scale of whole deployable units — the core depends only on a stable interface, never on concrete plugins — but that scale introduces a real versioning/compatibility risk that class-level DIP doesn't face.

**References:**
- [SOLID Principles of Object-Oriented and Agile Design, Robert C. Martin](https://gist.github.com/OddExtension5/5590a2a8197a31aa3bf1d4ca3ee20f83)

---

### Q17. Why is a domain-layer class importing a framework or ORM-specific type considered a layering violation? {#q17}

**Question:**
Why is a domain-layer class importing a framework or ORM-specific type considered a layering violation?

**Good answer:**
In a layered/Clean/Hexagonal design, the domain layer is meant to be the most stable part of the system — pure business rules with no knowledge of *how* they're persisted, transported, or displayed. If a domain entity is annotated with ORM-specific annotations (e.g. `@Entity`, `@Column`) or extends an ORM base class, the domain layer now has a compile-time dependency on a specific persistence technology's types and annotations. That inverts the intended dependency direction: instead of the ORM adapting to the domain's shape, the domain is now shaped by (and coupled to) the ORM's requirements — a no-argument constructor for reflection, mutable fields for lazy loading, specific collection types the ORM understands.

The practical cost shows up later: migrating databases or ORMs means touching domain classes (risking business-rule regressions in code that has nothing to do with persistence), and unit-testing a domain entity can pull in ORM-related concerns (lazy-loading proxies, session context) that have nothing to do with the business rule actually being tested.

**Follow-up question:**
What's the standard remediation pattern when a domain entity has become coupled to an ORM this way?

**Follow-up good answer:**
Introduce a separate persistence model (sometimes called a "persistence entity" or "record") that carries the ORM annotations and mapping concerns, and keep the domain entity as a plain object with no framework dependencies; a mapper (in the interface-adapters/infrastructure layer, not the domain layer) converts between the two at the repository boundary. This costs some boilerplate (two representations of similar data, a mapping step) but restores the Dependency Rule — the domain layer no longer needs to know the ORM exists, changing persistence technology means rewriting the mapper and persistence model, not the business rules.

**Glossary:**
- **ORM (Object-Relational Mapper)** — a library that maps between database rows and in-memory objects.
- **Persistence model** — a data representation shaped by storage/mapping needs, kept separate from the domain model.
- **Mapper** — code that converts between two representations of the same conceptual data at a boundary.

**Mental model:**
Checks whether the candidate can spot a very common, easy-to-miss real-world violation (ORM annotations on domain classes) and articulate the concrete future cost, not just recite "layers shouldn't depend on frameworks" as an abstract rule.

**TL;DR:**
ORM annotations on a domain entity invert the intended dependency direction, coupling business rules to a specific persistence technology; the fix is a separate persistence model plus a mapper at the boundary, keeping the domain layer framework-free.

**References:**
- [The Clean Architecture, Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

### Q18. How does an event-sourced system avoid re-triggering side effects (like a payment charge) when replaying events? {#q18}

**Question:**
How does an event-sourced system avoid re-triggering side effects (like a payment charge) when replaying events?

**Good answer:**
The key discipline is separating the *decision to act* (which happens once, when a command is originally handled) from *applying an event's effect to in-memory state* (which happens every time state is reconstructed, including during replay). The call to an external system like a payment gateway belongs in the command-handling step — it happens exactly once, produces a result, and that result is what gets captured in the resulting event (e.g. `PaymentCharged { amount, transactionId }`). Replaying that event later only means updating in-memory state to reflect "a charge of this amount with this transaction ID happened" — it must never mean calling the payment gateway again.

Practically, this means the code path used for replay (loading an aggregate from its event history) must be structurally incapable of calling out to external systems — often enforced by having the "apply event" methods be pure, synchronous, side-effect-free functions that only mutate in-memory fields, while all I/O happens in a separate command-handler code path that produces events but is never invoked during replay.

**Follow-up question:**
What happens if a bug is later discovered in the event-application logic, and you need to "fix" how a specific historical event type is interpreted? Why is this harder than fixing a normal bug?

**Follow-up good answer:**
Because the events themselves are immutable and already persisted — you can't retroactively edit history the way you'd fix a row in a mutable CRUD table. Changing how an event type is *applied* going forward is straightforward (fix the apply-function code), but if past aggregates need to reflect the corrected interpretation, you either accept that already-materialized state (e.g. old snapshots) is now inconsistent with a fresh replay under the new logic, or you run a one-time migration that replays all affected aggregates under the corrected logic and re-snapshots them. Some teams handle this by never truly "fixing" old event semantics in place — instead emitting a new compensating event or a new event schema version — precisely to avoid silently changing the meaning of history.

**Glossary:**
- **Command handler** — the code path that validates a command, performs any necessary I/O, and produces resulting event(s); runs once, at the time of the original action.
- **Apply function** — the pure function that updates in-memory state given an event; runs both live and during replay.
- **Compensating event** — a new event that corrects the effect of a prior one, rather than editing history.

**Mental model:**
Tests whether the candidate understands the architectural discipline (strict separation of decision-making/I-O from state-application) required to make event sourcing safe, not just the conceptual "replay events to get state" pitch.

**TL;DR:**
Side-effecting I/O must live only in the once-per-action command-handling path, never in the apply-function path used during replay, so reconstructing state never re-executes real-world effects like a payment charge.

**References:**
- [Event Sourcing, Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)

---

### Q19. How do read-model consistency guarantees differ between plain CQRS (no event sourcing) and CQRS built on top of Event Sourcing? {#q19}

**Question:**
How do read-model consistency guarantees differ between plain CQRS (no event sourcing) and CQRS built on top of Event Sourcing?

**Good answer:**
Plain CQRS (separate read/write models, but the write model just stores current state directly, no event log) *can* still be made strongly consistent if the write transaction also synchronously updates the read model in the same transaction or request — the split is purely about having two differently-shaped models, not necessarily about asynchrony. It's an architectural choice independent of consistency: you could have CQRS with immediate read-model updates.

CQRS layered on Event Sourcing almost always implies asynchronous projection: the write side's job is to append an event, full stop, and a separate projector process consumes that event (often via a message broker or event log like Kafka) to update the read model. Coupling the read-model update synchronously into the same transaction as the event append reintroduces the coupling ES is often used to avoid (multiple read models, projectors that can be rebuilt independently, projectors owned by different teams) — so in practice, ES-backed CQRS systems lean heavily toward eventual consistency as the default, while plain CQRS gives you the option (at the cost of some coupling) to stay strongly consistent if you choose to.

**Follow-up question:**
Why might a team deliberately choose plain CQRS with synchronous read-model updates instead of the "purer" event-sourced, eventually-consistent version?

**Follow-up good answer:**
When the product genuinely cannot tolerate any read-after-write lag (e.g. a user must see their own balance update instantly, with no UX workaround acceptable) and the team doesn't need the other benefits event sourcing provides — full audit history, time-travel debugging, rebuilding arbitrary new projections from history — paying for asynchronous eventual consistency buys nothing extra for that use case while adding real complexity (message infrastructure, projector lag monitoring, handling out-of-order or duplicate event delivery). Choosing the simpler, strongly-consistent version is the right trade-off when the extra capabilities of full event sourcing aren't actually needed by the business.

**Glossary:**
- **Strongly consistent** — a read immediately reflects the most recent write.
- **Projector** — a process/component that builds or updates a read model by consuming events.

**Mental model:**
Probes whether the candidate understands that CQRS and Event Sourcing are separable decisions — often bundled together in blog posts and talks — and can reason about consistency trade-offs independently of which combination is chosen.

**TL;DR:**
CQRS alone doesn't force eventual consistency — that's a separate choice — but CQRS built on Event Sourcing almost always leans eventually consistent because coupling the projector synchronously into the write transaction undoes much of what event sourcing is for.

**References:**
- [CQRS, Martin Fowler](https://martinfowler.com/bliki/CQRS.html)

---

### Q20. How would you evolve an event's schema in an event-sourced system without breaking replay of old, already-persisted events? {#q20}

**Question:**
How would you evolve an event's schema in an event-sourced system without breaking replay of old, already-persisted events?

**Good answer:**
Because events are immutable and permanently persisted, you can never retroactively change what was actually stored — schema evolution has to happen in the *code that reads* events, not the events themselves. The standard approach is upcasting: when an old event version is deserialized during replay, an explicit transformation step ("upcaster") converts it into the shape the current apply-logic expects, before it's applied — e.g. if `OrderPlaced` gained a new required `currency` field, an upcaster for the old version fills in a sensible default (like `"USD"`) for events that predate the field's existence, so current code never has to special-case "is this an old-format event."

This requires versioning events explicitly (an event type/version field) and keeping upcasters around indefinitely for every schema change ever made, since old events never disappear — which is a real, ongoing maintenance cost that grows with the system's age, and is one of the concrete reasons teams weigh event sourcing's audit/history benefits against this long-term schema-evolution tax.

**Follow-up question:**
What's a simpler alternative to indefinite upcaster maintenance, and what do you give up by choosing it?

**Follow-up good answer:**
Periodically "collapse" history by snapshotting all aggregates under the current schema and, in systems where full historical replay of the original raw events isn't a hard requirement, discarding (or archiving out of the live store) events older than the snapshot horizon — future replays only need to go back to the nearest snapshot, so old-format events stop needing to be understood by current code at all. The trade-off is that you lose the ability to fully replay history from the very beginning of time with the original raw event stream if that's ever needed (e.g. for a new report type that needs data older than the earliest surviving snapshot, or for regulatory audit requirements demanding the original records) — so this only works when "current state plus recent history" is enough, not "the complete original historical record forever."

**Glossary:**
- **Upcaster** — a transformation that converts an old event schema version into the shape current code expects.
- **Event versioning** — explicitly tagging events with a schema version so multiple historical shapes can coexist and be handled.

**Mental model:**
Tests whether the candidate has thought through the long-term maintenance reality of event sourcing (schema evolution is forever, not a one-time migration) rather than only understanding the initial "replay events to get state" pitch.

**TL;DR:**
Old events can never be edited, so schema evolution happens via upcasters in the read path (or by trading away full historical replay via snapshot-and-archive) — both are real, ongoing costs of committing to event sourcing.

**References:**
- [Event Sourcing, Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=architecture&tags=architectural-patterns-and-design-principles&autostart=1" | relative_url }})
