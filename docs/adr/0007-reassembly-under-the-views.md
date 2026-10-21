# 0007. Reassembly underneath the views

Date: 2026-10-21 · Status: Accepted

## Context
Some messages span several packets: TLS ClientHellos with post-quantum key shares, HTTP headers, Kerberos and SMB over TCP, QUIC Initials, IP fragments.

## Decision
- A view that needs more says so: `NeedMore(length)`, `NeedStream(state)` or `HandOff`.
- Three strategies, cheapest first: stream with a small state; buffer into a fixed slot from the tap's arena, placed by sequence number; hand off a copy to the background fold for heavy work (QUIC decryption, IP fragments).
- A reassembled message is a span, so the same views read it.

## Consequences
- Unsplit messages cost nothing extra.
- Slot memory is bounded (see 0006).
- An incomplete message is recorded as incomplete evidence, not an error.

## Alternatives considered
- **`System.IO.Pipelines` per flow:** heavy across many flows, and does not reorder.
- **Reassemble every TCP stream:** memory and work for traffic Tapster does not need.
