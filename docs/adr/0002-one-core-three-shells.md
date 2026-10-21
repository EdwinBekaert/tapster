# 0002. One I/O-free core, three thin shells

Date: 2026-10-21 · Status: Accepted

## Context
Tapster has three consumers: embedded in a device that sees all traffic it forwards, a server receiving ERSPAN, and a single machine. Their inputs differ; what they need to learn from a frame does not.

## Decision
- One core turns a frame plus its `FrameContext` into observations. It does no I/O, reads no clock and starts no threads.
- Each consumer is a shell that turns its input into frames plus context: the embedded C API, the ERSPAN collector (capture, reorder, decapsulate, steer), the single-device capture.
- What the frame cannot say (time, sensor, port, direction, a stripped VLAN) comes from the context, never from guessing.

## Consequences
- All three consumers share every view and every fix.
- The core is tested from recorded captures on any platform.
- Encapsulation never reaches the core: the collector shell hands it the inner frame.

## Alternatives considered
- **A separate program per consumer:** three parsers to keep correct.
- **The core does its own capture:** ties it to one platform and makes embedding impossible.
