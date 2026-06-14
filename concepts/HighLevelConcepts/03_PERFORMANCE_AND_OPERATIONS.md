# YARP Concepts — Part 3: Performance, Concurrency & Operations

> Parts 1 and 2 covered *what* YARP does and *how it decides* where traffic goes. Part 3 goes under the hood to the concepts that make it **fast, safe under concurrency, observable, secure, and operable at scale**. These are the deepest ideas in the project, and the ones most likely to be new if your background is application code rather than systems engineering. Take them slowly — each section is a self-contained concept you could spend a career deepening.

---

## 1. The Forwarding Core: Streaming, Not Buffering

### The "Why"

The single hottest path in any proxy is the actual act of forwarding: read the request from the client, write it to the backend, read the response from the backend, write it back to the client. This happens for *every* request, often tens of thousands per second per node, and request/response bodies can be enormous (file uploads, video, large API payloads). The way you move those bytes determines whether your proxy is fast and memory-stable or slow and prone to running out of memory.

### The Theory

The naive approach — read the entire body into a buffer, then write the buffer — is catastrophic at proxy scale. A 1 GB upload would consume 1 GB of proxy memory; a thousand concurrent ones would consume a terabyte you do not have. The correct model is **streaming**: copy data in small fixed-size chunks from the source to the destination as it arrives, never holding more than one chunk in memory at a time. Memory use becomes *O(chunk size × concurrent requests)* instead of *O(body size × concurrent requests)*.

This is the "move the bytes, don't process them" mindset from Part 1 in its purest form. The proxy is essentially a **pump** connecting two pipes. Several deeper performance ideas layer on top:

- **Backpressure.** If the client is slow to receive but the backend is fast to send (or vice versa), you must not read faster than you can write, or buffers grow without bound. Proper stream copying naturally couples the two: you read the next chunk only after the previous one is written, so the fast side is throttled to the slow side's pace. This is **flow control**, and it is what keeps a streaming proxy memory-stable under mismatched speeds.

- **Buffer reuse and avoiding allocations.** Allocating a fresh buffer per chunk per request creates immense **garbage-collection (GC)** pressure — in a managed runtime like .NET, allocations are cheap individually but ruinous in aggregate at this volume. High-performance .NET code rents buffers from a shared **array pool** and reuses them, and uses allocation-light types like `Span<T>` and `Memory<T>` (views over memory that don't copy) and `ArrayPool<T>`. When you read YARP's stream-copying code and see pooled buffers and span-based APIs, this is why: minimizing per-request allocation is minimizing GC pauses, which is minimizing tail latency.

- **Duplex (bidirectional) copying for upgraded connections.** For WebSockets and other upgraded protocols (Part 1, §5.4), the request/response model is gone; you must pump bytes in *both directions simultaneously* until either side closes. That means two concurrent copy loops, with careful handling of which side closing means what.

- **Asynchronous, non-blocking I/O.** A thread that blocks waiting for a slow network read is a thread doing nothing while consuming stack and scheduler resources. To handle tens of thousands of concurrent connections you cannot dedicate a thread to each. The model is **async I/O**: a small pool of threads services many connections by suspending a request's work when it is waiting on I/O and resuming it when data is ready. In .NET this is the `async`/`await` model over `Task`/`ValueTask`. Conceptually it is **cooperative concurrency**: thousands of logical operations in flight, mapped onto a handful of OS threads. This is *the* reason a single proxy node can serve so many connections, and it pervades every line of the forwarding path.

### The Implementation (conceptually, in YARP)

The forwarder reads the inbound request, applies transforms, and issues an outbound request using a pooled HTTP client (§2). Request and response bodies are moved by a **stream copier** that pumps fixed-size, pooled buffers between source and destination with backpressure, in an async loop. For upgraded connections it runs two copy loops for full-duplex traffic. The result of each copy is tracked as a structured outcome (success, canceled, the client gave up, the backend faulted) so that failures can be classified precisely — which feeds error reporting and the passive health checking from Part 2. The forwarder also distinguishes *stages* of the forwarding operation so telemetry can pinpoint exactly where time went or where a failure occurred.

---

## 2. The Outbound HTTP Client: Connection Pooling

### The "Why"

Every request the proxy forwards needs a connection to a backend. Opening a brand-new TCP connection (and a fresh TLS handshake) for every request would add tens of milliseconds and exhaust ports under load — a classic failure mode called **socket exhaustion**. The proxy must *reuse* connections aggressively.

### The Theory

The solution is **connection pooling**: keep a pool of open, warmed connections to each backend and hand them out for reuse, only opening new ones when the pool is empty. This amortizes the expensive TCP + TLS setup across many requests. With HTTP/2, a single pooled connection can carry many concurrent requests via multiplexing (Part 1, §5.3), so the pool can be small yet high-throughput. The pool must also handle connection lifetime (a connection held forever can become stale or pin to a backend that has since been replaced — so connections are recycled on a lifetime), and per-backend isolation (one slow backend must not starve connections to others).

In .NET, this machinery is `SocketsHttpHandler` underneath `HttpClient`. A vital and counterintuitive lesson the platform learned the hard way: `HttpClient` is meant to be **long-lived and shared**, not created per request (creating one per request reintroduces socket exhaustion). YARP manages these clients carefully for exactly this reason.

### The Implementation (conceptually, in YARP)

YARP creates and configures outbound HTTP clients through a **forwarder HTTP client factory**, one logical client per cluster, configured from the cluster's HTTP-client settings (allowed protocol versions, timeouts, TLS/certificate options, whether to follow redirects, etc.). The factory is an interface you can replace to control client creation entirely. Because the client (and thus its connection pool) is per-cluster and reused across requests, the expensive setup cost is paid rarely and the pool stays warm.

---

## 3. The Heart of YARP: Immutable Snapshots and Lock-Free Configuration

This is the most important concept in Part 3 and arguably in the whole project. It is where the "the world changes under me" mindset from Part 1 gets its rigorous solution. Read it twice.

### The "Why"

Recall that YARP's configuration — routes, clusters, destinations, and all their state (health, dynamic destination lists) — changes *continuously and concurrently with live request processing*. Thousands of requests are being routed right now, each reading the route table, the cluster's destination list, and each destination's health. Simultaneously, a config reload might be replacing routes, a DNS refresh might be changing destinations, and a health probe might be flipping a destination's status.

The obvious approach — protect shared state with **locks** — is a disaster at this scale. Every request would contend on the lock to read the route table; the lock would become the bottleneck; and you would risk deadlocks and priority inversions. Worse, a request that reads the destination list, then reads health, then reads config could see an **inconsistent mix** — a destination from the old list judged against new health data — if those pieces changed between reads.

### The Theory

YARP's answer is a well-known but underused pattern: **immutability plus atomic reference swap**, giving **snapshot consistency** with essentially zero read-side synchronization. The idea has several interlocking parts:

- **Immutable state.** Each piece of runtime state is an object that, once created, is *never modified*. A cluster's configuration, a route, a destination's health snapshot — all immutable. You never edit them in place.

- **Change by replacement, not mutation.** When something changes, you do not mutate the old object; you build a *new* immutable object with the new values and **atomically swap a single reference** to point at it. An atomic reference swap is a single hardware-level operation that is indivisible — no reader can ever observe a half-updated reference; it sees either the entirely-old object or the entirely-new one, never a tear.

- **Snapshot consistency for readers.** When a request begins, it reads the current references *once* and works against those snapshots for its entire lifetime. Even if a swap happens mid-request, the in-flight request keeps using the consistent snapshot it captured at the start. A new request arriving a microsecond later picks up the new snapshot. There is no locking, no blocking, and no possibility of mixing old and new state within one request. This is the "always the latest *and* consistent snapshot available when processing of a request starts" guarantee.

- **Atomic primitives for the mutable bits.** A few things genuinely must be mutable counters shared across threads — e.g., the per-destination in-flight request count that Least-Requests load balancing reads (Part 2), or a passive health failure tally. These use **atomic operations** (lock-free `Interlocked`-style increments/reads) and small thread-safe holder types, rather than locks, so they too avoid contention.

Why this is profound: it converts a hairy concurrency problem ("many readers, occasional writers, must stay consistent, must be fast") into a near-trivial one. Readers never coordinate with anyone. Writers never block readers. The only cost is allocating a new object on change — which is rare relative to reads — and a brief window where old and new snapshots coexist until in-flight requests drain. This is the same family of idea behind **read-copy-update (RCU)** in operating-system kernels and **persistent (immutable) data structures** in functional programming. If you internalize one systems concept from this whole series, make it this one.

### The Implementation (conceptually, in YARP)

YARP's runtime model is explicitly designed around this discipline. Its own internal documentation states the rule plainly: every runtime-state class must be immutable, or wrap an immutable value in an atomic holder, or be a thread-safe atomic counter. The model separates each abstraction (route, cluster, destination) into a stable identity object that holds atomic references to (a) the **config-derived** portion that changes only on config reload and (b) the **dynamic** portion that changes in reaction to runtime events (new destinations discovered, health changed). When config reloads, new config objects are built and the atomic holders are repointed; when health changes, a new dynamic-state object is built and swapped. Requests capture a feature object at the start that exposes the consistent snapshot for their entire duration. This is why YARP can hot-swap configuration "without explicit synchronization overhead across threads" — the architecture, not careful locking, is what makes it safe.

---

## 4. Configuration as a Pluggable, Hot-Reloadable Pipeline

### The "Why"

Part 1 established that configuration is a *living stream of snapshots* and can come from *anywhere*. Here is the machinery that makes both true, and why each piece exists.

### The Theory

A robust configuration system for a proxy needs four conceptual capabilities:

1. **Abstraction of source.** "Where does config come from?" must be a replaceable detail. File, database, Kubernetes, a remote control plane — the proxy core shouldn't care. This is the classic **provider pattern**: an interface that yields the current config and a way to be notified of changes.

2. **Change notification.** When the underlying source changes (the file is edited, the database row updates), the system must learn about it and produce a new snapshot. The standard .NET idiom is a **change token** — a lightweight signal that "the thing you're watching has changed, re-read it." A file-based provider, for instance, watches the file and raises a change token on edit.

3. **Validation before adoption.** A bad config (a route pointing at a nonexistent cluster, a malformed path pattern, contradictory options) must be **rejected before it goes live**, not after it has started breaking requests. So there is a validation stage that checks routes and clusters for structural and semantic correctness, and refuses to swap in an invalid snapshot. This is a guardrail around the atomic-swap mechanism of §3: only *valid* snapshots get swapped in.

4. **Filtering / transformation of config.** Sometimes you want to programmatically adjust config as it loads — inject defaults, rewrite values, merge sources. A **config filter** hook lets you intercept and modify the config on its way through.

The deep payoff is **hot reload**: because adoption is just an atomic swap of a validated, immutable snapshot (§3), configuration can change while the server runs with zero downtime and zero dropped requests. The reload pipeline is: *source changes → change token fires → new config read → validated → filtered → new immutable model built → atomic swap → in-flight requests drain on the old snapshot, new requests use the new one.*

### The Implementation (conceptually, in YARP)

YARP defines a config-provider interface that returns the current config object together with a change token. It ships an **in-memory** provider (you push config from code — ideal when your own system manages topology) and supports a file/`IConfiguration`-based provider (declarative config that hot-reloads on edit). A config-validator component checks routes and clusters before adoption; a config-filter interface lets you transform config as it loads; and change listeners let other components react to reloads. All of these are interfaces — the same "swap in your own implementation" seam as everywhere else — so binding YARP to a bespoke control plane is a first-class, supported path rather than a hack.

---

## 5. Observability: Telemetry, Metrics, Tracing, and Logging

### The "Why"

A proxy sits at the most consequential point in your network — *every* request passes through it. That makes it the ideal vantage point for understanding system behavior, and it makes its *own* health and performance critical to observe. When something is slow or failing, "where did the time go, and which hop failed?" must be answerable. **Observability** is the set of concepts for making the system's internal state externally visible.

### The Theory

Observability rests on three classic **pillars**, plus a cross-cutting concern:

- **Logs** — discrete, timestamped event records ("route X matched," "destination Y marked unhealthy"). High detail, high volume, good for *what happened* in a specific case.

- **Metrics** — numeric aggregates over time (requests per second, error rate, p99 latency, active connection count). Cheap to collect and store, good for *trends, alerting, and dashboards*. The relevant idea is **structured, low-overhead counters and histograms** that can be scraped by a monitoring system.

- **Traces** — the path of a *single request* across multiple services, broken into timed **spans**. This is what lets you see "the request spent 3 ms in the proxy, 200 ms in the backend, 190 ms of which was the database." For traces to span services, each hop must propagate a shared trace identity.

- **Distributed-tracing context propagation.** This is the subtle, important one for a proxy. A trace only stays connected across the proxy→backend hop if the proxy *forwards the trace context*. The modern standard is **W3C Trace Context** (the `traceparent`/`tracestate` headers), which carry a trace ID and span ID. A correct proxy must propagate (and correctly extend) these headers so the backend's spans attach to the same trace. Doing this *wrong* — dropping or mangling the context — silently breaks end-to-end tracing across your whole platform, which is why YARP has dedicated propagation logic.

A second platform-specific concept: **`EventSource` and `EventCounter`** are .NET's built-in, extremely low-overhead structured tracing/metrics mechanism. They let YARP emit fine-grained events (per forwarding stage, per WebSocket) that cost almost nothing when no one is listening but can be subscribed to by an in-process consumer or an external tool. This is the foundation of YARP's telemetry, and it integrates with **OpenTelemetry**, the vendor-neutral standard for collecting and exporting traces and metrics.

### The Implementation (conceptually, in YARP)

YARP instruments its pipeline with event sources that emit the forwarding **stages** (so you can see exactly where a request spent time or failed) and dedicated WebSocket telemetry (since upgraded connections need different measurements than request/response). A telemetry-consumption library lets you subscribe to these events in-process and turn them into metrics or traces. A propagator component carries distributed-tracing context across the proxy→backend hop so traces stay connected. The error path classifies failures into specific categories (request timeout, backend unreachable, client disconnected, body copy failed, …) and exposes them as a structured error feature, so logs and traces can say precisely *what* went wrong rather than "it failed." All of this is why a YARP node can be both deeply observable and observably cheap to observe.

---

## 6. Security at the Front Door

### The "Why"

As the public entry point to your infrastructure, the reverse proxy is your **primary security boundary**. It is where encryption is terminated, where untrusted input first arrives, and where many attacks are best stopped. Several concepts converge here.

### The Theory

- **TLS termination.** Clients connect over **HTTPS** (HTTP over TLS) for confidentiality and integrity. Decrypting (terminating) TLS at the proxy lets it read the request to route and transform it, and centralizes certificate management at one tier instead of every backend. The proxy may then talk to backends over plain HTTP (inside a trusted network) or re-encrypt (**TLS re-origination** / end-to-end TLS) for zero-trust environments. Understanding *where* TLS is terminated and whether the internal hop is encrypted is a core architectural decision the proxy embodies.

- **Mutual TLS (mTLS) and client certificates.** Normal TLS authenticates the *server* to the client. **mTLS** additionally authenticates the *client* to the server via a client certificate — common for service-to-service and high-security scenarios. When the proxy terminates a client-certificate-authenticated connection, the backend can no longer see that certificate directly, so the proxy can forward it (e.g., as a header) — a specific transform exists for exactly this. (This is the same "the backend lost sight of the original client" problem as forwarded headers, applied to identity.)

- **The forwarded-header trust boundary.** As stressed in Part 2, headers like `X-Forwarded-For` are security-sensitive: trusting a client-supplied one lets an attacker forge their apparent IP, defeat IP allow-lists, or poison logs. The proxy must decide which inbound forwarded headers to trust (only from known upstream proxies) and overwrite the rest. This is one of the most common real-world proxy misconfigurations, and YARP's per-action transforms exist to get it right.

- **Authorization and CORS at the edge.** Stopping unauthorized or cross-origin requests at the proxy (Part 2, §7) prevents them from ever reaching — and loading — backends. The ordering guarantee (these run *before* forwarding) is itself a security property.

- **Defense against resource-exhaustion attacks.** Limits on concurrency, request rate, body size, and timeouts (Part 2, §7) are not just performance tools; they are protections against denial-of-service. A proxy without them can be trivially overwhelmed.

### The Implementation (conceptually, in YARP)

Security in YARP is mostly about *correctly composing* the platform's robust primitives: Kestrel/HTTP.sys handle TLS and client-certificate negotiation; ASP.NET Core authorization and CORS policies are referenced per route; the data-protection system underpins encrypted session-affinity cookies (Part 2); the forwarded-header and client-certificate transforms manage the trust boundary and identity forwarding; and the limits/rate-limiting/timeout middleware bound resource use. YARP's contribution is placing these correctly in the pipeline and giving each route fine-grained, declarative control — so security is configured, auditable, and consistent rather than ad hoc.

---

## 7. Running at Scale: The Kubernetes Ingress Controller

### The "Why"

In a **Kubernetes** cluster, services and their backing instances (**pods**) are created, destroyed, and rescheduled constantly, and the cluster's desired routing is described declaratively in Kubernetes objects (like **Ingress** resources). Something must continuously translate "what Kubernetes says the routing should be" into "what the proxy is actually configured to do." YARP ships a **Kubernetes ingress controller** to be that something — a heavyweight, push-based form of the service discovery idea from Part 2.

### The Theory

This introduces one of the most important patterns in modern infrastructure: the **controller / reconciliation loop**, the foundation of Kubernetes itself.

- **Declarative desired state vs. observed state.** You declare *what you want* (these routes, these services exposed). The controller's job is to observe *what currently is* and continuously drive the actual state toward the desired state. It never assumes a one-shot apply; it **reconciles** repeatedly, because reality drifts (pods die, specs change).

- **The reconcile loop.** A controller **watches** the relevant Kubernetes objects, and whenever anything changes it recomputes the desired proxy configuration and applies it. The loop is *level-triggered*, not *edge-triggered*: it acts on the current total state, not on individual change events, so a missed or duplicated event cannot leave it permanently wrong. This idempotent, self-healing property is why the pattern is so robust.

- **Informers, caches, and work queues.** Watching the Kubernetes API directly for every object would overwhelm it. The standard machinery is an **informer** that maintains a **local cache** of the watched objects (kept current via a watch stream) and feeds changes into a **work queue**. Workers pull from the queue and reconcile. The queue provides **rate limiting** and **deduplication** (collapsing a flurry of changes into one reconcile) so the controller stays efficient and the API server stays healthy. These exact concepts — caches, queues, rate limiters, informers — are why the controller's structure mirrors the broader Kubernetes ecosystem.

- **Custom resources (CRDs).** Beyond standard Ingress objects, Kubernetes lets you define your own object types (**Custom Resource Definitions**) to express richer, YARP-specific routing intent. The controller watches these too.

### The Implementation (conceptually, in YARP)

The controller component watches Kubernetes Ingress (and related) resources through a cached, watch-based client, feeds changes into rate-limited work queues, and reconciles by **converting** the observed Kubernetes state into YARP's route/cluster/destination configuration. That generated configuration is then handed to a running YARP proxy through the same config-provider mechanism from §4 — so the proxy adopts it via the validated, atomic, hot-reload path of §3. In other words, the controller is "just another config source," and everything you learned about immutable snapshots and zero-downtime reload applies unchanged. This is the architecture paying off: a wildly dynamic source plugs into the same seam as a static file.

---

## 8. Two Advanced Topics Worth Knowing Exist

These are specialized but conceptually illuminating; you do not need mastery, only awareness of the idea.

**Tunneling.** Sometimes a backend lives on-premises behind a firewall that blocks inbound connections, but you want to reach it from the cloud. YARP's **tunnel** feature runs two cooperating proxies: an on-prem (back-end) proxy makes an *outbound* WebSocket connection to a cloud (front-end) proxy — outbound is allowed where inbound is blocked — and that long-lived connection is then used as a transport over which **HTTP/2 multiplexes** many proxied requests back to the on-prem services. Conceptually it inverts who-dials-whom to defeat the firewall, and reuses HTTP/2 multiplexing (Part 1) to carry many logical requests over one physical tunnel. The established tunnel appears to the front-end as a dynamically created **destination** of a cluster — so, once again, it reuses the core abstractions rather than inventing new ones.

**HTTP.sys request delegation (Windows).** On Windows, the **HTTP.sys** kernel-mode server supports *handing off* a request to another process **at the kernel level**, so the actual response bypasses the proxy entirely on the return path. This is an extreme performance optimization for specific Windows scenarios: instead of streaming the response back through YARP, YARP delegates the connection and steps out of the data path. It is niche, but it illustrates a general principle worth absorbing — the fastest proxy work is the work you can arrange *not* to do.

---

## 9. Mental Sandbox & Next Steps

You now have the full conceptual map. These closing challenges are deliberately open-ended — they are the kind of design reasoning that real YARP work demands.

1. **Defend the lock-free model.** Explain to a skeptical colleague who "just wants to put a lock around the config" why YARP uses immutable snapshots and atomic reference swaps instead. Cover: read-side cost, writer/reader contention, and — most importantly — the *consistency* guarantee for a request that spans a config change. Then identify the one category of state that genuinely *can't* be immutable (hint: §3) and explain how YARP handles it without locks.

2. **Trace a slow request.** A user reports a request that takes 4 seconds. Using the observability concepts from §5, lay out exactly how you would localize the delay: which signals (forwarding stages, traces, metrics, error feature) would distinguish "slow backend" from "exhausted connection pool" from "the proxy is GC-pausing" from "TLS handshake storm." Note where W3C trace-context propagation is essential to even see the backend's portion.

3. **Add a new config source.** Suppose your company has a custom service registry with a streaming change API. Sketch — conceptually, no code — how you would feed it into YARP. Which interface from §4 do you implement? How does your source signal changes? Where does validation happen? At what exact moment does a new destination become eligible for traffic, and which mechanism from §3 guarantees no in-flight request is disrupted when it does? If you can answer this cleanly, you understand how the Kubernetes controller (§7), the DNS resolver (Part 2), and the file provider are all the *same idea* wearing different clothes — and you are ready to open the source with confidence.

---

### Where to go from here

You have covered: the nature of a reverse proxy and its protocols (Part 1); routing, load balancing, health, affinity, discovery, and transforms (Part 2); and streaming, connection pooling, lock-free configuration, observability, security, and Kubernetes operation (Part 3). When you open the code, anchor yourself with the three core abstractions (Route, Cluster, Destination) and the request lifecycle, and remember that almost every type you meet is either an *immutable snapshot*, a *pluggable interface (a seam)*, or a *middleware stage in the pipeline*. Those three shapes account for the overwhelming majority of the design. Welcome to the project — you are ready.
