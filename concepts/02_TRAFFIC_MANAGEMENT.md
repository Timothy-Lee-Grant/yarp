# YARP Concepts — Part 2: Traffic Management & Resilience

> Part 1 built the foundations: what a reverse proxy is, the HTTP and ASP.NET Core ideas underneath it, the request lifecycle, and the Route → Cluster → Destination model. Part 2 opens up the *decision-making* stages of that lifecycle — how YARP chooses where a request goes, how it keeps that choice healthy and sticky, how it discovers backends, and how it rewrites requests on the way through. These are the concepts that turn "forward bytes to a server" into "intelligently manage live traffic across a fleet."

---

## 1. Routing and Request Matching

### The "Why"

A reverse proxy fronts many services behind one address. The very first intelligent thing it must do is decide, for each request, *which logical service is being asked for*. This is **routing**: mapping an inbound request to a **route**, and through it to a **cluster**. Get this wrong and every later stage is moot.

The difficulty is twofold. First, the matching criteria are rich: real systems route on path patterns (`/api/orders/{id}`), on host name (`orders.example.com` vs `images.example.com`), on HTTP method, on the presence or value of headers, and on query parameters — often in combination. Second, it must be **fast**, because matching happens on every single request, and the route table can be large.

### The Theory

The core computer-science problem is **multi-dimensional request classification**: given a request described by several attributes, find the most specific matching rule among many candidates, quickly. The naive approach — test every route against every request — is *O(number of routes)* per request and does not scale.

The established solution, which YARP adopts by building on ASP.NET Core's **endpoint routing**, is to compile the route table into an efficient matching structure. Path matching uses a **prefix tree (trie)**-style structure so that matching cost grows with the length of the path, not the number of routes. Non-path criteria (headers, query parameters, methods) are expressed as **matcher policies**: pluggable predicates that the routing engine consults to accept or reject candidate endpoints during matching. When multiple routes could match, **route order** and **specificity** break the tie deterministically — a more specific path or a route with explicit precedence wins.

Two subtleties worth holding:

- **Precedence and ambiguity.** If `/api/{**catchall}` and `/api/orders` both match `/api/orders`, the more specific one should win. Routing engines define precedence rules (segment specificity, then explicit order) so the outcome is predictable rather than dependent on registration order.
- **Header and query matching modes.** Matching a header is not just "present or not." YARP models match *modes* — exact value, prefix, presence-only, and so on — and likewise for query parameters. This is richer than path matching and is why header/query matching is implemented as custom matcher policies layered onto the base engine.

### The Implementation (conceptually, in YARP)

YARP translates each configured route into an ASP.NET Core **endpoint** and registers custom **matcher policies** for headers and query parameters so that those dimensions participate in the same fast matching pass as the path. Route configuration carries the path pattern plus optional host, method, header, and query-parameter constraints. The result of routing is not "run a handler" as in a normal web app, but "this request is bound to *this* route/cluster" — that binding is then read by the downstream proxy middleware. The key insight: **YARP does not reinvent matching; it expresses proxy routes in the platform's own routing vocabulary and extends it where proxying needs more (header/query matchers).**

---

## 2. Load Balancing

### The "Why"

A cluster typically has many interchangeable **destinations** so it can handle more load and survive individual failures. Once routing has chosen the cluster, something must choose *which destination* receives this particular request. That choice is **load balancing**, and its quality directly determines tail latency, throughput, and how evenly your backends wear out. A bad policy overloads one server while others idle; a good one spreads work so the slowest request stays fast.

### The Theory

Load balancing is a problem in **online assignment under uncertainty**: you must place each arriving request without knowing the future arrival pattern and with only imperfect, slightly stale knowledge of each backend's current load. Several classic policies trade off simplicity, fairness, and quality of information. YARP ships the canonical family, and understanding each one's theory tells you when to reach for it.

**Round Robin** cycles through destinations in order: 1, 2, 3, 1, 2, 3… It is trivially simple and gives perfectly even *request counts* if all requests cost the same. Its weakness is that requests are *not* equal — one slow request can pile onto a server that round-robin keeps feeding regardless of how busy it already is.

**Random** picks a destination uniformly at random. With enough requests the distribution evens out by the law of large numbers, and it needs no shared state (which matters across multiple proxy instances). Its weakness is variance: by chance it can transiently overload a server, and like round-robin it ignores current load.

**Least Requests** picks the destination with the fewest in-flight requests right now. This is **load-aware**: it actively steers away from busy servers, so a server stuck on slow requests stops receiving new ones. The cost is that it requires tracking a live counter of outstanding requests per destination — cheap, but it is shared mutable state that must be updated as requests start and finish.

**Power of Two Choices (P2C)** is the elegant compromise and often the best default. Instead of scanning *all* destinations for the least-loaded (which is costly at scale and contends on shared state), it picks **two** destinations at random and sends the request to the less-loaded of the two. The mathematics here is striking: choosing the better of just two random options reduces the maximum load from growing like $\frac{\log n}{\log\log n}$ (pure random) to growing like $\frac{\log\log n}{\log 2}$ — an exponential improvement in the worst-case imbalance, for almost no extra cost. This result is often called "the power of two choices," and it is why P2C is a staple of modern load balancers.

**First** simply uses the first available destination, only moving on when it is unavailable. This sounds naive but is exactly right for **active/passive failover**: you *want* all traffic on the primary, falling to the secondary only when the primary is down.

A summary you can carry:

| Policy | Information used | Best for | Main cost |
| --- | --- | --- | --- |
| First | order only | active/passive failover | no spreading |
| Round Robin | position counter | uniform, equal-cost requests | ignores real load |
| Random | none (stateless) | many proxy nodes, simplicity | variance |
| Least Requests | exact in-flight counts | uneven request costs | scans all, shared state |
| Power of Two Choices | two sampled in-flight counts | general default at scale | tiny sampling overhead |

### The Implementation (conceptually, in YARP)

Each policy is an implementation of a single load-balancing interface, registered by name, and a cluster selects one by name in its config. The load-balancing **middleware** sits in the pipeline after routing and health filtering: it receives the set of *available* (healthy) destinations and asks the configured policy to pick one. Because the policy is just a pluggable interface, writing a custom policy (say, weighted or geography-aware) means implementing that one interface and registering it — no other part of YARP changes. This is the §7 DI "seam" idea from Part 1 made concrete.

---

## 3. Health Checking

### The "Why"

Backends fail — they crash, get overloaded, deploy a bad build, lose their database connection. If the proxy keeps sending traffic to a sick destination, users get errors. **Health checking** is how YARP continuously decides which destinations are fit to receive traffic, so the load balancer only ever chooses among the healthy ones. This is the difference between a proxy that gracefully rides out a partial outage and one that amplifies it.

### The Theory

There are two complementary philosophies for assessing health, and YARP implements both because each catches what the other misses.

**Active health checking** *proactively probes* each destination on a schedule — sending a dedicated request (often to a special health endpoint like `/health`) and judging the response. Its strength is that it detects problems even when no real traffic is flowing to that destination, and it can detect *recovery* (a server that came back) so the destination can be reintroduced. Its cost is extra traffic and the need for a scheduler firing probes at intervals. YARP's default active policy is **Consecutive Failures**: a destination is marked unhealthy after *N* probes fail in a row, which avoids overreacting to a single blip.

**Passive health checking** *observes the real traffic already flowing* and infers health from outcomes — timeouts, connection failures, 5xx responses. Its strength is that it is free (no extra requests) and reflects exactly what real users are experiencing. Its weakness is that it only learns about destinations currently receiving traffic, and once it marks one unhealthy it has no traffic to that destination to notice recovery — so it typically pairs with a **reactivation timer** that tentatively returns the destination after a cool-off. YARP's default passive policy is **Transport Failure Rate**: it tracks the *proportion* of failed transport attempts in a sliding time window and trips when that rate crosses a threshold, which is more robust than counting raw failures because it accounts for volume.

This passive pattern is closely related to the **circuit breaker** idea from resilience engineering: when failures to a dependency exceed a threshold, you "open the circuit" and stop sending requests for a while, then "half-open" to test whether it has recovered before fully restoring traffic. Reading YARP's passive health code with the circuit-breaker mental model makes it click immediately.

A second important concept is the **health state has three values, not two**: `Healthy`, `Unhealthy`, and **`Unknown`** (not yet probed). What do you do with `Unknown` destinations? YARP makes this a policy too — the **available-destinations policy**. The default treats both `Healthy` and `Unknown` as usable (so a freshly added destination isn't excluded just because it hasn't been probed yet). And there is a crucial safety policy called **Healthy-Or-Panic**: if checking health would leave you with *zero* usable destinations, it is better to send traffic to *all* of them and hope, rather than return errors to every user. "Panic mode" — when everything looks unhealthy, treat everything as healthy — is a subtle but vital piece of production wisdom baked into the design.

### The Implementation (conceptually, in YARP)

Active checks are driven by a monitor that uses an **entity scheduler** to fire probes per cluster on its configured interval, sends probe requests via a pluggable probing-request factory, and feeds results into a destination health updater. Passive checks live in a **middleware** that inspects the outcome of each proxied request and updates the destination's passive health state, with reactivation scheduled after a period. Both active and passive states are combined to produce each destination's overall health, and the available-destinations policy decides the final usable set the load balancer sees. Every piece — active policy, passive policy, available-destinations policy, probing-request factory — is a named, swappable interface.

---

## 4. Session Affinity (Sticky Sessions)

### The "Why"

Load balancing assumes destinations are interchangeable, but sometimes they aren't *for a given user*. If a backend keeps per-user state in memory (an in-progress shopping cart, a cached session, an upload), then sending that user's next request to a *different* backend loses the state. **Session affinity** (also called "sticky sessions") ensures that once a client is associated with a destination, its subsequent requests keep going to the *same* destination.

### The Theory

Affinity is fundamentally about **mapping a client identity to a destination and remembering it across requests**, in a system that is otherwise stateless per request. Two questions define the design space.

**How do you recognize the same client again?** The proxy must carry an identifier between requests. The classic mechanism is a **cookie** the proxy sets on the first response and the client returns on every subsequent request; the cookie encodes which destination the client is bound to. An alternative is keying off an existing request **header** (e.g., a client-supplied identifier). YARP supports cookie-based and custom-header-based affinity.

**How do you keep the binding tamper-proof and private?** If the cookie literally said "you are bound to server #3," a client could forge it, and you would leak your topology. So affinity values are typically **encrypted** or **hashed** rather than stored in plaintext. YARP has hashed-cookie and encrypted variants precisely so the binding cannot be read or forged by the client. (The encryption reuses ASP.NET Core's **data protection** system — the platform's standard facility for protecting small pieces of data with managed keys.)

The hardest part is the failure case: **what happens when the affinitized destination is gone or unhealthy?** This is the **affinity failure policy**. There are two reasonable answers, and YARP offers both. One is to **redistribute** — give up on the dead destination and load-balance the request to a healthy one (good availability, but the user loses their server-side state). The other is to **return a 503 error** — refuse rather than silently route to a server that lacks the user's state (good correctness, worse availability). Which is correct depends entirely on what the state means, which is why it is a policy decision and not a fixed behavior.

### The Implementation (conceptually, in YARP)

Affinity is realized as a **middleware** that runs in the request lifecycle *around* destination selection: before load balancing, it checks for an existing affinity marker and, if found and valid, pins the request to that destination; after a destination is chosen for a new session, an **affinitize transform** stamps the marker onto the response (e.g., sets the cookie). The recognition mechanism (cookie/header), the protection mechanism (hash/encrypt), and the failure handling (redistribute/503) are each separate, named, swappable policies — once again the same plug-in architecture.

---

## 5. Service Discovery (Dynamic Destinations)

### The "Why"

In Part 1 we said the world changes under you. Nowhere is that more true than the **set of destinations**. In a modern deployment, backend instances are created and destroyed constantly — autoscaling, rolling deployments, crashes, container rescheduling. Hard-coding IP addresses in config is hopeless. **Service discovery** is the concept of *resolving the current set of backend addresses dynamically* rather than listing them statically.

### The Theory

Service discovery answers "where, right now, are the instances of this service?" There are several standard sources of truth. **DNS** is the most universal: a single service name resolves to a list of IP addresses (and that list changes as instances come and go). **Orchestrator APIs** like Kubernetes maintain authoritative, real-time membership and can push changes. **Service registries** (Consul, Eureka, etc.) are dedicated databases of live instances.

The recurring design problem is **staleness vs. cost**. Resolving fresh on every request is correct but expensive and slow; caching is fast but can route to instances that have vanished. The standard answer is to resolve on an interval (and/or on a change notification), cache the result, and keep it reasonably current — accepting a small staleness window in exchange for cheap per-request lookups. This is the same staleness/cost tension you will meet again in configuration (Part 3).

### The Implementation (conceptually, in YARP)

YARP models discovery behind a **destination resolver** interface. A destination configured with a hostname can be expanded into multiple concrete destination addresses by a resolver. The built-in **DNS resolver** periodically resolves configured host names into the live set of addresses, refreshing on an interval. Because resolution is behind an interface, you can plug in any source — a registry client, a custom API — and YARP will treat the resolved addresses as the cluster's destinations, feeding them into health checking and load balancing exactly as if they had been listed statically. (The Kubernetes ingress controller in Part 3 is a heavyweight, push-based form of this same idea.)

---

## 6. Transforms (Request and Response Rewriting)

### The "Why"

Recall from Part 1 that a proxy mediates *two* HTTP exchanges and must translate between them. Rarely is the outbound request a byte-for-byte copy of the inbound one. The backend may expect a different path, may need to be told the original client's IP, must not see certain client headers, and may add response headers the client should not. **Transforms** are YARP's general mechanism for *modifying requests on the way in and responses on the way out*.

### The Theory

A transform is conceptually a **function over the request (or response) context**: it receives the in-flight message and mutates it — adding, removing, or rewriting headers, path segments, or query parameters. The crucial design properties are **composability** and **ordering**: many small transforms chain together, and because some transforms depend on the effects of others, the order is significant. This is essentially a **middleware pipeline within the forwarding step** — a smaller echo of the request pipeline from Part 1.

The most conceptually important transforms are the **forwarded-headers** transforms, because they encode a genuinely tricky distributed-systems truth: *once a proxy is in the path, the backend can no longer see the original client directly.* The original client IP, the original `Host`, and the original scheme (http/https) are all about the client↔proxy hop and would otherwise be lost. The transforms re-inject this information in one of two competing conventions:

- The **`X-Forwarded-*`** family (`X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Proto`, `X-Forwarded-Prefix`) — the older, de facto standard, where each proxy *appends* its view, building a chain.
- The standardized **`Forwarded`** header (RFC 7239) — one header carrying the same facts in a structured form.

YARP lets you choose which to emit and whether to append to or replace existing values. And here is the security crux: a forwarded header arriving *from the client* may be a **lie** — an attacker spoofing a trusted internal IP. So the correct behavior is to *not blindly trust* inbound forwarded headers and to control whether you append to them or overwrite them. The transform's "action" (append, set, remove, off) exists precisely to manage this trust boundary. This connects directly to the security discussion in Part 3.

Beyond forwarded headers, transforms cover **path** rewriting (prefix removal, pattern-based rewrites), **query-parameter** manipulation (add from route values, add static, remove), **arbitrary header** add/remove/rewrite on both request and response, **HTTP method** changes, and forwarding the client's **TLS certificate** to the backend as a header. There are also **conditional** response transforms (only act when the response meets a condition) and **arbitrary function** transforms (run your own code), which is the ultimate escape hatch.

### The Implementation (conceptually, in YARP)

Transforms are built by **transform factories** that read declarative config (or fluent code) and produce concrete transform objects, which are then composed into a per-route transform pipeline. Request transforms run against a request-transform context just before forwarding; response transforms run against a response-transform context as the reply comes back. The default set already handles the correct, safe copying of headers (respecting the hop-by-hop vs. end-to-end distinction from Part 1) and the forwarded-headers conventions; you layer your custom transforms on top. As everywhere in YARP, factories and transforms are interfaces you can extend.

---

## 7. Cross-Cutting Request Controls: Rate Limiting, Timeouts, Concurrency, Authorization, CORS

These are not destination-selection concerns; they are **policies applied to the request as it flows through the pipeline**, mostly reusing ASP.NET Core's own middleware and exposed per-route by YARP. Grouping them clarifies that YARP's job here is **integration, not reinvention**.

**Rate limiting** protects backends from overload and abuse by capping how many requests a client (or route) may make in a time window. The classic algorithms are the **token bucket** (a bucket refills at a steady rate; each request spends a token; empty bucket means rejection — this naturally allows short bursts up to the bucket size), the **fixed window** (count per calendar window — simple but allows double-rate bursts at window boundaries), the **sliding window** (smooths the boundary problem), and **concurrency limiting** (cap simultaneous in-flight requests rather than rate over time). YARP exposes ASP.NET Core's rate-limiting middleware and lets each route select a named limiter policy.

**Timeouts** bound how long the proxy will wait on a backend before giving up, so a hung backend cannot tie up resources indefinitely. A request timeout is selected per route. This pairs conceptually with passive health checking: a timeout is exactly the kind of failure signal that should also inform whether a destination is healthy.

**Concurrency limits** (and request body size limits, etc.) protect the *proxy itself* from resource exhaustion — bounding how much work it will accept at once so it degrades gracefully under a flood rather than collapsing.

**Authorization** decides whether a caller is *allowed* to reach a route at all, and **CORS** (Cross-Origin Resource Sharing) governs which web origins a browser may call the route from. Both are standard ASP.NET Core policies that YARP lets routes reference by name. The conceptual point — and a recurring exam question for proxy correctness — is **ordering**: authorization and rate limiting must run *before* the request is forwarded to a backend, never after. YARP's pipeline placement guarantees this.

---

## 8. Mental Sandbox

1. **Pick a policy with a reason.** Service A serves cheap, uniform requests across 4 identical instances. Service B serves wildly variable requests (some take 5 ms, some take 5 s) across 40 instances. Service C has one primary and one warm standby. Which load-balancing policy fits each, and articulate *why* using the information-vs-cost trade-off from §2. For Service B, explain in one sentence why Power of Two Choices beats both Random and full Least-Requests at 40 instances.

2. **Design the failure case.** You enable cookie session affinity for a checkout service that holds the cart in server memory. A user's affinitized backend crashes mid-checkout. Walk through what the affinity *failure policy* options (redistribute vs. 503) would each do, and argue which is correct here. Now change the scenario to a stateless service where affinity was only a cache optimization — does your answer flip? Why?

3. **Trust the header or not.** Your proxy sits behind a cloud load balancer that *also* sets `X-Forwarded-For`, and directly receives traffic from the public internet on a second path. Explain why blindly appending to the inbound `X-Forwarded-For` is safe in one case and a spoofing vulnerability in the other, and how the transform "action" (append/set/off) lets you express the correct trust boundary for each ingress path.

When these feel natural, move to **Part 3: Performance, Concurrency & Operations**, where we go under the hood — streaming, lock-free configuration swaps, observability, security, and running YARP in Kubernetes.
