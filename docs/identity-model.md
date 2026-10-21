# Identity model

A MAC address is not a device, and an IP address is not either. Tapster keeps four layers
apart, each defined by what it can actually prove.

| Layer | Identified by | Example |
|---|---|---|
| **Segment** | (sensor, source port or ERSPAN session, VLAN path) | VLAN 10 behind switch A |
| **Interface** | (segment, MAC) | `3c:22:fb:…` on VLAN 10 behind switch A |
| **Binding** | (routing domain, IP) ↔ interface, with a time range | 10.0.10.23 held by that interface from 09:14 to 17:02 |
| **Device** | a cluster of interfaces and bindings, each link with a confidence | "the laptop", seen under three MACs |

## Why segments and routing domains

- **The same VLAN id on two switches without a shared trunk is two segments.** A VLAN id alone
  is not a scope; the segment includes where the frame was seen.
- **IP uniqueness belongs to the routing domain, not the VLAN.** Two VRFs can both use
  10.0.0.0/8. A routing domain is configured per sensor; by default each sensor is one domain.
- **"Namespaces"** on a host (Linux network namespaces, containers) are not visible on the
  wire. What the wire shows is the bridge or NAT in front of them: their MACs on a bridge, or
  one MAC and IP for all of them behind NAT.

## Where bindings come from

A MAC↔IP binding is trusted only from protocols that state it on the local segment:

- ARP requests and replies
- IPv6 neighbour discovery
- DHCPACK (`chaddr` ↔ `yiaddr`), and DHCPv6

A frame's source MAC belongs to the last hop, not to the sender's IP. On a mirror of a core
switch, most frames carry a router's MAC as their source. Deriving bindings from ordinary
traffic would bind hundreds of IPs to the router. Hosts on other segments are known by IP
only, until a sensor on their segment sees them.

## Evidence log

The identity layer is built from evidence, recorded from v1 on even though resolution comes
in v2. Per interface, Tapster keeps:

- first and last seen, per port and per segment
- the bindings it held, with time ranges
- hostnames (DHCP option 12, mDNS, NetBIOS, DHCPv6 FQDN)
- DHCP client identifier (option 61) and DHCPv6 DUID
- DHCP option 82 (relay agent information), which often names the switch port
- fingerprints: DHCP (options 55 and 60), TCP SYN, TLS ClientHello, each as an id
- the peers it talks to (from the [dependency map](dependency-map.md))

## Rotating MAC addresses

Phones, tablets and laptops use private MAC addresses that change per network or over time.
Each new MAC first becomes a new interface. Resolution then decides, per pair, whether two
interfaces belong to one device.

**This is a judgement with a confidence, never a hard merge.** A device is a view over its
interfaces; linking or unlinking two interfaces changes the view and rewrites nothing else.

### Evidence, strongest first

| Evidence | Points to |
|---|---|
| Both MACs active at the same moment | **different devices** (settles it) |
| Same DHCPv6 DUID or DHCP client identifier (option 61) | same device |
| Old MAC goes quiet, new MAC appears on the same port or relay circuit (option 82) shortly after | same device |
| Same hostname or mDNS name | same device, unless the name is generic ("iPhone") |
| Same set of peers and services | same device, weakly |
| Same DHCP or TLS fingerprint | **only** that they run the same OS or app version. Never enough alone: every phone on the same OS release shares it. |
| Different fingerprints | different devices, unless an OS update happened in between |

### Who decides

Resolution sits behind one interface so the method can change without touching the rest:

```csharp
interface IIdentityJudge
{
    IdentityVerdict Judge(in InterfaceEvidence a, in InterfaceEvidence b);
}

readonly record struct IdentityVerdict(Verdict Verdict, float Confidence, EvidenceSet Reasons);
enum Verdict { Same, Different, Unknown }
```

- Deterministic rules run first: simultaneous presence answers "different"; a matching DUID
  answers "same".
- Only ambiguous pairs reach a further judge (a scoring model, or a learned model later).
- Each verdict is stored as evidence, with the judge's name and version and the inputs it
  saw, so a link can be explained and undone.
- Judges are compared on a labelled set of pairs (known same, known different), built from
  captures where the answer is known.

Judging runs on the registry side, never inside `Observe`.
