# Performance

## What "high performance" means here

Device identification needs about one percent of the packets: ARP, ND, DHCP, LLDP/CDP, mDNS,
DNS, TCP SYN and TLS ClientHello. So the work splits in two:

- **Every packet:** parse to layer 4, update one flow counter, return. This is the path to
  make cheap.
- **Interesting packets:** full parse, fingerprint, copy out observations. Rare, so it may
  cost more.

The fastest packet is the one dropped early. Where the capture method allows it (a BPF filter
in the kernel, an XDP program), uninteresting packets can be counted and dropped before they
reach Tapster at all.

## Rough numbers

| Link | Average frame ~600 bytes | Minimum frames (64 bytes) |
|---|---|---|
| 1 Gbit/s | ~0.2 million packets/s | 1.5 million packets/s |
| 10 Gbit/s | ~2 million packets/s | 14.9 million packets/s |
| 100 Gbit/s | ~20 million packets/s | 149 million packets/s |

Header-only parsing in C# with no allocation runs in the tens of millions of packets per second
per core. The limits are elsewhere: getting packets into the process, and cache misses on
large state tables.

## Per consumer

### Embedded

`tapster_observe` runs inside the host's forwarding path, so its cost is added to every
forwarded packet. The budget is set by the host, not by Tapster; the design goal is that the
common path costs one parse to layer 4 and one counter update, with no lock, allocation or
system call. Memory is bounded and configured up front (ring sizes, table sizes), because an
embedded device cannot grow without limit.

### Forwarded traffic (ERSPAN collector)

| Capture method | Fits |
|---|---|
| `AF_PACKET` with TPACKET_V3 and `PACKET_FANOUT` | 1–10 Gbit/s; packets handed over in memory-mapped blocks, spread over cores |
| `AF_XDP` | 10 Gbit/s and up; frames delivered from the card into a shared ring |
| DPDK | Possible through interop; heavier to deploy (dedicated cores, huge pages) |

Fanout and receive-side scaling hash the outer headers, which are the same for a whole ERSPAN
session. Spreading the load needs steering by the inner five-tuple: an XDP program choosing the
`AF_XDP` socket, or a receive stage that decapsulates and dispatches (see
[reassembly](reassembly.md#flows-must-stay-on-one-tap)).

At 40–100 Gbit/s, the card's receive-side scaling, many cores and filtering in the kernel are
all required.

### Single device

Ordinary capture is enough; a single machine's traffic is small.

## What mirrors do to the data

A mirror is not a perfect copy, and the design accepts that:

- **Loss.** A mirror destination port that is oversubscribed drops packets silently.
  Observations are evidence, not a complete record; a missed DHCP exchange is caught at the
  next renewal.
- **Reordering.** Packets can arrive out of order. The ERSPAN shell restores mirror order from
  the GRE sequence number; what remains is handled per flow (see
  [reassembly](reassembly.md#when-the-first-segment-is-not-the-first-to-arrive)).
- **Duplicates.** One packet can arrive twice: mirrored on ingress and egress, or by two
  sessions. Flow counters deduplicate on a short window keyed by a hash of the inner frame.
- **Truncation.** Mirrors and snap lengths cut frames. Parsers read what is there; the
  context carries the original length.
- **Time.** ERSPAN type III carries the switch's timestamp. Otherwise the capture timestamp is
  used, never the clock when the packet happened to be processed.

## Measuring

- BenchmarkDotNet runs over recorded captures, asserting **zero allocated bytes** on the
  packet path.
- Replay at line rate with `tcpreplay` against the collector shell, measuring packets per
  second per core and dropped observations.
- Replay of split, reordered and duplicated segments, checking that slot memory stays within
  its pool.
- A fuzzer feeds malformed and truncated frames to every view; any exception or out-of-range
  read is a bug.
