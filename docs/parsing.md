# Parsing

The parser reads protocol headers through layered views over the caller's bytes. A view is a
`readonly ref struct` holding a `ReadOnlySpan<byte>`; moving to the next layer slices the span
and returns the next view. Nothing is copied and nothing is allocated.

```csharp
var eth = new EthernetFrame(frame);
if (eth.TryGetIp(out var ip) && ip.TryGetUdp(out var udp) && udp.IsDhcp)
{
    var dhcp = new DhcpPacket(udp.Payload);
    // read named fields; copy out only what identifies the device
}
```

The packet itself is never on the stack: only the view (a pointer and a length) is. The real
lifetime limit is the caller's buffer, which is reused as soon as `Observe` returns.

## Rules every view follows

1. **Every read is bounds-checked.** A view's `TryGet…` method returns `false` for a frame
   too short for its header. Malformed and truncated frames are normal input, not errors.
2. **Offsets live in one place.** Fields are named properties (`ClientHardwareAddress`,
   `ServerIdentifier`); raw `Slice(28, 6)` calls appear only inside the view that owns them.
3. **Variable-length lists are walked, not indexed.** DHCP options, TCP options, IPv6
   extension headers and TLS extensions are read by a `ref struct` enumerator over the span.
   When several options are needed, one pass fills a small table of offsets
   (an `[InlineArray]` of `(ushort Offset, ushort Length)` pairs), not `System.Range`.
4. **Offsets never leave the call.** They are meaningless once the buffer is reused. Only
   copied-out values cross into observations.
5. **A view may need more than one packet.** It then returns `NeedMore`, `NeedStream` or
   `HandOff` instead of guessing; see [reassembly](reassembly.md).
6. **Truncated frames are expected.** Mirrors and snap lengths cut frames. A view reports what
   it could read; a DHCP option list cut short still yields the options before the cut.

## Layer by layer

### Ethernet and VLAN

- Tag types: 0x8100 (802.1Q), 0x88A8 (802.1ad) and 0x9100 (older double tagging).
- With double tagging, keep **both** the outer (service) and inner (customer) VLAN ids. They
  identify different things.
- A VLAN from `FrameContext` (stripped by the card) and a tag inside the frame can both be
  present; the context VLAN is the outer one.
- Tag depth is limited (two, configurable), so a crafted frame cannot loop the parser.

### IPv4

- Header length comes from IHL, never a fixed 20 bytes.
- A non-first fragment (fragment offset ≠ 0) has no layer-4 header. Never parse it as TCP or
  UDP. The first fragment carries the ports; see [reassembly](reassembly.md#ip-fragments)
  for messages that need the rest.

### IPv6

- The Next Header field may name an extension header (hop-by-hop, routing, fragment,
  destination options). Walk the chain, with a depth limit, until a layer-4 protocol appears.
- ND (neighbour solicitation and advertisement) is the IPv6 equivalent of ARP and a source of
  MAC↔IP bindings.

### Encapsulation

Used by the ERSPAN collector shell; the core sees only the inner frame.

| Outer | How to recognise it | Inner |
|---|---|---|
| GRE (IP protocol 47), type 0x88BE | ERSPAN type I (no sequence bit) or II (8-byte ERSPAN header) | Ethernet |
| GRE, type 0x22EB | ERSPAN type III (12-byte header, optional 8-byte platform subheader) | Ethernet |
| GRE, type 0x6558 | Transparent Ethernet bridging | Ethernet |
| GRE, type 0x0800 / 0x86DD | A routed tunnel, not a mirror | IP, no Ethernet header |
| UDP 4789 | VXLAN | Ethernet |
| UDP 6081 | Geneve | Ethernet |

- The GRE header length depends on its C, K and S flags (4 to 16 bytes); never assume a fixed
  size.
- ERSPAN can run over IPv6 as well as IPv4.
- The outer headers fill `FrameContext`: switch address, ERSPAN session id, source port index
  and, for type III, the switch's timestamp.
- Encapsulation depth is limited (three by default).

### DHCP

- Fixed fields: `op` 0, `htype` 1, `hlen` 2, `xid` 4, `ciaddr` 12, `yiaddr` 16, `siaddr` 20,
  `giaddr` 24, `chaddr` 28. Magic cookie 0x63825363 at 236; options start at 240.
- Read the message type (option 53) before trusting anything else: a lease comes only from a
  DHCPACK.
- Option 52 (overload) can move options into the `sname` and `file` fields; honour it.
- What each message tells us is in [dependency map](dependency-map.md#dhcp).

## Value types for addresses

.NET's `IPAddress` and `PhysicalAddress` are classes, so each one allocates. Tapster uses
small `readonly record struct`s instead, read straight from a span.

| Type | Backed by | Notes |
|---|---|---|
| `MacAddress` | `ulong` (48 bits used) | `Oui` (top 24 bits), `IsLocallyAdministered`, `IsMulticast`, `IsBroadcast` |
| `IPv4Address` | `uint`, numeric (host order) | Read with `ReadUInt32BigEndian`; masks and ranges compare against numeric constants |
| `IPv6Address` | two `ulong`s | |
| `IpAddress` | `IPv6Address` + family | IPv4 stored as IPv4-mapped IPv6, so one key type serves both |
| `VlanTag` | `ushort` | `VlanId`, `Priority`, `DropEligible` |
| `FiveTuple` | addresses + ports + protocol | `ToCanonical()` orders the two ends so both directions share a key |

Notes that are easy to get wrong:

- **Reading a MAC.** Six bytes, not eight:
  `((ulong)ReadUInt16BigEndian(s) << 32) | ReadUInt32BigEndian(s[2..])`.
- **One byte order.** `IPv4Address` holds the numeric value `ReadUInt32BigEndian` returns.
  Never reverse it again before comparing with `0x0A000000` (10.0.0.0).
- **Locally administered is not randomised.** The 0x02 bit of the first octet is also set by
  virtual machines and containers. The type exposes `IsLocallyAdministered`; whether a MAC is a
  privacy-rotated one is a conclusion of the [identity model](identity-model.md).
- **Struct size includes padding.** `(ushort, ulong, uint)` is 24 bytes, not 14. Keys pack
  fields explicitly; for example, a 12-bit VLAN and a 48-bit MAC fit together in one `ulong`.

## Adding a protocol

A protocol is a `ref struct` view plus a small table of the fields it copies out. C# 13 lets a
`ref struct` implement an interface and be used as a generic argument (`allows ref struct`),
so protocol views can share one interface without boxing. When hand-written views become
repetitive, a source generator can produce them from a short field-layout description.
