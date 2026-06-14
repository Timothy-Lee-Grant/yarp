# YARP Project Components — Part 02: The Data Plane (the request pipeline)

> The data plane is the half of `Yarp.ReverseProxy` that *executes* the configuration the control plane (Part 01) produced. It runs on **every single request**, must be fast and allocation-light, and only ever **reads** the immutable runtime model. This document follows a live request through the pipeline, component by component, in the order the request actually encounters them, then covers the cross-cutting `Utilities/`.
>
> Each component is a folder in `src/ReverseProxy/`. We covered the *concepts* (load-balancing theory, health checking, transforms, etc.) in the Foundations Traffic-Management doc; here we map those concepts onto the **actual components and how they connect**.

---

## 1. The Pipeline, Drawn Once

ASP.NET Core processes requests through an ordered **middleware pipeline** (Foundations Part 1 §7.2). YARP inserts its own inner pipeline. Here is the order a request flows through, with each stage's owning folder:

```
   Kestrel parses request
        │
        ▼
  ┌──────────────────┐
  │ Endpoint Routing │  Routing/   ── match request → route → cluster
  └────────┬─────────┘
           ▼
  ┌──────────────────────────────┐
  │ ProxyPipelineInitializer      │  Model/ (Part 01) ── capture snapshot into the request feature
  └────────┬─────────────────────┘
           ▼
  ┌──────────────────┐
  │ Session Affinity │  SessionAffinity/  ── re-pin returning clients
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ Load Balancing   │  LoadBalancing/  ── pick ONE destination …
  └────────┬─────────┘            … from the HEALTH-filtered, DISCOVERY-resolved set
           │                         (Health/ + ServiceDiscovery/ feed this)
           ▼
  ┌──────────────────┐
  │ Passive Health   │  Health/  ── observe outcome of the forward (wraps the forward)
  │  + Limits        │  Limits/
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ Forwarder        │  Forwarder/  ── transform request → stream to backend → stream response back
  │  (+ Transforms)  │  Transforms/
  └──────────────────┘
```

The exact ordering is configurable and some stages only appear when their feature is enabled, but this is the canonical flow. Read each section below as "the request is now here."

---

## 2. `Routing/` — Matching the Request to a Route

### What it does and why

The first decision: *which route (and therefore cluster) does this request belong to?* YARP does **not** build its own matching engine — it plugs into ASP.NET Core's high-performance **endpoint routing** (a prefix-tree matcher), and extends it with the dimensions a proxy needs beyond path. This reuse is a deliberate architecture choice: get the platform's battle-tested, fast path matching for free, add only what's missing.

### The components

| File | Role |
| --- | --- |
| `ProxyEndpointFactory` | Turns each `RouteConfig` (Part 01) into an ASP.NET Core **endpoint** with the right matchers attached. Called by `ProxyConfigManager` when building/refreshing config. |
| `HeaderMatcherPolicy` + `HeaderMatcher` + `IHeaderMetadata`/`HeaderMetadata` | A custom **matcher policy** that lets routing accept/reject candidates based on request **headers** (with the match modes from Part 01) |
| `QueryParameterMatcherPolicy` + `QueryParameterMatcher` + `IQueryParameterMetadata`/`QueryParameterMetadata` | Same idea for **query parameters** |
| `ReverseProxyIEndpointRouteBuilderExtensions` + `ReverseProxyConventionBuilder` | The `MapReverseProxy()` API and the fluent conventions to customize the proxy pipeline |
| `DirectForwardingIEndpointRouteBuilderExtensions` | API for "direct forwarding" — mapping a single endpoint straight to the forwarder, bypassing the full route/cluster model (covered in §8) |

**The matcher-policy concept** is the thing to learn here. ASP.NET Core routing matches in two phases: build candidate endpoints by path, then let registered `MatcherPolicy` objects vote candidates in or out using other criteria. By implementing header and query matcher policies, YARP makes those dimensions first-class participants in the *same fast pass* as path matching, rather than a slow post-filter. The `*Metadata` types are how a route declares "I require header X" so the policy knows to evaluate it.

> **Output of this stage:** the request is now bound to a route → cluster. The `ProxyPipelineInitializerMiddleware` (Part 01, `Model/`) captures that into the per-request feature, establishing the consistent snapshot the rest of the pipeline reads.

---

## 3. `SessionAffinity/` — Sticky Sessions

### What it does and why

Some backends hold per-user in-memory state, so a returning client should go back to the *same* destination. Affinity runs *before* load balancing: if the request already carries an affinity marker, affinity pins the destination and load balancing is effectively skipped; if not, the request flows on and, after a destination is chosen, affinity stamps a marker for next time. (Concept: Foundations Traffic-Management §4.)

### The components, grouped by responsibility

| Concern | Files |
| --- | --- |
| **Pipeline stage** | `SessionAffinityMiddleware` — checks for an existing affinity, resolves or defers, and orchestrates the policies |
| **"How do we recognize the client?" (policies)** | `ISessionAffinityPolicy` + `CookieSessionAffinityPolicy`, `HashCookieSessionAffinityPolicy`, `ArrCookieSessionAffinityPolicy`, `CustomHeaderSessionAffinityPolicy` |
| **"Keep the marker tamper-proof" (base classes)** | `BaseEncryptedSessionAffinityPolicy`, `BaseHashCookieSessionAffinityPolicy` — share the protect/unprotect logic (uses ASP.NET Core **data protection**) |
| **Stamping the marker on responses** | `AffinitizeTransform` + `AffinitizeTransformProvider` — a transform (see §7) that writes the affinity cookie/header onto the response |
| **What to do when the pinned destination is gone (failure policies)** | `IAffinityFailurePolicy` + `RedistributeAffinityFailurePolicy` (re-balance to a healthy one) and `Return503ErrorAffinityFailurePolicy` (refuse) |
| **Result types / helpers** | `AffinityResult`, `AffinityStatus`, `AffinityHelpers`, `SessionAffinityConstants`, `Log` |

The structure literally encodes the three design questions from the concept doc: *recognize* (the policies), *protect* (the base classes), *handle failure* (the failure policies). The different cookie policies exist because real deployments need different cookie semantics — `ArrCookie` mimics the format Azure App Service's ARR affinity uses, for interop; `HashCookie` stores a hash rather than an encrypted blob; the encrypted base protects the binding from inspection/forgery.

---

## 4. `LoadBalancing/` — Choosing One Destination

### What it does and why

Once the cluster and the set of *available* destinations are known, load balancing picks exactly one. (Theory and policy trade-offs: Foundations Traffic-Management §2.) Every policy is an `ILoadBalancingPolicy`; the cluster names which one to use; the middleware applies it.

### The components

| File | Role |
| --- | --- |
| `LoadBalancingMiddleware` | The pipeline stage: reads the available destinations from the request feature, asks the configured policy to pick one, writes the choice back to the feature |
| `ILoadBalancingPolicy` | The seam — implement this for a custom policy |
| `RoundRobinLoadBalancingPolicy` | Cycles through destinations in order |
| `RandomLoadBalancingPolicy` | Uniform random pick (stateless) |
| `LeastRequestsLoadBalancingPolicy` | Picks the fewest in-flight requests (load-aware) |
| `PowerOfTwoChoicesLoadBalancingPolicy` | Samples two at random, picks the less loaded (the scalable default) |
| `FirstLoadBalancingPolicy` | Always the first available (active/passive failover) |
| `LoadBalancingPolicies` | Constants for the built-in policy names |
| `AppBuilderLoadBalancingExtensions` | Registration helpers |

**How it reads load:** the load-aware policies (`LeastRequests`, `PowerOfTwoChoices`) need each destination's live in-flight count. That counter lives on `DestinationState` as an `AtomicCounter` (`Utilities/`, §10), incremented when a request starts forwarding and decremented when it completes — a lock-free atomic, never a lock. This is the concrete link between the load-balancing theory and the concurrency primitives.

---

## 5. `Health/` — Keeping Destinations Healthy

### What it does and why

Load balancing must only choose among **healthy** destinations. `Health/` continuously assesses each destination via two complementary mechanisms — **active** probing and **passive** observation — and decides the *available* set the load balancer sees. (Theory: Foundations Traffic-Management §3, including the circuit-breaker analogy and "panic mode.")

### The components, grouped

| Group | Files | Role |
| --- | --- | --- |
| **Active probing** | `ActiveHealthCheckMonitor` (+`.Log`), `IActiveHealthCheckMonitor`, `ActiveHealthCheckMonitorOptions`, `DefaultProbingRequestFactory` / `IProbingRequestFactory`, `DestinationProbingResult` | A background monitor sends probe requests on a schedule and records the outcomes |
| **Active policy** | `IActiveHealthCheckPolicy`, `ConsecutiveFailuresHealthPolicy` (+`Options`) | Decide health from probe results (default: N failures in a row → unhealthy) |
| **Passive observation** | `PassiveHealthCheckMiddleware`, `IPassiveHealthCheckPolicy`, `TransportFailureRateHealthPolicy` (+`Options`) | A middleware wraps the forward, watches real outcomes, trips on failure *rate* in a window |
| **Applying results** | `DestinationHealthUpdater`, `IDestinationHealthUpdater`, `NewActiveDestinationHealth` | Combine active+passive into each destination's health (writes new immutable `DestinationHealth` and swaps it) |
| **Scheduling** | `EntityActionScheduler` | A reusable timer that fires per-cluster actions (probes, reactivations) on intervals |
| **Choosing the usable set** | `IAvailableDestinationsPolicy`, `HealthyAndUnknownDestinationsPolicy`, `HealthyOrPanicDestinationsPolicy`, `ClusterDestinationsUpdater` / `IClusterDestinationsUpdater` | Turn per-destination health into the cluster's *available* destination list the LB reads |
| **Constants** | `HealthCheckConstants` | Well-known policy names (`ConsecutiveFailures`, `TransportFailureRate`, `HealthyAndUnknown`, `HealthyOrPanic`) |

**The interaction worth tracing:** active and passive are independent inputs. `DestinationHealthUpdater` merges them into a `DestinationHealth` value (which has the three states Healthy/Unhealthy/Unknown). `ClusterDestinationsUpdater` then runs the `IAvailableDestinationsPolicy` to produce the `ClusterDestinationsState` (the dynamic slice from Part 01) that load balancing consumes. The `HealthyOrPanic` policy is the production safety net: if filtering leaves zero destinations, send to all rather than fail everyone. `EntityActionScheduler` is a nice reusable piece — a generic "do this action per entity on a recurring interval" timer, used both for probing and for passively-failed destinations' reactivation timers.

---

## 6. `ServiceDiscovery/` — Resolving Destinations Dynamically

### What it does and why

A `DestinationConfig` may name a *host* rather than fixed addresses, and the real instances behind that host change as the fleet scales. `ServiceDiscovery/` resolves abstract destinations into concrete addresses, refreshing over time. (Concept: Foundations Traffic-Management §5.)

### The components

| File | Role |
| --- | --- |
| `IDestinationResolver` | The seam — "given configured destinations, return the resolved concrete set (and a change signal)" |
| `DnsDestinationResolver` (+ `DnsDestinationResolverOptions`) | Built-in resolver that turns host names into IP addresses via DNS, on an interval |
| `ResolvedDestinationCollection` | The result type — the resolved addresses plus a change token |
| `NoOpDestinationResolver` | The default when discovery is off — passes destinations through unchanged |

The resolver feeds the **control plane**: when resolution produces a new set, it triggers a model update so the new destinations flow into health checking and load balancing exactly like statically-listed ones. This is why discovery sits at the boundary — it's a *source of dynamic destinations*, and the Kubernetes controller (Part 04) is a heavyweight, push-based sibling of this same idea.

---

## 7. `Transforms/` — Rewriting Requests and Responses

### What it does and why

The inbound request is rarely a byte-for-byte copy of the outbound one: headers must be added/removed/rewritten, paths and queries adjusted, the original client's identity re-injected via forwarded headers. `Transforms/` is the general request/response rewriting mechanism — a mini-pipeline inside the forward step. (Concept and the forwarded-header trust boundary: Foundations Traffic-Management §6.)

### The structure: factories build, transforms execute

This folder has two layers, and seeing the split makes it readable:

**`Transforms/Builder/`** — the *assembly* layer. It reads declarative/fluent transform config and produces the concrete transform objects for each route, composed into an ordered pipeline. This is where the `ITransformFactory` / `ITransformProvider` seams live and where the per-route transform pipeline is built once at config time (not per request).

**`Transforms/*` (the root)** — the *execution* layer: the concrete transforms, grouped by what they touch.

| Category | Representative files | What they do |
| --- | --- | --- |
| **Request headers** | `RequestHeaderTransform`, `RequestHeaderValueTransform`, `RequestHeaderRemoveTransform`, `RequestHeadersAllowedTransform`, `RequestHeaderRouteValueTransform` | Add/set/remove/allow-list request headers |
| **Forwarded headers** | `RequestHeaderXForwardedForTransform`, `…XForwardedHostTransform`, `…XForwardedProtoTransform`, `…XForwardedPrefixTransform`, `RequestHeaderForwardedTransform`, `ForwardedTransformActions/Extensions/Factory`, `NodeFormat` | Inject `X-Forwarded-*` / `Forwarded` with the correct append/set/off **action** (the trust boundary) |
| **Original host / client cert** | `RequestHeaderOriginalHostTransform`, `RequestHeaderClientCertTransform` | Preserve original `Host`; forward the client TLS cert to the backend |
| **Path** | `PathStringTransform`, `PathRouteValuesTransform`, `PathTransformFactory/Extensions` | Prefix removal, pattern rewrites, route-value substitution |
| **Query** | `QueryParameterFromStaticTransform`, `QueryParameterFromRouteTransform`, `QueryParameterRemoveTransform`, `QueryTransformContext`, `QueryTransformFactory/Extensions` | Add/remove query parameters |
| **HTTP method** | `HttpMethodChangeTransform`, `HttpMethodTransformFactory/Extensions` | Change the request method |
| **Response** | `ResponseHeaderValueTransform`, `ResponseHeaderRemoveTransform`, `ResponseCondition`, `ResponseFuncTransform` | Modify response headers, optionally conditionally |
| **Escape hatches** | `RequestFuncTransform`, `ResponseFuncTransform` | Run *your own* lambda as a transform |
| **Context + base** | `RequestTransform`, `RequestTransformContext`, `RequestTransformer`, `HttpTransformer` | Base classes and the per-request context the transforms mutate |

**The two-layer (factory/transform) design** matters: parsing config and validating transform syntax happens **once** at config time in the builder; the per-request work is just running the already-built transform objects against the request context. This keeps the hot path cheap — a recurring theme. `HttpTransformer` is the base type the forwarder calls; the default implementation already does correct header copying (respecting hop-by-hop vs end-to-end, Foundations Part 1 §5.2), and your transforms layer on top.

---

## 8. `Forwarder/` — The Hot Core

### What it does and why

This is the component that does the actual proxying: take the (transformed) request, send it to the chosen destination over a pooled HTTP client, and stream the response back — all without buffering bodies in memory. It is the most performance-critical folder in the repository. (Concepts: streaming/backpressure, connection pooling — Foundations Part 3 §1–2.)

### The components, grouped

| Group | Files | Role |
| --- | --- | --- |
| **The engine** | `HttpForwarder`, `IHttpForwarder`, `IHttpForwarderExtensions` | The core algorithm: build outbound request, apply transforms, send, copy bodies both ways, classify the result |
| **Pipeline stage** | `ForwarderMiddleware` | The middleware that invokes the forwarder for routed requests using the cluster's destination/client |
| **Outbound HTTP client** | `IForwarderHttpClientFactory`, `ForwarderHttpClientFactory`, `ForwarderHttpClientContext`, `CallbackHttpClientFactory`, `DirectForwardingHttpClientProvider` | Create/configure the per-cluster `HttpClient` (`SocketsHttpHandler`) — the connection pool |
| **Streaming** | `StreamCopier`, `StreamCopyHttpContent`, `StreamCopyResult`, `EmptyHttpContent`, `DelegatingStream` (in Utilities) | Pump bytes in fixed-size pooled buffers with backpressure; full-duplex for upgrades |
| **Request shaping** | `RequestTransformer`, `HttpTransformer`, `RequestUtilities`, `ProtocolHelper`, `ForwarderRequestConfig` | Translate inbound→outbound request semantics, protocol/version handling |
| **Error model** | `ForwarderError`, `ForwarderErrorFeature`, `IForwarderErrorFeature` | A precise classification of *what* failed (timeout, connect failure, body copy error, client disconnect, …) |
| **Telemetry** | `ForwarderTelemetry`, `ForwarderStage` | `EventSource` instrumentation of each forwarding stage (consumed by Part 03) |
| **Tracing propagation** | `ReverseProxyPropagator` | Carries W3C distributed-trace context across the proxy→backend hop (Foundations Part 3 §5) |

**The two ways to use the forwarder.** YARP supports a full mode (route → cluster → LB → health → forward, everything in this series) and a **direct forwarding** mode where you map a single endpoint straight to `IHttpForwarder.SendAsync(...)` with your own destination and transformer — no clusters, no load balancing. Direct forwarding (exposed via `Routing/DirectForwardingIEndpointRouteBuilderExtensions`) is for when you want YARP's fast, correct HTTP-forwarding engine but your app already decides where to send each request. Knowing both modes exist clarifies why the forwarder is a self-contained, independently-usable engine and not welded to the rest of the pipeline.

**The error feature** is a small but production-critical design: instead of failures collapsing into a generic exception, every failure is tagged with a specific `ForwarderError`, exposed on the request via `IForwarderErrorFeature`. This is what lets logs, traces, and **passive health** (§5) react precisely — e.g., "connection refused" should affect health differently than "client disconnected mid-response."

---

## 9. The Two Specialized Stages: `Limits/` and `Delegation/`

### `Limits/`

| File | Role |
| --- | --- |
| `LimitsMiddleware` | Applies request-level limits (e.g., max request body size) for the matched route/cluster before forwarding |

Small but important: it's the per-route enforcement point that protects the proxy and backends from oversized/abusive requests (the resource-exhaustion defense from Foundations Part 3 §6). Rate limiting and timeouts themselves reuse ASP.NET Core's middleware, referenced by the policy-name constants in `Configuration/`; `Limits/` covers what the platform doesn't.

### `Delegation/`

| File | Role |
| --- | --- |
| `HttpSysDelegator`, `IHttpSysDelegator`, `HttpSysDelegatorMiddleware` | On Windows with the **HTTP.sys** server, hand a request off to another process **at the kernel level**, so the response bypasses YARP entirely |
| `DelegationExtensions`, `AppBuilderDelegationExtensions` | Registration/wiring |
| `DummyHttpSysDelegator` | A no-op used on non-Windows platforms so the rest of the code can depend on the interface unconditionally |

This is a niche, Windows-only, extreme optimization (Foundations Part 3 §8): the fastest forwarding is forwarding you can avoid by delegating the OS connection. The `Dummy` implementation is a nice example of the **null-object pattern** — providing a do-nothing implementation so platform differences don't leak into the rest of the codebase.

---

## 10. `WebSocketsTelemetry/` — Instrumenting Upgraded Connections

### What it does and why

WebSocket (and other upgraded) connections break the request/response model — they're long-lived, bidirectional byte streams (Foundations Part 1 §5.4). Normal request metrics don't describe them, so this folder adds dedicated instrumentation: how long the connection lived, how much data flowed each way, why it closed.

| File | Role |
| --- | --- |
| `WebSocketsTelemetryMiddleware` | Wraps upgraded connections to observe them |
| `WebSocketsTelemetryStream` | A stream wrapper that counts bytes flowing in each direction |
| `WebSocketsParser`, `WebSocketCloseReason` | Understand WebSocket framing enough to detect close and reason |
| `HttpUpgradeFeatureWrapper`, `HttpConnectFeatureWrapper` | Intercept the upgrade/CONNECT handshake to insert the instrumentation |
| `WebSocketsTelemetry`, `WebSocketsTelemetryExtensions` | The `EventSource` + wiring |

The pattern here — **wrapping a stream to measure it without changing behavior** (the decorator pattern) — is worth noting; it's how you add observability to a byte pump without slowing it down or altering semantics.

---

## 11. `Utilities/` — The Cross-Cutting Toolbox

These aren't a pipeline stage; they're the low-level primitives the whole data plane is built from. As an embedded engineer you'll find these familiar in spirit — they're about doing things without allocating and without locking.

| File | What it is / why it matters |
| --- | --- |
| `AtomicCounter` | Lock-free integer counter (e.g., per-destination in-flight count for load balancing). The concrete concurrency primitive behind §4. |
| `ActivityCancellationTokenSource` | A pooled/reusable cancellation source tying request timeouts to .NET `Activity` tracing |
| `IClock` / `ValueStopwatch` | Abstract time + an allocation-free stopwatch (a `struct`), so timing code is testable and cheap |
| `IRandomFactory` / `RandomFactory` / `NullRandomFactory` | Abstracted randomness (for Random/P2C load balancing) so it's deterministic in tests |
| `ValueStringBuilder` | A stack-based, allocation-light string builder for hot-path string assembly |
| `ParsedMetadataEntry` | Cached parsing of route/cluster metadata so repeated reads don't re-parse |
| `TlsFrameHelper` | Parses TLS handshake frames (e.g., to read SNI) without a full TLS stack |
| `*EqualHelper`, `CollectionEqualityHelper`, `ConcurrentDictionaryExtensions` | Efficient equality/diffing used when the control plane compares old vs. new config to reuse unchanged objects |
| `DelegatingStream` | Base for stream wrappers (used by the forwarder/WebSocket instrumentation) |
| `Observability`, `EventIds` | Centralized logging/activity ids |
| `TaskUtilities`, `SkipLocalsInit` | Async helpers; a compiler hint to skip zeroing locals for speed |

**The recurring theme across `Utilities/`:** avoid allocations (lots of `struct`/`Value*` types), avoid locks (atomics), and abstract anything non-deterministic (clock, random) so the hot path is both fast and testable. This folder is where YARP's performance obsession is most visible at the primitive level. The use of `struct` types like `ValueStopwatch` and `ValueStringBuilder` is specifically to keep them on the stack and out of the garbage collector's way — directly relevant to your performance-optimization goals.

---

## 12. How the Data-Plane Components Interact (the full trace)

```
 Request ─▶ Routing/ matches route+cluster
          ─▶ ProxyPipelineInitializer captures snapshot into IReverseProxyFeature  [Model/]
          ─▶ SessionAffinity/ : returning client? pin destination, else continue
          ─▶ LoadBalancing/ : pick 1 destination from the available set …
                 ▲ available set produced by Health/ (ClusterDestinationsUpdater)
                 ▲ destinations resolved by ServiceDiscovery/ (e.g. DNS)
                 ▲ in-flight counts read via AtomicCounter [Utilities/]
          ─▶ Limits/ : enforce request limits
          ─▶ PassiveHealth middleware wraps the forward to observe outcome  [Health/]
          ─▶ Forwarder/ : Transforms/ rewrite request ▶ StreamCopier pumps body to backend
                          ▶ backend responds ▶ StreamCopier pumps response back
                          ▶ Transforms/ rewrite response ▶ SessionAffinity stamps marker
          ─▶ outcome classified (ForwarderError) → feeds PassiveHealth + telemetry
 Response ◀ back to client
```

Every arrow is a real handoff between folders, and every folder reads the same consistent snapshot captured at the top. That is the data plane: **a chain of single-purpose, swappable middleware stages, each reading an immutable snapshot, cooperating to route → select → transform → forward a request as fast as possible.**

> **Interview relevance.** Being able to narrate this trace — naming the stages, why each exists, and where the lock-free reads happen — is a strong answer to "design an API gateway / reverse proxy" and to "how would you structure a high-throughput request pipeline?" The factory-vs-execution split in `Transforms/`, the null-object pattern in `Delegation/`, and the decorator pattern in `WebSocketsTelemetry/` are reusable design-pattern talking points.

Next: **Part 03 — TelemetryConsumption**, the observability project that subscribes to the `EventSource` streams these components emit.
