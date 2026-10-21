# 0015. Capture through libpcap, loaded at run time

Date: 2026-10-21 · Status: Accepted

## Context
The collector and single-device shells must capture on Windows, Linux and macOS. Windows has no built-in libpcap; Npcap provides one under its own license.

## Decision
- Outside Linux's `AF_PACKET` and `AF_XDP`, capture goes through the libpcap API.
- The library (`libpcap`, or Npcap's `wpcap.dll` on Windows) is loaded at run time with `NativeLibrary.TryLoad`.
- Tapster never ships or installs Npcap; the user installs it and holds its license.

## Consequences
- Tapster builds and starts without a capture library, and says clearly when one is missing.
- One libpcap source serves three operating systems.

## Alternatives considered
- **Bundle Npcap:** its license limits redistribution.
- **Windows' own capture tools only:** a second capture path to maintain.
