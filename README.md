# Tapster

**A demonstration project about high-performance processing in C#.** Tapster is part of Edwin
Bekaert's portfolio: it shows how to design and build software that handles millions of events
per second without slowing the system it runs in. It is not a product.

The problem it works on is real: a passive network sensor. Tapster reads network traffic, works
out which devices are on the network and what they are, and records who depends on whom.

## What it demonstrates

| Technique | Where |
|---|---|
| Zero-allocation processing with `Span<T>` and `ref struct` views over memory owned by someone else | [Parsing](docs/parsing.md) |
| Bounded memory under hostile input: fixed pools, drop and count, never wait | [Building blocks](docs/building-blocks.md) |
| Lock-free hand-off between threads: single-producer, single-consumer rings, one owner per structure | [Architecture](docs/architecture.md#threading) |
| Reassembling and reordering data that arrives split and out of order, without slowing the common case | [Reassembly](docs/reassembly.md) |
| A C# core embedded in native software through NativeAOT and a C API | [Architecture](docs/architecture.md#embedded) |
| A pure core with thin shells, so one engine serves three very different deployments | [Architecture](docs/architecture.md) |
| Event-sourced state: facts in an append-only log, conclusions as a fold | [Deployment and outputs](docs/deployment-and-outputs.md) |
| Modelling a messy domain precisely: identity across VLANs, routing domains and rotating MAC addresses | [Identity model](docs/identity-model.md) |
| Decisions written down with their alternatives | [Decision records](docs/adr/README.md) |

It is built for places where every packet passes by: inside a router or other network
device, on a server that receives a switch's mirrored traffic, or on a single machine.
The packet parser reads headers where they already are in memory and copies out only the
few values that identify a device, so the hot path allocates nothing.

> **Status:** design. No code yet. These documents describe the architecture the first
> code will follow.

## Three consumers, one core

| Consumer | Where it runs | What it sees |
|---|---|---|
| **Embedded** | Inside a router or network device, as a native library | Every frame the device forwards |
| **Forwarded traffic** | A dedicated server receiving ERSPAN from switches | All mirrored traffic of those switches |
| **Single device** | One machine, capturing on its own interface | Its own traffic plus broadcast and multicast |

All three feed the same core. The core does no I/O: each consumer is a thin shell that turns
its input into a frame plus its context (where the frame came from), and the core turns that
into observations.

## Principles

- **Leave the packet where it is.** The parser works on a `ReadOnlySpan<byte>` over the
  caller's buffer. Only identified values (a MAC, an IP, a fingerprint) are copied out.
- **Nothing on the hot path allocates, locks or blocks.** Each calling thread writes to its
  own observation ring; when the ring is full, observations are dropped and counted, and the
  caller is never slowed.
- **Facts first, conclusions later.** The core records what it saw (observations, evidence,
  edges). Device identity and the dependency map are built from those facts, so a conclusion
  can be explained and revised.
- **Most packets are not interesting.** Device identification needs ARP, ND, DHCP, LLDP, mDNS,
  DNS, TCP SYN and TLS ClientHello: around one percent of traffic. Everything else costs a
  parse to layer 4 and one counter update.

## Documentation

- [Architecture](docs/architecture.md): the core, the three shells, the copy-out boundary
  and the threading model.
- [Parsing](docs/parsing.md): layered `ref struct` views, value types for addresses, and the
  rules every parser follows.
- [Reassembly](docs/reassembly.md): messages split over several packets, and how
  Tapster reads them without slowing the packet path.
- [Building blocks](docs/building-blocks.md): the arena, rings, reorder window, slots and
  timing wheel behind the packet path.
- [Identity model](docs/identity-model.md): segments, interfaces, bindings and devices, and
  how rotating MAC addresses are resolved.
- [Dependency map](docs/dependency-map.md): which edges are recorded from the first version,
  starting with DHCP.
- [Deployment and outputs](docs/deployment-and-outputs.md): sensors, the taproom that merges
  them, the fact log and what a user gets.
- [Fingerprint sources](docs/fingerprint-sources.md): fingerprint methods and signature data,
  and their licenses.
- [Platforms](docs/platforms.md): operating systems, CPUs, capture libraries and building.
- [Testing](docs/testing.md): unit tests, the capture corpus, fuzzing and benchmarks.
- [Observability](docs/observability.md): the counters Tapster keeps about itself.
- [A DHCP exchange, step by step](docs/walkthrough.md): one relayed exchange from frame to
  dependency edges.
- [Prior art](docs/prior-art.md): how Tapster relates to existing open-source tools.
- [Decision records](docs/adr/README.md): the decisions behind this design, and why.
- [Performance](docs/performance.md): targets, capture options and what happens under load.
- [Roadmap](docs/roadmap.md): what comes first and what comes last.
- [Glossary](docs/glossary.md): the words these documents use.

## Roadmap in one line

v1 identifies devices passively and records the evidence and edges that v2 needs; v2 resolves
rotating MAC addresses and builds the dependency map; active probing comes last.

## Data you collect

Tapster records MAC and IP addresses, hostnames, fingerprints and who talks to whom. Run it only
on infrastructure you own or are authorised to monitor, and check the rules that apply to you
(data protection, monitoring of employees) before you do. Tapster collects; what you do with
the data is your responsibility.

## Contributing and security

Tapster does not accept outside contributions; bug reports are welcome as issues (see
[CONTRIBUTING.md](CONTRIBUTING.md)). Report vulnerabilities privately, as described in
[SECURITY.md](SECURITY.md).

## License

[PolyForm Noncommercial 1.0.0](LICENSE.md). You may use, change and share Tapster for any
noncommercial purpose, such as personal use, study, research, and use by charities, schools
and public bodies. Commercial use is not allowed.
