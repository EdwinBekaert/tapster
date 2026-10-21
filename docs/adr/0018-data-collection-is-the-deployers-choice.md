# 0018. Data collection is the deployer's responsibility

Date: 2026-10-21 · Status: Accepted

## Context
Tapster records addresses, hostnames, fingerprints and dependencies. It is meant to run on infrastructure its user owns, with the right to collect this.

## Decision
- Tapster collects what device identification and dependency mapping need, without privacy-specific reduction features.
- The README warns to run it only on infrastructure the user owns or is authorised to monitor, and that what is done with the data is the user's responsibility.

## Consequences
- No hostname reduction, pseudonymisation or per-record erasure in the design for now.
- Payloads are still never stored, because Tapster does not need them.

## Alternatives considered
- **Privacy features in the core:** cost and complexity for a use case Tapster does not target.
