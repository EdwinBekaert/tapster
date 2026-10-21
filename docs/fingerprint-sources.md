# Fingerprint sources

A fingerprint has two parts, and they come with different terms:

- **The method**: how a fingerprint is computed from packets (which fields, in which order).
  Tapster implements methods itself, in its own code.
- **The signatures**: a database that maps fingerprints to an OS, device type or vendor. These
  are data, collected by others, under their own licenses.

Tapster is licensed under [PolyForm Noncommercial 1.0.0](../LICENSE.md). Every source below has
to fit that, and should not stand in the way of a commercial license from the licensor later.

> Status: researched October 2026, not legal advice. Items marked **verify** were not confirmed
> from the source itself and must be checked before any code or data depends on them.

## Methods

| Method | Terms | Use in Tapster |
|---|---|---|
| **DHCP fingerprint** (option 55 order, option 60, option 57) | A technique, not licensed | Implement |
| **TCP SYN fingerprint** in the style of p0f (TTL, window, options and their order, MSS, window scale) | A technique; p0f's code is LGPL-2.1, and Tapster does not use it | Implement from the published field list, not from p0f's code |
| **JA4** (TLS client) | BSD 3-Clause | Implement |
| **JA3** (TLS client) | BSD 3-Clause | Skip: browsers now shuffle TLS extension order, so JA3 changes between connections of the same client; JA4 sorts them |
| **JA4+** (JA4S, JA4H, JA4L, JA4X, JA4SSH and later methods) | FoxIO License 1.1: internal and academic use allowed, commercial use needs a license from FoxIO; FoxIO describes the methods as patent pending | Not in v1. Fits Tapster's noncommercial license, but would follow any commercial license of Tapster, so if added later, only as a separate optional module |

## Signatures and reference data

| Source | What it gives | Terms | Plan |
|---|---|---|---|
| **IEEE OUI registry** | Vendor of a MAC address prefix | Published by the IEEE Registration Authority. Debian's packaging states the IEEE asserts no copyright on the public listing (**verify** with the IEEE) | Download at install or update time, not stored in the repository |
| **Wireshark `manuf`** | The same, curated | GPL-2.0 | Not used: GPL data would not fit the license |
| **p0f signature database** (`p0f.fp`) | TCP fingerprints → OS | The pre-v3 file carries an LGPL header; the v3 file's terms are unconfirmed (**verify** in the v3 source) | Not shipped. Optional import from a file the user provides |
| **Fingerbank** | DHCP and other fingerprints → device | Commercial: a licensed database, or an API with a free tier (300 requests an hour at the time of writing) and paid tiers, with an API key | Optional integration on the taproom only, off by default, with the user's own key. Sends fingerprints only (see [privacy](privacy.md#exports)). Never on embedded sensors. |
| **Tapster's own signatures** | Fingerprints → OS and device type | Built from labelled captures made for the project | The default. Contributions under the project's license |

## Rules

- **No third-party signature data in the repository** unless its license clearly allows
  redistribution under PolyForm Noncommercial and does not block a commercial license.
- **Methods are implemented from their specifications**, never ported from code under an
  incompatible license.
- **Every source is recorded** with its license, version and download date, and shown in the
  sensor's health output, so a deployment can tell which data it runs on.
- **A signature never decides alone**: a match is evidence with a confidence, like everything
  else in the [identity model](identity-model.md).

## Sources consulted

- FoxIO JA4 repository, License FAQ: https://github.com/FoxIO-LLC/ja4
- p0f: https://lcamtuf.coredump.cx/p0f/ and https://en.wikipedia.org/wiki/P0f
- Debian `ieee-data` license notice, as shipped by Cumulus Linux:
  https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-43/Whats-New/licenses/ieee-data.txt
- Fingerbank licensing: https://fingerbank.org/license/
