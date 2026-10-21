# Glossary

| Term | Meaning in Tapster |
|---|---|
| **Arena** | One pinned block of memory per tap, cut into fixed-size chunks, holding bytes kept past a call. |
| **Binding** | A MAC↔IP pair within a routing domain, valid for a time range. Learned only from ARP, ND and DHCP. |
| **Configured edge** | A dependency the client was told about (for example by DHCP), whether or not it is used. |
| **Consumer** | One of the three ways Tapster is deployed: embedded, forwarded traffic, single device. |
| **Copy-out boundary** | The point where values are copied from the caller's buffer into observations. Above it, views; below it, owned values. |
| **Core** | The I/O-free part that turns a frame and its context into observations. Shared by all consumers. |
| **Device** | A cluster of interfaces and bindings judged to be one physical or virtual machine. A view, never a stored key. |
| **Edge** | A client → server relation for one service, between interfaces or bindings. |
| **ERSPAN** | Encapsulated remote mirroring: a switch sends copies of traffic inside GRE to a collector. |
| **Evidence log** | The per-interface history (ports, names, identifiers, fingerprints) that identity resolution works from. |
| **Fingerprint** | A compact id for how a client's software behaves (DHCP option order, TCP SYN fields, TLS ClientHello). Identifies an OS or app version, not one device. |
| **FrameContext** | What the shell knows about a frame that the frame cannot say: time, sensor, port, direction, stripped VLAN. |
| **Hand-off** | Copying a packet into the observation ring so heavy work (decryption, fragment reassembly) runs in the background fold. |
| **Interface** | A MAC address on a segment. |
| **Judge** | An implementation of `IIdentityJudge` that decides whether two interfaces are one device. |
| **Observation** | A small value-type fact the core copied out of a frame. |
| **Observed edge** | A dependency confirmed by traffic between the two ends. |
| **Ring** | The per-thread buffer of observations between the core and the background fold. |
| **Routing domain** | The scope in which an IP address is unique (a VRF, or by default one sensor). |
| **Segment** | One layer-2 broadcast domain as Tapster sees it: sensor, port or ERSPAN session, and VLAN path. |
| **Shell** | The consumer-specific code around the core: capture, decapsulation, context. |
| **Slot** | A fixed buffer from a tap's pool that collects one message split over several TCP segments. |
| **Tap** | In the embedded C API, the per-thread handle a forwarding thread calls `tapster_observe` on. |
| **View** | A `readonly ref struct` over the caller's bytes that reads one protocol layer. |
