# YARP Open Issues — Part 02: Bug & Feature Issues

> The meatier technical issues — real code changes touching the forwarder, transforms, routing, service discovery, and telemetry. Higher learning value and higher risk than the docs issues in Part 01. All carry `help wanted`. Tackle these *after* a merged docs/test PR or two, when you know the contribution loop and some of the codebase.
>
> Same template throughout. **Re-confirm status at the issue URL before starting.**

---

## Issue #2667 — Telemetry: use the request path for the name of the trace

**Link:** https://github.com/dotnet/yarp/issues/2667 · **Type:** Enhancement (Telemetry) · **Difficulty:** ★★★☆☆ · *(7 reactions)*

### What it is
When YARP is used with **OpenTelemetry** and the **Aspire dashboard**, every proxied request shows up in the trace view named by the **route match pattern** (e.g. `/{**catch-all}`) rather than the **actual request path** (e.g. `/api/orders/42`). From the issue: *"all requests show in the dashboard with the route match path as the display name… you have to drill in"* to see what was actually requested.

### Concepts involved
- **Distributed tracing & spans** (Foundations Part 3 §5): each operation becomes a named span; the *name* is what you see in dashboards.
- **.NET `Activity`**: the runtime's representation of a span. Something sets the Activity's `DisplayName`; here it's defaulting to the route pattern.
- **Span-name cardinality — the central design tension.** This is *why it's not trivial*. Using the raw request path as the span name gives readable traces, but if paths contain IDs (`/orders/42`, `/orders/43`, …) you get **unbounded unique span names**, which explodes the cardinality of your telemetry backend (cost, performance, unusable aggregation). The route *pattern* is low-cardinality but uninformative; the raw *path* is informative but high-cardinality. A good fix must navigate this trade-off — perhaps making the naming strategy configurable, or using the route template with parameters in a controlled way.

### Why it's a problem
Out of the box, YARP traces are hard to read in modern dashboards, hurting the observability story for the increasingly popular .NET Aspire stack.

### Why it's still open
The cardinality trade-off means there's **no obviously-correct default**, so it needs a small design decision (configurable behavior? opt-in?) before code. That design friction — plus it being a quality-of-life issue, not a breakage — keeps it parked under `help wanted`.

### How you'd fix it
Propose (in the issue) a naming strategy: likely an **opt-in option** to name spans by request path (or by a normalized template), defaulting to today's behavior to avoid surprising existing users with a cardinality spike. Then set the `Activity.DisplayName` accordingly at the point where YARP creates/owns the proxy Activity.

### Step-by-step
1. Reproduce with a tiny YARP + OpenTelemetry + Aspire setup; observe the span names.
2. Find where YARP names the proxy Activity (the forwarder/telemetry path, near `ReverseProxyPropagator` and the Activity creation — Components Part 02 §8, Part 03).
3. Propose the configurable behavior in the issue; get maintainer agreement on the default and option shape.
4. Implement the option; add tests asserting the span name under each setting.
5. PR "Fixes #2667."

### Fit for you ★★★★☆
**Strongly aligned with your observability goal.** You'll learn .NET `Activity`/OpenTelemetry internals and — more valuable — the **cardinality trade-off**, which is fundamental telemetry-engineering wisdom that comes up constantly in production and interviews. Contained scope; the main "difficulty" is the design conversation, which is good practice.

---

## Issue #2838 — Service discovery on `MapForwarder` disabled when `httpClient` is assigned

**Link:** https://github.com/dotnet/yarp/issues/2838 · **Type:** Bug · **Difficulty:** ★★★☆☆

### What it is
In **direct forwarding** (`MapForwarder` / `IHttpForwarder`, Components Part 02 §8), when a user supplies their **own configured `HttpClient`**, YARP's **service discovery** (e.g., resolving an Aspire/DNS service name to real addresses) stops working. The repro in the issue uses a .NET **Aspire** project where the forwarder references an `api` service by logical name; assigning a custom `httpClient` to the forwarder breaks the name resolution that would otherwise turn `api` into a concrete address.

### Concepts involved
- **Direct forwarding** vs. the full pipeline (Part 02 §8): direct forwarding is the lean path, but it still needs to resolve where to send the request.
- **Service discovery / name resolution** (Part 02 §6): turning a logical service name into actual endpoints. .NET has a service-discovery abstraction (used heavily by Aspire) that integrates via the `HttpClient`'s handler chain.
- **The likely root cause — handler-chain ownership.** Service discovery is typically injected as a **message handler** in the `HttpClient`'s pipeline. When YARP builds the client, it inserts that handler; when *you* supply your own `HttpClient`, YARP uses yours as-is and the discovery handler is never added — so resolution silently no-ops. The fix must reconcile "respect the user's client" with "still apply discovery."

### Why it's a problem
It's a surprising, silent failure: the same code works with YARP's default client but breaks the moment you customize the client — a sharp edge for the popular Aspire workflow.

### Why it's still open
It's a relatively recent bug at the **intersection of three subsystems** (forwarder, service discovery, HttpClient construction), so the correct fix needs investigation and care not to break the "bring your own client" contract. That investigation cost keeps it open.

### How you'd fix it
After reproducing, determine where the discovery handler is (or isn't) inserted when a custom client is provided, and decide the right behavior: e.g., document that discovery requires letting YARP build the client, *or* provide a way to apply discovery to a user-supplied client, *or* detect and warn. Maintainer input on the intended contract is essential.

### Step-by-step
1. Build the minimal Aspire repro from the issue; confirm discovery works without a custom client and breaks with one.
2. Trace how the default client gets the discovery handler vs. the custom-client path (`Forwarder/` client factory + the discovery integration).
3. Propose the intended behavior in the issue; align with maintainers.
4. Implement + add a test covering both client paths.
5. PR "Fixes #2838."

### Fit for you ★★★☆☆
A genuine **bug-with-a-repro** — the best kind for learning. It exercises the `Forwarder/` + `ServiceDiscovery/` interaction you studied in Components Part 02, and touches **service discovery** (on your list). Rung 3 because the fix direction needs investigation and a design call, not just a code tweak.

---

## Issue #1109 — Unable to transform the `Set-Cookie` response header via `ITransformProvider`

**Link:** https://github.com/dotnet/yarp/issues/1109 · **Type:** Bug/Limitation · **Difficulty:** ★★★☆☆

### What it is
A user writes a custom `ITransformProvider` (a structured response transform, Components Part 02 §7) to rewrite the **cookie path** in the backend's `Set-Cookie` response header so it matches the proxy's transformed path. They find they **cannot properly transform `Set-Cookie`** through the normal transform mechanism.

### Concepts involved
- **`Set-Cookie` is a special header.** Unlike most headers, a response can carry **multiple** `Set-Cookie` headers, and the value has rich internal structure (name, value, `Path`, `Domain`, `Secure`, `HttpOnly`, `SameSite`, expiry). HTTP rules even forbid folding multiple `Set-Cookie` values into one comma-separated header (commas are legal inside cookie attributes like `Expires`). So generic header-transform logic — which often assumes single or comma-joinable values — mishandles it.
- **Cookie `Path`/`Domain` rewriting**: when a proxy changes the public path/host, cookies scoped to the backend's path/domain won't be sent back by the browser unless rewritten. (This is the same family as the open "Cookie Domain Rewrites" issue.)
- **Transform extensibility** (Part 02 §7): the question is whether the transform API surfaces `Set-Cookie` in a way that lets you safely modify each cookie.

### Why it's a problem
Cookie-based auth/session through a path-rewriting proxy is a *very common* scenario, and broken cookie rewriting silently breaks logins. It's a real correctness gap with broad impact.

### Why it's still open
Doing it *right* means handling the multi-value, structured nature of `Set-Cookie` correctly — easy to implement naively and wrong. It likely needs a purpose-built cookie transform rather than a generic header transform, which is a small design effort. Hence it lingers despite being impactful.

### How you'd fix it
Investigate how response header transforms enumerate `Set-Cookie`. Likely add (or document) a cookie-aware transform that operates per-cookie and can rewrite `Path`/`Domain` safely, respecting the multi-header rule. Coordinate with maintainers on whether this becomes a first-class transform.

### Step-by-step
1. Reproduce: backend sets a cookie with a `Path`; proxy rewrites the path; observe the cookie isn't usable client-side.
2. Examine the response-transform code path for how `Set-Cookie` is read/written (`Transforms/`).
3. Propose a cookie-aware approach in the issue; align on API.
4. Implement + tests covering multiple cookies and attribute rewriting.
5. PR "Fixes #1109."

### Fit for you ★★★☆☆
Meaty and high-impact, and you'll learn a genuinely tricky corner of HTTP (the `Set-Cookie` special-casing) that trips up many engineers. Rung 3 — the multi-value handling demands care. Good follow-on if cookies/auth interest you.

---

## Issue #1695 — `StreamCopyHttpContent.CreateContentReadStreamAsync()` not implemented

**Link:** https://github.com/dotnet/yarp/issues/1695 · **Type:** Bug/Limitation · **Difficulty:** ★★★★☆

### What it is
A user is building a proxy that **signs upstream requests** (a security sidecar): it must compute a hash/signature over the **request body** before forwarding. To hash the body they need to *read* it, which calls `CreateContentReadStreamAsync()` on YARP's request content type — but `StreamCopyHttpContent` (the streaming body wrapper, Components Part 02 §8, Foundations Part 3 §1) **doesn't implement that method**, so body signing is blocked.

### Concepts involved
- **Streaming vs. buffering — the core tension** (Foundations Part 3 §1). `StreamCopyHttpContent` deliberately *streams* the body without ever materializing it, for memory safety at scale. `CreateContentReadStreamAsync()` is a .NET `HttpContent` method whose contract is "give me a readable, **buffered/seekable** stream of the content" — which is fundamentally at odds with "never hold the whole body." Implementing it naively (buffer the whole body) would reintroduce exactly the unbounded-memory risk YARP exists to avoid.
- **One-pass vs. multi-pass over a stream**: signing wants to read the body *and* still forward it. You either buffer (to read twice) or **tee** the stream (hash as it flows). The elegant fix hashes inline during the single copy rather than buffering.
- **`HttpContent` contract semantics**: understanding which methods callers rely on and what guarantees they expect.

### Why it's a problem
Request-signing/inspection sidecars are a legitimate, important use case (zero-trust architectures), and they're currently blocked by an unimplemented method.

### Why it's still open
This is the deepest tension on the list: the "obvious" implementation (buffer everything) **violates YARP's defining performance principle**, so it can't just be added. The *right* solution (bounded buffering with a size cap? an opt-in tee/hash hook? documented limitation?) requires real design thought and maintainer agreement. High design cost + niche audience = long-lived `help wanted`.

### How you'd fix it
This is a design conversation first, code second. Options to propose: (a) implement `CreateContentReadStreamAsync` with **bounded** buffering and a configurable max size (fail past it), (b) provide a transform/hook to observe the body as it streams (so signing happens without buffering), or (c) if neither is acceptable, document the limitation clearly. Let maintainers steer.

### Step-by-step
1. Reproduce the signing scenario and confirm the `NotImplemented`/missing behavior.
2. Study `StreamCopyHttpContent` / `StreamCopier` to understand the streaming contract (Part 02 §8).
3. Write up the design options *in the issue* with the streaming-vs-buffering trade-offs; get direction before coding.
4. Implement the agreed approach + tests (including the memory-bound behavior).
5. PR "Fixes #1695."

### Fit for you ★★★☆☆ (stretch / Rung 3)
**The most aligned with your async/streaming/performance interests** and an outstanding teacher of high-performance I/O design — but precisely *because* it's a real design tension, it's not a quick win. Read it early for the education even if you tackle it later. If you can drive the design discussion well here, it signals serious systems maturity.

---

## Issue #1240 — Support port routes

**Link:** https://github.com/dotnet/yarp/issues/1240 · **Type:** Enhancement (Routing) · **Difficulty:** ★★★☆☆ · *(2 reactions)*

### What it is
YARP can match routes by **host** (and path, headers, etc.), but not by the **port** the request arrived on. From the issue: *"Host routing is currently supported, but the host may change… the only thing you can know is the port currently listening."* The user wants to route based on which **listening port** received the request (e.g., port 8080 → service A, 8081 → service B).

### Concepts involved
- **Multi-dimensional request matching** (Components Part 02 §2): routes match on several attributes; this asks to add *port* as a matchable dimension.
- **Endpoint routing & matcher policies** (Part 02 §2): the likely implementation is a new matcher (like the existing header/query matcher policies) that inspects the connection's local port.
- **Listening endpoints / bindings**: the proxy can listen on multiple ports; the port a request came in on is available from the connection's local endpoint.

### Why it's a problem
Some deployment topologies distinguish services purely by port (common when host headers are unreliable or rewritten upstream). Without port matching those scenarios need awkward workarounds.

### Why it's still open
It's a **feature requiring API design**: a new route-match concept means new config schema, a new matcher policy, validation, and docs — plus maintainer buy-in that port-matching belongs in core. Modest demand (2 reactions) means it hasn't been prioritized, but it's well-scoped enough to be `help wanted`.

### How you'd fix it
Propose the config shape (e.g., a `Ports` criterion in `RouteMatch`) in the issue. Implement a matcher policy mirroring the existing header/query matchers that accepts/rejects candidates based on the request's local port, plus validation and docs.

### Step-by-step
1. Discuss the config/API design in the issue; get agreement before coding (essential for an enhancement).
2. Study `Routing/` — the header/query matcher policies are your template (Part 02 §2).
3. Add the port-match config to `RouteMatch`, a `PortMatcherPolicy`, a validator (`RouteValidators`), and wire it into endpoint construction.
4. Add tests + docs.
5. PR "Fixes #1240."

### Fit for you ★★★☆☆
A clean way to **learn the routing/matcher-policy system end-to-end** (config → validator → matcher → endpoint), reinforcing Components Parts 01–02. It's a real feature with a clear template to copy, but the upfront API-design agreement makes it Rung 3. Good once you've shipped smaller PRs.

---

## Issue #1548 — Built-in support for something like nginx `proxy_redirect default`

**Link:** https://github.com/dotnet/yarp/issues/1548 · **Type:** Enhancement · **Difficulty:** ★★★★☆

### What it is
Nginx has a `proxy_redirect default` behavior that automatically **rewrites the `Location` header** (and similar) in backend **redirect responses** so the redirect points at the proxy's public address instead of the hidden backend's internal address. The user wants a built-in YARP equivalent so 3xx redirects from backends don't leak internal URLs or send clients to unreachable addresses.

### Concepts involved
- **Redirect responses (3xx) and the `Location` header**: when a backend replies "go to URL X," X is often the backend's *internal* address. Behind a proxy, the client can't reach that — the proxy must rewrite X to its own public-facing equivalent. This mirrors the forwarded-headers problem (Foundations Part 2 §6) but for the *response* direction.
- **Response transforms** (Part 02 §7): the fix is a response transform that rewrites `Location` (and possibly other URL-bearing headers) by mapping backend address → public address.
- **The "default"/automatic part is the hard part**: doing it generically (inferring the mapping automatically, like Nginx's `default`) rather than requiring manual per-route rules involves nontrivial inference about which URLs to rewrite to what.

### Why it's a problem
Redirect leakage/breakage is a common proxy pitfall; users currently hand-roll `Location` rewriting. A built-in, ideally automatic, feature would remove a sharp edge.

### Why it's still open
Two reasons: it needs **API design** (manual mapping vs. automatic inference, how much to copy Nginx's semantics), and the *automatic* version is genuinely tricky to get right without surprising users. Enhancement + design ambiguity = parked.

### How you'd fix it
Likely start with the simpler, explicit version: a response transform that rewrites `Location` based on a configured backend→public mapping (this may largely exist as a manual transform already — confirm in the issue). The ambitious "default/automatic" version is a larger design effort to negotiate with maintainers.

### Step-by-step
1. Clarify scope in the issue: explicit mapping first, or attempt the automatic `default`?
2. Study response transforms (`Transforms/`, Part 02 §7) and any existing `Location`-rewriting capability.
3. Implement the agreed scope + tests covering 3xx with internal `Location` values.
4. Docs + PR "Fixes #1548."

### Fit for you ★★☆☆☆
Conceptually excellent (redirects + response rewriting are great to understand), but the **design ambiguity around the automatic behavior** makes it a poor *early* pick — you could sink time into a design that doesn't get accepted. Revisit after you have standing and can co-design with maintainers.

---

## Issue #1777 — .NET 6: YARP directing to a path with `%2F` rather than `/`

**Link:** https://github.com/dotnet/yarp/issues/1777 · **Type:** Bug · **Difficulty:** ★★★★★ · *(10 reactions, references #1617)*

### What it is
After upgrading to .NET 6, proxied requests whose path contains an **encoded slash** (`%2F`) return 404s. The encoded slash is being mishandled somewhere in the path's journey through the proxy — the encoding is preserved or transformed in a way that no longer matches the backend's expectations. The issue notes it duplicates an earlier unresolved bug (#1617): *"exactly the same issue but not solved."*

### Concepts involved
- **URL encoding and path normalization**: `%2F` is a percent-encoded `/`. Crucially, `%2F` and `/` are **not interchangeable** — a `%2F` is a literal slash *inside a single path segment*, whereas `/` is a segment separator. Servers, frameworks, and proxies disagree about whether/when to decode `%2F`, and getting it wrong changes the meaning of the URL.
- **Where decoding happens in the stack**: the request path passes through Kestrel, ASP.NET Core routing, YARP's transforms, and out via the forwarder. Each layer has opinions about encoding. A `.NET 6` behavior change in one of them shifted the result.
- **Round-trip fidelity**: a transparent proxy should forward the path to the backend with the *same* encoding the client sent. Preserving exact encoding through multiple parsing/re-serialization steps is genuinely hard.

### Why it's a problem
Real APIs use `%2F` in path segments (encoded IDs, file paths, keys). Breaking them causes 404s for legitimate requests, and the high reaction count shows real users are blocked.

### Why it's still open (the honest answer)
This is the textbook example of a **high-impact bug that's hard to fix safely**. Encoded-slash handling is a notorious cross-stack minefield: a change that "fixes" `%2F` for one scenario routinely **breaks** another (double-decoding, security implications of decoding slashes, interactions with routing and other proxies). It spans layers YARP doesn't fully own (Kestrel/ASP.NET Core behavior). Because any fix carries real regression risk and must be coordinated with platform behavior, it has lingered across multiple issue numbers (#1617 → #1777) despite clear demand. Difficulty and blast radius — not neglect — are why it's open.

### How you'd fix it
Carefully. First pin down *exactly* where the encoding changes (write a failing test that sends `%2F` and inspects the outbound path at the forwarder). Then determine whether the fix belongs in YARP (e.g., preserving the raw encoded path/target when building the outbound request) or is a platform interaction to be configured around. Expect heavy maintainer involvement and a thorough test matrix.

### Step-by-step
1. Build a minimal repro; add a test asserting the outbound path for a `%2F`-containing request.
2. Instrument each stage (Kestrel-decoded path → routing → transform → forwarder outbound) to find where `%2F` is altered.
3. Research the .NET 6 change and existing platform guidance on encoded slashes.
4. Propose a narrowly-scoped fix in the issue with the regression risks spelled out; get maintainer steering **before** coding.
5. Implement with an extensive encoding test matrix; PR referencing #1777/#1617.

### Fit for you ★★☆☆☆ (not yet)
Fascinating and high-impact, but a **★★★★★ trap for a newcomer**: the blast radius and cross-stack nature mean a well-intentioned fix can easily regress others, and you'd be working in territory the core team guards closely. Read it to understand *why hard bugs persist*, but don't make it an early target. Worth revisiting once you're established and can co-drive with maintainers.

---

## Summary: the bug/feature ladder

| Approachable | Meaty | Advanced / design-heavy |
| --- | --- | --- |
| **#2667** telemetry trace name (observability, contained) | **#2838** discovery + custom client (bug w/ repro) | **#1695** stream read (streaming-vs-buffering design) |
| | **#1109** Set-Cookie transform (HTTP cookie subtleties) | **#1548** proxy_redirect (API design) |
| | **#1240** port routes (routing feature, clear template) | **#1777** %2F encoding (cross-stack, high risk) |

**Suggested path:** after your Part 01 docs/test PRs, take **#2667** (plays to observability) or **#2838** (a clean bug with a reproduction). Use **#1695** as aspirational reading for streaming mastery. Avoid **#1777** and **#1548** until you have project standing — and you'll know *why* from the "why it's still open" sections above.
