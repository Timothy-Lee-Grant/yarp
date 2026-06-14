# Code Evaluation — Part 00: How Professionals Structure a .NET Project

> **Purpose of this series.** You've built plenty of personal projects. This series shows you what *production .NET engineered by Microsoft* looks like up close, using YARP's real code, so that when you walk into a mid-level role at a big company you already recognize the craft and conventions. Part 00 is about everything *around* the code — the build system, project configuration, and the disciplines a serious repo enforces before a single feature is written. These are exactly the things personal projects skip and professional ones live by.
>
> **How to use it.** Open the referenced files in the repo as you read. The goal is recognition: "ah, *that's* why that's there." Each section ends with a **why it matters professionally** note tied to your goal of being job-ready.

---

## 1. The Layered Build System (`Directory.Build.props` / `.targets`)

In a personal project, build settings live in each `.csproj`, copy-pasted and drifting apart. YARP uses MSBuild's **directory-scoped property files**, which apply automatically to every project at or below their folder. There's a hierarchy:

```
/Directory.Build.props            ← repo-wide defaults (lang version, signing, determinism)
/src/Directory.Build.props        ← settings for all shipping libraries (packaging, docs)
/src/ReverseProxy/Yarp.ReverseProxy.csproj  ← per-project specifics only
```

The child explicitly imports its parent so the chain composes:

```xml
<!-- src/Directory.Build.props -->
<Import Project="$(MSBuildThisFileDirectory)..\Directory.Build.props" />
```

The repo-root `Directory.Build.props` sets things *once* for the whole solution:

```xml
<LangVersion>12.0</LangVersion>
<Deterministic>true</Deterministic>
<EmbedUntrackedSources>true</EmbedUntrackedSources>
<IncludeSymbols>true</IncludeSymbols>
<StrongNameKeyId>Microsoft</StrongNameKeyId>
```

**What each buys you:**

- **`Deterministic`** — the same source always produces a byte-for-byte identical binary. This is the bedrock of *reproducible builds*: a build on your laptop and on the CI server are provably the same artifact, which matters for security (supply-chain verification) and debugging.
- **`EmbedUntrackedSources` + `IncludeSymbols`** — ships debugging symbols and source info so customers can step into the library with a debugger (Source Link). Personal projects never bother; products must.
- **`StrongNameKeyId`** — the assembly is cryptographically **strong-named** (signed), required for some enterprise/GAC scenarios and for verifiable identity.

> **Why it matters professionally:** the first thing that surprises engineers joining a big company is that "the build" is a real, owned, sophisticated system. Knowing that settings cascade through `Directory.Build.props` means you won't be lost when you need to change a compiler flag for 40 projects at once. This is table-stakes fluency.

---

## 2. Multi-Targeting (`TFMs.props`)

Personal projects target one framework. Libraries shipped to the world must run on several. YARP centralizes its **Target Framework Monikers (TFMs)** in one file:

```xml
<!-- TFMs.props -->
<LatestDevTFM>net9.0</LatestDevTFM>
<ReleaseTFMs>net8.0</ReleaseTFMs>
<TestTFMs>net8.0;net9.0</TestTFMs>
```

And the library consumes them:

```xml
<!-- Yarp.ReverseProxy.csproj -->
<TargetFrameworks>$(ReleaseTFMs)</TargetFrameworks>
```

**The concept:** a single codebase is compiled separately for each listed framework. Tests run on *both* net8.0 and net9.0 (`TestTFMs`) to guarantee the library behaves on the LTS release customers depend on *and* the newest one. Centralizing the moniker list means a framework bump is a one-line change, not a 30-file find-and-replace.

> **Why it matters professionally:** "support matrix" thinking is foreign to hobby projects and central to product work. You'll be asked "does this run on the LTS?" and multi-targeting + a test matrix is the answer. Recognize `$(ReleaseTFMs)` and you understand the project's support commitment at a glance.

---

## 3. The Project File Is a Declaration of Intent (`Yarp.ReverseProxy.csproj`)

Read this small file like a résumé of professional choices:

```xml
<PropertyGroup>
  <TargetFrameworks>$(ReleaseTFMs)</TargetFrameworks>
  <Nullable>enable</Nullable>
  <IsAotCompatible>true</IsAotCompatible>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>  <!-- from src/Directory.Build.props -->
</PropertyGroup>

<ItemGroup>
  <InternalsVisibleTo Include="DynamicProxyGenAssembly2" Key="$(MoqPublicKey)" />
  <InternalsVisibleTo Include="Yarp.ReverseProxy.Tests" />
  <InternalsVisibleTo Include="Yarp.ReverseProxy.FunctionalTests" />
</ItemGroup>

<ItemGroup>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

Four professional signals worth dwelling on:

**`<Nullable>enable</Nullable>`** — turns on **nullable reference types**, the single most important modern-C# safety feature. The compiler now tracks which references can be `null` and warns you when you might dereference one. This is why you'll see `DestinationState?` (nullable) vs `DestinationState` (non-null) all over the code — those `?`s are *contracts the compiler enforces*. It eliminates a whole class of `NullReferenceException` bugs at compile time. **Adopt this habit immediately**; it's the expected default in any modern .NET shop.

**`<IsAotCompatible>true</IsAotCompatible>`** — the library promises to work with **Native AOT** (ahead-of-time compilation to a native binary with no JIT). This forbids certain runtime tricks (unbounded reflection, runtime code-gen) and is enforced by analyzers. You'll see `[DynamicallyAccessedMembers(...)]` attributes (in the DI extensions) precisely to satisfy AOT/trimming analysis. AOT-awareness is increasingly expected for cloud/container libraries.

**`<InternalsVisibleTo>`** — grants specific *other* assemblies access to this assembly's `internal` members. Two uses here: the test projects (so unit tests can test internals without making them public), and `DynamicProxyGenAssembly2` keyed by `$(MoqPublicKey)` (so the Moq mocking library can subclass internal types). This is the professional answer to "how do I test internal code without polluting my public API?" — you don't make it public, you whitelist your test assembly.

**`<FrameworkReference Include="Microsoft.AspNetCore.App" />`** — depends on the *entire ASP.NET Core shared framework* rather than dozens of individual NuGet packages. This is how ASP.NET-adjacent libraries avoid version-conflict hell.

> **Why it matters professionally:** a mid-level engineer is expected to *read a csproj and understand the product's posture* — its framework support, its safety settings, its testability strategy. This file tells you YARP is null-safe, AOT-ready, strongly-tested, and ASP.NET-native, before you read one line of C#.

---

## 4. The Public API Is a Guarded Boundary

Notice the access modifiers in the code you'll read: the vast majority of classes are **`internal sealed`**, while only the intended extension points are `public`. Examples:

```csharp
internal sealed class PowerOfTwoChoicesLoadBalancingPolicy : ILoadBalancingPolicy   // hidden
public interface ILoadBalancingPolicy { ... }                                        // exposed
```

This is deliberate **API surface minimization**. Everything public is a *forever promise* — once shipped, you can't change it without breaking customers (semantic versioning). So professionals expose the *minimum*: the interfaces you're meant to implement and the registration methods you're meant to call. Implementations stay `internal sealed` so the team can rewrite them freely between versions.

- **`internal`** → invisible outside the assembly (but visible to tests via `InternalsVisibleTo`).
- **`sealed`** → cannot be inherited. This is both a performance win (the JIT can devirtualize calls) and a design statement ("don't extend this; implement the interface instead").

> **Why it matters professionally:** the discipline of "public is a contract; default to internal" is a hallmark of library engineers and a frequent code-review comment at companies like Microsoft. In personal projects everything is `public` by reflex. Breaking that reflex is a concrete level-up.

---

## 5. Code Style Is Mechanically Enforced (`.editorconfig`, analyzers, `.globalconfig`)

The repo root has a large `.editorconfig` and `eng/CodeAnalysis.*.globalconfig` files. These encode the team's style and correctness rules — naming, `var` usage, brace placement, banned APIs, nullable strictness — and the compiler/analyzers **fail the build** when violated. There's even a `.markdownlint.json` for docs.

The point: **style is not a matter of opinion or PR bickering; it's automated.** Everyone's code looks the same because a machine enforces it. **Roslyn analyzers** (static analysis that runs during compilation) catch real bugs too — disposed-object misuse, AOT-incompatible patterns, etc.

> **Why it matters professionally:** at a big company you will *not* argue about tabs vs spaces; the `.editorconfig` decides and the CI gate enforces it. Knowing this exists means your first PR won't be a wall of style nits. Run the build locally, fix what the analyzers flag, *then* push.

---

## 6. Everything Reproducible: SDK Pinning (`global.json`) and Arcade

`global.json` pins the exact .NET SDK version. The `restore` script downloads *that* SDK into a repo-local `.dotnet/` folder. The `eng/common/` Arcade scripts standardize the build across all of Microsoft's .NET repos.

The principle is **hermetic, reproducible builds**: every developer and the CI server use the *identical* toolchain, pinned in source control. No "works on my machine" caused by SDK drift — a problem you know well from embedded toolchains.

> **Why it matters professionally:** "the build is code, pinned and reproducible" is a core DevOps value. Recognizing `global.json` + Arcade means you'll know *why* you must run `restore` before `build`, and why the repo ships its own SDK rather than using your system one.

---

## 7. The Mental Checklist You Should Internalize

When you join a professional .NET codebase, scan for these in order — YARP has all of them, and so will your employer's repo:

| Look for | In YARP | Tells you |
| --- | --- | --- |
| Layered `Directory.Build.props` | yes | How settings cascade; where to change them |
| Centralized TFMs + test matrix | `TFMs.props` | The support commitment |
| `<Nullable>enable</Nullable>` | yes | Null-safety is enforced; respect the `?`s |
| `<IsAotCompatible>` / trimming attrs | yes | Reflection is constrained; watch the analyzers |
| `internal sealed` by default | pervasive | Minimal public surface; extend via interfaces |
| `InternalsVisibleTo` tests | yes | How internals get tested |
| `.editorconfig` + analyzers + `.globalconfig` | yes | Style/correctness is automated; don't fight it |
| `global.json` + a `restore` step | yes | Reproducible, pinned toolchain |
| Deterministic + Source Link + symbols | yes | Debuggable, verifiable releases |

> **The big takeaway:** professional .NET projects front-load an enormous amount of *invisible discipline* — null-safety, reproducibility, enforced style, guarded API surface, AOT-readiness — before any feature exists. Personal projects skip all of it. Closing that gap in your own work (start with `Nullable enable`, `internal` by default, and an `.editorconfig`) is the fastest way to *look and operate* like a mid-level engineer.

Next: **Part 01 — The C# language features and performance techniques** YARP uses in its hot paths, with the real `StreamCopier`, `AtomicCounter`, and `ValueStringBuilder` code.
