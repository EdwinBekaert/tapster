# 0016. Windows, Linux and macOS on x64 and Arm64

Date: 2026-10-21 · Status: Accepted

## Context
Tapster should run on as many platforms as possible, and NativeAOT makes each one a build target.

## Decision
- Supported: Windows, Linux (glibc and musl) and macOS, each on x64 and Arm64.
- CI builds and tests every combination on the platform it targets.
- 32-bit Arm is pursued where NativeAOT supports it; MIPS is out of reach for .NET.

## Consequences
- Platform specifics stay in the shells (see [platforms](../platforms.md)).
- The fastest collector capture (`AF_XDP`) is Linux only.

## Alternatives considered
- **Linux only:** excludes desktop single-device use and Windows-based appliances.
