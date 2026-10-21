# Messages split over several packets

Most of what Tapster needs fits in one packet: an ARP reply, a DHCP message, a TCP SYN, an
mDNS answer. Some messages do not:

- **TCP segments.** A TLS ClientHello used to fit in one segment. With post-quantum key shares
  (X25519MLKEM768, on by default in current browsers) it is often larger than one segment. HTTP
  request headers, SMB session setup, Kerberos and LDAP over TCP can also span segments.
- **QUIC.** The ClientHello is carried in CRYPTO frames, encrypted with keys anyone can derive
  from the connection id, and may span several Initial packets.
- **IP fragments.** Rare for the protocols Tapster reads, but large DNS answers over UDP can be
  fragmented.

Splitting is the exception, so it must cost nothing when it does not happen, and it must never
let the packet path grow without limit.

## The idea: reassembly produces a span

Every protocol view reads a `ReadOnlySpan<byte>`. A reassembled message is just another span,
over a buffer Tapster owns instead of the caller's. So the views, the copy-out boundary and the
observations stay exactly as they are; reassembly is a layer underneath them.

```
 segment 1 ─► view reads TLS record header: length 1,830, only 1,380 here
                 │  NeedMore(1,830)
                 ▼
            tap opens a slot for this flow, copies 1,380 bytes in at offset 0
 segment 2 ─► flow lookup hits the slot, copies 450 bytes in at offset 1,380
                 │  slot complete
                 ▼
            the same TlsClientHello view runs over the slot's span ─► observations
            slot returns to the pool
```

## A view says what it needs

Each view's parse returns one of:

| Result | Meaning |
|---|---|
| `Done` | Everything needed was in this packet. The common case. |
| `NotInterested` | Nothing to copy out from this flow. |
| `NeedMore(totalLength)` | A length-prefixed message continues; the header said how long it is. |
| `NeedStream(state)` | A text protocol: keep scanning the following segments with a small state. |
| `HandOff` | Too heavy for the packet path (decryption, decompression): copy and process later. |

## Four strategies

| The message | Strategy | Where it runs | Cost |
|---|---|---|---|
| fits in the first packet | none | packet path | nothing extra |
| a field in a text header that may cross a segment boundary (HTTP `User-Agent`) | **stream**: keep a small fixed state and the last partial line (a few hundred bytes at most) | packet path, per tap | a small copy per segment of a watched flow |
| a length-prefixed message (TLS record, Kerberos and SMB length prefixes, LDAP BER length) up to a configured size | **buffer**: copy segments into a fixed slot until complete, then run the normal view over it | packet path, per tap | one copy of the message |
| needs decryption or decompression (QUIC Initial), or is an IP fragment | **hand off**: copy the packet into the observation ring as a raw capture | background fold | one copy per packet; the work runs off the packet path |

Prefer streaming over buffering, and buffering over handing off: each step down costs more.

## Slots

- **A fixed pool per tap**, allocated up front: for example 256 slots of 16 KiB on a server, far
  fewer on an embedded device. The pool never grows.
- **Keyed by the canonical five-tuple** in a small open-addressing table owned by the tap. When
  the table is empty, which is almost always, the packet path checks one counter and skips the
  lookup. Only TCP packets with payload are looked up at all.
- **Placed by sequence number.** The slot records the starting sequence number and the total
  length, and each segment is copied to `sequence − start`. Out-of-order segments land in the
  right place; retransmissions and mirror duplicates overwrite the same bytes. A short list of
  covered ranges (up to four gaps) tells when the message is complete. More gaps than that
  means the flow is abandoned.
- **Bounded in time.** A slot expires when its flow is quiet for longer than a timeout
  (two seconds by default), measured in capture timestamps, never the wall clock.
- **Bounded in size.** Each protocol sets a maximum (a TLS handshake message is capped, for
  example, at 16 KiB). A larger length closes the slot at once.
- **When the pool is full**, the oldest slot is evicted and a counter goes up. Many half-open
  handshakes from one source can only fill the pool, never grow it.

An abandoned or expired message is not an error. Tapster records what the first packet gave
(the flow, the server name if it was in the part seen) and marks the evidence as incomplete.

## When the first segment is not the first to arrive

On a mirror, and on ERSPAN in particular, segments can arrive out of order: segment 2 before
segment 1. The slot above opens only when segment 1's header asks for more, so segment 2 would
already be gone. Tapster handles this in three places, each catching what the one before it
missed.

```
 shell (ERSPAN)   restores mirror order per session from the GRE sequence number
       │          catches most reordering, before the core sees anything
       ▼
 tap              a flow whose handshake was seen gets a slot on its first payload segment,
       │          whichever segment that is
       ▼
 fold             observations carry capture timestamps; order of arrival never matters
```

### 1. The shell restores mirror order

ERSPAN types II and III carry a sequence number in the GRE header, counted per mirror session.
The collector shell holds packets in a short reorder window (a millisecond or a few dozen
packets, configurable) and releases them in sequence order. A gap that the window closes on is
a lost packet: counted, not waited for. Most scrambling on ERSPAN is undone here. ERSPAN type I
has no sequence number, so its packets pass through as they come.

Reordering that happened before the mirror (traffic taking two paths through the network, or
the two directions of a flow mirrored by different sessions) is not undone by this; the tap
handles it.

### 2. The tap opens a slot on the handshake, not on the header

For TCP, the message Tapster wants is usually the first payload after the handshake (a TLS
ClientHello, an SMB negotiate, a Kerberos request). The SYN gives its starting sequence number:
the initial sequence number plus one.

- When the tap sees a SYN on a watched port, it records the flow in a small **new-flow table**:
  the five-tuple and the expected starting sequence number of each direction. One entry per
  new connection, not per packet.
- The first payload segment of that flow opens a slot, **whichever segment it is**. It is
  copied to `sequence − start`, as in any slot. The total length is still unknown, so the slot
  is bounded by the protocol's maximum.
- When the segment carrying the header arrives, the view reads the total length and the slot
  knows when it is complete. Then the normal view runs over it.
- The entry leaves the table when the first message is complete, when the flow's first
  payload turns out to be uninteresting, or when it times out.

The SYN itself can arrive after the data, since it was reordered too. A first payload segment
on a watched port with no entry yet is held for the reorder timeout in a small per-tap orphan
list, bounded in count and bytes, and only for flows that have **no** entry in the tap's flow
counters. The flow counter tells "a flow just starting" from "the middle of a long transfer",
which keeps this list empty under bulk traffic.

### 3. The fold does not care about order

Above the packet path, nothing depends on the order packets arrived in:

- Every observation carries its capture timestamp. "First seen" is the minimum, "last seen"
  the maximum, and binding time ranges come from timestamps.
- A DHCPACK seen before its REQUEST still yields the lease. The REQUEST, folded later, adds its
  fingerprint to the same interface.
- Client and server roles come from TCP flags (SYN versus SYN-ACK), never from which packet
  arrived first.

### What is still lost

If the segment carrying the header never arrives, or arrives after the slot timed out, the
message cannot be read. Tapster records what it had (the flow, the bytes it held), marks the
evidence as incomplete, and the fingerprint is taken from the next connection of that client.
Clients connect often, so one lost handshake rarely leaves a gap in identification.

## Flows must stay on one tap

Slots live in the tap, so every packet of a flow has to reach the same tap:

- **ERSPAN collector:** the outer headers are the same for every packet of a mirror session
  (switch to collector), so the card and `PACKET_FANOUT` would put a whole session on one
  core. The shell steers by the **inner** five-tuple instead: an XDP program reads the inner
  headers and picks the `AF_XDP` socket, or a receive stage per session decapsulates and
  dispatches to the worker taps. Each worker's reorder window then works on its own share of
  the session's sequence numbers, which still increase, with gaps that belong to other workers;
  so the window releases on time, not on a closed gap.
- **Embedded:** a host that spreads one flow over several forwarding threads must say so in
  its configuration. Tapster then hands off instead of buffering for that host, so reassembly
  happens in the background fold, where all packets meet.
- **Single device:** one tap; nothing to arrange.

## IP fragments

v1 does not reassemble IP fragments on the packet path. The first fragment carries the ports
and is parsed as far as it goes; later fragments are counted. When a protocol needs a
fragmented message (large DNS answers), its fragments are handed off and reassembled in the
background, keyed by (source, destination, identification, protocol), with the same limits on
memory, time and size as slots.

## QUIC

QUIC Initial packets are encrypted with keys derived from the destination connection id
(RFC 9001), so a passive sensor can read the ClientHello, but the work (key derivation and
AES-GCM per packet) does not belong on the packet path. The packet path recognises a QUIC
Initial, hands it off, and the background fold decrypts, orders the CRYPTO frames by offset
and runs the TLS ClientHello view over the result.

## Data structures

The arena, reorder heap, flow table, slots and timing wheel are described in
[building blocks](building-blocks.md).

## What this changes elsewhere

- **The copy-out boundary moves only for watched flows.** For them, Tapster copies payload
  bytes into a slot it owns, bounded per tap. Every other packet is still read in place.
- **Observations gain one type**, `RawCapture`, for hand-offs. It owns a copy of the packet
  and its context.
- **The fuzzer** also feeds split, reordered, duplicated and truncated segment sequences, and
  checks that no input grows memory beyond the configured pool.
