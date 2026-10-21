# 0019. No third-party runtime dependencies without justification

Date: 2026-10-21 · Status: Accepted

## Context
Tapster runs inside network devices and parses hostile input. Every dependency is code running there.

## Decision
- Tapster uses .NET's own libraries. A third-party runtime dependency needs a written justification: what it does, who maintains it, and why writing it ourselves is worse.
- Test and build tools follow the same rule, with a lighter bar.

## Consequences
- Small things are written in-house: the Prometheus text endpoint, the frame builder, the arena.

## Alternatives considered
- **Use packet libraries:** they allocate per packet and add code to every router.
