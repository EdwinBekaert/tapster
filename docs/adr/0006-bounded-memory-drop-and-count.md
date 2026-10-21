# 0006. Bounded memory: drop and count, never wait

Date: 2026-10-21 · Status: Accepted

## Context
Tapster runs inside a host whose forwarding must not slow down, on devices with fixed memory, while strangers choose its input.

## Decision
- Every structure on the packet path has a fixed size from configuration, allocated at start-up: rings, slot pools, flow tables, orphan lists, reorder windows.
- When a structure is full, Tapster evicts the oldest entry or drops the newest, and counts it. It never waits and never grows.
- Every loop over packet data has a depth or length limit.
- The flow-table hash is keyed with a random seed per process.

## Consequences
- Losing an observation is accepted; slowing the host is not.
- The [counters](../observability.md) tell when sizes need tuning.
- Floods cost missed fingerprints, never memory or forwarding.

## Alternatives considered
- **Growable collections:** memory under the attacker's control.
- **Back-pressure to the host:** slows the network.
