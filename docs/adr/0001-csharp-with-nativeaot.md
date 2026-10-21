# 0001. C# compiled with NativeAOT

Date: 2026-10-21 · Status: Accepted

## Context
Tapster has to run inside routers and appliances that have no .NET runtime, as well as on servers and desktops. It parses hostile input at high rates, where parsers written in C most often fail by reading outside their buffer.

## Decision
- Tapster is written in C#.
- Every deliverable is built with .NET NativeAOT: a native library with a C API for embedding, and native executables for the other shells.

## Consequences
- No runtime to install on any target; the embedded library is just another C library to its host.
- Spans and bounds checks make out-of-buffer reads impossible in views; `unsafe` code is confined to the arena.
- NativeAOT rules apply: no runtime code generation, reflection only where trimming keeps it.
- Each operating system is built on that operating system (see [platforms](../platforms.md)).

## Alternatives considered
- **C or C++:** the usual choice for packet parsing, and the usual source of memory-safety bugs.
- **Rust:** as safe and as fast, but not the language this project sets out to show.
- **Regular .NET with a runtime:** not embeddable in firmware without shipping the runtime.
