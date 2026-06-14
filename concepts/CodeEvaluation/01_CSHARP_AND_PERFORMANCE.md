# Code Evaluation — Part 01: C# Language Features & High-Performance Techniques

> This is where YARP will most expand you as a programmer. A reverse proxy lives or dies on throughput, so its hot paths use the *advanced, performance-oriented* corner of C# that most application developers never touch — and that big-company interviewers love to probe. Coming from embedded C/C++, much of this will feel familiar (manual buffers, avoiding allocations, memory layout) which is an advantage. We'll use YARP's real code to teach each technique.
>
> **The mental frame:** in a garbage-collected runtime, the enemy of throughput is **allocation** (every `new` object eventually costs a GC pause) and **copying**. The techniques below all serve one goal: *do the work without allocating or copying*.

---

## 1. `Span<T>` and `Memory<T>`: Windows Over Memory Without Copying

`Span<T>` is the most important modern-C# type to understand. It's a **view** over a contiguous block of memory — an array, part of an array, stack memory, or unmanaged memory — that lets you read/write it **without copying and without knowing where it lives**. Think of it as a safe, bounds-checked C pointer+length.

From `StreamCopier` (the proxy's hottest loop):

```csharp
read = await input.ReadAsync(buffer.AsMemory(), cancellation);
...
await output.WriteAsync(buffer.AsMemory(0, read), cancellation);
```

`buffer.AsMemory(0, read)` creates a `Memory<byte>` describing "bytes 0..read of this buffer" — **no new array, no copy**. The write consumes exactly those bytes.

**`Span<T>` vs `Memory<T>` — the distinction you must know:**

| | `Span<T>` | `Memory<T>` |
| --- | --- | --- |
| Where it can live | stack only (it's a `ref struct`) | anywhere (heap, fields, async state) |
| Can be used across `await`? | **No** | **Yes** |
| Use it for | synchronous, tight loops | async I/O, storing for later |

That's why `StreamCopier` uses `Memory<byte>` — it crosses `await` boundaries (async I/O). Synchronous parsing code (like `ValueStringBuilder`, §5) uses `Span<char>`. This rule — *Span for sync, Memory for async* — trips up many intermediate developers; knowing it cold marks you as fluent.

> **Embedded parallel:** `Span<T>` is the managed-world version of passing `(ptr, len)` instead of copying a buffer. You already think this way in C; C# now lets you do it *safely*.

---

## 2. `ArrayPool<T>`: Reusing Buffers Instead of Allocating Them

Allocating a 64 KB buffer per request, for thousands of requests per second, would bury the GC. Instead YARP **rents** buffers from a shared pool and returns them:

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(DefaultBufferSize);   // 65536
try
{
    // ... use buffer ...
}
finally
{
    if (buffer is not null)
    {
        ArrayPool<byte>.Shared.Return(buffer);                 // give it back
    }
}
```

`ArrayPool<T>.Shared` is a process-wide pool of reusable arrays. `Rent` gives you one (possibly larger than asked); `Return` puts it back for the next caller. The buffer is **reused** across millions of requests instead of allocated-then-collected each time. This converts a stream of GC garbage into a small fixed set of long-lived arrays.

Notice the **discipline** the code shows around it:

```csharp
// Take care not to return the same buffer to the pool twice in case zeroByteReadTask throws
var bufferToReturn = buffer;
buffer = null;
ArrayPool<byte>.Shared.Return(bufferToReturn);
```

Pooling is powerful but *dangerous*: return a buffer twice, or use it after returning, and you get memory corruption-class bugs (two callers writing the same array). The careful `buffer = null` dance and the `if (buffer is not null)` in `finally` exist to make double-return impossible. **This rent/return ownership rigor is exactly the manual-memory discipline you know from C** — applied in a GC language for speed.

> **Interview gold:** "How would you avoid GC pressure in a high-throughput service?" → pooled buffers via `ArrayPool<T>`, reused across requests, with strict return-once ownership. This single example answers it.

---

## 3. The Zero-Byte Read Trick: Allocating Even Later

This is a subtle, beautiful optimization most developers have never seen:

```csharp
// Issue a zero-byte read to the input stream to defer buffer allocation until data is available.
var zeroByteReadTask = input.ReadAsync(Memory<byte>.Empty, cancellation);
if (zeroByteReadTask.IsCompletedSuccessfully) { ... }
else
{
    // return the rented buffer to the pool while we WAIT for data
    ArrayPool<byte>.Shared.Return(bufferToReturn);
    await zeroByteReadTask;        // park here with NO buffer held
    buffer = ArrayPool<byte>.Shared.Rent(DefaultBufferSize);  // re-rent only when data is ready
}
```

**The insight:** a proxy may hold tens of thousands of *idle* connections waiting for data. If each idle connection held a 64 KB buffer, that's gigabytes of RAM doing nothing. By issuing a **zero-byte read** (which completes only when data is actually available) and *releasing the buffer while waiting*, YARP holds buffers only for connections with data ready to move. At scale this is an enormous memory saving.

> **Why it matters:** this is the kind of optimization that separates "works" from "works at 100k connections." Understanding *why* it exists — buffer-per-idle-connection is the enemy — is more valuable than memorizing it.

---

## 4. `ValueTask<T>`: Avoiding Allocation in the Async Hot Path

Every `async` method that returns `Task<T>` allocates a `Task` object. For a method called millions of times that *usually completes synchronously* (data already buffered), that allocation is wasteful. `ValueTask<T>` avoids it: it's a `struct` that holds *either* a synchronous result *or* a `Task` when it truly needs to go async.

YARP returns `ValueTask` throughout the forwarder:

```csharp
public ValueTask<ForwarderError> SendAsync(...)
public static ValueTask<(StreamCopyResult, Exception?)> CopyAsync(...)
```

When the operation completes synchronously (common case), **no `Task` is allocated** — the result rides inside the struct. Only when it genuinely suspends does it cost more.

**The catch you must know:** a `ValueTask` may be awaited **only once**, and you must not access its `.Result` before it completes. That's why you'll see careful handling like:

```csharp
if (zeroByteReadTask.IsCompletedSuccessfully)
{
    _ = zeroByteReadTask.Result;   // consume it exactly once
}
```

> **Rule of thumb (interview-ready):** return `ValueTask` from hot-path async methods that often complete synchronously; return `Task` for everything else. Never await a `ValueTask` twice.

---

## 5. `ref struct` + `stackalloc`: Building Strings Without the Heap

`ValueStringBuilder` assembles strings on the **stack** for short inputs, only falling back to a pooled heap array when they grow large:

```csharp
internal ref partial struct ValueStringBuilder
{
    public const int StackallocThreshold = 512;
    private char[]? _arrayToReturnToPool;   // rented only if we overflow the stack buffer
    private Span<char> _chars;               // the current backing store (stack or pooled)
    private int _pos;
```

Used (conceptually) like:

```csharp
Span<char> initialBuffer = stackalloc char[256];   // lives on the stack — zero heap allocation
var sb = new ValueStringBuilder(initialBuffer);
sb.Append(...);
string result = sb.ToString();                      // copies out once; disposes the buffer
```

Three advanced concepts in one type:

- **`stackalloc`** allocates a buffer on the *stack*, not the heap — instant, no GC, automatically freed when the method returns. (You know this as a stack array in C.)
- **`ref struct`** is a struct the compiler *guarantees* stays on the stack — it can never be boxed, stored in a field, or captured across `await`. That guarantee is what makes it safe to hold a `Span<char>` over `stackalloc` memory.
- **Graceful overflow**: if the string exceeds the stack buffer, it rents from `ArrayPool` and copies over — so it's fast in the common (small) case and still correct in the rare (large) case.

The `Append` method even shows micro-optimization:

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void Append(char c)
{
    var pos = _pos;
    var chars = _chars;
    if ((uint)pos < (uint)chars.Length)   // single unsigned compare = bounds check
    {
        chars[pos] = c;
        _pos = pos + 1;
    }
    ...
}
```

`[MethodImpl(AggressiveInlining)]` asks the JIT to inline this tiny method into its callers (no call overhead). The `(uint)pos < (uint)chars.Length` trick does a bounds check with a *single* unsigned comparison (a negative `pos` becomes a huge `uint`, failing the check) — a classic performance idiom you'll see throughout the .NET runtime.

> **Embedded parallel:** this is stack buffers + manual bounds checks + forced inlining — your daily bread in firmware, now expressed in C#. You're well-positioned to *get* this faster than a typical web developer.

---

## 6. `struct` Value Types to Avoid Allocation: `ValueStopwatch`

A normal `Stopwatch` is a class — using one per request allocates. `ValueStopwatch` is a `struct` that measures elapsed time with zero allocation:

```csharp
internal struct ValueStopwatch
{
    private readonly long _startTimestamp;
    private ValueStopwatch(long startTimestamp) => _startTimestamp = startTimestamp;

    public static ValueStopwatch StartNew() => new ValueStopwatch(Stopwatch.GetTimestamp());

    public TimeSpan Elapsed
    {
        get
        {
            if (_startTimestamp == 0)   // default(ValueStopwatch) guard
                throw new InvalidOperationException("...uninitialized...");
            var delta = Stopwatch.GetTimestamp() - _startTimestamp;
            return new TimeSpan((long)(_timestampToTicks * delta));
        }
    }
}
```

It stores a single `long` (a raw timestamp) and computes elapsed time on demand. Because it's a `struct`, declaring one is free — it lives inline in its containing frame/object, no heap object, no GC. Note the **defensive guard**: a `default(ValueStopwatch)` has `_startTimestamp == 0`, which is detected and rejected, because value types can be created uninitialized (you can't force a constructor to run on a struct).

> **The lesson:** when a small, short-lived helper would otherwise allocate, make it a `struct`. The trade-off (structs are copied by value, can be created uninitialized) requires care — hence the guard.

---

## 7. Lock-Free Counters: `AtomicCounter` and `Interlocked`

The load balancer needs each destination's live in-flight request count, read and written by many threads at once. A lock would serialize the hot path. Instead, `AtomicCounter` uses **atomic CPU instructions**:

```csharp
internal sealed class AtomicCounter
{
    private int _value;

    public int Value
    {
        get => Volatile.Read(ref _value);      // guaranteed to see latest value
        set => Volatile.Write(ref _value, value);
    }

    public int Increment() => Interlocked.Increment(ref _value);   // atomic ++ across threads
    public int Decrement() => Interlocked.Decrement(ref _value);
    public void Reset()     => Interlocked.Exchange(ref _value, 0);
}
```

- **`Interlocked.Increment`** performs `++` as a *single indivisible hardware operation*, so two threads incrementing simultaneously can never lose an update (the classic race). No lock needed.
- **`Volatile.Read/Write`** prevents the compiler/CPU from caching or reordering the read/write, so one thread's update is promptly visible to others (memory-ordering correctness).

> **Interview relevance:** "How do you increment a shared counter from many threads without a lock?" → `Interlocked`. "Why `Volatile`?" → visibility/ordering across cores. This is core concurrency literacy, and YARP shows it in 40 clean lines.

---

## 8. Putting Allocation-Avoidance Together: the `StreamCopier` Loop

Step back and see how the techniques compound in *one* method (Components Part 02 §8). The proxy's central byte-pump:

1. **Rents** a 64 KB buffer from `ArrayPool` (no allocation).
2. Issues a **zero-byte read** to defer buffer use until data is ready (memory saving at scale).
3. Reads into a **`Memory<byte>`** view (no copy).
4. Returns a **`ValueTask`** with a **tuple** `(StreamCopyResult, Exception?)` — structured result without throwing for control flow, without allocating a result object.
5. **Returns** the buffer in `finally`, with **return-once** safety.
6. Optionally records timing via the lightweight telemetry helper, gated on whether anyone's listening.

```csharp
return (StreamCopyResult.InputError,
        new InvalidOperationException("More bytes received than the specified Content-Length."));
```

That **tuple return** (`(StreamCopyResult, Exception?)`) is itself a pattern: instead of *throwing* for an expected failure (exceptions are expensive and meant for the *exceptional*), it returns a structured outcome the caller pattern-matches on. Using return values for expected failures and exceptions only for truly unexpected ones is a professional habit.

> **The synthesis:** every line is chosen so the per-request, per-chunk cost is *near zero allocation*. This is what "high-performance .NET" actually means in practice — not clever algorithms, but relentless elimination of allocation and copying on the hot path.

---

## 9. A Few More Idioms You'll See (quick hits)

| Idiom (real YARP usage) | What it does | Why pros use it |
| --- | --- | --- |
| `ConditionalWeakTable<ClusterState, AtomicCounter>` (RoundRobin) | Attaches per-cluster state without leaking memory when the cluster is GC'd | Associates data with objects you don't own, GC-safely |
| `(offset & 0x7FFFFFFF) % count` (RoundRobin) | Masks the sign bit to keep an index non-negative after `int` overflow | Correctness under integer wraparound — firmware thinking |
| `Debug.Assert(input is not null)` | Checks invariants in debug builds only (free in release) | Document + verify assumptions without release cost |
| `is not null` / `is { }` pattern matching | Modern null/type checks | Readability; the idiomatic modern style |
| `??=` (`RequestUri ??= ...`) | Assign only if currently null | Concise conditional initialization |
| `[MethodImpl(AggressiveInlining)]` | Force-inline a tiny hot method | Remove call overhead in loops |

The `ConditionalWeakTable` one is especially worth studying: `RoundRobinLoadBalancingPolicy` must keep a rotating counter *per cluster*, but it must not keep clusters alive forever (a memory leak) once they're removed from config. A `ConditionalWeakTable` holds its keys *weakly*, so when a cluster is collected its counter vanishes too. That's a sophisticated lifetime-management choice most developers don't know exists.

---

## 10. Your Practice Path

To convert reading into skill:

1. **Re-implement `ValueStopwatch` and `AtomicCounter` from memory.** They're small and teach `struct` semantics and `Interlocked`.
2. **Write a tiny stream-copy method** using `ArrayPool` + `Memory<byte>` + `ValueTask`, with correct rent/return. This single exercise touches half this document.
3. **Turn on `<Nullable>enable</Nullable>` in one of your projects** and fix every warning. You'll feel the design pressure that produces YARP's `?`-annotated APIs.
4. **Benchmark it.** Add [BenchmarkDotNet] (the standard .NET micro-benchmark library; YARP has a `BenchmarkApp`) and measure allocations with and without pooling. *Seeing* the allocation drop is what makes it stick.

> **Bottom line:** your embedded background is a genuine edge here — buffer ownership, stack allocation, bounds checks, and atomics are your native language. YARP shows you the *idiomatic C# expression* of instincts you already have. Master this document and you'll out-perform many web-first developers on exactly the topics big-company systems teams care about.

Next: **Part 02 — the design patterns** that organize all this code: DI, Strategy, Factory, Builder, Middleware, Null-Object, Decorator, and Adapter.
