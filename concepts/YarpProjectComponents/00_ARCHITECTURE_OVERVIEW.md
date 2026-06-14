# YARP Project Components — Part 00: Architecture Overview

> **Who this is written for.** You, Timothy — an embedded/firmware engineer (C/C++/Python/C#) moving deliberately toward backend, distributed systems, and cloud infrastructure. You read code well but want the *map before the streets*: overall architecture → major components → interactions → control flow, the way a senior engineer onboards a new teammate. This series is exactly that. We will not dwell on individual lines of code; we will explain **what every component and subcomponent is, why it exists, what problem it solves, and how the pieces talk to each other.**
>
> **Prerequisite.** The three files in `concepts/` (`01_FOUNDATIONS`, `02_TRAFFIC_MANAGEMENT`, `03_PERFORMANCE_AND_OPERATIONS`) taught the *concepts* (reverse proxy, L7, load balancing theory, immutable-snapshot concurrency, etc.). This new series (`YarpProjectComponents/`) maps those concepts onto the **actual code organization**. Where a concept was already taught, this series points back to it instead of re-teaching it.

---

## 1. The One-Paragraph Mental Model

YARP is shipped as a **library** (plus a ready-to-run app and a few satellite projects). You add it to an ASP.NET Core application, and it turns that application into a reverse proxy. Internally it has two halves that you should learn to see separately: a **control plane** that ingests configuration (routes, clusters, destinations) from some source, validates it, and turns it into an immutable in-memory model; and a **data plane** — a pipeline of middleware — that uses that model to route, balance, health-check, transform, and forward each live request. Around that core sit three companion projects: a **telemetry consumption** library for observability, a **Kubernetes controller** that generates config from a cluster's state, and a **sample/host application**. That is the entire repository in one breath. The rest of this series zooms into each piece.

---

## 2. The Four Source Projects (and how they relate)

Everything in `src/` is one of four .NET projects. This is the highest-level decomposition, and internalizing it makes the whole repo legible.

| Project (folder) | NuGet/assembly identity | Role in one line | Depends on |
| --- | --- | --- | --- |
| **`src/ReverseProxy`** | `Yarp.ReverseProxy` | **The product.** The actual reverse proxy library: config model, runtime model, and the request pipeline. | ASP.NET Core platform |
| **`src/TelemetryConsumption`** | `Yarp.Telemetry.Consumption` | **Observability add-on.** Subscribes to low-level .NET + YARP event streams and turns them into metrics/callbacks. | ReverseProxy (loosely), .NET diagnostics |
| **`src/Kubernetes.Controller`** | `Yarp.Kubernetes.Controller` | **A config *source*.** Watches Kubernetes and generates YARP config from Ingress resources. | ReverseProxy, Kubernetes client |
| **`src/Application`** | `Yarp.Application` | **A ready-to-run host.** A pre-built proxy app/template wiring the library together with sensible defaults. | ReverseProxy (+ others) |

The dependency direction is the key insight: **`ReverseProxy` is the hub, and the other three are spokes.** TelemetryConsumption observes it, Kubernetes.Controller feeds it config, and Application hosts it. None of the spokes depend on each other. If you ever feel lost, return to this picture.

```
                 ┌───────────────────────────┐
                 │   Yarp.ReverseProxy        │   ◀── THE CORE LIBRARY
                 │   (control + data plane)   │       (the "product")
                 └───────────────────────────┘
                    ▲          ▲          ▲
        observes it │  feeds   │  hosts   │
                    │  config  │  it      │
   ┌────────────────┴──┐  ┌────┴───────┐  ┌┴───────────────────┐
   │ Telemetry.        │  │ Kubernetes │  │ Yarp.Application   │
   │ Consumption       │  │ .Controller│  │ (host / template)  │
   └───────────────────┘  └────────────┘  └────────────────────┘
```

---

## 3. Control Plane vs. Data Plane — the most useful lens

This distinction comes from networking and is *the* organizing idea for reading YARP's core. Learn it once and the folder list inside `src/ReverseProxy` stops looking like a random pile.

**The control plane** decides *what the configuration is*. It answers: which routes exist? which clusters? which destinations? what policies apply? It runs occasionally — whenever configuration changes — and it is allowed to be relatively slow and careful, because correctness matters more than nanoseconds here. Its output is an **immutable model** of the proxy's intended behavior.

**The data plane** *executes* that configuration on every single request. It answers: for *this* request, which route, which destination, which transforms, then forward the bytes. It runs millions of times and must be brutally fast and allocation-light. It only *reads* the model the control plane produced — it never blocks waiting for the control plane to finish an update (that's the immutable-snapshot trick from the Foundations series, Part 3 §3).

```
            CONFIG SOURCE (file / code / K8s)
                       │
   ┌───────────────────▼────────────────────┐
   │              CONTROL PLANE              │   runs on change; careful & validating
   │  Configuration → Validation → Model     │
   │            (ProxyConfigManager)         │
   └───────────────────┬────────────────────┘
                       │ produces immutable snapshot, swapped atomically
   ┌───────────────────▼────────────────────┐
   │               DATA PLANE                │   runs per request; fast & lock-free
   │  Route ▶ Affinity ▶ LoadBalance ▶       │
   │  Health ▶ Transform ▶ Forward           │
   └─────────────────────────────────────────┘
```

In this series, **Part 01 covers the control plane** (`Configuration`, `Model`, `Management`) and **Part 02 covers the data plane** (`Routing`, `LoadBalancing`, `Health`, `SessionAffinity`, `ServiceDiscovery`, `Forwarder`, `Transforms`, `Limits`, `Delegation`, `WebSocketsTelemetry`, `Utilities`).

---

## 4. Inside `Yarp.ReverseProxy`: the subcomponent map

The core library has ~15 subfolders. Here is the full inventory, each tagged by plane and mapped to the document that covers it in depth. Skim this table now; it is the table of contents for Parts 01–02.

| Subfolder | Plane | What it is | Covered in |
| --- | --- | --- | --- |
| `Configuration/` | Control | The *declarative shape* of config (RouteConfig, ClusterConfig, DestinationConfig) + provider/validator/filter interfaces | Part 01 |
| `Configuration/ConfigProvider/` | Control | The built-in file/`IConfiguration`-based config source + snapshots | Part 01 |
| `Configuration/RouteValidators/` | Control | Pluggable checks that a route is well-formed before adoption | Part 01 |
| `Configuration/ClusterValidators/` | Control | Pluggable checks that a cluster is well-formed before adoption | Part 01 |
| `Model/` | Control→Data bridge | The *runtime* model: immutable RouteState/ClusterState/DestinationState + atomic holders | Part 01 |
| `Management/` | Control | **The orchestrator** (`ProxyConfigManager`) that ties source→validate→model→endpoints together | Part 01 |
| `Routing/` | Data | Custom matcher policies (header, query) + endpoint construction | Part 02 |
| `LoadBalancing/` | Data | The load-balancing policies + the middleware that applies one | Part 02 |
| `Health/` | Data | Active + passive health checking, availability policies | Part 02 |
| `SessionAffinity/` | Data | Sticky-session policies, affinity failure handling | Part 02 |
| `ServiceDiscovery/` | Data | Destination resolution (e.g., DNS) into concrete addresses | Part 02 |
| `Forwarder/` | Data | **The hot core**: the actual HTTP forwarding, stream copying, HTTP client factory | Part 02 |
| `Transforms/` | Data | Request/response rewriting; `Transforms/Builder/` assembles them | Part 02 |
| `Limits/` | Data | Request-level limits middleware (concurrency/size) | Part 02 |
| `Delegation/` | Data | Windows HTTP.sys kernel-level request hand-off | Part 02 |
| `WebSocketsTelemetry/` | Data | Instrumentation specifically for upgraded WebSocket connections | Part 02 |
| `Utilities/` | Cross-cutting | Low-level helpers: atomic counters, clocks, value types, TLS frame parsing | Part 02 |

A few orientation notes you'll appreciate as an embedded engineer used to reading unfamiliar trees:

- The **`I`-prefixed interfaces** scattered everywhere (`IProxyConfigProvider`, `ILoadBalancingPolicy`, `IDestinationResolver`, …) are the **extensibility seams**. In .NET, `I` means "interface." YARP's whole "customize anything" philosophy is implemented as: define behavior behind an interface, register a default in the DI container, let you replace it. When you see an `I...`, read it as "a place I'm allowed to plug in my own implementation."
- Files named **`*Middleware.cs`** are data-plane pipeline stages. Files named **`*Policy.cs`** are swappable strategies. Files named **`*Extensions.cs`** are the *registration* API (the `AddX()` / `UseX()` methods you call at startup). Files named **`*Config.cs`** are control-plane declarative data; **`*State.cs` / `*Model.cs`** are runtime data.

---

## 5. The companion projects at a glance

We give each its own document, but here is the elevator pitch for each so the overview is complete.

**`Yarp.Telemetry.Consumption` (Part 03).** YARP and the .NET runtime emit a firehose of structured diagnostic events via `EventSource` (sockets opened, DNS resolved, TLS handshakes, request stages, WebSocket lifecycle). This project is a *consumer* of that firehose: it provides `EventListenerService`s that subscribe to each event source and surface the data either as **metrics** (periodic aggregate numbers) or via **consumer interfaces** you implement to receive per-event callbacks. Its subfolders mirror the *areas* of telemetry: `Forwarder/`, `Http/`, `Kestrel/`, `NameResolution/` (DNS), `NetSecurity/` (TLS), `Sockets/`, `WebSockets/`. It is optional — pure observability — but it is the canonical example of the **"instrument with near-zero-cost events, subscribe out of band"** pattern that the Foundations series Part 3 §5 described.

**`Yarp.Kubernetes.Controller` (Part 04).** This is the most intricate companion, and the one whose subfolders mystified you. It is a **Kubernetes controller**: a background process that watches the Kubernetes API for `Ingress` (and related) resources and continuously translates the cluster's desired routing into YARP config. Its subfolders are the standard anatomy of *any* serious controller — `Client/` (the watch/informer machinery), `Caching/` (a local mirror of watched objects), `Queues/` + `Rate/` (rate-limited work queues), `Services/` (the reconcile loop), `Converters/` (Ingress→YARP translation), `Protocol/` + `Dispatcher` (pushing generated config to proxy instances), `ConfigProvider/` (the seam into ReverseProxy's control plane), `Certificates/` (TLS for the controller's own endpoints), and `Hosting/` (running it all as background services). Part 04 explains each as an instance of the controller/reconciler pattern — directly relevant to your Kubernetes and distributed-systems goals.

**`Yarp.Application` (Part 05).** A pre-assembled proxy application/host that wires the library together with config binding and a few extra features (static files, fallback, logging). Think of it as "YARP with the batteries included," useful as a template and as the thing packaged for container images. Part 05 also covers the **supporting cast** that isn't a runtime component but is essential to understanding a real enterprise repo: `src/Common/`, the `samples/` (each demonstrates one feature), the `test/` and `testassets/` projects (unit vs. functional tests, and the mock servers they need), and the `eng/` + build scripts + Azure pipelines (the CI/CD machinery — on your learning list).

---

## 6. The repository top level (so nothing looks mysterious)

Beyond `src/`, the root has the scaffolding of a large, professionally maintained .NET open-source project. You do not need to master these, but knowing what each *is* removes the "what is all this?" friction.

| Top-level folder/file | What it is | Why it's there |
| --- | --- | --- |
| `src/` | The four shipping projects | The actual product |
| `test/` | Unit + functional test projects | Correctness; mirrors `src/` |
| `testassets/` | Helper apps used *by* tests (mock backends, clients) | Tests need real servers to proxy to |
| `samples/` | Small runnable demos, one feature each | Documentation-by-example |
| `docs/` | Design docs (tunneling, config), operations runbooks | Design rationale & maintainer ops |
| `eng/` | Build engineering: versions, signing, the Arcade SDK `common/` scripts | Reproducible enterprise builds |
| `azure-pipelines*.yml`, `*.ps1`, `*.sh` | CI/CD pipeline + bootstrap scripts | Automated build/test/release |
| `*.props`, `*.targets`, `global.json`, `NuGet.config` | MSBuild + SDK + package configuration | How .NET builds the solution |
| `YARP.slnx` | The solution file listing all projects | The "table of contents" for the IDE |

The recurring `Directory.Build.props` / `Directory.Build.targets` files are a .NET convention: settings that **automatically apply to every project in that folder and below**, so common configuration (language version, analyzers, signing) is written once. This is the .NET equivalent of a shared CMake include in your embedded world.

---

## 7. End-to-end control flow (the 60-second tour)

To anchor everything, here is the life of the system from startup to serving a request, naming the components you'll meet:

1. **Startup / registration.** The host app calls `AddReverseProxy()` (from `Management/ReverseProxyServiceCollectionExtensions`) to register all services in the DI container, then `LoadFromConfig(...)` (or supplies a custom `IProxyConfigProvider`) to choose a config source, then `MapReverseProxy()` to place the proxy's middleware pipeline and endpoints into the ASP.NET Core request pipeline.

2. **Config load (control plane).** The `ProxyConfigManager` (`Management/`) pulls the current config from the provider(s), runs it through the **validators** (`Configuration/RouteValidators`, `ClusterValidators`) and any **config filters**, and builds the immutable **runtime model** (`Model/`). It registers each route as an ASP.NET Core **endpoint** (using `Routing/`).

3. **Config changes (control plane).** When the source signals a change (file edited, K8s controller pushes new config, code mutates in-memory config), the manager rebuilds the model and **atomically swaps** it. In-flight requests keep using their snapshot; new requests pick up the new one. No locks on the request path.

4. **Request arrives (data plane).** Kestrel parses it. Endpoint **routing** matches it to a route/cluster. Then the middleware chain runs: **session affinity** → **load balancing** (over **health**-filtered, **service-discovery**-resolved destinations) → **transforms** → **forwarder**. The forwarder streams the request to the chosen destination via a pooled HTTP client and streams the response back, applying response transforms.

5. **Observation (cross-cutting).** Throughout, `EventSource`s fire; if `Yarp.Telemetry.Consumption` is wired up, those become metrics and callbacks. Failures get classified and feed passive health.

Every noun in that tour is a folder in `src/`. The rest of this series is just zooming into each one.

---

## 8. How to read the rest of this series

| Document | Covers | Why read it in this order |
| --- | --- | --- |
| **00 (this file)** | The map: 4 projects, 2 planes, the subcomponent inventory | Orientation first |
| **01 — Control Plane** | `Configuration`, `Model`, `Management` | You must understand the model before the pipeline that reads it |
| **02 — Data Plane** | `Routing` → `Forwarder`, `Transforms`, `Limits`, `Delegation`, `WebSocketsTelemetry`, `Utilities` | The request lifecycle, component by component |
| **03 — TelemetryConsumption** | The observability project and its 7 area subfolders | Cross-cutting; safe to read anytime after 02 |
| **04 — Kubernetes.Controller** | The controller and all its subfolders | A self-contained distributed-systems case study |
| **05 — Application & Infra** | `Application`, `Common`, `samples`, `test`, `eng` | The "everything else," ties the repo together |

By the end you will be able to point at any folder in `src/` and say what it does, why it exists, which plane it belongs to, and what it talks to. That is precisely the senior-engineer onboarding fluency your persona is aiming for.

> **Interview relevance (a recurring section in this series).** "Walk me through the architecture of a reverse proxy / API gateway" is a staple system-design question. The control-plane/data-plane split, the immutable-snapshot config model, and the controller/reconciler pattern in Part 04 are all directly reusable talking points. We'll flag these as we go.
