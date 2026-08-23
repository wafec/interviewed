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

**Glossary:**
- **Representation** — a serialized form (e.g. JSON, XML) of a resource's current state.
- **HATEOAS** — Hypermedia As The Engine Of Application State; clients discover available actions from links in responses instead of out-of-band documentation.
- **Uniform interface** — the REST constraint that resources are manipulated through a small, standard set of operations (HTTP methods) and representations.

**Mental model:**
This question tests whether the candidate actually knows REST as an architectural style with constraints, versus having just memorized "REST = HTTP verbs + JSON." It also probes whether they can distinguish textbook purity from pragmatic industry practice without being dogmatic either way.

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

**Glossary:**
- **Sticky session** — a load balancer routing all requests from a client to the same backend instance to preserve server-held state, a common workaround when statelessness is violated.
- **Session affinity** — the same concept as sticky sessions, viewed from the load balancer's config.

**Mental model:**
Tests whether the candidate connects an architectural constraint to its concrete operational payoff (scalability, failover) rather than reciting it as a rule with no reason.

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

**Glossary:**
- **Safe method** — doesn't alter server-observable state (GET, HEAD, OPTIONS).
- **Idempotent method** — repeated identical requests produce the same end state as a single request.

**Mental model:**
Distinguishes candidates who know HTTP semantics precisely enough to reason about retry safety and correctness under network failure, not just "GET reads, POST writes."

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

**Glossary:**
- **JSON Merge Patch (RFC 7396)** — a common PATCH body format where the given object is recursively merged into the resource.
- **JSON Patch (RFC 6902)** — an alternative PATCH format expressing changes as an explicit list of add/remove/replace operations.

**Mental model:**
Checks whether the candidate understands PUT/PATCH semantics precisely enough to avoid the extremely common bug of using PUT with a partial body (silently nulling out unspecified fields).

**References:**
- [RFC 5789 — PATCH Method for HTTP](https://www.rfc-editor.org/rfc/rfc5789)
- [RFC 9110 §9.3.4 PUT](https://www.rfc-editor.org/rfc/rfc9110)

---

### Q5. When do you return 400 vs 422, and 401 vs 403?

**Question:**
When do you return 400 vs 422, and 401 vs 403?

**Good answer:**
**400 Bad Request** is a generic client error — the request is malformed at the protocol/syntax level (invalid JSON, missing required field, wrong type). **422 Unprocessable Entity** is for a request that's syntactically well-formed but semantically invalid — it parses fine, but violates a business rule (e.g. `end_date` before `start_date`, an email that's not actually deliverable-format valid per your rules). **401 Unauthorized** means the request lacks valid authentication credentials at all, or they're invalid/expired — the server doesn't know who you are (despite the confusing name, it's really "unauthenticated"). **403 Forbidden** means the server *does* know who you are, but you don't have permission to perform this action on this resource. Getting this right matters because clients (and monitoring/alerting) branch on these codes — a client should retry-with-fresh-token on 401 but not on 403.

**Follow-up question:**
Should a 403 response ever reveal that the resource exists (vs returning 404 to avoid leaking existence to unauthorized users)?

**Glossary:**
- **Authentication** — verifying who the caller is.
- **Authorization** — verifying what the (known) caller is allowed to do.

**Mental model:**
Tests precision with the client-error status code space, and whether the candidate treats status codes as meaningful contract rather than "200 for success, 500 for anything else, whatever else feels right."

**References:**
- [RFC 9110 §15.5 Client Error 4xx](https://www.rfc-editor.org/rfc/rfc9110)

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

**Glossary:**
- **Idempotency key** — a client-generated unique token identifying a logical operation so retries don't re-execute it.

**Mental model:**
This is a real production-failure-mode question: does the candidate know retries are unsafe by default for POST, and do they know the actual industry-standard mechanism (vs. hand-waving "just check if it already exists")?

**References:**
- [IETF draft-ietf-httpapi-idempotency-key-header-07: The Idempotency-Key HTTP Header Field](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.txt)
- [Stripe: Idempotent Requests](https://stripe.com/blog/rate-limiters)

---

### Q7. What are the main API versioning strategies, and what are the trade-offs?

**Question:**
What are the main API versioning strategies, and what are the trade-offs?

**Good answer:**
Common strategies: **URI versioning** (`/v1/orders`) — simplest, highly visible, but "leaks" version into every resource URI and encourages big-bang version bumps; **header versioning** (a custom header or `Accept` media-type versioning like `Accept: application/vnd.example.v2+json`) — keeps URIs stable/cacheable and is more RESTful (content negotiation), but is less discoverable/harder to test with a browser; **query parameter versioning** (`?version=2`) — easy to add, but caches may not vary correctly on it unless configured. A complementary, often-preferred approach for internal/evolving APIs is **additive, backward-compatible changes only** (add optional fields, never remove/rename, deprecate with sunset headers) so you rarely need a hard version bump at all. The trade-off is always: visibility/simplicity (URI) vs. purity/cacheability (headers) vs. minimizing versioning altogether (strict backward compatibility discipline).

**Follow-up question:**
How do you communicate and enforce a deprecation timeline for `v1` once `v2` ships, without breaking clients who haven't migrated?

**Glossary:**
- **Content negotiation** — client and server agreeing on representation format/version via `Accept`/`Content-Type` headers rather than the URI.
- **Sunset header** — an HTTP header (`Sunset`, per RFC 8594) signaling when a resource/endpoint will stop being available.

**Mental model:**
Tests whether the candidate has an opinion grounded in trade-offs (not just "I use URI versioning because that's what I've seen") and understands the operational cost of breaking changes.

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

**Glossary:**
- **Keyset pagination** — cursor-based pagination anchored to indexed column values rather than a row offset.
- **Opaque cursor** — a cursor value the client treats as a black box (often base64-encoded), so the server is free to change its internal encoding.

**Mental model:**
A classic performance-and-correctness-under-scale question — tests whether the candidate has actually hit the "OFFSET gets slow" wall in production, not just read about pagination in the abstract.

**References:**
- [GitHub REST API: Using pagination in the REST API](https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api)

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

**Glossary:**
- **Token bucket** — rate-limiting algorithm allowing bursts up to a capped capacity while enforcing a steady average rate.
- **Fixed window counter** — a simpler (flawed) rate limiter that resets a counter every fixed interval, vulnerable to boundary bursts.

**Mental model:**
Performance/internals question testing whether the candidate can explain the actual mechanism (not just "there's a rate limit"), including its failure modes and how it holds up in a distributed, multi-instance deployment.

**References:**
- [Stripe Engineering: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [RFC 9110 §15.5.30 — 429 Too Many Requests semantics context](https://www.rfc-editor.org/rfc/rfc9110)

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

**Glossary:**
- **ETag** — an opaque validator (fingerprint) representing a specific version of a resource's content.
- **Revalidation** — asking the origin whether a stale-but-cached representation is still current, without re-downloading it if so.

**Mental model:**
Performance question testing precise knowledge of the caching contract — many engineers know "ETags exist" but can't correctly explain the request/response cycle or the semantic difference between `no-cache` and `no-store`.

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

**Glossary:**
- **Same-origin policy** — the browser security model restricting scripts to interacting only with resources from the same scheme+host+port unless explicitly relaxed.
- **Preflight request** — an automatic `OPTIONS` request the browser sends to check permissions before a "non-simple" cross-origin request.

**Mental model:**
Tests whether the candidate understands CORS is a *browser*-enforced client-side protection (not a server security feature per se) and can reason about why a given request does or doesn't trigger a preflight.

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

**Glossary:**
- **Access token** — short-lived credential the client presents to the resource server.
- **Refresh token** — longer-lived credential used to obtain new access tokens without re-prompting the user.
- **PKCE (Proof Key for Code Exchange)** — an extension protecting the authorization code flow for clients that can't safely hold a client secret.

**Mental model:**
Security-fundamentals question testing whether the candidate can correctly map "who's the actor" (human vs. machine) to "which grant type," a very common real-world design decision.

**References:**
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)

---

### Q13. What is HATEOAS, and why does Roy Fielding say most "REST APIs" aren't actually REST?

**Question:**
What is HATEOAS, and why does Roy Fielding say most "REST APIs" aren't actually REST?

**Good answer:**
HATEOAS (Hypermedia As The Engine Of Application State) means a client should be able to navigate an entire API by following links embedded in the responses it receives, rather than by having URI structures ("`/users/{id}/orders`") hardcoded into the client or documented out-of-band. In a fully hypermedia-driven API, a response for an order would include links for "cancel," "pay," "view customer" — the available *actions* are discoverable at runtime, and the server is free to change its URI scheme without breaking clients, since clients never construct URIs themselves. Fielding has explicitly criticized APIs that call themselves "REST" while requiring out-of-band documentation describing fixed URI templates and which methods apply where — from his view, that's just "HTTP RPC," missing REST's core value: long-term evolvability, because the client-server coupling to specific URI shapes is exactly what REST's uniform interface + hypermedia constraints are meant to eliminate.

**Follow-up question:**
What's the actual practical cost of NOT doing HATEOAS — teams ship non-hypermedia "REST" APIs successfully all the time, so is this constraint just academic?

**Glossary:**
- **Hypermedia** — content containing links/controls that drive further interaction (as in HTML, where you don't hardcode form endpoints — you follow the `<form action>` given to you).
- **Media type** — per Fielding, this is where the descriptive/behavioral contract should live (e.g. a custom `application/vnd.example+json` type defining available relations), not in separate API docs.

**Mental model:**
Tests depth beyond "REST = CRUD over HTTP" — can the candidate articulate what real REST purity buys you (evolvability) and have an informed, non-dogmatic opinion on why most teams skip it anyway?

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

**Glossary:**
- **APM (Application Performance Monitoring)** — tooling (e.g. Datadog, New Relic) that tracks latency, throughput, and errors per endpoint.
- **Distributed trace / span** — a record of one unit of work (e.g. one DB call) within a request, linked into a tree showing where time was spent end-to-end.
- **p95/p99 latency** — the latency below which 95%/99% of requests complete; far more informative than an average for user-facing tail behavior.

**Mental model:**
This is the trending "how do you actually debug performance" question — it's testing methodology and tool fluency, not whether they can recite a single cause. A weak answer jumps straight to "add caching"; a strong answer starts with measurement.

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

**Glossary:**
- **N+1 problem** — 1 query/call to fetch a collection, plus N further calls (one per item) for related data.
- **Batch/bulk endpoint** — an API operation that accepts multiple identifiers and returns multiple resources in a single round trip.

**Mental model:**
Connects a very common real-world performance pitfall to general distributed-systems theory (round-trip cost dominates at scale) — tests whether the candidate recognizes the pattern by name and knows multiple mitigations, not just one.

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

**Glossary:**
- **TCP handshake** — the 3-way SYN/SYN-ACK/ACK exchange required to establish a TCP connection before any application data can flow.
- **Head-of-line blocking** — a slow/stuck response blocking all responses queued behind it on the same connection.

**Mental model:**
Internals question — tests whether the candidate can go one layer below "HTTP" into what a connection actually costs, which is foundational for reasoning about API latency at scale.

**References:**
- [RFC 9112 §9.3 Persistence](https://www.rfc-editor.org/rfc/rfc9112)

---

### Q17. Compare gRPC and REST — when would you choose gRPC for an API?

**Question:**
Compare gRPC and REST — when would you choose gRPC for an API?

**Good answer:**
gRPC is an RPC framework built on HTTP/2, using Protocol Buffers (a compact binary serialization format) by default instead of JSON, with the contract defined in a `.proto` file that generates strongly-typed client/server code in many languages. Compared to a typical REST+JSON API: gRPC payloads are smaller and faster to (de)serialize (binary vs. text), it natively supports streaming (client-streaming, server-streaming, and bidirectional streaming, all multiplexed over one HTTP/2 connection), and the generated code eliminates a lot of hand-written client boilerplate/type mismatches. The trade-offs: it's not browser-native (needs gRPC-Web + a proxy for browser clients), it's less human-debuggable (binary payloads vs. readable JSON in curl/devtools), and it couples client and server more tightly to a shared schema (which is a feature for internal microservice-to-microservice calls, but friction for a public API you want third parties to explore easily). Rule of thumb: gRPC shines for internal service-to-service communication where performance and strong typing matter and you control both ends; REST/JSON tends to win for public-facing APIs where accessibility, debuggability, and broad client compatibility matter more.

**Follow-up question:**
You're building a public third-party-facing API — would gRPC's performance advantage ever outweigh REST's accessibility advantage there?

**Glossary:**
- **Protocol Buffers (protobuf)** — Google's binary serialization format and IDL used to define gRPC service contracts.
- **HTTP/2 multiplexing** — multiple concurrent request/response streams over a single TCP connection, which gRPC relies on for streaming.

**Mental model:**
Trade-off/comparison question — tests whether the candidate picks a technology based on the actual constraints of a scenario (internal vs. public, streaming needs, client diversity) rather than "gRPC is faster so it's always better."

**References:**
- [gRPC: Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/)

---

### Q18. Compare GraphQL and REST — what real problem does GraphQL solve?

**Question:**
Compare GraphQL and REST — what real problem does GraphQL solve?

**Good answer:**
GraphQL exposes a single endpoint where the client specifies exactly the shape of data it wants in the query itself, rather than the server dictating a fixed response shape per endpoint. This directly solves REST's classic **over-fetching** (a REST endpoint returns a whole resource even if the client needs 2 of its 20 fields) and **under-fetching** (the client needs data spanning multiple resources, so it either needs a bespoke aggregating endpoint or multiple round trips — see the N+1/chatty API problem) — a GraphQL client can ask for exactly the nested fields it needs from multiple related resources in one round trip. The trade-offs: a naive GraphQL resolver implementation can *introduce* its own N+1 problem at the resolver level (needs solving with batching, e.g. Dataloader-style patterns), HTTP-level caching (which relies on distinct cacheable URLs) doesn't work the same way against a single POST endpoint, and query complexity/cost needs to be bounded server-side since clients can construct expensive nested queries.

**Follow-up question:**
How would you prevent a malicious or careless client from sending a GraphQL query so deeply nested it takes down your database?

**Glossary:**
- **Resolver** — a GraphQL server-side function responsible for fetching the data for one field in a query.
- **Dataloader pattern** — batching + caching resolver calls within a single request to avoid resolver-level N+1 queries.

**Mental model:**
Trade-off question testing whether the candidate understands GraphQL's actual value proposition (client-driven shape, solving over/under-fetching) versus just knowing "it's an alternative to REST," and whether they know its own performance pitfalls.

**References:**
- [GraphQL: Introduction to GraphQL](https://graphql.org/learn/)

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

**Glossary:**
- **Cascading failure** — one struggling service's slowness/failure propagating upstream and taking down callers that depend on it.
- **Half-open state** — the circuit breaker's probing state that tests recovery without fully re-exposing the system to failure.

**Mental model:**
Resilience/pitfalls question — tests whether the candidate distinguishes retry from circuit breaker (a very common conflation) and understands cascading failure as a systemic risk, not just a per-call annoyance.

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

**Glossary:**
- **Backpressure** — a consumer signaling capacity back to a producer so it throttles output instead of overwhelming the consumer.
- **Bounded buffer** — a fixed-capacity queue between producer and consumer; without backpressure, a bounded buffer must drop or block, and an unbounded one risks OOM.

**Mental model:**
Advanced/internals question — tests whether the candidate understands flow control as a first-class concern in streaming/async APIs, not just request/response thinking, which increasingly matters as APIs adopt streaming (SSE, websockets, gRPC streams).

**References:**
- [Reactive Streams Specification](https://www.reactive-streams.org/)
