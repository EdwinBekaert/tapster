# Roadmap

## v1: passive identification

Identify devices from traffic, without sending anything, for all three consumers.

- The core: layered views for Ethernet, VLAN, IPv4, IPv6, UDP, TCP, ARP, ND, DHCP, DHCPv6,
  LLDP, mDNS, DNS and TLS ClientHello.
- Observations and the copy-out boundary.
- The registry: segments, interfaces and bindings.
- Fingerprints: DHCP, TCP SYN and TLS ClientHello, stored as ids.
- Reassembly of messages split over TCP segments: streaming and fixed slots per tap. QUIC and
  IP fragments are handed off to the background fold.
- **Recorded for v2:** the evidence log per interface, and dependency edges (configured and
  observed), starting with DHCP.
- The three shells: embedded C API, ERSPAN collector, single device.

## v2: resolution and dependencies

- **Rotating MAC addresses**, through the `IIdentityJudge` interface: deterministic rules
  first, then a judge for ambiguous pairs. Must be solved; see
  [identity model](identity-model.md#rotating-mac-addresses).
- **Dependency map** built from the edges v1 recorded.
- Services named from ports, SNI and DNS answers.

## Last: active probing

Rate-limited probes (ICMP, TCP SYN to common ports, SNMP) to confirm details or reach hosts
that stay silent. It runs as a separate process, outside the packet path, and only where
probing is authorised.

## First steps

1. **Write the identity model down as types.** `SegmentId`, `MacAddress`, `IpAddress`,
   interface and binding keys, with their sizes checked in tests.
2. **Build a test set of captures with known answers.** ERSPAN types I, II and III, double
   tagging, IPv6 extension headers, IPv4 fragments, truncated frames, relayed DHCP and a
   device that rotates its MAC.
3. **Write the views and the core**, with BenchmarkDotNet checks that the packet path
   allocates nothing, and a fuzzer over every view.
4. **Add the single-device shell**: the quickest way to see real output.
5. **Add the ERSPAN collector shell** and replay the test set at line rate.
6. **Add the embedded C API**, built with NativeAOT, with a small C host that feeds it frames.
