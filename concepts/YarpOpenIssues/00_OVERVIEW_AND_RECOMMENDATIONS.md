# YARP Open Issues — Part 00: Landscape, Contribution Primer & Recommendations For You

> **What this series is.** A guided tour of the *currently open issues* on the public `dotnet/yarp` repository, written for you (Timothy) as a prospective first-time contributor. It explains the overall issue landscape, then deep-dives the issues you could realistically work on — the concepts each involves, why it's a problem, how hard it is, why it's still open, and the concrete steps to tackle it. By the end you should be able to pick an issue with confidence and know how to start.
>
> **Data freshness.** This is based on a live pull of the GitHub API. At the time of writing the repo had **182 open issues**. Issue *numbers* and the technical concepts behind them are stable, but **status changes** — before you start anything, open the issue URL and read the latest comments to confirm it's still open and unclaimed. Every issue below is linked.
>
> **The most important caveat up front:** dotnet/yarp is a Microsoft-maintained project. Many issues are *design discussions* or *backlog placeholders* that a drive-by PR will not resolve. The realistic contribution targets are the ones explicitly marked **`help wanted`** — there are **14** of them, and Parts 01–02 cover every single one.

---

## 1. How the YARP Team Organizes Issues (read this first)

Before judging any individual issue, you need to understand the repo's triage vocabulary, because it tells you which issues are *actually open for contribution* versus which are tracking/design artifacts.

| Signal | What it means | Should you touch it? |
| --- | --- | --- |
| **`help wanted` label** | Maintainers have decided this is well-scoped and they'd welcome a community PR | **Yes — these are your targets** |
| **`good first issue` label** | (YARP does **not** currently use this label — don't go looking for it) | n/a |
| **`Backlog` milestone** | "We acknowledge this but it's not scheduled." Most open issues sit here | Only if also `help wanted` |
| **`v.Next` / `YARP 3.0` milestone** | Slated for a specific release; usually maintainer-owned | Usually no |
| **Area labels** (`Area-*`) | Which subsystem it touches (Config, K8s, Telemetry, etc.) | Use to find issues matching your interests |
| **`Type: Bug` / `Type: Enhancement` / `Type: Docs`** | The nature of the work | Docs/Bug are friendlier starts than Enhancement |
| **No label, recent** | Freshly filed, not yet triaged | Wait — a maintainer hasn't confirmed it's real/wanted |

**The golden rule of contributing to a big corporate OSS repo:** for anything beyond a typo, **comment on the issue and get a maintainer's "go ahead" before writing code.** Enhancement/feature issues especially often need an agreed API design first; submitting a large unsolicited PR is the #1 way to have good work rejected. The `help wanted` label is essentially a pre-granted "go ahead," which is why it's the safe lane.

---

## 2. The 182 Open Issues at a Glance: Major Themes

Sampling the open issues reveals a handful of active *workstreams*. Knowing these helps you understand where the project's energy is and where a contribution is most welcome.

| Theme | What's going on | Example open issues |
| --- | --- | --- |
| **Container-image productionization** | YARP now ships an official container image; a cluster of issues is hardening it for production | "Improve logging/error output in the YARP container image," "Configuration model for the container image," "Improve static file & SPA hosting," "Checklist to make YARP container production ready," "502 Bad gateway only on Docker… Accept-Language header" |
| **Kubernetes ingress controller** | Ongoing improvements to the controller (Part 04 of your Components series) | "Use `WatchAsync` for both `ListAsync`/`WatchAsync` in `RunAsync`" (#3033), "EndpointSlices," "Unable to access YARP Ingress via SSL" |
| **Routing features** | Requests for richer match capabilities | "Multiple path matches for routes," "Support port routes" (#1240), "Cookie Domain Rewrites" |
| **Transforms** | Gaps/limitations in request/response rewriting | "Set-Cookie transform via ITransformProvider" (#1109), "customizable header exclusion list for transparent proxy" |
| **Observability / telemetry** | Better traces/metrics, OpenTelemetry & Aspire integration | "Telemetry – use request path for trace name" (#2667) |
| **Documentation** | Guides, migration docs, clarifications | Nginx/Ocelot migration guides, Swagger howto, WebSocket keep-alive docs, IHttpForwarder transform docs |
| **`YARP 3.0` (meta)** | An umbrella tracking the next major version | "YARP 3.0" |
| **Long-tail bugs & how-tos** | Many individual reports, often environment-specific | "%2F path encoding" (#1777), assorted Q&A-style issues |

**Why so many open issues for a healthy project?** This is normal and not a sign of neglect. Large OSS repos accumulate issues faster than any team can close them; "open" often means "acknowledged and parked," not "actively broken and ignored." Maintainers triage aggressively (labels, milestones) but deliberately leave well-scoped work open under `help wanted` *for the community*. Reading "182 open" as "182 fires" is a beginner's misread — most are backlog, design discussion, or support questions.

---

## 3. The 14 `help wanted` Issues — Your Realistic Targets

These are the issues maintainers have explicitly opened to contributors. Parts 01–02 deep-dive each. Here is the master table; the **Fit** column is my recommendation strength *for you specifically*, justified in §5.

| # | Title | Type | Difficulty | Fit for you |
| --- | --- | --- | --- | --- |
| **275** | Clean out usage of Autofac and Moq from tests | Test/Infra | ★★☆☆☆ | ★★★★★ **top pick** |
| **1764** | Doc WebSocket keep-alive requirement | Docs | ★☆☆☆☆ | ★★★★★ **top pick** |
| **2291** | Doc using structured transforms with `IHttpForwarder` | Docs | ★★☆☆☆ | ★★★★☆ |
| **2847** | LettuceEncrypt is now archived (update docs/sample) | Docs/Sample | ★★☆☆☆ | ★★★★☆ |
| **2667** | Telemetry – use the request path for name of trace | Feature (Telemetry) | ★★★☆☆ | ★★★★☆ |
| **2838** | Service discovery on `MapForwarder` disabled when `httpClient` assigned | Bug | ★★★☆☆ | ★★★☆☆ |
| **1109** | Unable to transform `Set-Cookie` response header via `ITransformProvider` | Bug | ★★★☆☆ | ★★★☆☆ |
| **1695** | `StreamCopyHttpContent.CreateContentReadStreamAsync()` not implemented | Bug | ★★★★☆ | ★★★☆☆ (stretch) |
| **1240** | Support port routes | Feature | ★★★☆☆ | ★★★☆☆ |
| **1548** | Built-in support for nginx `proxy_redirect default` | Feature | ★★★★☆ | ★★☆☆☆ |
| **1777** | .NET 6: directing to path with `%2F` rather than `/` | Bug | ★★★★★ | ★★☆☆☆ (hard) |
| **2037** | Nginx → YARP migration guide | Docs | ★★★☆☆ | ★★★☆☆ |
| **1842** | Ocelot → YARP (K8s) migration doc | Docs | ★★★☆☆ | ★★★☆☆ |
| **1789** | How to add Swagger for Gateway | Docs/Q&A | ★★☆☆☆ | ★★☆☆☆ |

*(Difficulty is the technical + risk weight: ★ = typo-tier, ★★★★★ = subtle, cross-cutting, high-regression-risk.)*

---

## 4. How To Actually Contribute (the mechanics, once)

This applies to any issue you pick. The Components series Part 05 covered the build/test machinery; here's the contribution workflow on top of it.

1. **Read `CONTRIBUTING.md`** in the repo root. It states the process, coding standards, and the **Contributor License Agreement (CLA)** you must sign (a one-time click for .NET Foundation projects — required before any PR is merged).
2. **Comment on the issue**: "I'd like to work on this — is it still available?" Wait for a maintainer ack. This avoids duplicate work and confirms the approach.
3. **Fork** the repo, **clone** your fork, create a **branch** (`git checkout -b issue-275-remove-autofac-moq`).
4. **Build locally**: run `restore.cmd`/`.sh` then `build.cmd`/`.sh` (downloads the pinned .NET SDK into `.dotnet/`, per the README). Confirm a clean build *before* changing anything.
5. **Make the change**, keeping the **diff minimal and focused** on the one issue. Match surrounding code style (the `.editorconfig` enforces it).
6. **Run the tests**: `build.cmd/sh -test`, or the targeted `dotnet build /t:Test /p:XunitMethodName=...` form from the README. Add/adjust tests for your change.
7. **Open a Pull Request** against `main`, referencing the issue ("Fixes #275"). Fill in the PR description: what, why, how tested.
8. **Respond to review.** Maintainers will comment; iterate. This is where you learn the most — treat review feedback as free senior mentorship.

> **Mindset:** your first PR's value is *learning the contribution loop and the codebase*, not the size of the change. A clean, well-tested, well-scoped 20-line PR that gets merged is worth far more to your growth than an ambitious 500-line PR that stalls in review.

---

## 5. My Recommendations For You — and the reasoning

I'm matching against your `persona.md`: embedded/firmware engineer, strong C/C++/Python/C#, comfortable reading code but newer to large enterprise repos and modern backend/distributed/async topics; goals include ASP.NET Core, Kubernetes, observability, async internals, and *contributing to major OSS*. The recommendations are sequenced as a **learning ladder** — each builds confidence and codebase familiarity for the next.

### Rung 1 — Build the contribution muscle (do one of these first)

**#1764 — Doc WebSocket keep-alive requirement** *(my #1 starter)*. The maintainer has *already written the technical explanation in the issue* (the 100-second idle activity timeout, why keep-alives must come from client/server not the proxy). The work is to fold that into the official docs. **Why you:** near-zero risk, teaches you the docs pipeline and the WebSocket-proxying concept (which is on your list), and gets a merged PR under your belt. You cannot really get this wrong, which is exactly what a first PR should be.

**#275 — Remove Autofac and Moq from tests** *(my #1 for codebase learning)*. A pure refactor: rewrite tests that use the Autofac DI container and Moq mocking library to use the built-in alternatives. **Why you:** it forces you to *read a lot of the test suite*, which is the fastest way to learn an unfamiliar codebase (your stated weakness), it involves zero product-behavior risk (tests must still pass), and it's squarely a C# skill. It can be done incrementally (a few test files per PR), so you can start tiny. This is the single best issue for "learn the repo by doing."

### Rung 2 — Apply your strengths to small features/bugs

**#2847 — LettuceEncrypt archived.** The Let's Encrypt sample/docs point at a now-archived library. Update them to a maintained alternative (or document the situation). **Why you:** touches TLS/certificate automation (your auth/security interest), modest scope, and involves *evaluating alternatives* — a real engineering judgment skill. Confirm the maintainers' preferred replacement in the issue first.

**#2667 — Telemetry: use request path for trace name.** When using OpenTelemetry/Aspire, traces are named by the *route match pattern* rather than the actual request path, making the dashboard hard to read. **Why you:** directly on your **observability** learning goal, exposes you to .NET `Activity`/OpenTelemetry naming (`ReverseProxyPropagator`, Components Part 02 §8 and Part 03), and is a contained change. Slightly harder because it needs a sensible default that doesn't explode cardinality (see Part 02 deep-dive).

### Rung 3 — Stretch into internals (after 1–2 merged PRs)

**#2838 — Service discovery disabled on `MapForwarder` + custom `httpClient`.** A real bug at the intersection of direct forwarding, service discovery, and HTTP client configuration. **Why you:** teaches the `Forwarder/` + `ServiceDiscovery/` interaction (Components Part 02 §6, §8), and bug-fixing with a reproduction is deeply educational. Needs investigation, hence Rung 3.

**#1695 — `StreamCopyHttpContent.CreateContentReadStreamAsync()` not implemented.** Request-signing sidecars need to read/hash the request body, but YARP's streaming content type doesn't implement the buffered-read method. **Why you:** it sits right on your **async/streaming/performance** interests and the zero-buffering design tension (Components Part 02 §8, Foundations Part 3 §1). It's genuinely hard because the *right* fix must respect YARP's no-buffering philosophy — but reading it will teach you more about high-performance I/O than a dozen tutorials.

### What I'd steer you *away* from (for now)

**#1777 (%2F path encoding)** is the highest-reaction bug here, but it's a ★★★★★ trap for a newcomer: URL-encoding/path-handling semantics interact with ASP.NET Core, Kestrel, and existing behavior, so a "fix" easily breaks other scenarios — that's precisely why it's still open despite many users wanting it (§Part 01 explains). **#1548 (proxy_redirect)** needs upfront API design agreement. Revisit these once you have standing in the project.

---

## 6. How To Read Parts 01 and 02

- **Part 01 — Documentation & Test/Infra issues**: the friendliest entry points (#1764, #275, #2291, #2847, #2037, #1842, #1789). Lower technical risk, high learning-per-effort.
- **Part 02 — Bug & Feature issues**: the meatier technical ones (#2667, #2838, #1109, #1695, #1240, #1548, #1777). Each with full concept background, fix strategy, and "why still open."

Each issue entry follows the same template so you can compare them: **What it is → Concepts involved → Why it's a problem → Why it's still open → Difficulty → How you'd fix it → Step-by-step → Fit for you.**

---

## 7. Sources

This analysis was built from a live GitHub API pull of `dotnet/yarp` open issues (the `help wanted` set and the most recent issues), cross-referenced with your codebase walkthrough in `concepts/YarpProjectComponents/`. Key issue links are embedded throughout Parts 01–02; the canonical list is the repo's [help-wanted filter](https://github.com/dotnet/yarp/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22). **Always re-check an issue's current state before starting.**
