# 0004. Own value types for addresses and keys

Date: 2026-10-21 · Status: Accepted

## Context
`IPAddress` and `PhysicalAddress` are classes: each one allocates. Flow and device lookups happen per packet.

## Decision
- `MacAddress`, `IPv4Address`, `IPv6Address`, `IpAddress`, `VlanTag` and `FiveTuple` are `readonly record struct`s read straight from spans.
- Keys pack fields explicitly; struct sizes are asserted in tests.
- Domain helpers live on the types (`Oui`, `IsLocallyAdministered`, subnet checks).

## Consequences
- Address handling allocates nothing and compares as integers.
- Conversion to .NET's types happens only at the edges (output, configuration).

## Alternatives considered
- **Use .NET's types:** an allocation per address, and no room for domain methods.
