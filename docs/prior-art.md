# Prior art

Passive network analysis is an old field with excellent open-source tools. Tapster borrows ideas
from them and differs in what it is for and where it runs.

| Tool | What it is | What Tapster takes from it |
|---|---|---|
| **Zeek** | Network security monitor: turns traffic into rich logs of connections, protocols and files, scriptable | Protocol coverage as a goal; the idea of logs of facts rather than packets |
| **Suricata** | Intrusion detection and prevention engine with signatures, multi-threaded | Flow-hash work distribution over cores, `AF_PACKET` and `AF_XDP` capture |
| **p0f** | Passive OS fingerprinting from TCP handshakes | The TCP SYN fingerprint fields |
| **nDPI** | Deep packet inspection library: which application a flow belongs to | Recognising services from the first packets of a flow |
| **Arkime** | Full packet capture and indexing for later search | Nothing directly: Tapster deliberately stores no packets |
| **Wireshark / tshark** | Interactive protocol analysis | The reference for how a protocol is dissected; its sample captures |
| **JA4** | TLS client fingerprinting method | The TLS fingerprint (see [fingerprint sources](fingerprint-sources.md)) |

## Where Tapster differs

- **It answers two questions only:** which devices are on the network, and which depend on
  which. It is not an intrusion detection system and not a logging platform, so most of the
  traffic it sees costs one counter update.
- **It embeds into the forwarding path.** The same core runs as a native library inside a
  router, with a C API and no runtime, as well as on a collector server or a single machine.
  The tools above run beside the traffic, not inside the device forwarding it.
- **It is written in safe, managed C#** with a packet path that allocates nothing. A malformed
  packet cannot make a view read outside its buffer, which is the most common class of bug in
  packet parsers written in C.
- **Identity is a model, not a key.** Segments, interfaces, bindings and devices are kept apart,
  and linking rotated MAC addresses is a scored, explainable judgement (see
  [identity model](identity-model.md)).
- **Dependencies are recorded from the first version**, with configured and observed edges kept
  apart.

## Using them together

Tapster's facts and Zeek's logs answer different questions about the same traffic. Nothing in
Tapster's design prevents running both on one ERSPAN feed, and the change stream (see
[deployment and outputs](deployment-and-outputs.md#what-a-user-gets)) is plain JSON lines that
other tools can read.
