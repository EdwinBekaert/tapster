# Building blocks

The data structures behind the packet path: the observation ring, the reorder window, slots,
the new-flow table and timeouts. They share three rules:

- **Allocated once, up front.** Sizes come from configuration. After start-up nothing on the
  packet path allocates, so the garbage collector never runs because of it.
- **Owned by one thread.** Each tap owns its structures. The only structure two threads touch is
  the observation ring, and it has exactly one writer and one reader.
- **Indices, not references.** Structures point at each other with `int` indices into arrays of
  structs, so the garbage collector has nothing to trace and nothing is boxed.

## Overview

| Need | Building block | Not this, because |
|---|---|---|
| Bytes held past the call (reorder window, slots, orphans) | **Arena**: one pinned block cut into fixed-size chunks, with a free list of chunk indices | `ArrayPool<byte>.Shared` rounds sizes up, caches per thread and allocates new arrays when empty: not bounded |
| A frame held in the arena | **`FrameRef`** struct: chunk index, length, capture timestamp, context | holding a `Span` is impossible past the call; a `byte[]` per frame allocates |
| Reorder window per ERSPAN session | **Min-heap by sequence number**: `PriorityQueue<FrameRef, uint>` with a custom comparer, capacity reserved up front | sorting a list per packet |
| New-flow table, slot lookup | **`Dictionary<FlowKey, int>`** with capacity reserved, read and written through `CollectionsMarshal.GetValueRefOrAddDefault`; later a custom open-addressing table if benchmarks ask for it | `ConcurrentDictionary`: locks and allocations, and not needed with one owner |
| Covered byte ranges in a slot | **`[InlineArray(4)]`** of `(uint Start, uint End)` | a `List<T>` per slot |
| Timeouts for slots, orphans, flow entries | **Timing wheel**: buckets of capture time, entries linked by index | scanning every slot on every packet |
| Orphan list | **Fixed ring of `FrameRef`**, oldest overwritten | an unbounded queue |
| Observations from a tap to the fold | **Single-producer, single-consumer ring** of structs | `Channel<T>` and `ConcurrentQueue<T>`: built for many producers, with locks or segment allocations |

## Arena

```csharp
sealed class Arena
{
    private readonly byte[] _memory;      // GC.AllocateUninitializedArray<byte>(n, pinned: true)
    private readonly int[] _free;         // stack of free chunk indices
    private int _freeCount;
    public readonly int ChunkSize;

    public bool TryRent(out int chunk);   // false when empty: the caller evicts or drops
    public void Return(int chunk);
    public Span<byte> Chunk(int chunk, int length) => _memory.AsSpan(chunk * ChunkSize, length);
}
```

- Pinned on the pinned object heap, so the address never moves and interop can use it. On an
  embedded host, `NativeMemory.AlignedAlloc` works the same way.
- Two arenas per tap: one with frame-sized chunks (2 KiB, or 9 KiB for jumbo frames) for the
  reorder window and orphans, one with message-sized chunks (16 KiB) for slots.
- `Chunk(...)` returns a span, so the same protocol views read held bytes and live bytes.
- **With `AF_XDP`, the shared packet memory is already an arena.** A held frame can stay where
  the card wrote it, and its address goes back to the kernel only when the frame is released.
  That is zero-copy reordering. With `AF_PACKET` (TPACKET_V3), blocks go back to the kernel as a
  whole, so held frames are copied into the arena.

## Sequence numbers wrap

GRE and TCP sequence numbers are 32 bits and wrap around. Every comparison uses serial-number
arithmetic (RFC 1982), never `<` on the raw values:

```csharp
static bool IsBefore(uint a, uint b) => (int)(a - b) < 0;
static uint Distance(uint from, uint to) => to - from;    // offset in a slot
```

The reorder heap's comparer and every slot offset use these.

## Reorder window

```
 arrive:  seq 7  seq 5  seq 6  seq 9          heap (min seq on top): 5 6 7 9
 release: when top is the next expected seq ─► 5 6 7
          or when top has waited longer than the window ─► 9 (8 counted as lost)
```

- One heap per (worker, ERSPAN session). Capacity is reserved up front, so `Enqueue` never
  allocates.
- Release is by time as well as by sequence, because a worker sees only its own share of a
  session's sequence numbers (see [reassembly](reassembly.md#flows-must-stay-on-one-tap)).
- Window length and size are configured; a full heap releases its oldest entry early.

## Flow keys and lookups

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
readonly record struct FlowKey(IpAddress Low, IpAddress High, ushort LowPort, ushort HighPort,
                               byte Protocol, SegmentId Segment) : IEquatable<FlowKey>;
```

- Canonical: the lower address and port come first, so both directions find the same entry.
- `GetHashCode` mixes the packed fields with a fast multiply-and-shift hash, **keyed with a
  random seed chosen at start-up**. The five-tuples come from strangers; without the seed,
  anyone could pick tuples that collide and make every lookup slow (see
  [threat model](threat-model.md#exhausting-resources)).
- `CollectionsMarshal.GetValueRefOrAddDefault` returns a `ref` to the value, so "find or add
  then update" is one lookup.
- The table has a fixed capacity. When it is full, the timing wheel's oldest entries are
  evicted first. It never grows past its configured size.
- If profiling shows the dictionary's indirection (buckets, then entries) matters, it is
  replaced by an open-addressing table over one array of structs, with linear probing and
  backward-shift deletion. The interface stays the same.

## Slot

```csharp
struct Slot
{
    public FlowKey Flow;
    public uint StartSeq;          // initial sequence number + 1
    public int TotalLength;        // -1 until the header segment arrives
    public int Chunk;              // arena chunk holding the bytes
    public Ranges Covered;         // [InlineArray(4)] of (uint Start, uint End)
    public long LastActivityNs;    // capture time
    public int NextInWheel;        // timing wheel link, -1 = none
}
```

Slots live in one `Slot[]` per tap; the flow table maps a `FlowKey` to an index into it.

## Timing wheel

- An array of buckets, each a span of capture time (for example 64 buckets of 50 ms). Each
  bucket holds the index of the first entry; entries link to the next with `NextInWheel`.
- Adding or moving an entry is constant time. As capture time advances past a bucket, its
  entries are checked and the expired ones are released.
- Driven by capture timestamps, so replaying a recorded capture expires slots exactly as live
  traffic did.

## Observation ring

```csharp
sealed class ObservationRing
{
    private readonly Observation[] _items;  // power-of-two length
    private PaddedLong _head;               // written by the tap only
    private PaddedLong _tail;               // written by the fold only
}
```

- One writer (the tap) and one reader (the fold), so `Volatile.Read` and `Volatile.Write` on
  head and tail are enough; no locks and no interlocked operations.
- Head and tail sit on separate cache lines (`[StructLayout(LayoutKind.Explicit, Size = 128)]`)
  so the two threads do not slow each other down.
- `Observation` is a fixed-size struct (a tag plus a union of observation types), so the ring is
  one contiguous array.
- Full means drop the observation and count it; the writer never waits.

## Checks

- Each building block has a BenchmarkDotNet run asserting zero allocated bytes after warm-up.
- Struct sizes (`FlowKey`, `Slot`, `Observation`, `FrameRef`) are asserted in tests, like the
  layout checks of a C header, so an added field cannot silently grow them.
- Sequence-number code is tested across the wrap from `0xFFFFFFFF` to `0`.
