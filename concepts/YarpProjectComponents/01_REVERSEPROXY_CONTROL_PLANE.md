# YARP Project Components — Part 01: The Control Plane (`Configuration`, `Model`, `Management`)

> The control plane is the half of `Yarp.ReverseProxy` that decides **what the configuration is**. It ingests config from a source, validates it, and turns it into an immutable in-memory model that the data plane (Part 02) reads on every request. This document walks the three subcomponents that make up the control plane — `Configuration/`, `Model/`, and `Management/` — explaining what each is, why it exists, and how they hand off to each other.
>
> Keep the **control plane vs. data plane** lens from Part 00 in mind: everything here runs *occasionally* (on config change), is allowed to be careful and validating, and never sits on the per-request hot path.

---

## 1. The Big Picture: Three Layers of "Configuration"

The single most important thing to understand before reading any file is that YARP has **three distinct representations of configuration**, and they are deliberately separate. Conflating them is the #1 source of confusion when reading the code.

```
   ┌──────────────────────────────────────────────────────────────┐
   │  1. RAW SOURCE        e.g. appsettings.json, code, K8s CRDs    │
   │     "text / external format"                                   │
   └───────────────────────────┬──────────────────────────────────┘
                               │  read by an IProxyConfigProvider
                               ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  2. DECLARATIVE CONFIG (Configuration/)                        │
   │     RouteConfig, ClusterConfig, DestinationConfig — immutable  │
   │     "what the user *asked for*", validated, source-agnostic    │
   └───────────────────────────┬──────────────────────────────────┘
                               │  compiled by ProxyConfigManager (Management/)
                               ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  3. RUNTIME MODEL (Model/)                                     │
   │     RouteState, ClusterState, DestinationState + AtomicHolders │
   │     "what the proxy is *doing right now*", incl. live health   │
   └──────────────────────────────────────────────────────────────┘
```

**Why three layers and not one?** Each layer has a different *rate of change* and a different *audience*, which is a classic separation-of-concerns argument:

- **Layer 1 (raw)** changes format-by-format. Keeping it isolated means the rest of YARP doesn't care whether config came from JSON, a database, or Kubernetes.
- **Layer 2 (declarative `*Config`)** is the *contract*. It is what a user writes and what gets validated. It only changes when the user changes config. It is immutable and comparable so YARP can diff old vs. new and react minimally.
- **Layer 3 (runtime `*State`/`*Model`)** carries things the user never wrote: live destination health, the currently-resolved destination list, dynamic state. It changes in reaction to *runtime events*, not just config edits.

This mirrors exactly the naming convention documented in `Model/README.md`: `*Config` = "the portion that changes only on config reload," `*DynamicState` = "the portion that changes in reaction to runtime state." Hold this three-layer picture; everything below slots into it.

---

## 2. `Configuration/` — The Declarative Contract (Layer 2)

### What problem it solves

A reverse proxy needs a **precise, source-independent description** of routes, clusters, destinations, and all their options — one that can be validated, compared, and built from any source. `Configuration/` is that description plus the *interfaces* that define how config enters and is checked.

### The data types (the "nouns")

These are immutable record-like types. They are the Route/Cluster/Destination abstractions from the Foundations series made concrete.

| Type (`Configuration/*.cs`) | Represents | Notable contents |
| --- | --- | --- |
| `RouteConfig` | One route | route id, the `RouteMatch`, cluster id, order, metadata, transforms, authorization/CORS/rate-limit/timeout policy names |
| `RouteMatch` | The match criteria of a route | path pattern, hosts, methods, `RouteHeader[]`, `RouteQueryParameter[]` |
| `RouteHeader` / `RouteQueryParameter` | A single header/query match rule | name, values, and a **match mode** |
| `HeaderMatchMode` / `QueryParameterMatchMode` | *How* to match | exact, prefix, presence, contains, etc. (enums) |
| `ClusterConfig` | One cluster | cluster id, dictionary of `DestinationConfig`, load-balancing policy name, health-check config, session-affinity config, HTTP client config |
| `DestinationConfig` | One backend address | the address URL, optional health endpoint, host, metadata |
| `HealthCheckConfig` → `ActiveHealthCheckConfig` / `PassiveHealthCheckConfig` | Health policy settings for a cluster | intervals, thresholds, policy names |
| `SessionAffinityConfig` / `SessionAffinityCookieConfig` | Sticky-session settings | mode, policy, cookie attributes |
| `HttpClientConfig` / `WebProxyConfig` | How the cluster's outbound HTTP client behaves | allowed HTTP versions, TLS options, proxy settings |

**Why immutable?** Because the data plane reads these without locks. An immutable object can be shared across thousands of concurrent requests safely — no reader can ever see it half-updated. This is the foundation of the lock-free design (Foundations Part 3 §3). Note also the `*Constants.cs` files (`AuthorizationConstants`, `CorsConstants`, `RateLimitingConstants`, `TimeoutPolicyConstants`) — these hold the well-known string keys used to reference ASP.NET Core policies by name, the integration seam described in the Foundations Traffic-Management doc §7.

### The interfaces (the "verbs" / seams)

This is where YARP's extensibility lives. Each is a place you can plug in your own code.

| Interface | Responsibility | When you'd implement it |
| --- | --- | --- |
| `IProxyConfigProvider` | **The config source.** Returns the current `IProxyConfig` and a *change token* to signal updates. | To load config from a custom source (DB, registry, your control plane) |
| `IProxyConfig` | A single immutable snapshot: the list of routes + clusters + a change token | Returned by a provider; rarely implemented directly |
| `IProxyConfigFilter` | A hook to **programmatically transform** config as it loads (inject defaults, rewrite values) | To mutate config in-flight before adoption |
| `IConfigValidator` | The top-level validation entry point | Usually use the default |
| `IConfigChangeListener` | Be notified when config is loaded/applied/fails | To react to reloads (logging, cache invalidation) |
| `IYarpRateLimiterPolicyProvider` / `IYarpOutputCachePolicyProvider` | Bridges to ASP.NET Core's rate-limiter / output-cache policy lookups | Advanced policy integration |

The two **built-in providers** live right here too:

- `InMemoryConfigProvider` (+ its extensions) — you hold the config in memory and push updates from code. This is the provider you use when *your own system* (or the Kubernetes controller, Part 04) manages topology programmatically. It exposes an `Update(...)` method that swaps the config and fires the change token.
- The file-based provider lives in the `ConfigProvider/` subfolder (next section).

### `ConfigProvider/` — the built-in declarative source

| File | Role |
| --- | --- |
| `ConfigurationConfigProvider` | An `IProxyConfigProvider` that reads routes/clusters from an ASP.NET Core `IConfiguration` section (e.g., `appsettings.json`) and **hot-reloads** when the file changes |
| `ConfigurationReadingExtensions` | The binding logic: turn raw config keys into `RouteConfig`/`ClusterConfig` objects |
| `ConfigurationSnapshot` | An immutable `IProxyConfig` capturing one read of the file + its change token |

**The change-token mechanism** is worth pausing on because it recurs throughout .NET. A *change token* is a tiny object that lets a consumer register a callback to be invoked once when "the thing changed." The file provider watches the file; on edit it produces a *new* snapshot with a *new* token and signals the old one. The `ProxyConfigManager` (§4) is listening, so it re-reads and rebuilds. This is the plumbing that makes **zero-downtime hot reload** possible — the concept from Foundations Part 3 §4, now you can see exactly which files implement it.

### `RouteValidators/` and `ClusterValidators/` — the guardrails

Before any config becomes live, it is **validated**, because a bad config that goes live breaks real traffic. Validation is broken into many small, single-purpose validators, each implementing `IRouteValidator` or `IClusterValidator`. This is the **strategy pattern applied to validation**: each rule is isolated, independently testable, and the set is extensible.

| Route validators | Checks |
| --- | --- |
| `PathValidator` | The path pattern is well-formed |
| `HostValidator` | Host rules are valid |
| `MethodsValidator` | HTTP methods are recognized and non-contradictory |
| `HeadersValidator` / `QueryParametersValidator` | Header/query match rules are coherent |
| `AuthorizationPolicyValidator` / `CorsPolicyValidator` | Referenced auth/CORS policies actually exist |
| `RateLimitPolicyValidator` / `OutputCachePolicyValidator` / `TimeoutPolicyValidator` | Referenced policies exist and are valid |

| Cluster validators | Checks |
| --- | --- |
| `DestinationValidator` | Destination addresses are valid |
| `HealthCheckValidator` | Active/passive health settings are sane (intervals, thresholds) |
| `LoadBalancingValidator` | The named load-balancing policy exists |
| `SessionAffinityValidator` | Affinity settings are coherent |
| `ProxyHttpClientValidator` / `ProxyHttpRequestValidator` | HTTP client/request options are valid |

`ConfigValidator.cs` is the coordinator that runs all the registered validators and aggregates their errors. **Why so granular?** Because validation rules are exactly the kind of thing that grows over time and that third parties want to extend. Isolating each rule means adding one without touching the others — the Open/Closed Principle in practice.

> **Common mistake this design prevents:** adopting a route that points at a nonexistent cluster, or a cluster naming a load-balancing policy that isn't registered. Without pre-adoption validation, these would fail *per request* at runtime, intermittently and confusingly. Validating up front converts a runtime mystery into a startup/reload error with a clear message.

---

## 3. `Model/` — The Runtime Model (Layer 3)

### What problem it solves

The declarative `*Config` objects describe *intent*, but they lack the things that change at runtime: which destinations are currently healthy, the live in-flight request counts, the currently-resolved destination set. `Model/` holds the **runtime state** the data plane actually reads on every request — and it is engineered to be read concurrently, lock-free, while being updated. This folder is the **bridge between the two planes** and the literal embodiment of the immutable-snapshot concurrency pattern.

### The structure: identity vs. config vs. dynamic state

Read `Model/README.md` once — it states the discipline explicitly. Every runtime entity is split into three conceptual parts, following a strict rule: **every member must be immutable, OR an `AtomicHolder<T>` wrapping an immutable `T`, OR a thread-safe primitive like `AtomicCounter`.**

For each of the three abstractions you get a trio:

| Abstraction | "Identity / live" object | "Config slice" object | "Dynamic slice" |
| --- | --- | --- | --- |
| Route | `RouteState` | `RouteModel` (wraps a `RouteConfig` + compiled transforms) | — |
| Cluster | `ClusterState` | `ClusterModel` (wraps a `ClusterConfig` + resolved HTTP client) | `ClusterDestinationsState` (the current usable destination set) |
| Destination | `DestinationState` | `DestinationModel` (wraps a `DestinationConfig`) | `DestinationHealth` / `DestinationHealthState` (live active+passive health) |

The `State` object is the stable handle the data plane holds; inside it, **atomic holders** point at the current config slice and dynamic slice. When config reloads, a new `ClusterModel` is built and the holder is repointed — *atomically*. When new destinations are discovered or health changes, a new `ClusterDestinationsState` / `DestinationHealth` is built and swapped. A request that started a microsecond before the swap keeps using the old slice (consistent snapshot); a request a microsecond after sees the new one. **No locks, no torn reads, no stalls.** This is the single most important engineering idea in the whole project, and `Model/` is where it lives.

```
   ClusterState  (stable handle the pipeline holds)
     ├── Model           : AtomicHolder<ClusterModel>          ◀ swapped on config reload
     ├── DestinationsState: AtomicHolder<ClusterDestinationsState> ◀ swapped on discovery/health change
     └── (per-destination) DestinationState
            ├── Model  : AtomicHolder<DestinationModel>        ◀ config reload
            └── Health : DestinationHealth (active+passive)    ◀ runtime events
```

### The other Model files (the data-plane handoff)

These wire the runtime model into the actual request pipeline:

| File | Role |
| --- | --- |
| `IReverseProxyFeature` / `ReverseProxyFeature` | A per-request "feature" object placed on the HTTP context that carries the chosen route/cluster/destination snapshot for *this* request. This is the consistency anchor — the pipeline reads everything through it. |
| `HttpContextFeaturesExtensions` | Convenience accessors to get the proxy feature off the request |
| `ProxyPipelineInitializerMiddleware` | The first proxy middleware: it captures the matched route/cluster into the feature, establishing the snapshot the rest of the pipeline uses |
| `IReverseProxyApplicationBuilder` / `ReverseProxyApplicationBuilder` | The builder used to assemble the inner proxy middleware pipeline |
| `IClusterChangeListener` | A hook to be notified when clusters are added/changed/removed at runtime |

**The "feature" concept** is an ASP.NET Core idiom worth learning: the `HttpContext` carries a collection of *features* — typed objects attached to the request that middleware use to share state. YARP uses one to pass "here is the route, cluster, and destination snapshot bound to this request" down the pipeline without globals and without re-reading shared state. It is how the data plane gets its consistent snapshot at request start (Foundations Part 3 §3).

---

## 4. `Management/` — The Orchestrator

### What problem it solves

Something has to *coordinate* the whole control plane: pull from the provider(s), run validation and filters, build the runtime model, diff against the previous model, register routes as ASP.NET Core endpoints, perform the atomic swap, and listen for change tokens to do it all again. That coordinator is `Management/`, and its centerpiece is `ProxyConfigManager` — the most important single class in the project.

### `ProxyConfigManager` — the heart of the control plane

Its own summary comment captures its dual nature: it "provides a method to apply Proxy configuration changes" **and** is an implementation of ASP.NET Core's `EndpointDataSource` "that supports being dynamically updated in a thread-safe manner while avoiding locks on the hot path."

Unpack that:

- **As a config applier:** it holds the array of `IProxyConfigProvider`s, and on startup (and on every change-token signal) it: reads each provider's current `IProxyConfig` → runs `IProxyConfigFilter`s → runs the `IConfigValidator` → builds/updates the `Model/` runtime objects (creating new immutable slices and swapping atomic holders) → notifies `IConfigChangeListener`s and `IClusterChangeListener`s.

- **As an `EndpointDataSource`:** ASP.NET Core's routing system consumes *endpoint data sources*. By being one, `ProxyConfigManager` feeds YARP's routes directly into the platform's matching engine (Part 02, `Routing/`). When config changes, it raises the endpoint-source's change signal so routing rebuilds its match table — again, atomically, no hot-path locks. This is how a config edit instantly changes which routes match, with zero downtime.

The `_syncRoot` lock you'll see exists only to serialize *writers* (config updates happen rarely and one at a time); **readers — the request pipeline — never take it.** That is the precise realization of "lock the cold path, never the hot path."

### The rest of `Management/`

| File | Role |
| --- | --- |
| `ReverseProxyServiceCollectionExtensions` | Defines `AddReverseProxy()` — the entry point that registers *every* control-plane and data-plane service in the DI container. This is where all the interface→implementation bindings (the seams) are declared. |
| `IReverseProxyBuilder` / `ReverseProxyBuilder` | The fluent builder returned by `AddReverseProxy()`, onto which you chain `.LoadFromConfig(...)`, `.AddTransforms(...)`, custom policies, etc. |
| `IReverseProxyBuilderExtensions` | The chainable methods (`LoadFromConfig`, `LoadFromMemory`, add-policy helpers) that hang off the builder |
| `IProxyStateLookup` | A read API to query the live model — "give me the cluster with this id," "list all routes." Used by tools, the K8s controller, and tests to introspect current state. |

**Why a builder pattern?** Because configuration of *services* in .NET is fluent and incremental: `AddReverseProxy().LoadFromConfig(...).AddTransforms(...)`. Each call registers more into the DI container. The builder is just a typed handle that makes those additions discoverable and ordered. As an embedded engineer, think of it as a structured init sequence where each method wires one more peripheral.

---

## 5. How the Control Plane Components Interact (control flow)

Putting it together — the life of a configuration change, end to end:

```
 (1) Source changes (file edited / code calls Update() / K8s pushes)
        │  change token fires
        ▼
 (2) ProxyConfigManager wakes up  [Management/]
        │  reads current IProxyConfig from each IProxyConfigProvider
        ▼
 (3) IProxyConfigFilter(s) transform the declarative config  [Configuration/]
        ▼
 (4) IConfigValidator runs all Route/Cluster validators  [Configuration/*Validators/]
        │  invalid?  → reject, keep old model live, raise error to IConfigChangeListener
        │  valid?    ↓
 (5) Build/refresh runtime Model: new immutable slices  [Model/]
        │  diff vs. previous; reuse unchanged objects
        ▼
 (6) Atomically swap AtomicHolders + raise EndpointDataSource change
        │  in-flight requests keep old snapshot; routing rebuilds match table
        ▼
 (7) Notify IClusterChangeListener / IConfigChangeListener
        ▼
 (8) New requests now see the new model.  No locks were taken on the request path.
```

Notice how each numbered step lives in a specific subfolder, and how validation (step 4) is the gate that protects the atomic swap (step 6). This is the control plane's entire job: **turn an untrusted, changing, source-specific description into a validated, immutable, lock-free runtime model — continuously and safely.**

---

## 6. What To Pay Attention To / Production Notes

- **The three layers are not redundant — they have different change rates.** If you only remember one thing: `*Config` = user intent (changes on edit); `*State`/`*Model` = live runtime (changes on health/discovery). The `AtomicHolder` is the hinge between them.
- **Validation is a security and reliability boundary**, not a formality. In production, a malformed config push that bypassed validation could take down every route at once. The validator/atomic-swap design ensures a bad config is *rejected* and the last-good model stays live.
- **The `EndpointDataSource` integration** is the clever bit that earns YARP the platform's high-performance route matching for free, while still supporting hot reload. In an interview, "how do you change routing rules at runtime without dropping requests?" is well answered by: immutable snapshot + atomic reference swap + dynamic endpoint data source.

> **Interview relevance.** This document is essentially a worked example of three classic topics on your list: **separation of concerns** (3-layer config), **lock-free concurrency / snapshot consistency** (the `Model/` discipline), and the **control-plane/data-plane split** (a core distributed-systems architecture pattern). Be able to draw the §1 and §5 diagrams from memory.

Next: **Part 02 — The Data Plane**, where we follow a live request through `Routing` → `SessionAffinity` → `LoadBalancing` → `Health` → `ServiceDiscovery` → `Transforms` → `Forwarder`, plus `Limits`, `Delegation`, `WebSocketsTelemetry`, and the `Utilities` that make it all fast.
