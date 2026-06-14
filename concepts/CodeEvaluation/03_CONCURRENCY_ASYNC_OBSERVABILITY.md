# Code Evaluation — Part 03: Concurrency, Async & Observability

> This is the most advanced — and most career-defining — material in the series. Your `persona.md` lists async internals, lock-free programming, race conditions, and synchronization as priority growth areas, and YARP is a master-class in exactly those. We'll study how YARP changes shared configuration *while serving traffic* without locks on the hot path, how it uses `async`/`await` and `CancellationToken` correctly, and how it instruments everything for near-zero cost. Real code throughout.
>
> Read Part 01 (`Interlocked`, `Volatile`, `ValueTask`) first — this builds directly on it.

---

## 1. The Core Problem: Mutable Shared State Under Massive Concurrency

A reverse proxy has thousands of requests in flight, each *reading* configuration (routes, clusters, destinations, health). Meanwhile that configuration *changes* — a file is edited, Kubernetes pushes an update, a health probe flips a destination. Two threads, one writing and many reading the same data, is the textbook setup for **race conditions** and **torn reads** (seeing a half-updated state).

The naive fix — a lock around all shared state — would make every request contend on one lock, destroying throughput. YARP's solution is the pattern you should tattoo on your brain as a systems engineer: **immutable snapshots swapped atomically, with locks only on the rare writer path.**

---

## 2. Immutable Snapshot + Atomic Swap (the heart of YARP)

The discipline is documented in the code itself (`Model/README.md`): every runtime-state object must be **immutable**, or wrap an immutable value in an **atomic holder**, or be a **thread-safe atomic** counter. Nothing is mutated in place.

Here is the actual swap, in `ProxyConfigManager` (lightly trimmed):

```csharp
private CancellationTokenSource _endpointsChangeSource = new();
private IChangeToken _endpointsChangeToken;
private List<Endpoint>? _endpoints;
private readonly object _syncRoot = new();

// READERS call this on every request — note: NO LOCK.
public override IChangeToken GetChangeToken() => Volatile.Read(ref _endpointsChangeToken);

// WRITER path — runs only when config changes.
private void UpdateEndpoints(List<Endpoint> endpoints)
{
    lock (_syncRoot)                                  // serialize WRITERS only
    {
        var oldCancellationTokenSource = _endpointsChangeSource;

        Volatile.Write(ref _endpoints, endpoints);    // publish the new immutable list atomically

        _endpointsChangeSource = new CancellationTokenSource();
        Volatile.Write(ref _endpointsChangeToken,
                       new CancellationChangeToken(_endpointsChangeSource.Token));

        oldCancellationTokenSource?.Cancel();         // signal "config changed" to subscribers
    }
}
```

Unpack the genius:

- **Readers never lock.** They call `Volatile.Read(ref _endpoints)` (or read through the change token) and get *whatever the current published reference is*. A reader either sees the entirely-old list or the entirely-new list — never a mix — because reference assignment is atomic and `Volatile` enforces visibility/ordering.
- **The writer builds a brand-new list** and **publishes it with a single `Volatile.Write`**. The old list is not touched; in-flight requests holding it keep working against a consistent snapshot until they finish. New requests pick up the new reference. This is **snapshot consistency** with zero reader coordination.
- **The lock guards only writers** (`_syncRoot`) — and config changes are rare, so contention is essentially nil.
- **`Volatile.Write`** is what makes the swap *safe*: it guarantees the new list is fully constructed and visible to other cores *before* the reference is seen pointing at it (preventing a reader from seeing the new reference but stale contents — the subtle bug that sinks naive double-checked locking).

This is the production-correct version of the double-checked locking from Part 02 §10, and it's the same family of idea as **RCU (read-copy-update)** in the Linux kernel and **persistent data structures** in functional programming.

> **This is your single highest-leverage takeaway.** "How do you mutate shared state read by thousands of concurrent requests without locking readers?" → immutable snapshot, build-new-then-atomic-swap-the-reference, `Volatile` for ordering, lock only writers. Being able to explain *and* defend this (why not a `lock`? why not `ConcurrentDictionary`? why `Volatile`?) is a senior-level concurrency conversation.

---

## 3. Change Notification via `CancellationToken` (a clever idiom)

Look again at how the swap *signals* "config changed": it uses a `CancellationTokenSource` not to cancel work, but as a **one-shot broadcast signal**:

```csharp
_endpointsChangeSource = new CancellationTokenSource();
_endpointsChangeToken  = new CancellationChangeToken(_endpointsChangeSource.Token);
...
oldCancellationTokenSource?.Cancel();   // fires the OLD token → "something changed, re-read"
```

The pattern: a `CancellationToken` is really just "a thing that fires once and lets listeners register callbacks." ASP.NET Core's routing subscribes to `GetChangeToken()`; when the old source is `Cancel()`led, routing re-reads the endpoints and rebuilds its match table. After signaling, a *fresh* token is installed for the next change.

This is the same `IChangeToken` mechanism the file-config provider uses (Components Part 01). Repurposing `CancellationToken` as a general "change happened" notifier is an idiomatic .NET move worth recognizing — it's used all over the framework.

> **Lesson:** `CancellationToken` is a general-purpose *signal*, not just a cancel button. Recognizing this unlocks a lot of framework code that uses change tokens for config reload, cache invalidation, and lifecycle events.

---

## 4. `async` / `await`: Cooperative Concurrency at Scale

A proxy must handle tens of thousands of *simultaneous* connections, most of them just *waiting* on slow network I/O. Dedicating a thread to each would exhaust memory and the scheduler. `async`/`await` is the answer: a small thread pool services many connections by *suspending* a request when it's waiting on I/O and *resuming* it when data is ready.

YARP is `async` end to end. The forwarder's signature:

```csharp
public async ValueTask<ForwarderError> SendAsync(
    HttpContext context, string destinationPrefix,
    HttpMessageInvoker httpClient, ForwarderRequestConfig requestConfig,
    HttpTransformer transformer, CancellationToken cancellationToken)
```

**The mental model you must build (it's on your priority list):**

- `await someIoTask;` does **not** block a thread. It *returns the thread to the pool* and registers a continuation. When the I/O completes, the runtime grabs *a* pool thread and resumes after the `await`. So one thread can advance hundreds of requests, each parked at its own `await`.
- The compiler rewrites each `async` method into a **state machine** — an object that remembers "where was I" across each `await`. (This is why hot-path methods avoid capturing unnecessary variables: each captured variable enlarges that state-machine object. You can see YARP deliberately minimizing captures: *"Avoid capturing 'isRequest' and 'timeProvider' in the state machine when telemetry is disabled."*)
- **`ValueTask`** (Part 01 §4) avoids allocating that state-machine/`Task` when the method completes synchronously — common when data is already buffered.

> **Embedded parallel:** this is *cooperative multitasking* — exactly like a cooperative RTOS where tasks yield at await points — but the compiler generates the yield/resume bookkeeping for you. Your firmware intuition about "don't block; yield and come back" maps perfectly. The difference is the scheduler is the .NET thread pool, and the "yield points" are `await`s.

---

## 5. `CancellationToken` for Timeouts and Teardown

Async work must be cancellable — a client disconnects, a timeout fires, the app shuts down. The proxy threads a `CancellationToken` through every async call so the whole forward can be torn down promptly. YARP wraps this in `ActivityCancellationTokenSource` (a pooled, reusable source tied to request timeouts), and the stream copier *resets* the timeout on every successful chunk:

```csharp
read = await input.ReadAsync(buffer.AsMemory(), cancellation);
...
activityToken.ResetTimeout();   // progress was made → push the idle deadline out
```

This implements the **idle/activity timeout** (the same 100s behavior behind open issue #1764): a request isn't killed for taking long *overall*, only for going *silent* — the correct way to reap dead connections without punishing slow-but-alive ones. And on cancellation, the error is classified precisely:

```csharp
if (activityToken.CancelledByLinkedToken)
    return (StreamCopyResult.Canceled, ex);
```

> **Lesson:** professional async code is *cancellation-aware everywhere*. A `CancellationToken` parameter on every async method is not boilerplate — it's how the system stays responsive and leak-free under failure. Adopt the habit of accepting and honoring `CancellationToken` in your own async APIs.

---

## 6. Structured Error Handling: Classify, Don't Just Throw

Notice YARP almost never lets a raw exception escape to mean "it failed." It **classifies** failures into a precise enum and returns/reports them. From the forwarder's failure handler:

```csharp
return await ReportErrorAsync(
    requestBodyCanceled ? ForwarderError.RequestBodyCanceled : ForwarderError.RequestCanceled,
    StatusCodes.Status502BadGateway);
...
return await ReportErrorAsync(ForwarderError.RequestTimedOut, StatusCodes.Status504GatewayTimeout);
...
return await ReportErrorAsync(
    failedDuringRequestCreation ? ForwarderError.RequestCreation : ForwarderError.Request,
    StatusCodes.Status502BadGateway);
```

And exceptions are caught *narrowly*, with **exception filters** (`when (...)`):

```csharp
catch (HttpRequestException hre) when (tryDowngradingH2WsOnFailure) { ... }
```

The `when (...)` filter means the `catch` only runs if the condition holds — otherwise the exception keeps propagating *without unwinding the stack into this handler*. This is more precise (and cheaper) than catching broadly and re-throwing.

Two professional habits here:

1. **Expected failures are return values, not exceptions.** `ForwarderError` + a `(result, exception)` tuple (Part 01 §8) communicates "what went wrong" structurally, so callers (logging, passive health, metrics) can react precisely — "connection refused" vs "client disconnected" lead to different health decisions. Throwing for *expected* conditions is slow and loses that structure.
2. **Catch narrowly, filter precisely.** Broad `catch (Exception)` that swallows everything is a code smell; YARP catches specific types with filters and always *does something deliberate* (classify, report, sometimes retry like the HTTP/2 WebSocket downgrade).

> **Interview relevance:** "When should you use exceptions vs return codes?" → exceptions for the genuinely exceptional/unrecoverable; structured return values for expected, recoverable outcomes on a hot path. YARP's `ForwarderError` is the canonical example.

---

## 7. Near-Free Observability: `EventSource`

YARP instruments every forwarding stage, but it can't afford instrumentation that costs anything when no one's watching. The trick (Components Part 03) is `EventSource`, gated on subscription:

```csharp
// Only build the telemetry helper if someone is actually listening.
var telemetry = ForwarderTelemetry.Log.IsEnabled(EventLevel.Informational, EventKeywords.All)
    ? new StreamCopierTelemetry(isRequest, timeProvider)
    : null;
...
telemetry?.AfterRead(contentLength);   // null-conditional: no-op when not listening
ForwarderTelemetry.Log.ForwarderStage(ForwarderStage.SendAsyncStart);
```

When no listener is attached, `IsEnabled(...)` is false, the telemetry object is never allocated, and every `telemetry?.X()` is a cheap null check. Only when a consumer subscribes does the machinery spin up. This is how YARP can fire fine-grained events (per forwarding stage, per stream read/write) *in the hot path* without measurable cost in production.

> **Lesson:** "instrument generously, pay only when observed" is the professional approach to telemetry. The `?.` null-conditional + an `IsEnabled` gate is the idiom. (Compare to naive logging that formats strings on every call regardless of log level — a real performance bug.)

---

## 8. Allocation-Free, Structured Logging: `LoggerMessage`

Ordinary logging (`logger.LogWarning($"... {clusterId} ...")`) allocates a string and boxes arguments *every call*, even if the log level is disabled. YARP uses the **`LoggerMessage` pattern** to pre-compile log delegates once:

```csharp
internal static class Log
{
    private static readonly Action<ILogger, string, Exception?> _affinityCannotBeEstablished =
        LoggerMessage.Define<string>(
            LogLevel.Warning,
            EventIds.AffinityCannotBeEstablishedBecauseNoDestinationsFoundOnCluster,
            "The request affinity cannot be established because no destinations are found on cluster `{clusterId}`.");

    public static void AffinityCannotBeEstablishedBecauseNoDestinationsFound(ILogger logger, string clusterId)
        => _affinityCannotBeEstablished(logger, clusterId, null);
}
```

`LoggerMessage.Define<string>(...)` builds a strongly-typed, cached delegate *once* (static readonly). Each log call invokes that delegate, which **skips all work if the level is disabled** and avoids re-parsing the message template and boxing. Every log site also carries a stable **`EventId`** (from a central `EventIds` class), so logs are filterable and correlatable by ID, not just by message text.

Modern .NET takes this further with a **source generator** (`[LoggerMessage]` attribute) that writes this boilerplate for you at compile time — you'll see both styles in current code.

> **Lesson:** in high-throughput services, logging is a performance surface. The `LoggerMessage` pattern (cached delegates, structured `EventId`s, no work when disabled) is the professional standard. Naive interpolated-string logging in a hot loop is a classic junior mistake.

---

## 9. The Concurrency/Async Cheat Sheet

| Technique | YARP example | The principle |
| --- | --- | --- |
| Immutable snapshot + atomic swap | `ProxyConfigManager.UpdateEndpoints` | Mutate shared state without locking readers |
| `Volatile.Read/Write` | the endpoint swap | Visibility + ordering across cores |
| Lock the writer, never the reader | `_syncRoot` around updates only | Hot path stays contention-free |
| `Interlocked` counters | `AtomicCounter` | Lock-free shared counters |
| `CancellationToken` as change signal | `_endpointsChangeSource.Cancel()` | One-shot broadcast notification |
| `async`/`await` + `ValueTask` | the whole forwarder | Cooperative concurrency, no thread-per-connection |
| Cancellation everywhere | `ActivityCancellationTokenSource`, `ResetTimeout` | Responsive teardown, idle timeouts |
| Classify failures, filter catches | `ForwarderError`, `catch ... when` | Structured, precise error handling |
| Gated `EventSource` | `IsEnabled(...)` + `telemetry?.` | Free-when-unobserved telemetry |
| `LoggerMessage` delegates + `EventId` | `SessionAffinity/Log.cs` | Allocation-free, structured logging |

> **The synthesis for your career:** these are *precisely* the topics (lock-free programming, async internals, synchronization, race conditions) your persona flags as growth areas and that big-company systems interviews hammer. YARP gives you correct, production reference implementations of every one. Don't just read them — re-derive the config-swap from scratch and explain out loud why each `Volatile` is necessary. When you can do that, you've genuinely leveled up.

Next: **Part 04 — testing, the seams that make it possible, and a concrete self-study plan** to convert all this reading into demonstrable mid-level ability.
