# Architecture

Tapster is one core with no I/O and three thin shells around it. Each shell turns its own
input into the same thing, a frame plus its context, and the core turns that into small
value-type observations. Everything after the core is built from those observations.

```
 embedded host               ERSPAN collector            single device
 native C API (NativeAOT),   AF_PACKET / AF_XDP; peels   capture on own interface
 the host embeds it          GRE + ERSPAN -> inner frame (own traffic + broadcast)
       │ frame* + ctx               │ frame + ctx                │ frame + ctx
       └───────────────┬────────────┴────────────────┬───────────┘
                       ▼
 core: Observe(ReadOnlySpan<byte> frame, in FrameContext ctx)    no I/O, no allocation
       views: Ethernet → VLAN → IP → UDP/TCP → DhcpPacket, TlsClientHello …
                       │ copy-out: small value-type observations
                       ▼
 observation ring, one per calling thread (single writer, drops + counts when full)
                       │
       ┌───────────────┼─────────────────────┐
       ▼               ▼                     ▼
 registry          edge store             evidence log
 segments,         configured | observed  per-interface history
 interfaces,           │                         │
 L3 bindings           ▼                         ▼
                 dependency map (v2)      identity judge (v2)
```

## The core contract

```csharp
public static void Observe(ReadOnlySpan<byte> frame, in FrameContext context, ObservationWriter writer);
```

- `frame` starts at the Ethernet header. It is valid only during the call; the core keeps no
  reference to it.
- `context` says where the frame came from. The core never guesses this from the frame.
- `writer` appends observations to the calling thread's ring.

`FrameContext` carries what the frame alone cannot tell:

| Field | Why it is needed |
|---|---|
| `TimestampNs` | Capture time. Every "first seen" and "last seen" uses it, never the clock at processing time. |
| `SensorId` | Which Tapster instance saw it. Part of the segment (see [identity model](identity-model.md)). |
| `Port` | Interface index on the host, or the ERSPAN source port. The strongest signal for MAC rotation. |
| `Direction` | Ingress, egress or unknown. Mirrors often deliver both, which matters for deduplication. |
| `VlanId` | Set when the network card or host stripped the tag before Tapster saw the frame. |
| `Flags` | `VlanStripped`, `Truncated`, … |
| `OriginalLength` | Length on the wire when the frame was truncated (ERSPAN and capture snap lengths). |

## The copy-out boundary

Everything above the boundary is a view over the caller's bytes; everything below is a value
that owns its data. Values cross threads; views and offsets never do, because the buffer they
point into is reused as soon as `Observe` returns.

Observations are small `readonly struct`s, for example:

```csharp
readonly record struct ArpBinding(SegmentId Segment, MacAddress Mac, IPv4Address Ip, long TimestampNs);
readonly record struct DhcpExchange(SegmentId Segment, MacAddress ClientMac, DhcpMessageType Type,
                                    IPv4Address ServerId, IPv4Address YourIp, uint LeaseSeconds,
                                    FingerprintId Fingerprint, long TimestampNs);
readonly record struct FlowCounter(FiveTuple Flow, uint Packets, ulong Bytes, long TimestampNs);
```

Variable-length values (hostnames, vendor class strings, option lists) are interned once into
a table and referenced by id, so a repeated hostname costs no allocation after the first time.

## The shells

### Embedded

The host (router firmware, a network appliance) loads Tapster as a native library built with
.NET NativeAOT, so it needs no .NET runtime. The host does the embedding; Tapster provides
the C API. The API follows the memory-ownership conventions of a plain C library: the frame
is caller-owned and read only during the call.

```c
/* PROPOSED - tapster.h */
typedef struct tapster tapster_t;          /* one per process: registry and settings   */
typedef struct tapster_tap tapster_tap_t;  /* one per forwarding thread: its own ring  */

typedef struct tapster_frame_ctx {
    uint64_t timestamp_ns;
    uint32_t sensor_id;
    uint32_t port;
    uint16_t vlan_id;        /* 0 = none */
    uint8_t  direction;      /* 0 unknown, 1 ingress, 2 egress */
    uint8_t  flags;          /* TAPSTER_CTX_VLAN_STRIPPED, TAPSTER_CTX_TRUNCATED */
    uint32_t original_len;
} tapster_frame_ctx_t;       /* 24 bytes, no padding */

tapster_status tapster_open(const tapster_config_t* config, tapster_t** out);
tapster_status tapster_tap_open(tapster_t* tapster, tapster_tap_t** out_tap);
tapster_status tapster_observe(tapster_tap_t* tap, const uint8_t* frame, size_t len,
                               const tapster_frame_ctx_t* ctx);
void           tapster_tap_close(tapster_tap_t* tap);
void           tapster_close(tapster_t* tapster);
```

`tapster_observe` runs inside the host's forwarding path, so it never blocks, locks or
allocates. A tap is used by one thread at a time; a host with four forwarding threads opens
four taps. Because nothing on this path allocates, the NativeAOT garbage collector is never
triggered by it.

Open question: how the embedded host gets results out (a drain callback, a periodic export
file, or a small local API). This does not affect the core.

### Forwarded traffic (ERSPAN collector)

A Linux server receives ERSPAN from one or more switches. The shell captures with
`AF_PACKET` (TPACKET_V3, with fanout over cores) or `AF_XDP`, removes the outer IP, GRE and
ERSPAN headers, and fills `FrameContext` from them: the switch, the ERSPAN session id and the
source port index. The core never knows the frame was tunnelled. Decapsulation rules are in
[parsing](parsing.md#encapsulation).

Two more jobs belong to this shell:

- **Restore mirror order.** ERSPAN types II and III number their packets per session; a short
  reorder window releases them in order (see
  [reassembly](reassembly.md#when-the-first-segment-is-not-the-first-to-arrive)).
- **Steer by the inner flow.** Every packet of a session has the same outer headers, so
  hashing on them would put a whole session on one core. The shell hashes the inner five-tuple
  to choose the tap.

### Single device

Captures on the machine's own interface. It sees only its own traffic plus broadcast and
multicast, which still includes ARP, DHCP, mDNS and LLDP: enough to identify most neighbours.

## Messages split over several packets

A view that finds a message continuing in the next packet asks the tap to follow that flow.
The tap streams, buffers into a fixed slot, or hands the packet to the background fold;
the reassembled message is a span, so the same views read it. See
[reassembly](reassembly.md).

## Threading

- Each calling thread has its own ring and its own reassembly slots, with a single writer. No locks and no shared writes on
  the packet path.
- The ERSPAN collector spreads flows over cores by flow hash (`PACKET_FANOUT` or the card's
  receive-side scaling), so one flow always lands on the same core.
- A background thread drains the rings and folds observations into the registry, the edge
  store and the evidence log. It uses capture timestamps, so the order packets arrived in never
  changes the result. This side may allocate and lock; it runs at the rate of interesting
  events, not the rate of packets.
- When a ring is full, the newest observations are dropped and a per-tap counter goes up.
  Losing an observation is acceptable; slowing the host's forwarding is not.

## Pure core, thin shells

The core is plain C# with no I/O, no clock and no threads of its own. That makes it testable
with recorded packet captures on any platform, and lets all three shells share it unchanged.
Capture, decapsulation, clocks and storage live in the shells.
