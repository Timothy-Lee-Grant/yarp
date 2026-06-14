# YARP Concepts — Part 1: Foundations

> **How to read this series.** These three documents teach the *ideas* behind YARP (Yet Another Reverse Proxy), not its code or folder layout. The goal is that when you eventually open the source, every term, pattern, and design decision already feels familiar. Part 1 builds the bedrock: what a reverse proxy *is*, the network and protocol concepts it lives on top of, and the hosting model it is built from. Part 2 covers traffic management and resilience. Part 3 covers performance, concurrency, and operations.
>
> **A note on tailoring.** No `persona.md` was found in this repository, so this document assumes you have a solid general programming background (you can read code, understand functions and data structures) but may be newer to distributed systems, networking internals, and the ASP.NET Core platform. If you create a `persona.md` describing your exact background and goals, a regenerated version of this document can target your specific blind spots far more precisely.

---

## 1. Executive Overview: What YARP Actually Is

YARP is a **toolkit for building reverse proxies** in .NET. That phrasing matters. It is not a finished product you configure and forget; it is a *library* you compose into your own application, giving you a proxy whose every behavior you can intercept, replace, or extend in code.

A **reverse proxy** is a server that sits in front of one or more backend servers and forwards client requests to them. To the outside world it *looks like* the real server — clients connect to it, not to the backends. It receives each incoming HTTP request, decides which backend should handle it, forwards the request, receives the backend's response, and relays that response back to the client. Everything YARP does — routing, load balancing, health checking, header rewriting, TLS termination — is in service of doing that one loop well, fast, and flexibly.

The reason a whole project exists for this is that "forward the request to a backend" hides an enormous amount of difficulty once you care about **performance** (tens of thousands of requests per second per node), **correctness** (HTTP is a deceptively subtle protocol), **resilience** (backends fail, networks partition), and **dynamism** (the set of backends changes constantly as services scale up and down). YARP's central design bet is that no single fixed feature set can satisfy every team, so it exposes its internal pipeline as extension points rather than locking behavior behind configuration flags.

---

## 2. Your Personal Mindset Shift

If your background is mostly writing application logic — functions that take input, compute, and return output — a proxy will stretch you in three specific directions, and it is worth naming them up front because they recur throughout the codebase.

**From "process the data" to "move the bytes."** In ordinary application code you own the data; you parse it, transform it, and emit it. A proxy frequently must move gigabytes of request and response bodies *without ever materializing them in memory*. The mental model shifts from "load it, work on it, save it" to "open two pipes and let bytes flow between them." This is the difference between thinking in **values** and thinking in **streams**, and it dominates the performance-critical paths.

**From "call a function" to "traverse a pipeline."** A request in YARP is not handled by one method. It passes through an ordered chain of independent components (middleware), each of which can inspect it, modify it, short-circuit it, or pass it along. You will stop asking "what function handles this request?" and start asking "what does the pipeline do to this request, in what order?"

**From "the world is static" to "the world changes under me."** The list of backends, their health, and the routing rules can all change *while requests are in flight*. Code that assumes a fixed configuration is wrong here. A recurring theme — explored deeply in Part 3 — is how YARP lets configuration change continuously without locks, race conditions, or stalls. If you have only ever worked with single-threaded or request-scoped state, this is the largest stretch, and it is genuinely beautiful once it clicks.

Keep these three shifts in mind. Almost every concept below is an instance of one of them.

---

## 3. Forward Proxy vs. Reverse Proxy

These two terms are confusingly similar and worth pinning down precisely, because YARP is firmly in the second camp.

A **forward proxy** acts on behalf of *clients*. Your corporate network might route all outbound web traffic through a forward proxy that enforces policy ("no social media"), caches popular content, and hides individual machines behind one IP. The client knows it is using a proxy; the destination servers generally do not.

A **reverse proxy** acts on behalf of *servers*. Clients think they are talking to the real service, but they are actually talking to the proxy, which fans their requests out to a pool of backends behind it. The backends are hidden; the client is generally unaware a proxy exists at all.

```
Forward proxy:   [many clients] → [proxy] → the whole internet
Reverse proxy:   the whole internet → [proxy] → [your backend pool]
```

The direction of "who is being represented" is the whole distinction. A reverse proxy is the gatekeeper *in front of your infrastructure*. That single position lets it perform an outsized set of jobs: it is the natural place to terminate TLS, balance load, enforce rate limits, check backend health, rewrite URLs, add security headers, and present a unified front door to a sprawl of internal services. Understanding *why* the reverse proxy is the right home for each of these duties is half of understanding YARP.

---

## 4. Layer 4 vs. Layer 7: Where YARP Operates

Networking is described in **layers** (the OSI model). Two matter intensely for proxies.

**Layer 4 (the transport layer)** deals with raw connections — TCP and UDP. A Layer 4 (L4) load balancer or proxy forwards *bytes* between a client connection and a backend connection without understanding what those bytes mean. It can see IP addresses and ports but not URLs, headers, or cookies. It is extremely fast and protocol-agnostic, but blind to application meaning.

**Layer 7 (the application layer)** deals with the actual protocol — for the web, that is HTTP. A Layer 7 (L7) proxy *parses* each request: it reads the method, the path, the headers, the host. Because it understands the request, it can make intelligent decisions ("requests for `/api/billing` go to the billing service; requests for `/images` go to the CDN tier") and modify requests and responses.

**YARP is a Layer 7 reverse proxy.** This is its defining characteristic. Everything it offers — path-based routing, header matching, transforms, session affinity — is only possible because it parses and understands HTTP. The cost of L7 is that parsing takes work, which is exactly why YARP obsesses over doing that work efficiently. When you read the source and see careful avoidance of string allocations and re-parsing, remember: it is paying the L7 tax as cheaply as possible.

---

## 5. HTTP: The Protocol YARP Speaks

Because YARP lives at Layer 7, you cannot fully understand it without understanding HTTP. Here are the conceptual pieces that the codebase assumes you know.

### 5.1 The request/response model

HTTP is a **request/response** protocol. A client sends a request — a **method** (GET, POST, PUT, DELETE, …), a **target** (the path and query string, e.g. `/api/orders?id=5`), a set of **headers** (key/value metadata like `Host`, `Authorization`, `Content-Type`), and optionally a **body** (the payload, e.g. JSON or an uploaded file). The server replies with a response — a **status code** (200 OK, 404 Not Found, 503 Service Unavailable, …), its own headers, and optionally a body.

A reverse proxy handles *two* request/response exchanges per proxied call, joined at the hip: the **inbound** exchange between client and proxy, and the **outbound** exchange between proxy and backend. Much of YARP's subtlety is in deciding how the inbound request is translated into the outbound one, and how the backend's response is translated back. That translation is the job of **transforms** (Part 2).

### 5.2 Headers, and why they are dangerous

Headers carry the metadata that makes proxying tricky. Some headers describe the *connection* (like `Connection`, `Keep-Alive`, `Transfer-Encoding`) and must **not** be blindly copied from one hop to the next, because they describe the client↔proxy hop, not the proxy↔backend hop. These are called **hop-by-hop headers**, as opposed to **end-to-end headers** that should be preserved. A correct proxy must know the difference. Getting this wrong causes some of the most baffling bugs in all of networking, and a large amount of YARP's request-handling code exists precisely to handle headers correctly.

A second header subtlety: the backend often needs to know facts about the *original* client that are otherwise lost once the proxy is in the middle — the client's real IP address, the original `Host` it asked for, whether the original connection used HTTPS. These are communicated with **forwarded headers** (`X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Proto`, and the standardized `Forwarded` header). Managing them correctly — and *not* trusting spoofed ones from untrusted clients — is a recurring security theme.

### 5.3 The protocol versions: HTTP/1.1, HTTP/2, HTTP/3

HTTP has evolved, and a proxy must straddle multiple versions because the client side and the backend side can each speak a different one.

**HTTP/1.1** is text-based and uses one request per connection at a time (with keep-alive reusing the connection for sequential requests). Its limitation is **head-of-line blocking**: a slow response holds up the connection.

**HTTP/2** introduces **multiplexing** — many concurrent requests (called **streams**) share a single TCP connection, interleaved as binary **frames**. This is a major efficiency gain and is why HTTP/2 appears repeatedly in YARP's internals (for example, the tunneling feature multiplexes many proxied requests over one HTTP/2 connection between two proxies). HTTP/2 still suffers TCP-level head-of-line blocking because all streams share one ordered TCP byte stream.

**HTTP/3** moves off TCP entirely onto **QUIC**, a protocol built on UDP that provides independent streams so a lost packet on one stream does not stall the others. It also folds TLS in by default.

The conceptual takeaway: a proxy is a **protocol translation point**. A client might connect over HTTP/3, while the backend only speaks HTTP/1.1; YARP terminates one and originates the other. You do not need to memorize frame formats, but you must hold the idea that "the version on the way in need not match the version on the way out," because it explains why YARP configures protocol versions per cluster and why so much care goes into translating semantics rather than bytes.

### 5.4 WebSockets and other upgraded/streaming protocols

Normal HTTP is one request, one response. But some interactions are **long-lived and bidirectional**. **WebSockets** start as an ordinary HTTP request carrying an `Upgrade` header, then "upgrade" the connection into a raw two-way channel that stays open, over which both sides send messages whenever they like (chat, live dashboards, multiplayer games). **gRPC** and **Server-Sent Events** are other long-lived streaming patterns built on HTTP.

For a proxy these are special because the "request then response" assumption breaks: after the upgrade, there is just a duplex stream of bytes to be shuttled in both directions until one side closes. YARP must detect the upgrade, stop treating the exchange as request/response, and switch into a **byte-pumping** mode that copies data both ways simultaneously. This is a direct instance of the "move the bytes, don't process them" mindset shift, and YARP has dedicated machinery (and even dedicated telemetry) for WebSocket traffic.

---

## 6. The Reverse Proxy Pipeline (Conceptual Lifecycle)

Now we assemble the pieces into the **life of a request** through YARP. This is the single most important mental model in the entire system. Read it slowly; everything in Parts 2 and 3 hangs off one of these stages.

1. **Acceptance.** A client connection arrives at the web server (Kestrel — see §7). The raw bytes are parsed into a structured HTTP request.

2. **Routing / matching.** YARP examines the request — its path, host, headers, method, query — and matches it against the configured **routes** to decide *which route* applies. A route identifies a logical destination: a **cluster**. (Part 2, §Routing.)

3. **Affinity resolution.** If **session affinity** is enabled, YARP checks whether this client was previously "stuck" to a particular backend (e.g., via a cookie) and, if so, steers it back there. (Part 2.)

4. **Destination selection / load balancing.** The chosen cluster has a pool of **destinations** (concrete backend addresses). A **load-balancing policy** picks exactly one healthy destination to receive this request. (Part 2.)

5. **Health filtering.** Only destinations currently considered **healthy** are eligible. Health is continuously assessed by active probes and by observing real traffic outcomes. (Part 2.)

6. **Request transformation.** The inbound request is transformed into the outbound request: headers added/removed/rewritten, path and query adjusted, forwarded headers stamped on. (Part 2.)

7. **Forwarding.** The transformed request is sent to the chosen destination using a pooled HTTP client. The backend's response streams back. Request and response bodies are copied as streams, not buffered. (Part 3.)

8. **Response transformation.** The backend's response is transformed on its way back to the client (e.g., stripping or adding response headers).

9. **Observation.** Throughout, the outcome (success, latency, failure type) is recorded — feeding metrics, traces, logs, and passive health assessment.

The elegance is that each numbered stage is a **replaceable component**. You can swap the load-balancing policy, inject a custom transform, provide your own health policy, or hook the forwarding step — without rewriting the rest. That composability is the product.

---

## 7. The Platform Underneath: ASP.NET Core

YARP is not built from scratch. It stands on **ASP.NET Core**, Microsoft's web framework, and reuses its highest-performance machinery. You will understand YARP much faster if you understand the four platform concepts it leans on most.

### 7.1 Kestrel: the web server

**Kestrel** is ASP.NET Core's built-in, cross-platform, high-performance HTTP server. It does the low-level work of accepting TCP connections, speaking HTTP/1.1, HTTP/2, and HTTP/3, doing TLS, and turning raw bytes into structured request objects. YARP delegates all of that to Kestrel and concentrates on the proxy logic above it. When you read about YARP "receiving a request," it is Kestrel that physically did the receiving. (An alternative server, **HTTP.sys**, is used on Windows for certain advanced features like request delegation — Part 3.)

### 7.2 The middleware pipeline

ASP.NET Core processes every request through a **middleware pipeline**: an ordered sequence of components, each a function that receives the request context and a reference to "the next component." Each middleware may do work before calling the next, after it returns, both, or neither (short-circuiting the pipeline entirely). Conceptually it is like a set of nested wrappers — often pictured as an onion — where the request travels inward through each layer and the response travels back outward through the same layers in reverse.

```
request  →  [ Auth ]→[ RateLimit ]→[ Routing ]→[ LoadBalance ]→[ Forward ]
response ←  [ Auth ]←[ RateLimit ]←[ Routing ]←[ LoadBalance ]←[ Forward ]
```

This is the concrete realization of the "traverse a pipeline" mindset. YARP's stages from §6 — load balancing, session affinity, health, the final forwarder — are each implemented as middleware. The **order** of middleware is semantically critical: authentication must run before forwarding; rate limiting must run before you commit a backend to the work. Much of configuring YARP is really about arranging this pipeline correctly.

### 7.3 Endpoint routing

ASP.NET Core has a built-in **endpoint routing** system: a two-phase mechanism where one early middleware *matches* the request to an endpoint (using an efficient matching structure over all registered routes), and a later middleware *executes* that endpoint. The matching supports constraints and custom **matcher policies** that can accept or reject candidate endpoints based on headers, query parameters, and so on. YARP plugs its own route table and custom matcher policies into this system rather than reinventing request matching — so YARP routes get the same fast, well-tested matching engine that the rest of ASP.NET Core uses. (Details in Part 2.)

### 7.4 Dependency injection and the host

ASP.NET Core is built around **dependency injection (DI)**: components declare the services they need (by interface), and a central **container** supplies concrete implementations at runtime. This is *the* mechanism by which YARP achieves its "customize anything" goal. Nearly every behavior — the config provider, the load-balancing policies, the health policies, the transform factories, the HTTP client factory — is registered in the DI container behind an interface. To change a behavior, you register your own implementation; YARP picks it up automatically. When you see interfaces like `I…Policy`, `I…Provider`, `I…Factory` throughout the source, understand them as **seams**: deliberate places where the platform lets you substitute your code for the default.

The **host** and the **builder** are the startup-time objects that wire all this together: you describe the services and the pipeline, and the host runs the resulting application. The recurring `AddReverseProxy()` and `MapReverseProxy()` style calls you will see are just the registration (services) and pipeline-placement (middleware/endpoints) steps of this host model.

---

## 8. The Three Core Abstractions: Routes, Clusters, Destinations

If you remember nothing else, remember these three nouns. They are the vocabulary of the entire system, and every configuration file, every model class, and every diagram is organized around them.

A **Route** is a *rule for matching incoming requests*. It says, in effect, "requests that look like *this* (this path pattern, optionally these hosts/headers/methods) belong to *that* cluster." Routes are about the **client-facing** side: what the outside world asks for. A route also carries an **order** (for resolving overlaps) and can attach metadata, authorization requirements, CORS policy, rate-limiter and timeout selections, and a chain of transforms.

A **Cluster** is a *named group of interchangeable backends* that can all serve the same logical service, plus the **policies** for how to treat that group: which load-balancing policy to use, how to health-check it, whether to apply session affinity, what HTTP version and client settings to use when talking to it. A cluster is the **service-facing** abstraction: "the orders service" might be one cluster.

A **Destination** is a *single concrete backend address* within a cluster — one URL, one server instance. Clusters contain many destinations; load balancing chooses among them; health checking marks individual ones up or down; service discovery adds and removes them as the backend fleet scales.

```
Route  ──matches──▶  Cluster  ──contains──▶  Destination
"/api/* → orders"    "orders"               https://10.0.0.5:443
                                            https://10.0.0.6:443
                                            https://10.0.0.7:443
```

The relationship is layered exactly as the request lifecycle uses it: routing picks the **route** (hence the cluster), and load balancing + health picks the **destination**. Almost every feature in Parts 2 and 3 is "a smarter way to do one of those two selections," or "a way to keep these three things accurate as the world changes."

---

## 9. Configuration as a First-Class, Living Concept

A final foundational idea, because it pervades the design. YARP treats **configuration** — the set of routes, clusters, and destinations and all their options — as **data that lives and changes at runtime**, not as static startup settings.

Two consequences follow, and both are deliberate design goals stated in YARP's own README. First, configuration can come from **anywhere**: a JSON file, a database, a service registry, a Kubernetes API, or your own management system. YARP abstracts "where config comes from" behind a provider interface so you can supply your own source. Second, configuration can be **swapped while the server runs** — new routes appear, destinations come and go, health-check intervals change — *without restarting and without dropping in-flight requests*. The README explicitly frames programmatic, in-process, hot-reloadable configuration as the primary scenario, not an afterthought.

Making "the world changes under me" safe and cheap is one of YARP's deepest engineering achievements, built on immutable snapshots and atomic pointer swaps. That machinery is the centerpiece of Part 3. For now, simply internalize the principle: **in YARP, configuration is a stream of immutable snapshots, not a fixed object.**

---

## 10. Mental Sandbox

Test your conceptual grip before moving on. You do not need to write code — reason through these.

1. **The header trap.** A client sends a request with `Connection: keep-alive` and `Authorization: Bearer abc`. As the proxy author, which of these two headers should be forwarded to the backend and which should not, and *why*? (Hint: hop-by-hop vs. end-to-end, §5.2.) What real-world bug occurs if you get it backwards?

2. **Protocol mismatch.** A client connects over HTTP/3, but a particular cluster's destinations only speak HTTP/1.1. At which stage of the request lifecycle (§6) does this matter, and what does it mean to say the proxy "terminates one protocol and originates another"? What information might be lost in translation, and how would forwarded headers help recover it?

3. **Drawing the model.** Sketch, from memory, the Route → Cluster → Destination relationship for a system with two services (`web` and `api`), where `api` has three backend instances and `web` has one. Which abstraction does load balancing operate on? Which does routing operate on? Where would a health check mark something unhealthy?

When those three feel obvious, you are ready for **Part 2: Traffic Management & Resilience**, where each selection step — routing, load balancing, health, affinity — is opened up in depth.
