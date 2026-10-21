# 0014. Fingerprint methods in-house, signature data outside

Date: 2026-10-21 · Status: Accepted

## Context
Fingerprints have a method (how to compute them) and signature data (what they mean), under different terms. The license must stay clear for noncommercial use and for a later commercial license.

## Decision
- Methods are implemented in Tapster from their specifications: DHCP, TCP SYN in the style of p0f, JA4.
- JA3 is skipped; JA4+ (FoxIO License 1.1) stays out of v1.
- No third-party signature data in the repository unless its license clearly allows it; OUI data is downloaded, p0f data imported from a user's file, Fingerbank optional on the taproom with the user's key.
- Details in [fingerprint sources](../fingerprint-sources.md).

## Consequences
- Tapster's own signatures, built from labelled captures, are the default.
- Every data source is recorded with license, version and date.

## Alternatives considered
- **Ship existing databases:** GPL and unclear licenses.
