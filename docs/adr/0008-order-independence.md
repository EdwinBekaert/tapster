# 0008. Order independence in three layers

Date: 2026-10-21 · Status: Accepted

## Context
Mirrored traffic, ERSPAN in particular, arrives out of order: a later segment can arrive before the first.

## Decision
- The ERSPAN shell restores mirror order per session from the GRE sequence number, in a short window released by time.
- The tap opens a slot on a flow's first payload segment after a seen handshake, whichever segment that is; a bounded orphan list covers a SYN that arrives late.
- The fold uses capture timestamps, never arrival order.
- Sequence numbers are compared with serial-number arithmetic.

## Consequences
- Most reordering is undone before the core sees it.
- The result of folding does not depend on the order facts arrived in.

## Alternatives considered
- **Only buffer from the first segment:** loses every message whose first segment arrives late.
