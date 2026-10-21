# Architecture decision records

Each record captures one decision: context, decision, consequences and the alternatives considered. Records are immutable once accepted; a change of mind is a new record that supersedes or amends the old one, and says so in its own status line.

The first records were drawn from the design documents as they stood on 2026-10-21: the documents explain how, the records why.

| # | Decision | Status |
|---|---|---|
| [0001](0001-csharp-with-nativeaot.md) | C# compiled with NativeAOT | Accepted |
| [0002](0002-one-core-three-shells.md) | One I/O-free core, three thin shells | Accepted |
| [0003](0003-zero-allocation-packet-path.md) | A zero-allocation packet path over the caller's buffer | Accepted |
| [0004](0004-own-address-value-types.md) | Own value types for addresses and keys | Accepted |
| [0005](0005-per-thread-taps.md) | Per-thread taps: one writer, no locks | Accepted |
| [0006](0006-bounded-memory-drop-and-count.md) | Bounded memory: drop and count, never wait | Accepted |
| [0007](0007-reassembly-under-the-views.md) | Reassembly underneath the views | Accepted |
| [0008](0008-order-independence.md) | Order independence in three layers | Accepted |
| [0009](0009-identity-model.md) | Segments, interfaces, bindings and devices | Accepted |
| [0010](0010-identity-judge.md) | Device identity as scored, explainable links | Accepted |
| [0011](0011-dependency-edges-from-v1.md) | Dependency edges recorded from v1 | Accepted |
| [0012](0012-sensors-and-taproom.md) | Sensors send facts, the taproom folds them | Accepted |
| [0013](0013-polyform-noncommercial.md) | PolyForm Noncommercial 1.0.0 | Accepted |
| [0014](0014-fingerprint-methods-and-data.md) | Fingerprint methods in-house, signature data outside | Accepted |
| [0015](0015-libpcap-loaded-at-run-time.md) | Capture through libpcap, loaded at run time | Accepted |
| [0016](0016-platform-support.md) | Windows, Linux and macOS on x64 and Arm64 | Accepted |
| [0017](0017-wrapper-and-tapster-responsibilities.md) | The wrapper secures the environment, Tapster its own behaviour | Accepted |
| [0018](0018-data-collection-is-the-deployers-choice.md) | Data collection is the deployer's responsibility | Accepted |
| [0019](0019-no-third-party-runtime-dependencies.md) | No third-party runtime dependencies without justification | Accepted |
