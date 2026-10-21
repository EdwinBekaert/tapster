# 0003. A zero-allocation packet path over the caller's buffer

Date: 2026-10-21 · Status: Accepted

## Context
Every forwarded or mirrored packet passes through Tapster. Allocating per packet makes the garbage collector run in the host's forwarding path.

## Decision
- Protocol layers are read through `readonly ref struct` views over a `ReadOnlySpan<byte>` of the caller's buffer.
- Only identified values are copied out, at the copy-out boundary, into small value-type observations. Offsets never leave the call.
- Repeated variable-length values (hostnames, option lists) are interned and referenced by id.
- The packet path allocates nothing; benchmarks assert zero allocated bytes.

## Consequences
- The garbage collector is never triggered by packet processing.
- Views cannot cross `await`; everything after the boundary is plain values.
- Bytes needed after the call (reassembly, reordering) need a buffer Tapster owns (see 0007).

## Alternatives considered
- **Parse into objects per packet**, as most packet libraries do: simplest to write, and one or more allocations per packet.
