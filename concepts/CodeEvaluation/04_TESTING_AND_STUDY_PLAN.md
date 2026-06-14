# Code Evaluation — Part 04: Testing, Seams & Your Study Plan

> The final piece. First, how a professional project is *tested* — the design seams that make testing possible, which is itself a hallmark of senior design. Then a concrete, sequenced **study plan** that turns everything in this series from "I read it" into "I can do it," aimed squarely at walking into a mid-level role at a company like Microsoft and being up to speed.

---

## 1. Testability Is a Design Property, Not an Afterthought

The single biggest difference between personal-project code and professional code is that professional code is **designed to be tested**. You can't bolt good tests onto tangled code; you build the seams in from the start. YARP is full of these seams, and recognizing them teaches you to write testable code yourself.

### Seam 1 — Inject your dependencies (so tests can substitute fakes)

Every class takes its collaborators as constructor parameters typed as *interfaces* (Part 02 §1). That means a test can pass *fake* collaborators. The load-balancing policy takes `IRandomFactory` rather than calling `Random.Shared` directly — so a test can make randomness deterministic:

```csharp
public PowerOfTwoChoicesLoadBalancingPolicy(IRandomFactory randomFactory) { ... }
```

In production, `RandomFactory` returns `Random.Shared`. In a test, you inject a fake `IRandomFactory` whose `Random` returns a *scripted* sequence, so "pick two at random, choose the less busy" becomes **deterministically assertable**. Without that seam, the policy would be untestable (you can't assert on true randomness).

### Seam 2 — Abstract the uncontrollable: time

Tests must not depend on wall-clock time (slow, flaky). YARP abstracts time behind `IClock`/`TimeProvider`:

```csharp
[Obsolete("For testing only. Use TimeProvider instead.")]
public interface IClock
{
    DateTimeOffset GetUtcNow();
    long TickCount { get; }
    Task Delay(TimeSpan delay, CancellationToken cancellationToken);   // testable "wait"
}
```

The comment *"facilitates unit tests that use virtual time"* says it outright. A health-check test that needs "advance 30 seconds" uses a fake/virtual time provider and advances it **instantly** — no real waiting, no flakiness. (Note the professional touch: `IClock` is marked `[Obsolete]` pointing at the framework's newer `TimeProvider`, showing how a mature project *migrates* abstractions over time while keeping the old one working.)

> **The principle:** anything nondeterministic or slow — randomness, time, the network, the filesystem — must sit behind an interface so tests can replace it. Spotting "what's uncontrollable here?" and putting a seam there is a senior design skill.

### Seam 3 — `InternalsVisibleTo` to test internals without exposing them

From the csproj (Part 00 §3):

```xml
<InternalsVisibleTo Include="Yarp.ReverseProxy.Tests" />
<InternalsVisibleTo Include="DynamicProxyGenAssembly2" Key="$(MoqPublicKey)" />
```

This lets the test assembly (and the Moq mocking engine) see `internal` types. So YARP keeps its implementations `internal sealed` (minimal public surface, Part 00 §4) *and* still unit-tests them directly. You don't have to choose between "testable" and "encapsulated."

---

## 2. The Test Pyramid in Practice

YARP's test projects (Components Part 05 §4) implement the classic **test pyramid**:

```
            ╱╲          few, slow, high-confidence
           ╱  ╲         ReverseProxy.FunctionalTests  — real proxy + real backends, real HTTP
          ╱────╲        Kubernetes.Tests / Application.Tests — subsystem integration
         ╱──────╲       ReverseProxy.Tests — many, fast unit tests (one component, faked deps)
        ╱────────╲      testassets/BenchmarkApp — performance is tested too
```

| Layer | What it verifies | Trade-off |
| --- | --- | --- |
| **Unit** (`ReverseProxy.Tests`) | One class in isolation with faked deps (the seams above) | Fast + pinpoints the break; can't catch integration bugs |
| **Functional** (`ReverseProxy.FunctionalTests`) | A real proxy forwarding to real in-process backends over real HTTP | High confidence; slower + harder to localize failures |
| **Benchmarks** (`testassets/BenchmarkApp`) | Throughput/latency/allocations don't regress | Performance treated as a tested property |

The functional tests are why `testassets/` exists: `TestServer` (a configurable backend) and `TestClient` give the proxy something real to proxy. **The lesson:** unit tests prove components are *correct*; functional tests prove they're correct *together*; benchmarks prove they're *fast*. A serious product needs all three, and the architecture's interface seams serve double duty — they enable customization *and* unit testing.

> **Interview relevance:** "How do you test a high-throughput networked component?" → pyramid: many fast unit tests over faked seams (time, randomness, network), fewer functional tests with real servers, plus benchmarks guarding performance. Cite YARP's structure.

---

## 3. What "Reading a Big Codebase" Should Look Like (your stated weakness)

Your persona flags navigating large enterprise repos as a growth area. Here's the senior approach, demonstrated on YARP, that you can apply to *any* unfamiliar codebase:

1. **Find the architecture seams first, not the code.** Identify the projects and their dependencies (Components Part 00). Don't read top-to-bottom; build the map.
2. **Trace one request end-to-end.** Pick the primary control flow (here: a request through the pipeline) and follow it across components. One real trace teaches more than reading ten files in isolation.
3. **Learn the naming conventions and let them guide you.** Once you know `I*Policy` = Strategy, `*Middleware` = pipeline stage, `*Config`/`*State` = immutable data (Components Part 00 §4), you can predict what a file does from its name. Conventions are a map.
4. **Read the tests to learn intended behavior.** A class's unit test is executable documentation of how it's *meant* to be used and what edge cases matter.
5. **Read the `docs/designs/` and `README`s.** Maintainers explain the *why* there (e.g., `Model/README.md`'s immutability rule). The why is the hardest thing to reverse-engineer from code alone.
6. **Use the debugger as a microscope.** Set a breakpoint in `HttpForwarder.SendAsync`, send one request, and step. Watching real values flow makes the abstractions concrete.

> This *is* the onboarding playbook at big companies. Nobody reads the whole repo; they map it, trace it, and learn the conventions. Practicing this deliberately on YARP is rehearsal for your first week on the job.

---

## 4. Your Sequenced Study Plan

A path from "read these docs" to "demonstrably mid-level." Each phase has a concrete, *buildable* deliverable, because skill comes from producing, not reading.

### Phase 1 — Foundations you can prove (1–2 weeks)
- **Turn on `<Nullable>enable</Nullable>`** in one of your existing projects and fix every warning. Feel where null-safety changes your designs.
- **Re-implement, from memory:** `AtomicCounter`, `ValueStopwatch`, and a minimal `ILoadBalancingPolicy` with `RoundRobin` + `PowerOfTwoChoices`. These are small and touch DI, `struct`s, `Interlocked`, and Strategy at once.
- **Deliverable:** a tiny console app that registers two policies via DI, picks destinations, and has a unit test that injects a *fake* `IRandomFactory` to make `PowerOfTwoChoices` deterministic. *Now you've done the seam-for-testability move yourself.*

### Phase 2 — Async & performance (2–3 weeks)
- **Write a streaming copier**: `ValueTask<long> CopyAsync(Stream in, Stream out, CancellationToken)` using `ArrayPool<byte>` + `Memory<byte>`, with correct rent/return and cancellation. Compare against a naive `byte[]`-per-call version.
- **Add [BenchmarkDotNet]** and measure allocations both ways. *Seeing* the GC drop is the lesson.
- **Deliverable:** a short write-up (for yourself) explaining, in your words, why `Memory` not `Span` across `await`, why `ValueTask`, and why pooling — as if answering an interviewer.

### Phase 3 — Concurrency (2–3 weeks, the big one)
- **Implement a `ConfigSnapshot` holder** with an immutable record swapped via `Volatile.Write` under a writer-only lock, read lock-free by many tasks. Hammer it with concurrent readers + a writer and assert no torn reads.
- **Write up** why this beats a `lock`, why `Volatile`, and what RCU is. This is your headline concurrency story for interviews.
- **Deliverable:** the holder + a stress test + the written explanation.

### Phase 4 — Build a real (small) thing (3–4 weeks)
- **Build a mini reverse proxy** on ASP.NET Core: config-driven routes → clusters → destinations, one load-balancing policy, basic health checking, and `IHttpForwarder` (or hand-rolled forwarding). Add structured logging via `LoggerMessage` and one `EventSource`.
- **Deliverable:** a working proxy with unit + one functional test (a real in-process backend). This single project exercises *every* document in this series and is a portfolio piece you can discuss in depth.

### Phase 5 — Contribute (ongoing)
- **Ship the `help wanted` issues** from the `concepts/YarpOpenIssues/` series — start with **#1764** (docs) and **#275** (test cleanup), then **#2667** (telemetry) or **#2838** (a real bug). A merged PR into a Microsoft repo is concrete, verifiable evidence of mid-level capability — exactly what hiring managers want.

---

## 5. The Habits That Mark a Mid-Level Engineer (carry these forward)

Distilled from everything YARP demonstrates:

1. **Program to interfaces at every seam where behavior might change.** (DI, Strategy, Factory.) This one instinct produces most of "good architecture."
2. **Default to `internal sealed`; treat `public` as a permanent promise.** Minimize surface area.
3. **On the hot path, hunt allocations and copies.** Pool buffers, use `Span`/`Memory`/`ValueTask`, make small helpers `struct`s.
4. **Never lock readers if you can swap an immutable snapshot instead.** Lock writers only.
5. **Make the uncontrollable injectable** — time, randomness, network — so it's testable.
6. **Classify failures structurally**; reserve exceptions for the truly exceptional.
7. **Instrument generously but pay only when observed** (`EventSource` gating, `LoggerMessage`).
8. **Honor `CancellationToken` everywhere** in async code.
9. **Let conventions and tests document intent**; write yours so the next person benefits.
10. **Build the test seams in from the start** — testability is a design decision, not a phase.

> **Final word.** You came in with a real advantage: embedded engineers already think in buffers, ownership, stack vs heap, atomics, and cooperative scheduling — the exact instincts that YARP's hardest code rewards. What you were missing was the *idiomatic .NET expression* of those instincts and the *professional scaffolding* (DI, testing seams, immutable-snapshot concurrency, enforced conventions) that big-company code is built on. This series mapped all of it onto real, production Microsoft code. Now do the Phase 1–5 deliverables: reading made it familiar; *building* will make it yours. When you can re-derive the config-swap and explain every `Volatile`, defend `internal sealed`, and ship a merged YARP PR, you are not "preparing to be" mid-level — you are operating at that level. Go build.
