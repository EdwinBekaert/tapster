# Threat model

Tapster reads packets written by strangers. On a router, the embedded library parses traffic
from the internet, inside the router's forwarding path. Every byte it reads is attacker
controlled, and a failure does not just stop Tapster: it can take the network down.

## What is protected

| Asset | Why it matters |
|---|---|
| **The host's forwarding** | On an embedded sensor, a crash or a slow path in Tapster is a network outage |
| **Collected data** | Addresses, names and dependencies of a whole network (see [privacy](privacy.md)) |
| **The inventory's correctness** | Wrong devices or dependencies lead to wrong decisions |
| **The taproom** | It holds the data of every sensor |

## Who attacks

- **Anyone who can send a packet past a sensor.** For a router's WAN side, that is the whole
  internet. They control every byte of the frames Tapster parses.
- **A device on the monitored network.** It can also send ARP, DHCP, mDNS and LLDP, and so make
  claims about itself and others.
- **Someone between a sensor and the taproom.**
- **A host that misuses the C API**, by mistake more often than on purpose.

## Threats and mitigations

### Crafted packets

| Threat | Mitigation |
|---|---|
| Out-of-bounds read | Views are safe C# over spans: a bad offset cannot read outside the buffer. `unsafe` code is not allowed in views; it is limited to the arena and reviewed on its own. |
| An exception on the packet path takes down the host | Views use `TryGet…` and return results, never throw on malformed input. At the C boundary every exception is caught and becomes a status code. An exception that reaches the boundary is a bug, reported by a counter. |
| Endless or deep loops (VLAN tags, encapsulation, IPv6 extension headers, TLV lists with a zero length) | Every loop has a depth or length limit (see [parsing](parsing.md#rules-every-view-follows)). A TLV with length zero still advances. |
| Huge length fields | Lengths are checked against the bytes present and the protocol's maximum before use. |

### Exhausting resources

| Threat | Mitigation |
|---|---|
| Flooding with half-open handshakes or split messages to fill slots | Fixed pools per tap, eviction of the oldest, counters. Memory never grows (see [reassembly](reassembly.md#slots)). |
| SYN flood filling the new-flow table | Fixed capacity, oldest evicted first. A flood costs some missed fingerprints, never memory or forwarding. |
| Hash flooding: five-tuples chosen to collide in the flow table | Flow-table hashes are keyed with a random seed per process, so collisions cannot be computed from outside (see [building blocks](building-blocks.md#flow-keys-and-lookups)). |
| Expensive work per packet (decryption, decompression) | Handed off to the background fold, with its own bounded queue, never done on the packet path. |
| Event storms toward the taproom | Batched, bounded backlog, oldest dropped and counted. |

### False claims

A device on the network can lie: send ARP replies for another device's IP, use another
device's MAC, invent a hostname, or copy a fingerprint.

| Threat | Mitigation |
|---|---|
| Fake devices or bindings | Every fact keeps where it was seen (sensor, segment, port). A binding that moves between ports, or two MACs claiming one IP at once, is kept as a **conflict**, not resolved silently. Conflicts are reported: they are what ARP spoofing looks like. |
| Making two devices look like one, or one like two | The identity judge never links on evidence the device controls alone (hostname, fingerprint). Links need evidence from the network side as well: port, timing, simultaneous presence. |
| Fingerprint mimicry | A fingerprint is evidence with a confidence, never a verdict. |

### Data leaving the sensor

| Threat | Mitigation |
|---|---|
| Hostnames and names with control characters, invalid UTF-8 or huge lengths, aimed at dashboards, logs or spreadsheets | Values are stored as bytes with a length cap, and encoded for each output: escaped in JSON and HTML, and in CSV prefixed so a value never starts a formula. |
| Eavesdropping or tampering between sensor and taproom | Mutual TLS: the taproom knows each sensor, each sensor knows the taproom. Batches are numbered, so a replayed or missing batch is noticed. |
| A stolen sensor identity | Per-sensor credentials, revocable at the taproom. |

### The C API

| Threat | Mitigation |
|---|---|
| Null pointers, wrong lengths, a handle used after close | Arguments are checked and refused with a status code. Use after close is undefined, as in any C library, and documented as such. |
| A tap used by two threads at once | Documented as not allowed; checked in debug builds. |

### Supply chain

Tapster has no third-party runtime dependencies. Each one added needs its own justification:
what it does, who maintains it, and why writing it ourselves is worse.

### Active probing

When [active probing](roadmap.md#last-active-probing) arrives, it is a tool that could be turned
into a scanner. It runs as a separate process, only against ranges that are explicitly allowed,
with rate limits, and every probe is logged.

## Testing the model

- A fuzzer feeds every view, and the whole `Observe` path, malformed, truncated, split,
  reordered and duplicated input. Any exception, out-of-range read or growth beyond configured
  memory is a bug.
- Flood tests replay SYN floods, slot floods and colliding five-tuples, and check that forwarding
  cost and memory stay flat.
- Spoofing scenarios (ARP spoofing, MAC cloning, copied hostnames) are part of the capture test
  set, with the conflicts Tapster is expected to report.

## Reporting a vulnerability

Please report vulnerabilities privately, not in a public issue.
