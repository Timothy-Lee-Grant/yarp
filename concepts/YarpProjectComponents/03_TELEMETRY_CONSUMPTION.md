# YARP Project Components — Part 03: `Yarp.Telemetry.Consumption`

> This is the observability companion project (`src/TelemetryConsumption/`). It is **optional** — YARP proxies fine without it — but it is the canonical example of a pattern every backend engineer must know: **instrument the system with near-zero-cost structured events, and subscribe to them out-of-band to produce metrics and traces.** This document explains the project's purpose, the .NET diagnostics machinery it's built on, and each of its seven area subfolders.
>
> Concept background: Foundations Part 3 §5 (the three pillars — logs, metrics, traces — plus `EventSource`/`EventCounter`). Here we map that onto the actual components.

---

## 1. The Problem This Project Solves

A reverse proxy sits on the critical path of every request, so two things are true at once: it is the **best place to observe** system behavior, and its own instrumentation must be **almost free**, or it would slow down the very traffic it measures.

These two requirements are in tension. Verbose logging is informative but expensive (string formatting, I/O) on a hot path. The resolution, baked into .NET, is **`EventSource`**: a mechanism for emitting strongly-typed, structured diagnostic events that cost essentially **nothing when no one is listening**, and only do work when a subscriber attaches. YARP (and the .NET runtime itself — sockets, DNS, TLS, HTTP, Kestrel) emit a rich stream of such events.

`Yarp.Telemetry.Consumption` is the **consumer** side: it subscribes to those event streams and turns them into something you can use — periodic **metrics** and per-event **callbacks** into code you write. It is deliberately a *separate* package from `Yarp.ReverseProxy` because observation is optional and you may not want its overhead or dependencies in every deployment.

```
   PRODUCERS (emit EventSource events, ~free when unobserved)
   ┌──────────────────────────────────────────────────────────┐
   │ .NET runtime: Sockets · DNS · TLS · HTTP                  │
   │ ASP.NET Core: Kestrel                                     │
   │ YARP: Forwarder stages · WebSockets                       │
   └───────────────────────────────┬──────────────────────────┘
                                   │  (event stream)
                                   ▼
   CONSUMER  =  Yarp.Telemetry.Consumption
   ┌──────────────────────────────────────────────────────────┐
   │ EventListenerService(s) subscribe & decode the events     │
   │   → produce METRICS (periodic aggregates)                 │
   │   → invoke your IXxxTelemetryConsumer callbacks per event │
   └──────────────────────────────────────────────────────────┘
```

---

## 2. The Core .NET Concepts (so the folder makes sense)

Three platform ideas underpin everything in this project. Learn them once; you'll meet them across all of .NET observability.

**`EventSource`.** A class that *produces* structured events. Each event has a name and typed payload. Crucially, emitting is gated on whether any listener has enabled that source at a given level, so an unobserved event is a cheap branch, not a formatting + write. This is why YARP can instrument every forwarding stage without measurable overhead in production where no consumer is attached.

**`EventListener`.** A class that *consumes* events. You override callbacks to be told when an event source is created (so you can choose to enable it) and when an event is written (so you can read its payload). `Yarp.Telemetry.Consumption`'s `EventListenerService` types are `EventListener`s wrapped as hosted background services.

**`EventCounter` / counters.** A sub-mechanism for *metrics* specifically: sources periodically publish aggregate numbers (counts, rates, means, percentiles) over a polling interval. The consumer subscribes with a desired interval and receives the rolled-up values. This is how the `*Metrics` types in each subfolder get populated.

**Two output modes.** For each area, the project offers both:

| Output mode | Type shape | Use it for |
| --- | --- | --- |
| **Metrics** | `XxxMetrics` snapshot + `IMetricsConsumer<XxxMetrics>` | Dashboards, alerting, trends (periodic aggregates) |
| **Per-event callbacks** | `IXxxTelemetryConsumer` interface you implement | Detailed per-request tracing, custom correlation |

You implement whichever consumer interface you care about, register it in DI, and the matching `EventListenerService` pushes data to it. This is the same **interface-seam** pattern as the rest of YARP: the project doesn't decide what you do with telemetry, it just delivers it to your implementation.

---

## 3. The Top-Level Components

| File (`TelemetryConsumption/*.cs`) | Role |
| --- | --- |
| `EventListenerService` | The shared base: a hosted `EventListener` that knows how to attach to a named event source, decode events, and dispatch to consumers. Every area's listener derives from this. |
| `IMetricsConsumer<TMetrics>` | The generic seam for receiving periodic metric snapshots of type `TMetrics` |
| `MetricsOptions` | Configuration for metrics collection (e.g., the polling interval) |
| `TelemetryConsumptionExtensions` | The registration API — `AddTelemetryConsumer(...)` / `AddTelemetryListeners(...)` that wire your consumer implementations and start the listener services |

The design is uniform: a generic base (`EventListenerService`), a generic metrics seam (`IMetricsConsumer<T>`), and then **one subfolder per telemetry *area*** that specializes both. Once you understand one subfolder, you understand all seven — they differ only in *which* event source they listen to and *what* their payload/metrics contain.

---

## 4. The Seven Area Subfolders

Each subfolder corresponds to one **producer** of events. Together they give end-to-end visibility across the entire network stack a proxied request touches — from the raw socket up to YARP's own forwarding logic. This breadth is the point: when a request is slow, you can localize *which layer* the time went to.

### A request's journey, mapped to the telemetry areas

```
 client
   │  TCP connect ............... Sockets/      (connections, bytes)
   │  TLS handshake ............. NetSecurity/  (handshake count/duration)
   │  (server side) accepted by . Kestrel/      (inbound connection/request metrics)
   ▼
 YARP forwards:
   │  DNS resolve backend ....... NameResolution/ (lookups, duration, failures)
   │  outbound HTTP ............. Http/         (outbound request metrics, HttpClient)
   │  forwarding stages ......... Forwarder/    (per-stage timings, proxy-specific)
   │  if upgraded ............... WebSockets/   (duration, bytes each way, close reason)
   ▼
 backend
```

### The subfolders in detail

| Subfolder | Listens to | What it tells you | Key files |
| --- | --- | --- | --- |
| **`Sockets/`** | .NET Sockets `EventSource` | Connections established, bytes sent/received at the TCP layer — the rawest I/O view | `SocketsEventListenerService`, `SocketsMetrics`, `ISocketsTelemetryConsumer` |
| **`NameResolution/`** | .NET DNS `EventSource` | DNS lookups: how many, how long, failures. Catches "discovery/DNS is slow" problems | `NameResolutionEventListenerService`, `NameResolutionMetrics`, `INameResolutionTelemetryConsumer` |
| **`NetSecurity/`** | .NET TLS `EventSource` | TLS handshakes: count, duration, protocol/cipher. Catches "handshake storms" and TLS perf issues | `NetSecurityEventListenerService`, `NetSecurityMetrics`, `INetSecurityTelemetryConsumer` |
| **`Http/`** | .NET `HttpClient`/`SocketsHttpHandler` `EventSource` | Outbound HTTP requests YARP makes to backends: timings, request/response start, connection pool behavior | `HttpEventListenerService`, `HttpMetrics`, `IHttpTelemetryConsumer` |
| **`Kestrel/`** | ASP.NET Core Kestrel `EventSource` | Inbound side: connections and requests *arriving* at the proxy | `KestrelEventListenerService`, `KestrelMetrics`, `IKestrelTelemetryConsumer` |
| **`Forwarder/`** | YARP's own `ForwarderTelemetry` `EventSource` | Proxy-specific: time spent in each **forwarding stage** (Part 02 §8), proxied request counts, error categories. The most YARP-specific area | `ForwarderEventListenerService`, `ForwarderMetrics`, `ForwarderStage`, `IForwarderTelemetryConsumer` |
| **`WebSockets/`** | YARP's WebSocket telemetry (Part 02 §10) | Upgraded-connection lifetime, bytes each direction, close reason | `WebSocketsEventListenerService`, `WebSocketCloseReason`, `IWebSocketsTelemetryConsumer` |

**Why split by area rather than one giant consumer?** Two reasons. First, **producers are independent** — each event source is owned by a different layer (OS sockets vs. DNS vs. YARP's forwarder), so subscribing is naturally per-source. Second, **you rarely want all of it** — a team debugging TLS perf enables `NetSecurity/` only, paying zero cost for the rest. The per-area split lets you opt in granularly.

### The `ForwarderStage` concept (the most useful one)

`Forwarder/ForwarderStage` enumerates the *phases* of a single forward — e.g., sending request headers, sending request body, receiving response headers, receiving response body, response upgrade. Because the forwarder fires an event at each stage boundary, a consumer can compute "how long did we spend waiting on the backend's first byte vs. streaming the body?" This is what turns a vague "the request was slow" into "the backend took 190 ms to start responding." It's the proxy-aware equivalent of a distributed trace's spans, and it's why the `Forwarder/` area is the one you'll reach for most when diagnosing latency.

---

## 5. How It Connects to the Rest of YARP

The relationship is intentionally **one-directional and loose**: `Yarp.ReverseProxy` (and the runtime) *emit* events; `Yarp.Telemetry.Consumption` *listens*. The core library has no dependency on the consumption library — it would emit the same events whether or not anyone listens. This is good architecture: the observed system doesn't know or care about the observer, so observability can be added, removed, or replaced without touching the proxy.

```
   Yarp.ReverseProxy ──emits EventSource events──▶  (nothing, if no listener)
                                                │
                          (optional) ───────────┘
                                ▼
   Yarp.Telemetry.Consumption listens ──▶ your IMetricsConsumer / IXxxTelemetryConsumer
                                                ▼
                          your exporter (OpenTelemetry, Prometheus, logs, …)
```

The `samples/ReverseProxy.Metrics.Sample` and `samples/Prometheus/` projects (Part 05) show this end-to-end: implement a consumer, forward the data to Prometheus, scrape it into Grafana. **OpenTelemetry** is the modern, vendor-neutral standard you'd typically bridge to here.

---

## 6. What To Pay Attention To / Production Notes

- **"Free when unobserved" is the whole trick.** The reason YARP can afford to instrument every socket, handshake, and forwarding stage is that `EventSource` events are gated on subscription. Internalize this — it's the standard answer to "how do you add deep observability without hurting performance?"
- **Metrics vs. events is a real trade-off.** Metrics are cheap, aggregate, and great for dashboards/alerts but lose per-request detail. Per-event consumers give full fidelity (per-request tracing) but cost more and can flood you with data. Production systems use metrics for *everything always-on* and per-event tracing *sampled* or *on-demand*.
- **Layered visibility localizes faults.** The seven areas exist so you can answer "which layer is slow?" — socket, DNS, TLS, inbound (Kestrel), outbound (Http), proxy logic (Forwarder), or WebSocket. Drawing the §4 journey diagram is a great way to reason about observability coverage in *any* system.

> **Interview relevance.** This project is a compact, real-world illustration of: the **three pillars of observability**, the **low-overhead-instrumentation** pattern (`EventSource`), the **producer/consumer decoupling** of observed system vs. collector, and **metrics-vs-tracing trade-offs**. These are directly on your learning list (Observability) and come up constantly in system-design discussions. Connect it to distributed tracing: the `Forwarder/` stages are the proxy's contribution to an end-to-end trace, and `ReverseProxyPropagator` (Part 02 §8) is what keeps that trace connected across the backend hop.

Next: **Part 04 — Kubernetes.Controller**, the most intricate companion project, and a self-contained tour of the controller/reconciler pattern that powers Kubernetes itself.
