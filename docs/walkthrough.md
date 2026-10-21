# A DHCP exchange, step by step

One laptop joins the network. This page follows its DHCP exchange through Tapster, from the
first frame to the bindings and dependency edges it leaves behind.

## The setup

```
 laptop  ──── port 3, VLAN 20 ────  router R  ─────────────  DHCP server S
 MAC 3c:22:fb:00:00:01              runs Tapster embedded      10.0.100.10
 hostname build-07                  relays DHCP, giaddr 10.0.20.1
                                    the network card strips VLAN tags
```

The router's forwarding thread calls `tapster_observe` for every frame. For the frames below
it passes this context:

| Field | Value |
|---|---|
| `sensor_id` | R |
| `port` | 3 |
| `direction` | ingress for frames from the laptop, egress for frames to it |
| `vlan_id` | 20, with `TAPSTER_CTX_VLAN_STRIPPED`, because the card removed the tag |
| `timestamp_ns` | when the card received the frame |

From the context alone, the core knows the **segment**: (sensor R, port 3, VLAN 20).

## Four frames

### 1. DISCOVER, laptop to everyone

```
Ethernet  dst ff:ff:ff:ff:ff:ff  src 3c:22:fb:00:00:01  type 0x0800
IPv4      0.0.0.0 → 255.255.255.255  UDP 68 → 67
DHCP      op 1  chaddr 3c:22:fb:00:00:01  options: 53=1 (DISCOVER), 61=client id,
          12="build-07", 60="MSFT 5.0", 55=[1,3,6,15,31,33,43,44,46,47,119,121,249,252]
```

The views run `EthernetFrame` → `IpPacket` → `UdpDatagram` → `DhcpPacket`. One pass of the
option enumerator finds options 53, 61, 12, 60 and 55. The core copies out:

- **`DhcpClientSeen`**: segment, client MAC, message type, client id, hostname id, fingerprint
  id. The hostname and the option 55 list are interned: the first time costs one copy, every
  later DISCOVER with the same values costs a table lookup.

Nothing else from this frame is kept. The frame stays in the router's buffer, and the router
forwards it.

### 2. OFFER, relay to laptop

```
Ethernet  src = router R's MAC       ← not the DHCP server's
IPv4      10.0.20.1 → 255.255.255.255
DHCP      op 2  yiaddr 10.0.20.57  giaddr 10.0.20.1  options: 53=2 (OFFER), 54=10.0.100.10, ...
```

`giaddr` is set, so the exchange is relayed: the frame's source MAC belongs to the relay, and
the server is option 54. An OFFER is only a proposal, so the core copies out the server and the
relay, but no lease.

- **`DhcpServerSeen`**: segment, server id 10.0.100.10, relay 10.0.20.1.

### 3. REQUEST, laptop to everyone

The same fields as the DISCOVER, plus option 54 (the server the laptop chose) and option 50
(the address it asks for). The core copies out another `DhcpClientSeen`, with the chosen
server.

### 4. ACK, relay to laptop

```
DHCP      op 2  yiaddr 10.0.20.57  giaddr 10.0.20.1
          options: 53=5 (ACK), 54=10.0.100.10, 51=86400, 3=10.0.20.1,
                   6=[10.0.100.53, 10.0.100.54], 42=10.0.100.123, 15="corp.example"
```

An ACK is the only message that grants a lease. The core copies out:

- **`DhcpLease`**: segment, client MAC 3c:22:fb:00:00:01, address 10.0.20.57, lease 86400 s,
  server 10.0.100.10.
- **`DhcpConfigured`**: the same client, gateway 10.0.20.1, DNS 10.0.100.53 and 10.0.100.54,
  NTP 10.0.100.123, domain `corp.example`.

## What the fold makes of it

The background fold drains the tap's ring and updates the registry, the evidence log and the
edge store. It orders by capture timestamp, so it would reach the same result if the ACK had
been folded before the REQUEST.

**Registry**

| Record | Value |
|---|---|
| Interface | (R, port 3, VLAN 20) + `3c:22:fb:00:00:01` |
| Binding | routing domain of R: `10.0.20.57` ↔ that interface, from the ACK's timestamp, for 86400 s, source `dhcp` |

**Evidence** for the interface: hostname `build-07`, client id (option 61), DHCP fingerprint
(option 55 order with option 60 `MSFT 5.0`), first and last seen on port 3.

**Edges**

| Client | Server | Service | Kind |
|---|---|---|---|
| the interface | 10.0.100.10 | DHCP | observed, through relay 10.0.20.1 |
| the interface | 10.0.20.1 | gateway | configured |
| the interface | 10.0.100.53, 10.0.100.54 | DNS | configured |
| the interface | 10.0.100.123 | NTP | configured |

## What it does not conclude

- **The router's MAC is not bound to 10.0.20.1.** The OFFER and ACK came from that MAC and
  address, but bindings come only from ARP, ND and DHCP leases (see
  [identity model](identity-model.md#where-bindings-come-from)). The laptop's first ARP request
  for its gateway will give that binding a moment later.
- **The DHCP server has no interface yet.** It sits on another segment; Tapster knows it by IP
  only, until a sensor on its own segment sees it.
- **The DNS edges are only configured.** They become observed when the laptop's first DNS query
  passes the router.
- **No device yet.** The interface is a device of its own until there is a reason to link it to
  another (see [rotating MAC addresses](identity-model.md#rotating-mac-addresses)).

## Cost on the router

Four frames, four calls to `tapster_observe`. Each parsed headers in place, walked one option
list, and wrote one or two small structs into the tap's ring. No allocation, no lock, and no
copy of the frame.
