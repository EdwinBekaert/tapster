# 0009. Segments, interfaces, bindings and devices

Date: 2026-10-21 · Status: Accepted

## Context
A MAC address is not unique across broadcast domains, an IP address is unique only within a routing domain, and on a mirror most frames carry a router's MAC.

## Decision
- A **segment** is (sensor, port or ERSPAN session, VLAN path); an **interface** is a MAC on a segment; a **binding** is an IP in a routing domain bound to an interface for a time range; a **device** is a cluster of interfaces and bindings.
- Bindings are learned only from ARP, IPv6 neighbour discovery and DHCP leases.
- Conflicting facts are kept and reported, never resolved silently.

## Consequences
- Overlapping VLANs and VRFs are handled by construction.
- Hosts on other segments are known by IP only.
- ARP spoofing shows up as a conflict.

## Alternatives considered
- **(VLAN, MAC, IP) as the identity:** wrong across switches, VRFs and routed traffic.
