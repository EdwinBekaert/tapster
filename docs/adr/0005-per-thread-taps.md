# 0005. Per-thread taps: one writer, no locks

Date: 2026-10-21 · Status: Accepted

## Context
An embedded host calls Tapster from several forwarding threads at once. Shared, locked state would make every call wait on the others.

## Decision
- Each calling thread opens its own tap, with its own observation ring, reassembly slots, flow table and counters.
- The only structure two threads share is the ring between a tap and the fold: one writer, one reader, no locks.
- All packets of a flow must reach the same tap. Hosts that cannot guarantee this say so, and Tapster hands off instead of reassembling for them.

## Consequences
- No lock or interlocked operation on the packet path.
- The collector must steer by the inner five-tuple, since ERSPAN's outer headers are the same for a whole session.

## Alternatives considered
- **`ConcurrentDictionary` and shared tables:** locks on writes, allocations, and contention between forwarding threads.
