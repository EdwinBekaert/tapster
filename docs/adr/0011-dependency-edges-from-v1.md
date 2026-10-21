# 0011. Dependency edges recorded from v1

Date: 2026-10-21 · Status: Accepted

## Context
The dependency map is a v2 feature, but v1 already sees its evidence: one DHCP exchange shows a client, its DHCP server, gateway, DNS and NTP servers.

## Decision
- v1 records edges: client → server for one service.
- Edges connect interfaces or bindings, never devices.
- Each edge is **configured** (the client was told to use it) or **observed** (traffic was seen).

## Consequences
- v2 builds the map from history instead of starting empty.
- Identity links never require rewriting edges.
- A configured edge without traffic is a finding of its own.

## Alternatives considered
- **Wait for v2:** throws away months of evidence.
- **Key edges on devices:** every identity change would rewrite edges.
