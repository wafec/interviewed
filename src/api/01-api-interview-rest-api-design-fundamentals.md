---
layout: default
title: "API Interview: REST API Design Fundamentals"
---

# API Interview — REST API Design Fundamentals

Twenty questions covering the foundations of designing and reasoning about
HTTP/REST APIs: the constraints that make an API "RESTful," correct HTTP
semantics, caching, security, versioning, pagination, rate limiting, and how
these decisions play out under real production load — plus the trade-offs
against gRPC and GraphQL.

### Q1. What is REST, and what makes an API "RESTful" rather than just "an API over HTTP"?

**Question:**
What is REST, and what makes an API "RESTful" rather than just "an API over HTTP"?

**Good answer:**
REST (Representational State Transfer) is an architectural style defined by Roy Fielding in his 2000 doctoral dissertation, describing constraints for building distributed hypermedia systems. An API is "RESTful" when it follows constraints like: client-server separation, statelessness (no client session state stored on the server between requests), cacheability, a uniform interface (resources identified by URIs, manipulated through representations, using standard methods), a layered system, and — the constraint most real-world "REST" APIs skip — hypermedia as the engine of application state (HATEOAS), where clients navigate the API via links returned in responses rather than hardcoded URI templates. Most APIs that call themselves "REST" are really "HTTP + JSON CRUD APIs" — they satisfy statelessness and the uniform interface loosely, but skip HATEOAS entirely. That's fine in practice, but it's worth knowing you're describing a pragmatic subset, not full REST.

**Follow-up question:**
If most production "REST" APIs don't implement HATEOAS, does that mean they're not really REST — and does it matter?

**Follow-up good answer:**
Strictly, yes — per Fielding, without hypermedia controls driving state transitions, an API is not REST by his definition; it's more accurately "HTTP RPC" or a pragmatic JSON-over-HTTP API. Whether that matters depends on the actual payoff: full HATEOAS buys you client/server decoupling from URI structure, so the server can restructure or move endpoints without breaking clients and without a version bump, since clients never hardcode URIs — they follow links. That's valuable for long-lived, public APIs with many independent third-party clients you don't control the release cadence of. For most internal or product APIs with a small number of clients deployed in lockstep, that evolvability isn't worth the added complexity (media-type design, link-following client tooling), so most teams reasonably choose the pragmatic non-HATEOAS subset and document endpoints instead. It's largely a terminology-purity issue, not a correctness bug — but worth recognizing as an intentional trade-off of evolvability for simplicity, not "doing REST wrong" by accident.

**Glossary:**
- **Representation** — a serialized form (e.g. JSON, XML) of a resource's current state.
- **HATEOAS** — Hypermedia As The Engine Of Application State; clients discover available actions from links in responses instead of out-of-band documentation.
- **Uniform interface** — the REST constraint that resources are manipulated through a small, standard set of operations (HTTP methods) and representations.

**Mental model:**
This question tests whether the candidate actually knows REST as an architectural style with constraints, versus having just memorized "REST = HTTP verbs + JSON." It also probes whether they can distinguish textbook purity from pragmatic industry practice without being dogmatic either way.

**TL;DR:**
Full REST requires HATEOAS-driven hypermedia, not just HTTP verbs + JSON — most production 'REST' APIs are a pragmatic subset that skips it.

**References:**
- [Fielding's dissertation, Chapter 5: Representational State Transfer (REST)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)

---

### Q2. Why is statelessness a core REST constraint, and what breaks if you violate it?

**Question:**
Why is statelessness a core REST constraint, and what breaks if you violate it?

**Good answer:**
Statelessness means each request from a client to server must contain all the information needed to understand and process it — the server holds no client session context between requests. This enables horizontal scalability (any server instance can handle any request, so you can load-balance freely and add/remove instances without session affinity), improves visibility for intermediaries (a proxy/cache can understand a request in isolation), and improves reliability (a server crash doesn't lose in-flight session state). Violating it — e.g. storing "logged in user" in server-side memory/session tied to a specific instance — forces sticky sessions, breaks horizontal autoscaling cleanly, and complicates failover. The trade-off is that the client (or a shared external store like a token or a database-backed session) now carries more responsibility.

**Code example:**
```http
# Stateless: every request carries its own auth context
GET /orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOi...
```

**Follow-up question:**
If you need a multi-step workflow (e.g. a checkout wizard), how do you preserve statelessness while still tracking progress across requests?

**Follow-up good answer:**
Move the workflow state out of any single server's memory and into something either the client carries or a shared external store holds, so any server instance can handle any request in the sequence. Two common approaches: (1) client-held state — return the accumulated draft/workflow data to the client (e.g. in a signed/encrypted token or the response body) and have the client send it back with each subsequent request, so the server reconstructs full context purely from what arrived; (2) server-side state persisted in a shared external store (a database row or Redis entry keyed by a workflow/session ID) that every instance can read and write — no server instance holds anything in local memory, it just looks up shared state by ID per request. Both preserve REST's statelessness constraint, because no request depends on a *specific* server instance having seen prior requests — the difference is only where the durable state physically lives.

**Glossary:**
- **Sticky session** — a load balancer routing all requests from a client to the same backend instance to preserve server-held state, a common workaround when statelessness is violated.
- **Session affinity** — the same concept as sticky sessions, viewed from the load balancer's config.

**Mental model:**
Tests whether the candidate connects an architectural constraint to its concrete operational payoff (scalability, failover) rather than reciting it as a rule with no reason.

**TL;DR:**
Statelessness means every request is self-contained, enabling free load-balancing and clean failover — violating it forces sticky sessions.

**References:**
- [Fielding's dissertation, §5.1.3: Stateless](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)

---

### Q3. What's the difference between a "safe" and an "idempotent" HTTP method, and why does it matter for retries and caches?

**Question:**
What's the difference between a "safe" and an "idempotent" HTTP method, and why does it matter for retries and caches?

**Good answer:**
A **safe** method is one that doesn't change server state — it's read-only (GET, HEAD, OPTIONS). An **idempotent** method is one where making the same request N times has the same effect as making it once (GET, HEAD, PUT, DELETE — all safe methods are automatically idempotent, but PUT/DELETE are idempotent without being safe, since they do change state). POST and PATCH are, by default, neither safe nor idempotent — calling POST /orders twice creates two orders. This matters enormously for retries: an HTTP client (or an intermediary proxy) can safely auto-retry an idempotent request after a timeout/connection failure without risking a duplicate side effect, but it cannot safely auto-retry a POST — that requires an idempotency key mechanism instead. It also matters for caching: only safe (and specifically cacheable, like GET) requests are candidates for HTTP caching.

**Follow-up question:**
Your DELETE endpoint returns 404 on the second call because the resource is already gone — does that break idempotency?

**Follow-up good answer:**
No. RFC 9110 §9.2.2 defines idempotency in terms of the *intended effect on server state* of repeated identical requests, not in terms of identical response codes or bodies. After the first DELETE, the resource is gone; after the second DELETE, the resource is still gone — server state is unchanged between the two outcomes, so the method is idempotent even though the HTTP status differs (e.g. 204 the first time, 404 the second). This trips people up because "idempotent" sounds like it should mean "returns the same response every time," but the spec only guarantees the same end state, not the same response. Returning 404 on a repeat DELETE is correct and common.

**Glossary:**
- **Safe method** — doesn't alter server-observable state (GET, HEAD, OPTIONS).
- **Idempotent method** — repeated identical requests produce the same end state as a single request.

**Mental model:**
Distinguishes candidates who know HTTP semantics precisely enough to reason about retry safety and correctness under network failure, not just "GET reads, POST writes."

**TL;DR:**
Safe = read-only; idempotent = repeating it has the same end state — idempotent requests are safe to auto-retry, POST/PATCH by default are neither.

**References:**
- [RFC 9110 §9.2 Common Method Properties (Safe/Idempotent Methods)](https://www.rfc-editor.org/rfc/rfc9110)

---

### Q4. Walk through PUT vs PATCH vs POST for updating a resource — when would you use each?

**Question:**
Walk through PUT vs PATCH vs POST for updating a resource — when would you use each?

**Good answer:**
**PUT** replaces the entire resource with the representation sent in the request body — the client sends the full, final state, and any field omitted is treated as cleared. It's idempotent. **PATCH** applies a *partial* modification — the body is a set of instructions describing changes (a JSON Merge Patch, JSON Patch, or custom diff format), not a full replacement; it's not idempotent by default, though many implementations design it to be. **POST** is used when the operation doesn't map cleanly to "replace" or "modify" a known resource — e.g. creating a new resource under a collection where the server assigns the ID, or triggering an action/side effect. Practical rule of thumb: if the client can send the complete new state and you're fine discarding unspecified fields, use PUT; if the client should only send the fields that changed, use PATCH; if you're creating something new or running a non-idempotent operation, use POST.

**Code example:**
```http
PUT /users/42 HTTP/1.1
Content-Type: application/json

{"name": "Ada", "email": "ada@example.com", "role": "admin"}

PATCH /users/42 HTTP/1.1
Content-Type: application/merge-patch+json

{"email": "ada@example.com"}
```

**Follow-up question:**
A client PATCHes a resource with a stale copy of a field, unintentionally overwriting a concurrent update from another client — how do you defend against that?

**Follow-up good answer:**
Use optimistic concurrency control via conditional requests. The server returns an `ETag` representing the resource's current version on GET; the client must send that value back in an `If-Match` header on the PATCH/PUT. Per RFC 9110's conditional-request semantics, if the resource's current ETag no longer matches (meaning someone else changed it since the client last read it), the server rejects the write with `412 Precondition Failed` instead of applying it. This forces the client to re-fetch the latest version, reconcile, and retry rather than silently overwriting a field with stale data — the classic "lost update" problem — without needing full database transactions across the request boundary.

**Glossary:**
- **JSON Merge Patch (RFC 7396)** — a common PATCH body format where the given object is recursively merged into the resource.
- **JSON Patch (RFC 6902)** — an alternative PATCH format expressing changes as an explicit list of add/remove/replace operations.

**Mental model:**
Checks whether the candidate understands PUT/PATCH semantics precisely enough to avoid the extremely common bug of using PUT with a partial body (silently nulling out unspecified fields).

**TL;DR:**
PUT replaces the whole resource, PATCH applies a partial change, POST is for creation or non-idempotent actions — mixing up PUT with a partial body silently nulls fields.

**References:**
- [RFC 5789 — PATCH Method for HTTP](https://www.rfc-editor.org/rfc/rfc5789)
- [RFC 9110 §9.3.4 PUT](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9110 §13.1.1 If-Match (optimistic concurrency via ETag)](https://www.rfc-editor.org/rfc/rfc9110#section-13.1.1)

---

### Q5. When do you return 400 vs 422, and 401 vs 403?

**Question:**
When do you return 400 vs 422, and 401 vs 403?

**Good answer:**
**400 Bad Request** is a generic client error — the request is malformed at the protocol/syntax level (invalid JSON, missing required field, wrong type). **422 Unprocessable Entity** is for a request that's syntactically well-formed but semantically invalid — it parses fine, but violates a business rule (e.g. `end_date` before `start_date`, an email that's not actually deliverable-format valid per your rules). **401 Unauthorized** means the request lacks valid authentication credentials at all, or they're invalid/expired — the server doesn't know who you are (despite the confusing name, it's really "unauthenticated"). **403 Forbidden** means the server *does* know who you are, but you don't have permission to perform this action on this resource. Getting this right matters because clients (and monitoring/alerting) branch on these codes — a client should retry-with-fresh-token on 401 but not on 403.

**Follow-up question:**
Should a 403 response ever reveal that the resource exists (vs returning 404 to avoid leaking existence to unauthorized users)?

**Follow-up good answer:**
It depends on the sensitivity of what "existence" itself reveals. For most authorization failures, 403 is fine and more honest to the client — the resource exists, you're just not allowed to see it, which helps legitimate clients debug their own permissions. But when the *existence* of the resource is itself sensitive information (e.g. a private repository's name, another user's private document ID, an admin-only record), returning 403 leaks that something is there at that URI even though the caller can't access it — an attacker can enumerate valid IDs just by distinguishing 403 from 404 responses. In that case, the safer pattern (used by GitHub for private repos, for example) is to return 404 for both "doesn't exist" and "exists but you can't see it," so an unauthorized caller can't distinguish the two. The trade-off is debuggability: legitimate users lose the clearer "you don't have permission" signal.

**Glossary:**
- **Authentication** — verifying who the caller is.
- **Authorization** — verifying what the (known) caller is allowed to do.

**Mental model:**
Tests precision with the client-error status code space, and whether the candidate treats status codes as meaningful contract rather than "200 for success, 500 for anything else, whatever else feels right."

**TL;DR:**
400 is malformed syntax, 422 is well-formed but business-invalid; 401 means unauthenticated, 403 means authenticated but not permitted.

**References:**
- [RFC 9110 §15.5 Client Error 4xx](https://www.rfc-editor.org/rfc/rfc9110)
- [GitHub REST API docs — Troubleshooting: 404 returned instead of 403 for private/unauthorized resources](https://docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api)

---

### Q6. A client's POST to create a payment times out — they don't know if it succeeded. How do you make this safe to retry?

**Question:**
A client's POST to create a payment times out — they don't know if it succeeded. How do you make this safe to retry?

**Good answer:**
POST isn't idempotent by default, so blindly retrying risks double-charging. The standard solution is an **idempotency key**: the client generates a unique key (typically a UUID) per logical operation and sends it in a header (conventionally `Idempotency-Key`). The server stores the result of the first request keyed by that value; if the same key arrives again — with the same payload — the server returns the original cached response instead of re-executing the operation. If the key is reused with a *different* payload, the server should reject it as a conflict. This pattern is how Stripe and most payment APIs make POST-based operations retry-safe, and it's now being formalized as an IETF draft standard header.

**Code example:**
```http
POST /payments HTTP/1.1
Idempotency-Key: 8f14e45f-ceea-467f-b0c4-13b3f77f34b3
Content-Type: application/json

{"amount": 5000, "currency": "usd"}
```

**Follow-up question:**
How long should the server retain an idempotency key's stored result, and what happens if two requests with the same key arrive concurrently?

**Follow-up good answer:**
Stripe retains idempotency keys for at least 24 hours before they're eligible for pruning; if a key is reused after that window, it's treated as a brand-new operation. That window is a trade-off between storage cost and how long a client's retry logic might plausibly wait before giving up — 24 hours comfortably covers realistic retry/backoff scenarios (including a client retrying after being offline overnight) without keeping unbounded state forever. For concurrent requests with the same key, the server needs a lock or an atomic "claim" on the key: the first request to arrive marks the key as in-progress and executes the operation; if a second request with the same key arrives while the first is still executing, the server should not execute it a second time — Stripe's behavior here is to treat the concurrent duplicate as a conflict rather than serving a cached result that doesn't exist yet, so the client should retry rather than assume success or failure.

**Glossary:**
- **Idempotency key** — a client-generated unique token identifying a logical operation so retries don't re-execute it.

**Mental model:**
This is a real production-failure-mode question: does the candidate know retries are unsafe by default for POST, and do they know the actual industry-standard mechanism (vs. hand-waving "just check if it already exists")?

**TL;DR:**
An idempotency key lets the server cache and replay the first response to a retried POST, making non-idempotent operations like payments safe to retry.

**References:**
- [IETF draft-ietf-httpapi-idempotency-key-header-07: The Idempotency-Key HTTP Header Field](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt)
- [Stripe API Reference: Idempotent requests (24h retention, concurrent-request handling)](https://docs.stripe.com/api/idempotent_requests)

---

### Q7. What are the main API versioning strategies, and what are the trade-offs?

**Question:**
What are the main API versioning strategies, and what are the trade-offs?

**Good answer:**
Common strategies: **URI versioning** (`/v1/orders`) — simplest, highly visible, but "leaks" version into every resource URI and encourages big-bang version bumps; **header versioning** (a custom header or `Accept` media-type versioning like `Accept: application/vnd.example.v2+json`) — keeps URIs stable/cacheable and is more RESTful (content negotiation), but is less discoverable/harder to test with a browser; **query parameter versioning** (`?version=2`) — easy to add, but caches may not vary correctly on it unless configured. A complementary, often-preferred approach for internal/evolving APIs is **additive, backward-compatible changes only** (add optional fields, never remove/rename, deprecate with sunset headers) so you rarely need a hard version bump at all. The trade-off is always: visibility/simplicity (URI) vs. purity/cacheability (headers) vs. minimizing versioning altogether (strict backward compatibility discipline).

**Follow-up question:**
How do you communicate and enforce a deprecation timeline for `v1` once `v2` ships, without breaking clients who haven't migrated?

**Follow-up good answer:**
Deprecation typically has two stages: first, mark `v1` as no longer recommended while it's still fully operational (communicate this out-of-band — changelog, email, dashboard banners — since there's no dedicated "not recommended" header); second, once an actual retirement date is set, add the `Sunset` header (RFC 8594) to every `v1` response with the exact date/time it will stop working, ideally alongside a `Link` header pointing to migration documentation. In the interim, track which clients/API keys are still calling `v1` so you can proactively reach out to the heaviest remaining users before flipping it off — don't rely on the header alone. After the sunset date passes, `v1` should return `410 Gone` (not a silent 404) with a body pointing to `v2`, so clients get an unambiguous, actionable failure instead of a confusing generic error.

**Glossary:**
- **Content negotiation** — client and server agreeing on representation format/version via `Accept`/`Content-Type` headers rather than the URI.
- **Sunset header** — an HTTP header (`Sunset`, per RFC 8594) signaling when a resource/endpoint will stop being available.

**Mental model:**
Tests whether the candidate has an opinion grounded in trade-offs (not just "I use URI versioning because that's what I've seen") and understands the operational cost of breaking changes.

**TL;DR:**
URI versioning is simple but leaks into every path, header/content-negotiation versioning is purer but less discoverable — additive, backward-compatible changes avoid needing a bump at all.

**References:**
- [RFC 8594 — The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)

---

### Q8. Offset-based vs. cursor-based pagination — how do they work, and why does it matter at scale?

**Question:**
Offset-based vs. cursor-based pagination — how do they work, and why does it matter at scale?

**Good answer:**
**Offset-based pagination** (`?page=3&per_page=50`, or `LIMIT/OFFSET` under the hood) is simple and lets you jump to an arbitrary page, but has two real problems at scale: performance (the database still has to scan and discard all the skipped rows, so deep pages get slower — `OFFSET 1000000` is expensive) and correctness (if rows are inserted/deleted between page requests, results can shift, causing skipped or duplicated items). **Cursor-based (keyset) pagination** returns an opaque cursor pointing at the last-seen item (typically an encoded primary key / sort key), and the next request says "give me the next N after this cursor" — this translates to an indexed `WHERE id > :cursor ORDER BY id LIMIT N` style query, which stays fast regardless of depth, and is stable against concurrent inserts/deletes because it's anchored to a real row, not a row count. The trade-off: cursors don't support jumping to an arbitrary page number, and they're slightly more complex to implement (especially with multi-column sort).

**Code example:**
```http
GET /repos/octocat/hello-world/issues?per_page=50 HTTP/1.1

# response Link header:
Link: <https://api.github.com/...&page=2>; rel="next"
```

**Follow-up question:**
How would you design a cursor for a feed sorted by `(created_at, id)` where `created_at` isn't unique?

**Follow-up good answer:**
Encode both columns into the cursor, not just `created_at` — e.g. a base64 blob of `{"created_at": "...", "id": 12345}`. The next-page query then becomes a compound keyset comparison: `WHERE (created_at, id) < (:cursor_created_at, :cursor_id) ORDER BY created_at DESC, id DESC LIMIT N` (row-value comparison), or equivalently `WHERE created_at < :c OR (created_at = :c AND id < :i)`. This works correctly even when many rows share the same `created_at`, because `id` acts as a tiebreaker that makes the composite key unique — without it, rows with an identical timestamp could be skipped or duplicated across page boundaries. The underlying index needs to match: a composite index on `(created_at, id)` so the comparison can still be served efficiently rather than falling back to a full scan.

**Glossary:**
- **Keyset pagination** — cursor-based pagination anchored to indexed column values rather than a row offset.
- **Opaque cursor** — a cursor value the client treats as a black box (often base64-encoded), so the server is free to change its internal encoding.

**Mental model:**
A classic performance-and-correctness-under-scale question — tests whether the candidate has actually hit the "OFFSET gets slow" wall in production, not just read about pagination in the abstract.

**TL;DR:**
Offset pagination scans and discards skipped rows, getting slower and less stable at depth; cursor pagination anchors to an indexed value and stays fast and stable regardless of depth.

**References:**
- [GitHub REST API: Using pagination in the REST API](https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api)
- [PostgreSQL docs §9.24.5 — Row and Array Comparisons (row-wise keyset comparison)](https://www.postgresql.org/docs/current/functions-comparisons.html)

---

### Q9. How does rate limiting actually work under the hood — walk through the token bucket algorithm?

**Question:**
How does rate limiting actually work under the hood — walk through the token bucket algorithm?

**Good answer:**
A token bucket holds up to `B` tokens (the burst capacity) and refills at a steady rate `R` tokens/second. Each incoming request must take one token to proceed; if the bucket is empty, the request is rejected (typically with `429 Too Many Requests`, often with a `Retry-After` header). This allows short bursts up to the bucket size while enforcing a steady long-term rate — which better matches real client traffic patterns than a naive fixed-window counter (which allows a burst of `2x` the limit right at a window boundary, since a client can spend its whole quota at the end of one window and the start of the next). Stripe and most large API providers implement exactly this, usually with the bucket state held in a fast shared store (e.g. Redis) so it works correctly across multiple API server instances, not just per-instance. An alternative is sliding-window-log, which is more precise (no boundary-burst issue) but more expensive to store (every request timestamp), whereas token bucket is O(1) state per client.

**Code example:**
```
# Simplified token bucket check (pseudocode)
now = current_time()
elapsed = now - bucket.last_refill
bucket.tokens = min(bucket.capacity, bucket.tokens + elapsed * refill_rate)
bucket.last_refill = now
if bucket.tokens >= 1:
    bucket.tokens -= 1
    allow_request()
else:
    reject_with_429()
```

**Follow-up question:**
You're rate-limiting per API key across a fleet of stateless servers — where does the bucket state live, and what happens if that store goes down?

**Follow-up good answer:**
The bucket state lives in a fast shared store all instances can reach — typically Redis, using an atomic script (e.g. Lua) so the check-and-decrement is a single atomic operation and safe under concurrent access from multiple app servers. If that store goes down, you face a fundamental availability-vs-protection trade-off with no universally correct answer: **fail open** (let all requests through when the limiter is unreachable) keeps the service available but removes the only thing protecting downstream systems from a load spike right when the store's own failure may itself be load-related; **fail closed** (reject everything) protects the backend but turns your own rate limiter into a self-inflicted outage for every caller. A common middle ground is falling back to per-instance in-memory counters using `global_limit / num_instances` as each instance's local limit — inaccurate (since instances don't coordinate) but bounded, so you get some protection without a hard dependency on the shared store's availability.

**Glossary:**
- **Token bucket** — rate-limiting algorithm allowing bursts up to a capped capacity while enforcing a steady average rate.
- **Fixed window counter** — a simpler (flawed) rate limiter that resets a counter every fixed interval, vulnerable to boundary bursts.

**Mental model:**
Performance/internals question testing whether the candidate can explain the actual mechanism (not just "there's a rate limit"), including its failure modes and how it holds up in a distributed, multi-instance deployment.

**TL;DR:**
A token bucket allows bursts up to its capacity while enforcing a steady refill rate, avoiding the boundary-burst flaw of a naive fixed-window counter.

**References:**
- [Stripe Engineering: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [RFC 9110 §15.5.30 — 429 Too Many Requests semantics context](https://www.rfc-editor.org/rfc/rfc9110)
- [Envoy proxy docs — rate limit filter `failure_mode_deny` (fail-open default vs. fail-closed)](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_filter)

---

### Q10. Explain HTTP caching: Cache-Control, ETag/If-None-Match, and when a 304 gets returned.

**Question:**
Explain HTTP caching: Cache-Control, ETag/If-None-Match, and when a 304 gets returned.

**Good answer:**
`Cache-Control` directives govern how a response can be cached: `no-store` means never cache it at all; `no-cache` means it *can* be stored but must be revalidated with the origin before reuse (it doesn't mean "don't cache," despite the name); `max-age=N` sets freshness for N seconds, after which it's stale and needs revalidation; `private` restricts caching to the end-user's own cache (not shared/CDN caches), `public` explicitly allows shared caching. For revalidation, the server sends an `ETag` (an opaque fingerprint of the resource's current state) with the response; on the next request, the client sends that value back in `If-None-Match`. If the resource hasn't changed, the server responds `304 Not Modified` with no body, saving bandwidth — the client just reuses its cached copy. `Last-Modified`/`If-Modified-Since` is the older, weaker equivalent based on timestamps rather than a content fingerprint.

**Code example:**
```http
# First request
GET /orders/42 HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: private, max-age=60
ETag: "33a64df551"

# After max-age expires, revalidate:
GET /orders/42 HTTP/1.1
If-None-Match: "33a64df551"

HTTP/1.1 304 Not Modified
```

**Follow-up question:**
Why must a `304 Not Modified` response never include a body, and what does that imply about how you generate the ETag?

**Follow-up good answer:**
RFC 9110 specifies that a `304` response's whole purpose is to tell the client "your cached copy is still valid" without re-transmitting the representation — the entire point is to save the bandwidth/time of resending a body the client already has, so including one would defeat the mechanism (and the spec explicitly forbids a message body in a 304). This means the server must be able to determine whether the resource has changed *without* having to (re)generate the full response body — the ETag must be computable cheaply and independently of rendering the whole representation, e.g. derived from a stored `updated_at`/version column or a hash the server already maintains, not by generating the full JSON payload and hashing it every request (which would erase the whole performance benefit of caching in the first place).

**Glossary:**
- **ETag** — an opaque validator (fingerprint) representing a specific version of a resource's content.
- **Revalidation** — asking the origin whether a stale-but-cached representation is still current, without re-downloading it if so.

**Mental model:**
Performance question testing precise knowledge of the caching contract — many engineers know "ETags exist" but can't correctly explain the request/response cycle or the semantic difference between `no-cache` and `no-store`.

**TL;DR:**
Cache-Control governs storability/freshness, ETag + If-None-Match let the server issue a bodyless 304 when the cached copy is still valid.

**References:**
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)

---

### Q11. What is CORS, why does the browser enforce it, and what triggers a preflight request?

**Question:**
What is CORS, why does the browser enforce it, and what triggers a preflight request?

**Good answer:**
CORS (Cross-Origin Resource Sharing) is a browser-enforced mechanism that relaxes the same-origin policy in a controlled way. By default, browsers block a script on `origin-a.com` from reading responses from `origin-b.com` to prevent malicious sites from silently using a logged-in user's credentials against another site (e.g. reading their bank balance via an authenticated request made from an attacker's page). CORS lets `origin-b.com`'s server explicitly opt certain origins into cross-origin access via response headers like `Access-Control-Allow-Origin`. A **preflight** — an automatic `OPTIONS` request the browser sends before the real one — is triggered for "non-simple" requests: any method other than GET/HEAD/POST, custom headers, or a `Content-Type` other than `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`. The server answers the preflight with `Access-Control-Allow-Methods`/`-Headers` to declare what it permits; only if that succeeds does the browser send the real request.

**Code example:**
```http
OPTIONS /doc HTTP/1.1
Origin: https://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type,x-pingother

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
```

**Follow-up question:**
A request with `credentials: include` is failing CORS even though `Access-Control-Allow-Origin: *` is set — why?

**Follow-up good answer:**
Per the Fetch/CORS spec (as documented by MDN), the wildcard `*` is explicitly disallowed for `Access-Control-Allow-Origin` on credentialed requests — a request carrying cookies, HTTP auth, or client TLS certs, sent with `credentials: "include"` (or `XMLHttpRequest.withCredentials = true`). If the server responds with `*`, the browser blocks access to the response, because a wildcard combined with credentials would let *any* origin read privileged, cookie-scoped data — exactly the cross-site data leak CORS exists to prevent. The fix is for the server to echo back the specific requesting `Origin` value (validated against an allowlist) instead of `*`, and to also send `Access-Control-Allow-Credentials: true`.

**Glossary:**
- **Same-origin policy** — the browser security model restricting scripts to interacting only with resources from the same scheme+host+port unless explicitly relaxed.
- **Preflight request** — an automatic `OPTIONS` request the browser sends to check permissions before a "non-simple" cross-origin request.

**Mental model:**
Tests whether the candidate understands CORS is a *browser*-enforced client-side protection (not a server security feature per se) and can reason about why a given request does or doesn't trigger a preflight.

**TL;DR:**
CORS lets a server opt specific origins into cross-origin access; the browser preflights any 'non-simple' request (non-GET/HEAD/POST, custom headers, or an unusual Content-Type) with an OPTIONS check first.

**References:**
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)

---

### Q12. Walk through the OAuth 2.0 authorization code flow, and explain when you'd use client credentials instead.

**Question:**
Walk through the OAuth 2.0 authorization code flow, and explain when you'd use client credentials instead.

**Good answer:**
OAuth 2.0 defines four roles: the **resource owner** (the user), the **client** (the app requesting access), the **authorization server** (issues tokens), and the **resource server** (hosts the protected API). In the **authorization code flow**, the client redirects the resource owner to the authorization server to authenticate and consent; the authorization server redirects back to the client with a short-lived authorization code (not a token) via the browser; the client then exchanges that code for an access token in a direct, back-channel server-to-server call — meaning the access token itself never passes through the browser/user-agent. This is the right flow whenever a human user needs to grant an app access to their own resources. The **client credentials** grant, by contrast, has no user involved at all: the client authenticates as itself (client ID + secret) to get a token representing the client's own access — used for service-to-service/machine-to-machine calls, like a backend job calling another internal API.

**Code example:**
```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
client_id=s6BhdRkqt3
```

**Follow-up question:**
Why was the implicit grant deprecated in favor of authorization code + PKCE for public clients like SPAs?

**Follow-up good answer:**
RFC 9700 (the OAuth 2.0 Security Best Current Practice) formally deprecates the implicit grant. Its core flaw: the implicit grant returns the access token directly in the redirect URI fragment, meaning the token passes through the browser's address bar/history and can be exposed via browser history, referrer headers, or malicious browser extensions/scripts — with no back-channel exchange step to keep it out of the user-agent entirely. It's also more exposed to interception and replay since there's no proof that the party redeeming the response is the same party that started the flow. The replacement — authorization code + PKCE — keeps the token exchange in a back-channel request and adds a cryptographic `code_verifier`/`code_challenge` pair: the client generates a secret verifier up front, sends only its hashed challenge in the initial request, and must present the original verifier when exchanging the code for a token, so an attacker who intercepts just the authorization code (e.g. via a malicious app registering the same redirect scheme) can't redeem it without also having the verifier. RFC 9700 now mandates PKCE for all authorization code flows, including confidential clients, not just public ones like SPAs.

**Glossary:**
- **Access token** — short-lived credential the client presents to the resource server.
- **Refresh token** — longer-lived credential used to obtain new access tokens without re-prompting the user.
- **PKCE (Proof Key for Code Exchange)** — an extension protecting the authorization code flow for clients that can't safely hold a client secret.

**Mental model:**
Security-fundamentals question testing whether the candidate can correctly map "who's the actor" (human vs. machine) to "which grant type," a very common real-world design decision.

**TL;DR:**
Authorization code flow keeps the access token off the browser via a back-channel exchange, for use whenever a human user grants access; client credentials skips the user entirely for service-to-service calls.

**References:**
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security (implicit grant deprecation, mandatory PKCE)](https://www.rfc-editor.org/rfc/rfc9700.html)

---

### Q13. What is HATEOAS, and why does Roy Fielding say most "REST APIs" aren't actually REST?

**Question:**
What is HATEOAS, and why does Roy Fielding say most "REST APIs" aren't actually REST?

**Good answer:**
HATEOAS (Hypermedia As The Engine Of Application State) means a client should be able to navigate an entire API by following links embedded in the responses it receives, rather than by having URI structures ("`/users/{id}/orders`") hardcoded into the client or documented out-of-band. In a fully hypermedia-driven API, a response for an order would include links for "cancel," "pay," "view customer" — the available *actions* are discoverable at runtime, and the server is free to change its URI scheme without breaking clients, since clients never construct URIs themselves. Fielding has explicitly criticized APIs that call themselves "REST" while requiring out-of-band documentation describing fixed URI templates and which methods apply where — from his view, that's just "HTTP RPC," missing REST's core value: long-term evolvability, because the client-server coupling to specific URI shapes is exactly what REST's uniform interface + hypermedia constraints are meant to eliminate.

**Follow-up question:**
What's the actual practical cost of NOT doing HATEOAS — teams ship non-hypermedia "REST" APIs successfully all the time, so is this constraint just academic?

**Follow-up good answer:**
The real cost shows up specifically as coupling: every client that hardcodes a URI template (`/users/{id}/orders/{orderId}/cancel`) instead of following a `cancel` link returned in a response is now coupled to that exact URI shape staying stable forever, or every client breaks simultaneously on any restructuring. For an API with a handful of clients you control and deploy alongside, that's a manageable, even preferable trade — you get simpler client code and a smaller surface to design (no media-type/link-relation vocabulary to invent) in exchange for accepting that URI changes require coordinated client updates. The cost becomes real once you have many independent, slowly-updating third-party clients (the scenario Fielding was actually addressing) — then every URI-shape change becomes a breaking-change event requiring a version bump, deprecation cycle, and migration comms, which HATEOAS would have avoided by letting clients discover the current URI at runtime. So it's not academic, but it's also not free — it's a genuine trade-off most teams correctly evaluate as not worth it for their actual client population.

**Glossary:**
- **Hypermedia** — content containing links/controls that drive further interaction (as in HTML, where you don't hardcode form endpoints — you follow the `<form action>` given to you).
- **Media type** — per Fielding, this is where the descriptive/behavioral contract should live (e.g. a custom `application/vnd.example+json` type defining available relations), not in separate API docs.

**Mental model:**
Tests depth beyond "REST = CRUD over HTTP" — can the candidate articulate what real REST purity buys you (evolvability) and have an informed, non-dogmatic opinion on why most teams skip it anyway?

**TL;DR:**
HATEOAS means clients navigate via links returned in responses instead of hardcoded URIs — without it, Fielding considers an API 'HTTP RPC,' not true REST.

**References:**
- [Roy Fielding: REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)

---

### Q14. Production users report an API endpoint is "slow." Walk through how you'd diagnose it.

**Question:**
Production users report an API endpoint is "slow." Walk through how you'd diagnose it.

**Good answer:**
First, quantify it: pull p50/p95/p99 latency for that endpoint from your APM/metrics system (not just averages — averages hide tail latency that's often the actual complaint) and correlate with traffic volume and deploy timestamps to see if it's a regression or load-related. Then use **distributed tracing** (e.g. OpenTelemetry spans) to break the request down: how much time is in your app code vs. downstream calls (database, cache, third-party APIs) vs. network? This usually immediately narrows it to one layer. If it's the database, pull the actual query and run `EXPLAIN ANALYZE` to check for missing indexes or bad plans. If it's downstream service calls, check whether they're serialized when they could be parallelized, or whether there's a chatty N+1 pattern (one API call fanning out into many backend calls). If it's CPU-bound app code, profile it (flame graph) rather than guessing. Finally, reproduce under load in a staging environment before shipping a fix, and validate the fix against the same p95/p99 metrics post-deploy — "I fixed it" isn't done until the dashboard confirms it.

**Follow-up question:**
Your trace shows the database query itself is fast (2ms), but the endpoint is still 800ms — where do you look next?

**Follow-up good answer:**
With the DB ruled out, work outward from the trace's unaccounted-for time. Common culprits, in rough order of likelihood: (1) other downstream calls not yet instrumented as spans — a third-party API, an internal microservice, a cache lookup that's actually slow (e.g. cold cache or a network-partitioned Redis); (2) serialization/deserialization cost — large response payloads being JSON-encoded, or ORM object hydration turning a fast query into a slow object-mapping step; (3) connection acquisition — the app waiting on a connection pool (DB or HTTP client pool) that's exhausted, which shows up as "wait time" not "query time" and is easy to miss if your tracing only wraps the query execution itself; (4) app-level CPU work — synchronous business logic, validation, or serialization that's blocking the event loop/thread; (5) queueing before the request even reaches your handler — a thread pool or async runtime backlog. The fix is to widen the trace's instrumentation (span connection-pool wait time, serialization, and every downstream call explicitly) rather than assume the gap is "network," and if that still doesn't explain it, profile the request handler directly (flame graph / CPU profiler) to see where wall-clock time is actually going.

**Glossary:**
- **APM (Application Performance Monitoring)** — tooling (e.g. Datadog, New Relic) that tracks latency, throughput, and errors per endpoint.
- **Distributed trace / span** — a record of one unit of work (e.g. one DB call) within a request, linked into a tree showing where time was spent end-to-end.
- **p95/p99 latency** — the latency below which 95%/99% of requests complete; far more informative than an average for user-facing tail behavior.

**Mental model:**
This is the trending "how do you actually debug performance" question — it's testing methodology and tool fluency, not whether they can recite a single cause. A weak answer jumps straight to "add caching"; a strong answer starts with measurement.

**TL;DR:**
Diagnosing 'slow' starts with p95/p99 latency and distributed tracing to isolate which layer (DB, downstream call, app code) is actually responsible, then fix and re-measure against the same metrics.

**References:**
- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)

---

### Q15. What's a "chatty" API, and how does it relate to the N+1 problem?

**Question:**
What's a "chatty" API, and how does it relate to the N+1 problem?

**Good answer:**
A chatty API is one that forces clients (or forces the API's own backend) into many small round trips to accomplish one logical task, instead of returning what's needed in one call. The N+1 pattern is the classic instance: fetching a list of N items (1 call), then making one additional call *per item* to get related data (N more calls) — e.g. fetching 50 orders, then calling `/customers/{id}` 50 times to get each order's customer name. Each round trip pays full network latency (and, if it's a backend calling a database, a full query round trip), so this scales linearly-badly with list size. Fixes include: batching/bulk endpoints (`/customers?ids=1,2,3`), embedding related data in the list response (accepting some over-fetching), using GraphQL so the client specifies the full shape it needs in one request, or, at the database layer, replacing a per-row query with a single JOIN or `WHERE IN (...)`.

**Code example:**
```
# N+1 (bad): 1 + 50 round trips
GET /orders            -> 50 orders, each with customer_id
GET /customers/1
GET /customers/2
... x50

# Fixed: 2 round trips
GET /orders
GET /customers?ids=1,2,3,...,50
```

**Follow-up question:**
Your team wants to fix this by embedding the full customer object in every order response — what's the downside of that approach?

**Follow-up good answer:**
It trades the N+1 round-trip problem for over-fetching and payload bloat: every order response now carries a full, possibly large customer object even when the caller only needed the customer's name, and if 50 orders share the same 5 customers, that customer data gets duplicated 50 times in the response instead of fetched once. It also couples the orders endpoint's response shape to the customer schema, so an unrelated change to customer fields now affects every consumer of orders, and different callers that want different subsets of customer data (or none at all) all pay the same fixed cost. Better middle grounds: a bulk `/customers?ids=...` endpoint the client calls once after fetching orders (2 round trips total, no duplication, no coupling), or letting the caller opt in to embedding via a field-selection parameter (`?include=customer`) so the cost is only paid by callers who need it — or GraphQL, if the client population is diverse enough that different callers routinely want different shapes.

**Glossary:**
- **N+1 problem** — 1 query/call to fetch a collection, plus N further calls (one per item) for related data.
- **Batch/bulk endpoint** — an API operation that accepts multiple identifiers and returns multiple resources in a single round trip.

**Mental model:**
Connects a very common real-world performance pitfall to general distributed-systems theory (round-trip cost dominates at scale) — tests whether the candidate recognizes the pattern by name and knows multiple mitigations, not just one.

**TL;DR:**
A chatty/N+1 API pays one round trip per related item instead of batching them — fixed with bulk endpoints, embedding, or a query language that lets the client shape one request.

**References:**
- [Google Cloud API Design Guide — Resource-oriented design](https://docs.cloud.google.com/apis/design)

---

### Q16. Why do persistent (keep-alive) connections matter for API performance, and what's actually happening at the TCP/HTTP layer?

**Question:**
Why do persistent (keep-alive) connections matter for API performance, and what's actually happening at the TCP/HTTP layer?

**Good answer:**
HTTP/1.0 opened a new TCP connection per request by default — each request paid a full TCP handshake (and, over HTTPS, a TLS handshake too) before any data moved. HTTP/1.1 defaults to **persistent connections**: the same TCP connection is reused for multiple sequential requests/responses, so that setup cost is paid once, not per request. This matters a lot for latency, especially over high-RTT networks or TLS, where handshake cost can dominate a small request's total time. The connection stays open until either side sends `Connection: close`, or it's closed due to idle timeout. The trade-off is server resource usage — each open connection consumes memory/file descriptors, so a server handling many concurrent clients needs to manage connection limits and idle timeouts carefully. HTTP/2 takes this further with multiplexing multiple requests over a *single* connection concurrently, avoiding HTTP/1.1's per-connection head-of-line blocking.

**Follow-up question:**
Why can HTTP/1.1 pipelining (sending multiple requests without waiting for responses) still suffer head-of-line blocking, and how does HTTP/2 solve it differently?

**Follow-up good answer:**
HTTP/1.1 pipelining lets a client send several requests back-to-back on one connection without waiting for each response, but the protocol still requires responses to come back **strictly in the order the requests were sent** — there's no way to tag a response as belonging to a specific request. So if the first request is slow, every response behind it is stuck waiting even if their own work finished first: head-of-line blocking at the application/HTTP layer. HTTP/2 (RFC 9113) fixes this by introducing **streams**: each request/response exchange gets its own independent stream ID multiplexed over the same TCP connection, so responses can be interleaved and returned in any order — a slow stream no longer blocks unrelated streams at the HTTP layer. The catch, per RFC 9113 itself, is that this only solves HOL blocking *above* TCP: since all streams still share one TCP connection, a single lost/delayed TCP segment still stalls the entire connection at the transport layer (TCP guarantees in-order byte delivery), affecting every HTTP/2 stream regardless of which stream's data was actually lost. That residual TCP-level HOL blocking is exactly what HTTP/3 (built on QUIC/UDP instead of TCP) was designed to eliminate, by giving each stream its own independent loss-recovery.

**Glossary:**
- **TCP handshake** — the 3-way SYN/SYN-ACK/ACK exchange required to establish a TCP connection before any application data can flow.
- **Head-of-line blocking** — a slow/stuck response blocking all responses queued behind it on the same connection.

**Mental model:**
Internals question — tests whether the candidate can go one layer below "HTTP" into what a connection actually costs, which is foundational for reasoning about API latency at scale.

**TL;DR:**
HTTP/1.1 keep-alive reuses a TCP connection across requests to avoid repeated handshake cost; HTTP/2 goes further by multiplexing multiple requests over one connection at once.

**References:**
- [RFC 9112 §9.3 Persistence](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 9113 — HTTP/2 (stream multiplexing; TCP-level head-of-line blocking not addressed by HTTP/2)](https://www.rfc-editor.org/rfc/rfc9113.html)

---

### Q17. Compare gRPC and REST — when would you choose gRPC for an API?

**Question:**
Compare gRPC and REST — when would you choose gRPC for an API?

**Good answer:**
gRPC is an RPC framework built on HTTP/2, using Protocol Buffers (a compact binary serialization format) by default instead of JSON, with the contract defined in a `.proto` file that generates strongly-typed client/server code in many languages. Compared to a typical REST+JSON API: gRPC payloads are smaller and faster to (de)serialize (binary vs. text), it natively supports streaming (client-streaming, server-streaming, and bidirectional streaming, all multiplexed over one HTTP/2 connection), and the generated code eliminates a lot of hand-written client boilerplate/type mismatches. The trade-offs: it's not browser-native (needs gRPC-Web + a proxy for browser clients), it's less human-debuggable (binary payloads vs. readable JSON in curl/devtools), and it couples client and server more tightly to a shared schema (which is a feature for internal microservice-to-microservice calls, but friction for a public API you want third parties to explore easily). Rule of thumb: gRPC shines for internal service-to-service communication where performance and strong typing matter and you control both ends; REST/JSON tends to win for public-facing APIs where accessibility, debuggability, and broad client compatibility matter more.

**Follow-up question:**
You're building a public third-party-facing API — would gRPC's performance advantage ever outweigh REST's accessibility advantage there?

**Follow-up good answer:**
Occasionally, yes — when the API's consumers are themselves sophisticated backend systems (not browsers or ad-hoc scripts) and raw throughput/latency genuinely matters more than exploratory debuggability: high-frequency trading feeds, real-time telemetry ingestion, or infrastructure APIs consumed primarily by other well-resourced platforms (e.g. Google's own public Cloud APIs offer gRPC alongside REST for exactly this reason). But for the median public API — where third-party developers need to explore it with curl/Postman, debug it by eye, and integrate quickly without generating client stubs from a `.proto` file — REST/JSON's accessibility usually wins, so the common real-world pattern is offering *both*: a REST/JSON facade (often generated from the same underlying gRPC service via a transcoding gateway) for broad accessibility, and native gRPC for consumers who specifically need the performance and are equipped to use it.

**Glossary:**
- **Protocol Buffers (protobuf)** — Google's binary serialization format and IDL used to define gRPC service contracts.
- **HTTP/2 multiplexing** — multiple concurrent request/response streams over a single TCP connection, which gRPC relies on for streaming.

**Mental model:**
Trade-off/comparison question — tests whether the candidate picks a technology based on the actual constraints of a scenario (internal vs. public, streaming needs, client diversity) rather than "gRPC is faster so it's always better."

**TL;DR:**
gRPC trades REST's human-debuggable JSON and browser-native accessibility for binary protobuf, native streaming, and strong typing — best for internal service-to-service calls, not public APIs.

**References:**
- [gRPC: Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/)
- [Google Cloud Endpoints docs — gRPC Transcoding (REST/JSON facade over a gRPC service)](https://cloud.google.com/endpoints/docs/grpc/transcoding)

---

### Q18. Compare GraphQL and REST — what real problem does GraphQL solve?

**Question:**
Compare GraphQL and REST — what real problem does GraphQL solve?

**Good answer:**
GraphQL exposes a single endpoint where the client specifies exactly the shape of data it wants in the query itself, rather than the server dictating a fixed response shape per endpoint. This directly solves REST's classic **over-fetching** (a REST endpoint returns a whole resource even if the client needs 2 of its 20 fields) and **under-fetching** (the client needs data spanning multiple resources, so it either needs a bespoke aggregating endpoint or multiple round trips — see the N+1/chatty API problem) — a GraphQL client can ask for exactly the nested fields it needs from multiple related resources in one round trip. The trade-offs: a naive GraphQL resolver implementation can *introduce* its own N+1 problem at the resolver level (needs solving with batching, e.g. Dataloader-style patterns), HTTP-level caching (which relies on distinct cacheable URLs) doesn't work the same way against a single POST endpoint, and query complexity/cost needs to be bounded server-side since clients can construct expensive nested queries.

**Follow-up question:**
How would you prevent a malicious or careless client from sending a GraphQL query so deeply nested it takes down your database?

**Follow-up good answer:**
Depth limiting alone (capping how many levels of nesting a query can have) is a blunt instrument and not sufficient on its own, because fields vary wildly in how expensive they are to resolve — a shallow query can still be catastrophically expensive if it fans out wide (e.g. requesting a field that triggers thousands of resolver calls) rather than deep. The more robust approach is **query complexity/cost analysis**: assign a cost to each field in the schema (fields backed by expensive service calls or large fan-out get a high cost, cheap scalar fields get a low cost, often defaulting to 1 if unspecified), walk the incoming query's AST before execution to sum the total cost, and reject the query outright if it exceeds a configured maximum — so expensive queries are rejected before any resolver runs, not after the damage is done. This is commonly combined with per-client/per-API-key rate limiting based on cumulative query cost (not just request count), since a single complex query can be equivalent to hundreds of simple REST calls.

**Glossary:**
- **Resolver** — a GraphQL server-side function responsible for fetching the data for one field in a query.
- **Dataloader pattern** — batching + caching resolver calls within a single request to avoid resolver-level N+1 queries.

**Mental model:**
Trade-off question testing whether the candidate understands GraphQL's actual value proposition (client-driven shape, solving over/under-fetching) versus just knowing "it's an alternative to REST," and whether they know its own performance pitfalls.

**TL;DR:**
GraphQL lets the client specify exactly the fields/shape it needs in one request, solving REST's over-fetching and under-fetching — at the cost of new caching and query-cost risks.

**References:**
- [GraphQL: Introduction to GraphQL](https://graphql.org/learn/)
- [GraphQL.js docs — Operation Complexity Controls](https://www.graphql-js.org/docs/operation-complexity-controls/)

---

### Q19. A downstream API your service depends on starts failing intermittently — how does the circuit breaker pattern help, and how is it different from just retrying?

**Question:**
A downstream API your service depends on starts failing intermittently — how does the circuit breaker pattern help, and how is it different from just retrying?

**Good answer:**
A **retry** pattern assumes the failure is transient and the operation will likely succeed if attempted again — useful for a single blip. But if a downstream dependency is genuinely down or overloaded, blind retries from every caller make it worse (retry storms), and every caller pays the full timeout latency on every failed attempt. A **circuit breaker** wraps calls to that dependency and tracks failure rate; once failures cross a threshold, it "opens" the circuit — subsequent calls fail fast immediately (no network call, no waiting for a timeout) for a cooldown period, protecting both your service (freed-up threads/latency) and the struggling downstream service (no added load while it's already failing). After the cooldown, it moves to "half-open," letting a small number of test calls through to check if the dependency has recovered before fully closing the circuit again. In short: retry handles transient blips, circuit breaker handles sustained failures and prevents cascading overload.

**Code example:**
```
State machine: CLOSED --(failures > threshold)--> OPEN
OPEN --(cooldown elapsed)--> HALF_OPEN
HALF_OPEN --(test call succeeds)--> CLOSED
HALF_OPEN --(test call fails)--> OPEN
```

**Follow-up question:**
Should the circuit breaker's failure threshold be a raw count or a rate (percentage of recent calls), and why does that choice matter under varying traffic volume?

**Follow-up good answer:**
A raw count ("open after 5 failures") is dangerously miscalibrated across varying traffic: during low traffic, 5 failures might represent 100% of recent calls (a clearly dead dependency) but the breaker takes just as long to trip as it would during high traffic, where 5 failures might be a negligible 0.1% blip that shouldn't trip anything. A rate-based threshold ("open if more than 50% of the last N calls failed," evaluated over a rolling window) scales correctly with actual traffic volume and better reflects the real health signal — Microsoft's Azure Architecture Center guidance for the circuit breaker pattern recommends this rate/rolling-window approach for production use. The practical refinement most implementations add is a **minimum call volume** gate (e.g. "don't evaluate the rate at all until at least 10 calls have been made in the window"), so the breaker doesn't trip on a statistically meaningless sample like 1 failure out of 1 call.

**Glossary:**
- **Cascading failure** — one struggling service's slowness/failure propagating upstream and taking down callers that depend on it.
- **Half-open state** — the circuit breaker's probing state that tests recovery without fully re-exposing the system to failure.

**Mental model:**
Resilience/pitfalls question — tests whether the candidate distinguishes retry from circuit breaker (a very common conflation) and understands cascading failure as a systemic risk, not just a per-call annoyance.

**TL;DR:**
Retry handles transient blips; a circuit breaker trips open after sustained failures to fail fast and protect both caller and callee from cascading overload, then half-opens to probe recovery.

**References:**
- [Microsoft Learn / Azure Architecture Center: Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)

---

### Q20. What is backpressure, and why does it matter when one API/service produces data faster than a downstream consumer can process it?

**Question:**
What is backpressure, and why does it matter when one API/service produces data faster than a downstream consumer can process it?

**Good answer:**
Backpressure is a flow-control mechanism where a slower consumer signals its actual processing capacity back to a faster producer, so the producer throttles itself instead of overwhelming the consumer's buffers (which otherwise either grow unboundedly — risking OOM — or silently drop data). Without backpressure, a fast producer (e.g. a streaming API endpoint, a message queue publisher) can flood a slow consumer, causing unbounded memory growth, cascading latency, or crashes. The Reactive Streams specification formalizes this for JVM-based streaming: a `Subscriber` calls `request(n)` to tell the `Publisher` exactly how many items it's ready to receive next, and the publisher must never push more than that — pull-based flow control baked into the protocol, rather than the producer just pushing everything it has. This matters directly for streaming API endpoints (server-streaming gRPC, chunked HTTP responses, websockets/SSE) where the producer and consumer run at different speeds and are decoupled by a network hop.

**Follow-up question:**
If you can't modify the producer to support backpressure signaling (e.g. it's a third-party firehose), what are your options for protecting your consumer?

**Follow-up good answer:**
When the producer can't cooperate, you have to absorb or shed excess load on the consumer side instead of signaling upstream. Reactive libraries built on Reactive Streams (e.g. Project Reactor) expose exactly this as explicit operators: **buffer** the overflow up to a bounded size (with an overflow strategy like dropping the oldest or newest item once the buffer is full, to keep memory bounded rather than growing unboundedly); **drop** new items outright once the consumer can't keep up (`onBackpressureDrop`); keep only the **latest** value and discard stale intermediate ones if only the freshest data matters (`onBackpressureLatest`); or **error out** the stream entirely once demand is exceeded, forcing an explicit failure instead of silent data loss (`onBackpressureError`). Which one is correct depends on the data's semantics: a sensor reading stream can usually drop-to-latest safely, but a payment event stream generally can't drop anything, so you'd bound the buffer and fail loudly (error) rather than silently lose events.

**Glossary:**
- **Backpressure** — a consumer signaling capacity back to a producer so it throttles output instead of overwhelming the consumer.
- **Bounded buffer** — a fixed-capacity queue between producer and consumer; without backpressure, a bounded buffer must drop or block, and an unbounded one risks OOM.

**Mental model:**
Advanced/internals question — tests whether the candidate understands flow control as a first-class concern in streaming/async APIs, not just request/response thinking, which increasingly matters as APIs adopt streaming (SSE, websockets, gRPC streams).

**TL;DR:**
Backpressure lets a slow consumer signal capacity back to a fast producer so it throttles instead of overwhelming buffers — without it, unhandled producers must be dropped, buffered, or errored on the consumer side.

**References:**
- [Reactive Streams Specification](https://www.reactive-streams.org/)
- [Project Reactor Flux API docs — onBackpressureBuffer / onBackpressureDrop / onBackpressureLatest / onBackpressureError](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=api&tags=rest-api-design-fundamentals&autostart=1" | relative_url }})
