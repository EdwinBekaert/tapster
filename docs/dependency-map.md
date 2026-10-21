# Dependency map

The dependency map shows which devices rely on which services: this laptop uses that DHCP
server, that DNS server and that database. The map itself is a v2 feature, but its edges are
recorded from v1 on. When a DHCP exchange shows a client and a server, not recording that
would throw away the evidence the map will need.

## Edges

An edge connects two endpoints for one service:

```csharp
readonly record struct EdgeKey(EndpointRef Client, EndpointRef Server, Service Service);
```

- **Endpoints are interfaces or bindings, never devices.** A device is a view over interfaces
  (see [identity model](identity-model.md)). When two interfaces are later judged to be one
  device, no edge needs rewriting.
- **An endpoint outside the network is a binding with an IP only.** Most peers on the
  internet, and hosts on segments without a sensor, are known only by IP.
- **Every edge has a kind:**

| Kind | Meaning | Example |
|---|---|---|
| **configured** | Something told the client to use this server | DHCP option 6 names a DNS server |
| **observed** | Traffic between the two was seen | DNS queries from the client to that server |

A configured edge with no observed counterpart is a finding of its own: a DNS server handed
out but never used, or a client using a different one than it was given.

## DHCP

One exchange already gives most of a client's first dependencies.

| Message | Device facts | Edges |
|---|---|---|
| DISCOVER | client MAC (`chaddr`), hostname (12), fingerprint (55, 60), client id (61), requested IP (50) | none yet: the server is unknown |
| REQUEST | the same, plus the chosen server (54) | client → that DHCP server |
| OFFER / ACK | lease: `yiaddr`, lease time (51); server identifier (54) | client → DHCP server (**observed**) |
| ACK options | gateway (3), DNS servers (6), NTP servers (42), domain (15) | client → gateway, DNS, NTP (**configured**) |
| relayed (`giaddr` ≠ 0) | the frame's source MAC is the relay's, not the server's; option 82 often names the switch port | client → relay → server |
| RELEASE / DECLINE | the binding ends early, or the address was in use | ends the binding's time range |

DHCPv6 follows the same pattern, with the DUID as client identity and its own options for DNS
servers.

## Other sources recorded in v1

| Source | Edges |
|---|---|
| ARP for the gateway | client → gateway (observed) |
| DNS queries and answers | client → DNS server; the answer links a name to the IP the client then contacts |
| TCP SYN / SYN-ACK | client → server on that port; the SYN side is the client |
| TLS ClientHello | client → server; the SNI names the service |
| LLDP / CDP | device ↔ switch port (physical topology, not a dependency) |

## Who is the client

The SYN decides it when the handshake is seen. On a mirror it often is not: flows that started
before capture, asymmetric routing, or mirror loss. Then, in order:

1. a well-known or registered port on one side and an ephemeral port on the other
2. the side with many distinct peers on the same port is the server
3. otherwise the direction stays unknown on the edge

## Counters

Edges carry packet and byte counts, first and last seen. Counters are updated per thread and
merged by the background fold, never with shared `++` on the packet path.
