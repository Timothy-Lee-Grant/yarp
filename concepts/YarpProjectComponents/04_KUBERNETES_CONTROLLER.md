# YARP Project Components — Part 04: `Yarp.Kubernetes.Controller`

> This is the most intricate companion project, and the one whose subfolders (Caching, Certificates, Client, Protocol, Queues, Rate, …) looked mysterious. They stop being mysterious the moment you realize they are **the standard anatomy of a Kubernetes controller** — the same pattern Kubernetes itself is built from. This document teaches that pattern first, then walks every subfolder as a part of it.
>
> This is a self-contained distributed-systems case study and is squarely on your learning list (Kubernetes, service discovery, controllers, event-driven coordination). Concept background: Foundations Part 3 §7 (the reconcile loop, informers, work queues, declarative state).

---

## 1. What This Project Is and Why It Exists

In Kubernetes, you don't run servers by hand — you *declare desired state* (objects like `Ingress`, `Service`, `Endpoints`) and the cluster makes reality match. An **`Ingress`** object says, in effect, "expose this HTTP routing to the outside world." But an Ingress is just data; something must *read* those objects and *configure an actual proxy* to implement the routing. That something is an **ingress controller**.

`Yarp.Kubernetes.Controller` is an ingress controller backed by YARP. Its entire job:

> **Continuously watch the Kubernetes API, translate the cluster's `Ingress`/`Service`/`Endpoints` state into YARP routes/clusters/destinations, and push that config into a running YARP proxy — keeping them in sync forever as the cluster changes.**

It is, in YARP's terms, a **dynamic configuration source** (Part 01) — a heavyweight, push-based sibling of the DNS service discovery in Part 02. Everything it produces ultimately flows into `Yarp.ReverseProxy`'s control plane via an `IProxyConfigProvider`.

---

## 2. The Controller Pattern (learn this once, it's everywhere)

Every serious Kubernetes controller — including this one — is built from the same handful of moving parts. Internalize this diagram and the subfolders become self-explanatory.

```
        Kubernetes API server  (source of truth for desired state)
                │   ▲
   watch stream │   │ status updates
                ▼   │
   ┌─────────────────────────────────────────────────────────────┐
   │  INFORMER  (Client/)                                         │
   │   • LIST all objects once, then WATCH for changes            │
   │   • reconnect/resync on disconnect (resourceVersion)         │
   └───────────────┬─────────────────────────────────────────────┘
                   │ add/update/delete events
                   ▼
   ┌───────────────────────────┐      ┌──────────────────────────┐
   │  CACHE  (Caching/)        │◀────▶│  WORK QUEUE (Queues/+Rate/)│
   │  local mirror of objects  │      │  rate-limited, deduplicated│
   └───────────────────────────┘      └─────────────┬─────────────┘
                                                    │ dequeue
                                                    ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  RECONCILER  (Services/)                                     │
   │   • read current cache state                                 │
   │   • CONVERT (Converters/) Ingress→YARP config                │
   │   • push to proxy (ConfigProvider/ + Protocol/)              │
   └─────────────────────────────────────────────────────────────┘
```

Three principles make this robust, and they're worth stating because they're the "why" behind the whole design:

1. **Level-triggered, not edge-triggered.** The reconciler acts on the *current total state* in the cache, not on the individual event that woke it. So a missed, duplicated, or out-of-order event cannot corrupt it — the next reconcile reads the full truth and converges. This is the single most important reliability property of controllers.

2. **Desired vs. observed, converging continuously.** The loop never assumes "done." It repeatedly drives observed state (what the proxy is configured with) toward desired state (what Kubernetes declares). Drift (a pod dies, a spec changes) is automatically corrected on the next pass.

3. **Decoupled via cache + queue.** The fast event stream (informer) is decoupled from the potentially-slow reconcile (which talks to the proxy) by a cache (so reconcile reads a consistent local snapshot) and a queue (so bursts are absorbed, deduplicated, and rate-limited). This is a classic **producer/consumer with backpressure** — directly relevant to your event-driven-systems and message-queue goals.

Now every subfolder is just "which part of this picture am I?"

---

## 3. The Subfolders, Mapped to the Pattern

| Subfolder | Role in the pattern | One-line purpose |
| --- | --- | --- |
| `Client/` | **Informer + API client** | Watch Kubernetes resources; maintain the watch connection |
| `Caching/` | **Cache** | Local in-memory mirror of the watched objects |
| `Queues/` | **Work queue** | Buffer + deduplicate items needing reconciliation |
| `Rate/` | **Rate limiter** | Token-bucket limiter that paces the queue |
| `Services/` | **Reconciler + controller loop** | Dequeue, reconcile, drive convergence |
| `Converters/` | **Translation** | Turn Ingress/Service/Endpoints into YARP config |
| `ConfigProvider/` | **Output seam** | Hand generated config to YARP's control plane |
| `Protocol/` | **Distribution** | Push config from controller to (possibly many) proxy instances |
| `Certificates/` | **TLS support** | Manage certs for the controller's own dispatch endpoint |
| `Hosting/` | **Lifecycle** | Run informers/controller as background services |
| `Management/` | **Composition root** | DI registration + wiring of everything above |
| `Protocol`/`Properties`/`NamespacedName.cs`/`YarpOptions.cs` | misc/support | Shared types, options, assembly metadata |

The rest of this document takes them in roughly data-flow order.

---

## 4. `Client/` — The Informer (the watch machinery)

### What problem it solves

You cannot poll the Kubernetes API ("list all ingresses") every second — it would hammer the API server and be slow. The **informer** pattern solves this: do one initial **LIST** to get everything, then open a long-lived **WATCH** stream that pushes only *changes*. The informer keeps a local cache current and notifies subscribers of add/update/delete events. This is the foundational efficiency trick of the entire Kubernetes ecosystem.

### The components

| File | Role |
| --- | --- |
| `ResourceInformer<TResource, TListResource>` | The generic informer base: LISTs, then WATCHes, tracks `resourceVersion` to resume after disconnects, reconnects on failure, and dispatches change events. Its own doc-comment describes exactly this. Runs as a background service. |
| `IResourceInformer<T>` / `IResourceInformerRegistration` | The seam to subscribe to a resource's change events and manage the subscription lifetime |
| `V1IngressResourceInformer`, `V1ServiceResourceInformer`, `V1EndpointsResourceInformer`, `V1IngressClassResourceInformer`, `V1SecretResourceInformer` | Concrete informers, one per Kubernetes resource type the controller cares about |
| `ResourceSelector<T>` | Narrows *which* objects to watch (e.g., by namespace/label) |
| `GroupApiVersionKind` | Identifies a Kubernetes resource type (its API group/version/kind) |
| `KubernetesClientOptions` | Configuration for the API client |
| `IIngressResourceStatusUpdater` / `V1IngressResourceStatusUpdater` | Writes **status back** to the Ingress object (e.g., "this ingress is being served by load balancer X") — controllers report status, closing the loop |

**Why five separate informers?** Because computing routing requires correlating five resource types: an `Ingress` references `Service`s, a `Service`'s actual pod addresses live in `Endpoints`, `IngressClass` decides whether *this* controller should handle a given ingress, and `Secret`s hold TLS certs. The controller must watch all five and react when *any* of them changes. The `resourceVersion` tracking you'll see in `ResourceInformer` is how Kubernetes watches resume exactly where they left off after a dropped connection — a small but critical correctness detail for not missing or replaying changes.

---

## 5. `Caching/` — The Local Mirror

### What problem it solves

The reconciler needs to read a **consistent, fast, local** view of all relevant objects without calling the API server. The cache is that view, kept current by the informers' change events.

### The components

| File | Role |
| --- | --- |
| `ICache` / `IngressCache` | The cache abstraction and its implementation: stores the current ingresses, services, endpoints, etc., and answers "what does the world look like right now?" |
| `NamespaceCache` | Per-namespace partition of the cache (Kubernetes objects live in namespaces) |
| `IngressData`, `ServiceData`, `Endpoints`, `IngressClassData` | The cached representations of each resource type |

The cache is also where **filtering logic** lives — e.g., "does this ingress's `IngressClass` mean we should handle it?" Only relevant objects are retained. When a reconcile runs, it reads from here, not from Kubernetes — that's what makes reconciles fast and gives them a stable snapshot to work from (echoes of the immutable-snapshot idea from the core library, applied to cluster state).

---

## 6. `Queues/` and `Rate/` — Backpressure and Pacing

### What problem they solve

Changes can arrive in bursts (a deployment rolls 50 pods at once). You don't want 50 reconciles; you want **one** reconcile against the final state, and you don't want to overwhelm the API server or the proxy. The work queue **deduplicates** and the rate limiter **paces**.

### `Queues/`

| File | Role |
| --- | --- |
| `IWorkQueue<T>` / `WorkQueue<T>` | A queue that **collapses duplicates** (adding the same item twice while it's pending is a no-op) and supports delayed re-queue (for retries with backoff) |
| `ProcessingRateLimitedQueue<T>` | A `WorkQueue` wrapped with a rate limiter so items are dequeued no faster than allowed |

**The deduplication property is the key insight.** Because the loop is level-triggered (§2), processing "ingress changed" once after a burst is just as correct as processing it 50 times — and far cheaper. The queue exploits this: many change events collapse to a single pending work item. The controller even uses a single shared "ingress changed" queue item (you can see `_ingressChangeQueueItem` in `IngressController`) — any change just ensures that one item is queued, and one reconcile recomputes everything.

### `Rate/`

| File | Role |
| --- | --- |
| `Limiter` | A **token-bucket** rate limiter (Foundations Traffic-Management §7): tokens refill at a steady rate; each action spends one; empty bucket means wait |
| `Limit` | The rate/burst configuration |
| `Reservation` | A "reservation" of a future token — lets a caller ask "when may I proceed?" and wait that long |

This is a from-scratch token-bucket implementation (mirroring the one in Go's Kubernetes client libraries). Studying it is a genuinely good exercise: it's the same algorithm used for API rate limiting everywhere, implemented compactly. It paces both the watch-reconnect attempts (so a flapping connection doesn't hammer the API server) and the reconcile rate.

---

## 7. `Services/` — The Reconciler (the heart)

### What problem it solves

This is where the loop actually lives: pull work items, read the cache, compute the desired YARP config, and apply it. Its doc-comments lay out the architecture explicitly — the controller "receives notifications from informers," saves data "in an `ICache`," queues resources "which need to be reconciled," and a "background task dequeues items and passes them to an `IReconciler`."

### The components

| File | Role |
| --- | --- |
| `IngressController` | The **controller loop**, a background service. Wires informers → cache → queue, and runs the dequeue loop. Holds registrations to all five informers. |
| `IReconciler` / `Reconciler` | The **reconcile step**: read the cache, convert to YARP config (via `Converters/`), and push it out (via `IUpdateConfig`). Also updates Ingress status. |
| `QueueItem` / `ReconcileData` | The unit of work and the data gathered for one reconcile pass |

**Trace the loop:** an informer fires "ingress changed" → `IngressController` enqueues the shared change item (deduped) → the rate limiter releases it → `IngressController` dequeues and calls `Reconciler.ProcessAsync` → the reconciler reads the whole cache, builds the complete YARP config, and calls `IUpdateConfig` to publish it → it writes status back to the Ingress objects. The loop is *whole-world recompute*, which is exactly the level-triggered robustness from §2. Notice the dependencies in `Reconciler`'s constructor: `ICache` (read state), `IUpdateConfig` (publish), `IIngressResourceStatusUpdater` (report status) — the three responsibilities of any reconcile.

---

## 8. `Converters/` — Ingress → YARP Translation

### What problem it solves

Kubernetes `Ingress` objects describe routing in *Kubernetes' vocabulary* (rules, paths, backend service references, TLS secrets). YARP needs it in *its* vocabulary (`RouteConfig`/`ClusterConfig`/`DestinationConfig`). `Converters/` is the translation layer — the semantic bridge between the two systems.

### The components

| File | Role |
| --- | --- |
| `YarpParser` | The core translator: walks Ingress rules + the correlated Service/Endpoints data and emits YARP routes and clusters |
| `YarpIngressContext` / `YarpIngressOptions` | The working context + the per-ingress options (often from annotations) that tune the translation |
| `YarpConfigContext` | Accumulates the YARP config being built across all ingresses |
| `ClusterTransfer` | Helper for assembling cluster/destination data during conversion |

This is where Kubernetes-specific knowledge concentrates: how an `Ingress` path maps to a YARP route match, how a `Service` + its `Endpoints` become a cluster with destinations (the actual pod IPs come from `Endpoints`, not the `Service`), and how Ingress **annotations** (the standard Kubernetes extension mechanism — arbitrary key/value metadata on objects) select YARP features like load-balancing policy or health checks. If you want to understand "how does a YAML Ingress become a live route," this folder is the answer.

---

## 9. `ConfigProvider/` and `Protocol/` — Getting Config to the Proxy

There are two deployment shapes for an ingress controller, and these two folders support them. This is a subtle but important distinction.

### `ConfigProvider/` — the in-process seam

| File | Role |
| --- | --- |
| `IUpdateConfig` | The seam the reconciler calls to publish new config |
| `KubernetesConfigProvider` | An `IProxyConfigProvider` (Part 01!) that hands the generated config to a YARP proxy **running in the same process** |

When controller and proxy run together, `KubernetesConfigProvider` simply *is* a YARP config source — the reconciler updates it, and YARP's `ProxyConfigManager` picks up the change through the exact same `IProxyConfigProvider` mechanism as a JSON file would. **This is the punchline of the whole project:** a wildly dynamic source (Kubernetes) plugs into the identical seam as a static file. Everything you learned about validation and atomic hot-swap in Part 01 applies unchanged.

### `Protocol/` — the distributed (out-of-process) shape

| File | Role |
| --- | --- |
| `Dispatcher` / `IDispatcher` / `IDispatchTarget` | Holds the set of currently-connected proxy instances and **broadcasts** new config to all of them |
| `DispatchController` | An HTTP/endpoint that proxy instances connect to in order to receive config |
| `Receiver` / `ReceiverOptions` | The **proxy-side** client: connects to the controller and receives config updates |
| `DispatchConfigProvider` / `MessageConfigProviderExtensions` | On the proxy side, expose received messages as an `IProxyConfigProvider` |
| `Message` / `DispatchActionResult` | The wire format and result types for config messages |

This supports the **separated** topology: **one** controller computing config, and **many** stateless proxy pods receiving it over a connection. The `Dispatcher` keeps a list of connected targets (note in its code the `ImmutableList<IDispatchTarget>` swapped under a lock — the same immutable-snapshot discipline) and pushes the latest config to each; a newly-connected proxy immediately gets the current config. This is a small **pub/sub / fan-out** system — directly on your distributed-systems learning list. The separation matters operationally: you can scale proxies horizontally without each one independently hammering the Kubernetes API; only the single controller watches the cluster.

```
   SEPARATED TOPOLOGY
                    ┌──────────────┐  config broadcast   ┌─────────┐
   K8s API ──watch──│  Controller  │────────────────────▶│ Proxy 1 │
                    │ (Dispatcher) │────────┐            └─────────┘
                    └──────────────┘        │             ┌─────────┐
                                            └────────────▶│ Proxy 2 │ … N
                                                          └─────────┘
```

---

## 10. `Certificates/` and `Hosting/` — Support Infrastructure

### `Certificates/`

| File | Role |
| --- | --- |
| `ICertificateHelper` / `CertificateHelper` | Load/parse certificates (e.g., from Kubernetes `Secret`s) |
| `IServerCertificateSelector` / `ServerCertificateSelector` | Choose the right server certificate per incoming TLS connection (SNI-based selection) for the controller's own dispatch endpoint |

The controller's dispatch endpoint (§9) is itself an HTTPS server that proxies connect to, so it needs TLS. The certificate selector picks the correct cert based on the requested hostname — the same SNI mechanism real multi-tenant servers use.

### `Hosting/`

| File | Role |
| --- | --- |
| `BackgroundHostedService` | A base class for long-running background services with clean start/stop and lifetime handling. The informers and the controller loop all derive from it. |
| `HostedServiceAdapter` / `ServiceCollectionHostedServiceAdapterExtensions` | Glue to register the same object as both a typed service *and* a hosted background service in DI |

**The hosted-service concept** is the .NET way to run continuous background work inside an app: implement the hosted-service contract and the host starts/stops you with the application. Informers (long-lived watches) and the controller loop (long-lived dequeue) are exactly this. As an embedded engineer, think of these as the app's "tasks" or "threads of execution" that the runtime supervises — `BackgroundHostedService` is the shared scaffolding so each one gets correct startup ordering, cancellation, and shutdown.

---

## 11. `Management/`, options, and shared types

| File | Role |
| --- | --- |
| `Management/KubernetesCoreExtensions`, `KubernetesReverseProxyServiceCollectionExtensions`, `KubernetesReverseProxyWebHostBuilderExtensions` | The **composition root**: the `AddKubernetes...()` registration methods that wire informers, cache, queue, reconciler, converters, dispatcher, and the config provider into the DI container and the host. This is the "main wiring" that assembles the whole pattern. |
| `YarpOptions` | Top-level controller configuration (namespace scoping, the controller's class name, server cert settings, etc.) |
| `NamespacedName` | The Kubernetes identity of an object: namespace + name. A tiny but pervasive value type. |
| `Properties/` | Assembly metadata |

`Management/` is where you'd start reading if you wanted to see how the pieces are bolted together, because DI registration is effectively an index of every component and its chosen implementation.

---

## 12. End-to-End: a pod scales up

To cement it, trace a real event — Kubernetes adds a pod to a service the controller exposes:

```
 1. Kubernetes updates the Service's Endpoints (new pod IP added)
 2. V1EndpointsResourceInformer's WATCH stream pushes the change   [Client/]
 3. IngressCache updates its mirror of that Endpoints object       [Caching/]
 4. The change enqueues the shared "ingress changed" item (deduped) [Queues/]
 5. Rate/ releases it when a token is available                     [Rate/]
 6. IngressController dequeues → Reconciler.ProcessAsync            [Services/]
 7. Reconciler reads the whole cache, YarpParser rebuilds full config [Converters/]
 8. Reconciler calls IUpdateConfig                                  [ConfigProvider/]
 9a. In-process: KubernetesConfigProvider fires its change token →
     YARP's ProxyConfigManager validates + atomically swaps the model  [→ Part 01]
 9b. Separated: Dispatcher broadcasts the config to all proxies     [Protocol/]
10. New pod IP is now a healthy destination; load balancing includes it [→ Part 02]
11. Reconciler writes status back to the Ingress object            [Client/]
```

Notice how it ends by flowing into the core library you already understand. **The Kubernetes controller is "just another config source"** — the most dynamic one, but it reuses the identical control-plane → data-plane machinery from Parts 01–02.

---

## 13. What To Pay Attention To / Production & Interview Notes

- **Level-triggered reconciliation is the headline lesson.** "Recompute the whole desired state from cache, idempotently, on any change" is *the* controller pattern, and it's why controllers are so robust to dropped/duplicate events. Be able to explain why this is safer than reacting to individual deltas.
- **The cache+queue+ratelimiter trio is a reusable producer/consumer-with-backpressure design** you'll see far beyond Kubernetes. The dedup-on-enqueue + token-bucket-pacing combination is worth being able to reproduce.
- **Informers (LIST then WATCH with resourceVersion resume)** are the efficiency backbone of Kubernetes. Knowing how they avoid polling and how they recover from disconnects is common interview territory for infra roles.
- **In-process vs. separated (Dispatcher) topology** is a real architectural decision: co-located simplicity vs. horizontally-scalable, centrally-watched proxies. The `Protocol/` fan-out is a compact pub/sub example.
- **It all funnels into `IProxyConfigProvider`** — the same seam as a file. This is excellent evidence for "good abstractions let a static config file and a live Kubernetes cluster be interchangeable inputs."

> This project alone touches: Kubernetes, informers/controllers, service discovery, event-driven coordination, work queues, rate limiting, pub/sub fan-out, background services, and TLS/SNI — a large slice of your distributed-systems and cloud-infrastructure goals, in one readable codebase.

Next: **Part 05 — Application & Supporting Infrastructure**, covering the `Yarp.Application` host, `Common`, and the samples/tests/build machinery that surround the product.
