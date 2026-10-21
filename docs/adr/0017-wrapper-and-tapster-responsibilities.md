# 0017. The wrapper secures the environment, Tapster its own behaviour

Date: 2026-10-21 · Status: Accepted

## Context
Tapster runs inside software that others build: router firmware, services, desktop tools. Security work can sit in either.

## Decision
- The wrapper owns isolation, the transport between sensor and taproom and its credentials, access to outputs, and the authorisation to monitor the network.
- Tapster owns what only the parser can: never throwing on input, no exception across the C boundary, bounded memory and work per packet, a seeded flow-table hash, and reporting conflicting facts.

## Consequences
- No separate threat-model document; Tapster's own rules live in the design documents they apply to.
- See [architecture](../architecture.md#who-protects-what).

## Alternatives considered
- **Tapster secures everything:** duplicates what every host already has.
- **The wrapper secures everything:** impossible for parser behaviour.
