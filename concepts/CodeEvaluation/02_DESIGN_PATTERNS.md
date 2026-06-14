# Code Evaluation — Part 02: Design Patterns You Should Learn From This Code

> Design patterns are the vocabulary senior engineers use to discuss structure. YARP is an unusually clean catalog of them because its entire reason for existing — *be customizable* — forces disciplined, pattern-based design. This document walks the patterns that actually appear in the code, each with the real YARP example, what problem it solves, and how to recognize and reuse it. Learn to *name* these and you'll communicate like a senior engineer in design reviews and interviews.
>
> A theme to hold: most of these patterns are different answers to one question — **"how do I let behavior be swapped out without rewriting the code that uses it?"** That is the heart of extensible software.

---

## 1. Dependency Injection (DI) — the pattern everything else rides on

**Problem it solves:** a class needs collaborators (a logger, a clock, a policy). If it `new`s them itself, it's welded to specific implementations — untestable and inflexible. **Dependency Injection** inverts this: a class *declares* what it needs (usually via constructor parameters typed as interfaces), and a container *supplies* concrete instances at runtime.

YARP's `PowerOfTwoChoicesLoadBalancingPolicy` doesn't create its randomness source; it asks for one:

```csharp
internal sealed class PowerOfTwoChoicesLoadBalancingPolicy : ILoadBalancingPolicy
{
    private readonly IRandomFactory _randomFactory;

    public PowerOfTwoChoicesLoadBalancingPolicy(IRandomFactory randomFactory)  // injected
    {
        _randomFactory = randomFactory;
    }
    ...
}
```

The **container** (Microsoft.Extensions.DependencyInjection) is configured in the registration methods:

```csharp
services.TryAddSingleton<IHttpForwarder, HttpForwarder>();
services.TryAddSingleton(TimeProvider.System);
```

`TryAddSingleton<IHttpForwarder, HttpForwarder>()` means "when anyone asks for `IHttpForwarder`, give them one shared `HttpForwarder`." Two professional details:

- **`TryAdd...`** registers *only if not already registered*. This lets a user override a default by registering their own implementation *first* — the mechanism behind YARP's "customize anything." Plain `Add` would clobber a user's choice; `TryAdd` respects it.
- **Lifetimes** (`Singleton`, `Scoped`, `Transient`) declare how long an instance lives. `Singleton` = one for the app; `Scoped` = one per request; `Transient` = a fresh one each time. Choosing correctly is a common source of real bugs (e.g., capturing request state in a singleton).

> **Why it matters:** DI *is* how modern .NET apps are wired. Every class you see taking interfaces in its constructor is participating. Internalize "depend on interfaces, let the container supply them" — it's the precondition for testability (Part 04) and for every other pattern below.

---

## 2. Strategy / Policy — interchangeable algorithms behind one interface

**Problem it solves:** you have several ways to do the same job (pick a destination) and want to choose at runtime/config time without `if/switch` sprawl. **Strategy** defines one interface and many implementations; callers depend only on the interface.

YARP's load-balancing policies are a textbook Strategy:

```csharp
public interface ILoadBalancingPolicy
{
    string Name { get; }   // how config refers to this strategy
    DestinationState? PickDestination(HttpContext context, ClusterState cluster,
                                      IReadOnlyList<DestinationState> availableDestinations);
}
```

Implementations — `RoundRobinLoadBalancingPolicy`, `RandomLoadBalancingPolicy`, `PowerOfTwoChoicesLoadBalancingPolicy`, `LeastRequestsLoadBalancingPolicy`, `FirstLoadBalancingPolicy` — each encapsulate one algorithm. The middleware that uses them knows *nothing* about which one it's calling:

```csharp
// conceptually, in LoadBalancingMiddleware:
ILoadBalancingPolicy policy = /* resolved by name from the cluster's config */;
DestinationState? chosen = policy.PickDestination(context, cluster, availableDestinations);
```

The same shape appears all over YARP: `ISessionAffinityPolicy`, `IActiveHealthCheckPolicy`, `IPassiveHealthCheckPolicy`, `IAvailableDestinationsPolicy`, `IAffinityFailurePolicy`, `IDestinationResolver`. **Whenever you see `I...Policy`, you're looking at Strategy.**

The `Name` property is a nice professional touch: it lets config select a strategy by string ("RoundRobin") while code stays type-safe, and it lets the container hold *all* implementations and pick the right one by name.

> **Interview relevance:** "How would you let users choose between load-balancing algorithms?" → Strategy: one interface, many implementations, selected by name, resolved via DI. You can literally cite this code.

---

## 3. Factory — encapsulating *how to create* something

**Problem it solves:** sometimes *creating* an object is itself a decision with logic, configuration, or polymorphism. A **Factory** centralizes that creation so callers don't hard-code `new`.

YARP has several. The cleanest to study is `IRandomFactory`:

```csharp
public interface IRandomFactory
{
    Random CreateRandomInstance();
}

internal sealed class RandomFactory : IRandomFactory
{
    public Random CreateRandomInstance() => Random.Shared;
}
```

Why wrap something as trivial as getting a `Random`? **Testability.** Production uses `RandomFactory` (real randomness); a test injects a fake factory returning a *deterministic* sequence, so a test of `PowerOfTwoChoices` can assert exactly which destination is chosen. The factory turns an uncontrollable dependency (randomness) into an injectable, controllable one (Part 04 expands this).

A heavier factory is `IForwarderHttpClientFactory` / `ForwarderHttpClientFactory`, which builds the per-cluster `HttpClient` (with its connection pool, TLS, protocol settings) from a cluster's config. Creating a correct `HttpClient` is genuinely complex, so it's encapsulated behind a factory you can replace.

> **Recognize it by:** `I...Factory` with a `Create...` method. Use it when object creation has logic worth isolating, or when you need to make creation swappable/testable.

---

## 4. Builder + Fluent Interface — readable, incremental configuration

**Problem it solves:** configuring a complex subsystem with many optional parts is awkward with constructors. A **Builder** lets you assemble it step by step, and a **fluent interface** (methods returning the builder) makes it read like a sentence.

`AddReverseProxy` returns a builder you chain onto:

```csharp
public static IReverseProxyBuilder AddReverseProxy(this IServiceCollection services)
{
    var builder = new ReverseProxyBuilder(services);
    builder
        .AddConfigBuilder()
        .AddRuntimeStateManagers()
        .AddConfigManager()
        .AddSessionAffinityPolicies()
        .AddActiveHealthChecks()
        .AddPassiveHealthCheck()
        .AddLoadBalancingPolicies()
        .AddDestinationResolver()
        .AddProxy();
    ...
    return builder;
}
```

And the user continues the chain:

```csharp
services.AddReverseProxy()
        .LoadFromConfig(configuration)
        .AddTransforms(...);
```

Each method registers some services and **returns the builder** so the next call chains. The builder itself is minimal — it just holds the `IServiceCollection`:

```csharp
internal sealed class ReverseProxyBuilder : IReverseProxyBuilder
{
    public ReverseProxyBuilder(IServiceCollection services)
    {
        ArgumentNullException.ThrowIfNull(services);
        Services = services;
    }
    public IServiceCollection Services { get; }
}
```

The actual chainable methods are **extension methods** (`LoadFromConfig`, `AddConfigFilter<T>`, …) on `IReverseProxyBuilder`. This is a deliberate, very .NET pattern: a tiny core interface, extended by many discoverable extension methods, so the API can grow without changing the interface.

> **Why it matters:** this fluent `Add...().Use...()` style *is* the ASP.NET Core configuration idiom. Recognizing "builder + extension methods + return-this fluency" means you can both read and design the configuration surface of any .NET library.

---

## 5. Extension Methods — adding capability without inheritance

Worth calling out on its own because it's pervasive and C#-specific. An **extension method** is a static method that *appears* to be an instance method on a type you don't own:

```csharp
public static IReverseProxyBuilder LoadFromConfig(this IReverseProxyBuilder builder, IConfiguration config)
{
    ...
    return builder;
}
```

The `this IReverseProxyBuilder builder` first parameter is the magic: now anyone can write `builder.LoadFromConfig(config)`. The entire `AddReverseProxy().LoadFromConfig()...` fluent API, and indeed most of ASP.NET Core's `services.AddX()` / `app.UseX()` surface, is built from extension methods. They let a library expose a rich, discoverable API around a small set of core types, and let *you* add methods to types you can't modify.

> **Recognize it by:** `public static ... this SomeType x ...`. Master this and the whole "`services.AddThis().AddThat()`" ecosystem stops being mysterious.

---

## 6. Middleware / Chain of Responsibility — the request pipeline

**Problem it solves:** a request must pass through many independent processing steps (auth, routing, load balancing, forwarding), each able to act, modify, or stop the flow, in a configurable order. **Chain of Responsibility** links handlers so each decides whether to handle and/or pass along.

ASP.NET Core's middleware (Components Part 02) is this pattern. Each YARP stage is a middleware — `LoadBalancingMiddleware`, `SessionAffinityMiddleware`, `PassiveHealthCheckMiddleware`, `ForwarderMiddleware` — and they're composed into a pipeline where each calls the "next." The order is configured once; each stage is independent and replaceable.

> **Why it matters:** the pipeline/middleware model is the backbone of every ASP.NET Core app. Seeing it as Chain of Responsibility (with the option to *short-circuit*) lets you reason about ordering bugs ("why does auth need to run before forwarding?") structurally rather than by trial and error.

---

## 7. Null Object — a do-nothing implementation that removes special-casing

**Problem it solves:** code depends on an interface that sometimes has no real implementation (e.g., a Windows-only feature on Linux). Sprinkling `if (delegator != null)` everywhere is ugly and error-prone. The **Null Object** pattern provides a *do-nothing* implementation so callers never need null checks.

YARP's HTTP.sys delegation is Windows-only. On Linux it registers a dummy:

```csharp
internal sealed class DummyHttpSysDelegator : IHttpSysDelegator
{
    public void ResetQueue(string queueName, string urlPrefix) { }   // intentionally empty
}
```

```csharp
// in AddReverseProxy:
if (OperatingSystem.IsWindows())
    builder.AddHttpSysDelegation();                                   // real implementation
else
    builder.Services.TryAddSingleton<IHttpSysDelegator, DummyHttpSysDelegator>();  // no-op
```

Now any code that injects `IHttpSysDelegator` can call it unconditionally; on Linux it harmlessly does nothing. The platform difference is handled *once* at registration, not scattered through the codebase.

> **Recognize it by:** an implementation whose methods are deliberately empty/no-op, registered to satisfy a dependency that isn't meaningful in some context. Cleaner than null checks; a favorite in cross-platform code.

---

## 8. Decorator — wrapping to add behavior without changing the original

**Problem it solves:** you want to add behavior (measurement, logging) to an object without modifying it or its callers. A **Decorator** implements the same interface, wraps an instance, and adds behavior around the delegated calls.

YARP's WebSocket telemetry (Components Part 02 §10) wraps the connection's stream to *count bytes* as they flow, without changing how the bytes are copied: `WebSocketsTelemetryStream` is a stream that delegates reads/writes to the inner stream while tallying throughput. The base `DelegatingStream` (in `Utilities/`) exists precisely to make such wrappers easy — it forwards every `Stream` member to an inner stream so a decorator only overrides the one or two it cares about.

> **Why it matters:** decoration is how you add cross-cutting concerns (metrics, retries, caching) *non-invasively*. "Wrap, delegate, augment" is a reusable move you'll apply constantly.

---

## 9. Adapter — making one interface look like another

**Problem it solves:** you have a component speaking interface A, but the surrounding system expects interface B. An **Adapter** translates between them.

In the Kubernetes controller (Components Part 04), `HostedServiceAdapter` lets an object be registered as both a typed service *and* a hosted background service. The Kubernetes config provider is itself an adapter: it makes the controller's generated config *look like* a standard YARP `IProxyConfigProvider`, so the core proxy consumes Kubernetes the same way it consumes a JSON file. The converter layer (`YarpParser`) adapts Kubernetes `Ingress` objects to YARP's route/cluster model.

> **Recognize it by:** a class whose job is purely "speak the language the other side expects." Adapters are the glue that lets independently-designed systems compose.

---

## 10. Double-Checked Locking — initialize-once, read-cheap (advanced)

**Problem it solves:** lazily create an expensive shared thing exactly once, with many threads possibly racing to trigger it, *without* paying a lock on every subsequent read. This is an advanced concurrency pattern and YARP uses it correctly in `ProxyConfigManager`:

```csharp
public override IReadOnlyList<Endpoint> Endpoints
{
    get
    {
        if (_endpoints is null)            // 1st check: no lock, fast path
        {
            lock (_syncRoot)               // only race losers enter here
            {
                if (_endpoints is null)    // 2nd check: did someone else init while we waited?
                {
                    InitialLoad...();
                }
            }
        }
        return _endpoints;
    }
}
```

The first `if` (no lock) handles the overwhelmingly common case — already initialized — at zero synchronization cost. Only the rare first-time callers take the lock, and the *second* `if` inside the lock ensures the work happens once even if several threads got past the first check simultaneously. Getting this subtly wrong is a famous source of bugs (it requires the field's writes to be properly ordered, which `Volatile`/the swap logic ensures — see Part 03).

> **Why it matters:** double-checked locking is a canonical interview/whiteboard topic. Seeing it *correctly* implemented in production — paired with the `Volatile.Write` swap in Part 03 — is the best way to actually understand it.

---

## 11. The Pattern Map (your cheat sheet)

| Pattern | YARP example | The question it answers |
| --- | --- | --- |
| **Dependency Injection** | every ctor taking interfaces; `TryAddSingleton` | How do collaborators get supplied + swapped? |
| **Strategy / Policy** | `ILoadBalancingPolicy` and the `*Policy` family | How do I make an algorithm interchangeable? |
| **Factory** | `IRandomFactory`, `IForwarderHttpClientFactory` | How do I make *creation* swappable/testable? |
| **Builder + Fluent** | `AddReverseProxy().LoadFromConfig()...` | How do I configure a complex thing readably? |
| **Extension methods** | `LoadFromConfig(this IReverseProxyBuilder ...)` | How do I grow an API around small core types? |
| **Middleware / Chain of Responsibility** | the request pipeline, `*Middleware` | How do steps process + pass + short-circuit? |
| **Null Object** | `DummyHttpSysDelegator` | How do I avoid null-checks for absent features? |
| **Decorator** | `WebSocketsTelemetryStream`, `DelegatingStream` | How do I add behavior without touching callers? |
| **Adapter** | K8s `KubernetesConfigProvider`, `YarpParser` | How do I make A look like B? |
| **Double-checked locking** | `ProxyConfigManager.Endpoints` | How do I init-once but read-cheap, thread-safely? |

> **The meta-lesson:** notice how many entries are variations of "**program to an interface, supply the implementation later**." DI, Strategy, Factory, Null Object, Adapter, Decorator are all that idea in different costumes. When you design your own systems, reach for an interface at the seam where behavior might change — that single instinct, applied well, is most of what separates junior structure from senior structure.

Next: **Part 03 — concurrency, async, and observability**: the immutable-snapshot config swap, lock-free reads, `CancellationToken`-as-change-signal, and the near-free `EventSource`/`LoggerMessage` instrumentation.
