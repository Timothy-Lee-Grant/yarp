# YARP Open Issues — Part 01: Documentation & Test/Infrastructure Issues

> These are the **friendliest entry points** — low product-behavior risk, high learning-per-effort, and several are explicitly on your learning list. Every issue here carries the `help wanted` label. Each entry uses the same template so you can compare them.
>
> Re-confirm current status at the issue URL before starting — labels and assignees change.

---

## Issue #1764 — Document the WebSocket keep-alive requirement

**Link:** https://github.com/dotnet/yarp/issues/1764 · **Type:** Docs · **Difficulty:** ★☆☆☆☆

### What it is
YARP closes idle connections after a default **100-second activity timeout**. For ordinary request/response traffic that's invisible, but for **WebSocket** connections — which are long-lived and can sit silent for minutes (Foundations Part 1 §5.4) — an idle socket gets closed by the proxy, surprising users. The maintainer has already written the explanation directly in the issue:

> *"100s is the default activity timeout to close idle requests. Without this timeout the proxy would be subject to resource leaks… WebSocket or application-level keep-alives are required to keep an idle WebSocket from being closed by the proxy. These can be enabled on either the client or server (not the proxy)."*

The work is to turn that explanation into official documentation.

### Concepts involved
- **WebSocket proxying** and why a proxy treats an upgraded connection as a long-lived byte pump.
- **Idle/activity timeouts** as a resource-leak defense: a proxy can't cheaply detect a dead peer without traffic, so it closes connections that go silent. This is a real distributed-systems concern — *half-open connections* (one side gone, the other unaware) waste resources and must be reaped.
- **Keep-alives**: periodic no-op frames (WebSocket ping/pong, or app-level heartbeats) that prove a connection is alive and reset the idle timer. The key subtlety the doc must convey: the keep-alive must come from the **client or backend**, not the proxy.

### Why it's a problem
Users hit mysteriously dropped WebSocket connections and have no idea the 100s timeout exists or how to avoid it. It's a documentation gap, not a code bug.

### Why it's still open
Classic low-priority docs task: the *answer is known* (it's in the issue), it doesn't block anyone who reads the issue, and writing docs is nobody's urgent job. It's `help wanted` precisely because it's perfect for a community member — it just needs someone to do it.

### How you'd fix it
Add a short section to the WebSocket/limits documentation explaining the activity timeout, the symptom (idle WebSockets closing), and the remedy (enable keep-alives on client/server; or, if appropriate, adjust the activity timeout). Reference the relevant config knob (the request activity/idle timeout).

### Step-by-step
1. Comment on the issue offering to write the docs; confirm where docs live (the repo's `docs/`-driven site / Microsoft Learn content).
2. Find the existing WebSocket and timeout documentation pages.
3. Draft a concise section: symptom → cause (100s default) → fix (keep-alives, who sets them) → how to change the timeout if needed.
4. Cross-link from the WebSocket page and the limits/timeouts page.
5. PR with "Fixes #1764."

### Fit for you ★★★★★
**The ideal first PR.** The technical answer is handed to you, risk is near zero, and you'll internalize the WebSocket-proxying and idle-timeout concepts (both on your list) in the process. Start here to learn the contribution loop.

---

## Issue #275 — Clean out usage of Autofac and Moq from tests

**Link:** https://github.com/dotnet/yarp/issues/275 · **Type:** Test/Infra refactor · **Difficulty:** ★★☆☆☆

### What it is
The test suite historically used two third-party libraries: **Autofac** (an alternative dependency-injection container) and **Moq** (a mocking library). The maintainer wants them removed in favor of the built-in equivalents (the .NET DI container, and hand-written test doubles / a lighter approach). From the issue:

> *"This is a style issue more than anything, but it's really not clear that these libraries help when writing the tests. The tests I have re-written result in a similar amount of code and don't require learning two additional frameworks."*

### Concepts involved
- **Dependency injection containers**: Autofac vs. Microsoft.Extensions.DependencyInjection. You'll learn how the same registration/resolution is expressed in each — directly reinforcing the DI material from Components Part 01.
- **Mocking and test doubles**: Moq generates fake implementations of interfaces at runtime so a unit test can isolate one component (Components Part 05 §4). Removing it means replacing mocks with either hand-written stub classes or a different technique. You'll learn *what mocking actually does* by doing it manually.
- **Minimizing dependencies**: a real engineering value — fewer libraries means less to learn, fewer version conflicts, smaller attack surface. This issue is a small lesson in that philosophy.

### Why it's a problem
Extra test dependencies raise the barrier to entry for contributors (you must learn Autofac + Moq to write tests) without clear benefit. It's technical-debt cleanup.

### Why it's still open
It's been open a long time (issue #275 — very early in the project's life). Cleanup/style work is *never* urgent, the surface area is large (many test files), and it's easy to perpetually deprioritize behind features and bugs. That combination — low urgency, broad-but-shallow scope — is exactly what makes an issue linger for years *and* exactly what makes it a great community contribution.

### How you'd fix it
Incrementally. Find usages of Autofac and Moq across the test projects, and rewrite each to use the built-in DI container and hand-rolled test doubles, ensuring the tests still pass and read clearly. The maintainer explicitly notes the rewrites end up similar in size — so this is mechanical, not inventive.

### Step-by-step
1. Comment to claim it; propose doing it **in small batches** (e.g., a few test files or one test project per PR) so reviews stay manageable.
2. Search the test projects for `using Autofac` and `using Moq` to scope the work.
3. Pick one small cluster of tests. Replace Moq mocks with simple stub/fake classes implementing the needed interface; replace Autofac registrations with `ServiceCollection`.
4. Run the affected tests (`build.cmd/sh -test` or targeted `XunitMethodName`) — they must stay green.
5. PR the batch ("Part of #275"). Repeat. Each merged batch is a win and teaches you more of the codebase.

### Fit for you ★★★★★
**The best issue for learning the repo.** It pushes you through large amounts of real test code (your stated weakness: navigating big codebases), is pure C# with zero product-behavior risk, teaches DI and mocking from the inside, and is naturally incremental so you can start very small. Highly recommended as your codebase-onboarding PR.

---

## Issue #2847 — LettuceEncrypt is now archived

**Link:** https://github.com/dotnet/yarp/issues/2847 · **Type:** Docs / Sample · **Difficulty:** ★★☆☆☆

### What it is
YARP's documentation and the `ReverseProxy.LetsEncrypt.Sample` demonstrate automatic TLS certificates using the third-party library **LettuceEncrypt** (by natemcmaster). That library's repository has been **archived** (no longer maintained). From the issue: the docs reference it, *"as it happens"* it's now archived — so the guidance points users at a dead-end dependency.

### Concepts involved
- **ACME / Let's Encrypt**: the protocol for *automatically* obtaining and renewing free TLS certificates. The proxy proves it controls a domain (an ACME "challenge"), and a certificate authority issues a cert. LettuceEncrypt automates this inside an ASP.NET Core app.
- **TLS termination at the proxy** (Foundations Part 3 §6): the reverse proxy is the natural place to manage certificates, which is why YARP ships a sample for it.
- **Dependency lifecycle / supply-chain hygiene**: an archived dependency is a maintenance and security risk. Recognizing this and migrating off it is a real production skill.

### Why it's a problem
New users following the official Let's Encrypt guidance adopt an unmaintained library. At minimum the docs should warn; ideally the sample should migrate to a maintained alternative.

### Why it's still open
It requires a small judgment call (which maintained alternative to recommend) and isn't blocking core functionality, so it sits in `help wanted`. It's recent-ish and just needs someone to do the legwork of evaluating replacements.

### How you'd fix it
First, **ask in the issue what the maintainers want** — a docs warning, or a full sample migration, and to which library. Then either update the docs to flag the archive and point at the alternative, or rewrite the sample against the maintained library and verify it still obtains a cert.

### Step-by-step
1. Comment proposing an approach; get the maintainers' preferred replacement direction (this is a must — don't guess the target library).
2. Reproduce the current sample to understand what it does.
3. Update docs and/or sample accordingly; if migrating the sample, test that certificate acquisition still works end-to-end (or document how to test it).
4. PR "Fixes #2847."

### Fit for you ★★★★☆
Good Rung-2 pick. It touches **TLS/certificate automation** (your auth/security interest), is modest in scope, and exercises the real-world skill of *migrating off a dead dependency*. The only friction is needing maintainer input on the target — which is itself good practice at OSS collaboration.

---

## Issue #2291 — Document structured transforms with `IHttpForwarder`

**Link:** https://github.com/dotnet/yarp/issues/2291 · **Type:** Docs · **Difficulty:** ★★☆☆☆

### What it is
YARP has two usage modes (Components Part 02 §8): the full pipeline, and **direct forwarding** via `IHttpForwarder`. The transforms documentation explains structured transforms (the request/response rewriting system, Part 02 §7) *only in the context of the full pipeline*. The direct-forwarding docs show transforms only via deriving a custom `HttpTransformer`. From the issue: both docs *"talk about transforms and direct forwarding, but only in the context of deriving"* — there's no guidance on using the nicer **structured** transforms together with `IHttpForwarder`.

### Concepts involved
- **The two transform authoring styles**: deriving from `HttpTransformer` (low-level, override methods) vs. **structured transforms** (declarative/builder-based, Part 02 §7). The docs gap is that the convenient structured style isn't shown for the direct-forwarding path.
- **Direct forwarding** (`IHttpForwarder.SendAsync`) as a standalone engine for when your app picks the destination itself.

### Why it's a problem
Users doing direct forwarding don't realize they can reuse the structured transform system and instead hand-roll transformers. A docs/sample gap.

### Why it's still open
Docs polish for an advanced scenario; low urgency, `help wanted`.

### How you'd fix it
Add documentation (and ideally a small sample snippet) showing how to build a structured transform pipeline and use it with `IHttpForwarder`, bridging the two existing doc pages.

### Step-by-step
1. Claim it; confirm the intended audience/page.
2. Study how the transform builder produces an `HttpTransformer` and how `IHttpForwarder.SendAsync` accepts one.
3. Write a worked example wiring structured transforms into direct forwarding.
4. PR "Fixes #2291."

### Fit for you ★★★★☆
Solid Rung-2 docs task that *deepens your own* understanding of both transforms and direct forwarding (both core to the Components series). Requires enough comprehension to write a correct example — which is good for you — but no risky code changes.

---

## Issues #2037 & #1842 — Migration guides (Nginx → YARP, Ocelot → YARP)

**Links:** https://github.com/dotnet/yarp/issues/2037 · https://github.com/dotnet/yarp/issues/1842 · **Type:** Docs · **Difficulty:** ★★★☆☆

### What they are
Two requests for **migration guides** helping users move to YARP from popular alternatives:
- **#2037 (Nginx → YARP):** "Have tips and tricks for common scenarios — How-Tos; config translation." Users coming from Nginx need a map from Nginx config directives to YARP's route/cluster model.
- **#1842 (Ocelot → YARP, Kubernetes):** a user running **Ocelot** (a .NET API gateway) behind a K8s ingress wants a short guide to swap in YARP.

### Concepts involved
- **Comparative gateway architecture**: Nginx (a C-based, config-file-driven reverse proxy) and Ocelot (a .NET API gateway) express routing very differently from YARP. Writing the guide forces you to *map concepts across systems* — e.g., an Nginx `location` block → a YARP route; `upstream` → a cluster; `proxy_pass` → a destination. This is genuinely valuable system-design literacy.
- **Config translation**: the heart of both guides is a side-by-side "this in Nginx/Ocelot = this in YARP" table.

### Why they're a problem
Migration friction is a real adoption barrier. Good migration docs are high-leverage for the project but require cross-system expertise to write well.

### Why they're still open
Writing these *well* requires actually knowing Nginx or Ocelot config in depth, which narrows the pool of capable contributors. They're valuable but not blocking, so they wait for someone with the relevant background.

### How you'd fix it
Produce a guide with a concept-mapping table plus worked examples translating a representative config from the source system into YARP's JSON/code config.

### Step-by-step
1. Claim it; agree scope with maintainers (how comprehensive).
2. Build the concept-mapping table (directive → YARP equivalent).
3. Translate 2–3 realistic example configs end to end.
4. PR "Fixes #2037" / "#1842."

### Fit for you ★★★☆☆
A good pick **only if** you already know (or are willing to learn) Nginx/Ocelot config. The Nginx one is more broadly useful. Upside: writing a translation guide cements your understanding of the route/cluster/destination model better than almost anything. Downside: requires source-system expertise you may need to build first.

---

## Issue #1789 — How to add Swagger for the gateway

**Link:** https://github.com/dotnet/yarp/issues/1789 · **Type:** Docs / Q&A · **Difficulty:** ★★☆☆☆ · *(10 reactions — popular)*

### What it is
A frequently-wanted **how-to**: aggregate the **Swagger/OpenAPI** documentation of the backend services *behind* the proxy and expose it at the gateway, so the proxy presents a unified API surface. From the issue: *"Can you provide example using Swagger with proxy?"*

### Concepts involved
- **OpenAPI/Swagger**: a machine-readable description of a REST API (endpoints, schemas) that powers interactive docs and client generation — on your REST/API list.
- **API gateway aggregation**: a gateway often needs to *merge* multiple backends' API docs into one, and rewrite the server URLs in each spec so they point at the gateway, not the hidden backend. This is a real gateway pattern (and relates to path/host transforms, Part 02 §7).

### Why it's a problem
It's a very common need with no official example, so the question recurs. The popularity (10 reactions) shows demand.

### Why it's still open
It's partly a *support question* that the team would like answered with a sample/doc rather than a code change. It needs someone to build a clean, correct example — Swagger aggregation has fiddly URL-rewriting details that make a *good* example non-trivial.

### How you'd fix it
Create a sample (or doc) showing the proxy fetching backends' OpenAPI documents, optionally merging them, and serving a Swagger UI at the gateway with server URLs rewritten to the gateway address.

### Step-by-step
1. Claim it; confirm whether maintainers want a sample, docs, or both.
2. Stand up 1–2 backends with Swagger and a YARP gateway in front.
3. Expose/aggregate their OpenAPI specs at the gateway, handling the URL rewriting.
4. Document the approach; PR "Fixes #1789."

### Fit for you ★★☆☆☆
Educationally rich (OpenAPI + gateway aggregation are on your list), but the *good-example bar* is higher than it looks because of the spec-rewriting subtleties. A reasonable Rung-2/3 pick if the Swagger/OpenAPI angle excites you; otherwise prefer #1764/#275 first.

---

## Summary: the docs/infra ladder

| Start here | Then | If the domain interests you |
| --- | --- | --- |
| **#1764** (WebSocket keep-alive) — answer is handed to you | **#2847** (LettuceEncrypt) — TLS, light judgment | **#2037/#1842** (migration guides) — needs Nginx/Ocelot knowledge |
| **#275** (Autofac/Moq) — learn the repo, incremental | **#2291** (IHttpForwarder transforms) — deepens transforms | **#1789** (Swagger) — OpenAPI aggregation |

Do **#1764** and a first batch of **#275** to get two merged PRs and real familiarity, then move to Part 02's bug/feature issues.
