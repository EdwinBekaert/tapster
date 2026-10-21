# Platforms

Tapster runs on as many platforms as .NET NativeAOT supports: Windows, Linux and macOS, on x64
and Arm64. NativeAOT produces a native binary or library with no .NET runtime to install, so
each platform is a build target, not a porting effort.

## Support matrix

| | Windows x64 | Windows Arm64 | Linux x64 | Linux Arm64 | macOS x64 | macOS Arm64 |
|---|---|---|---|---|---|---|
| **Core** | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| **Embedded** (C API library) | ✔ | ✔ | ✔ (glibc and musl) | ✔ (glibc and musl) | ✔ | ✔ |
| **ERSPAN collector** | ✔ libpcap | ✔ libpcap | ✔ AF_PACKET, AF_XDP, libpcap | ✔ AF_PACKET, AF_XDP, libpcap | ✔ libpcap | ✔ libpcap |
| **Single device** | ✔ libpcap | ✔ libpcap | ✔ | ✔ | ✔ | ✔ |
| **Taproom** | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |

- **musl** matters for embedded: router firmware such as OpenWrt uses musl, not glibc.
  NativeAOT supports `linux-musl-x64` and `linux-musl-arm64`.
- **32-bit Arm** (`linux-arm`, `linux-musl-arm`) is common on smaller routers. NativeAOT support
  for it is newer and less documented (**verify** with the .NET version in use before promising
  it).
- **MIPS** is not supported by .NET at all, so MIPS routers cannot run the embedded library.
- **The ERSPAN collector is fastest on Linux**, where `AF_XDP` and `AF_PACKET` exist. Elsewhere
  it works through libpcap at lower rates, which suits testing and small sites.

## Capture libraries

The embedded consumer needs no capture library: its host hands Tapster the frames. The other
shells capture through one interface, `IFrameSource`, with an implementation per mechanism:

| Mechanism | Platforms | Comes from |
|---|---|---|
| `AF_PACKET` (TPACKET_V3) | Linux | the kernel |
| `AF_XDP` | Linux | the kernel |
| libpcap | Linux, macOS | the operating system (macOS) or the distribution's package (Linux) |
| libpcap API through Npcap | Windows | **Npcap, installed by the user** |

### Npcap on Windows

Windows has no built-in libpcap. Npcap provides one (`wpcap.dll`) with the libpcap API, so
Tapster's libpcap source works on Windows unchanged. Npcap has its own license, which limits
redistribution. Tapster therefore:

- never ships or installs Npcap;
- loads `wpcap.dll` at run time with `NativeLibrary.TryLoad`, so it builds and starts without
  it;
- reports clearly when it is missing, with a link to Npcap;
- leaves the Npcap license to the user, who installs it themselves.

The same run-time loading is used for libpcap on Linux and macOS, so no platform needs capture
headers or libraries at build time.

## Building

- **NativeAOT compiles on the operating system it targets.** A Windows binary is built on
  Windows, a macOS binary on macOS. Building for the other CPU on the same operating system
  (x64 ↔ Arm64) works with the right native toolchain installed.
- **CI builds and tests every cell of the matrix** on GitHub-hosted runners: Windows, Linux and
  macOS, each on x64 and Arm64 where GitHub offers a runner, and cross-built on the same
  operating system where it does not.
- **musl** builds run in an Alpine container.
- Each build runs the unit tests and the capture tests (see [testing](testing.md)) on the
  machine that built it, so every published binary has been run on its own platform.

## Platform differences that reach the code

- **Clocks.** Capture timestamps come from the capture mechanism; Tapster never reads the wall
  clock on the packet path.
- **Paths, services and permissions** (raw capture needs administrator rights or Linux
  capabilities) live in the shells, never in the core.
- **Byte order.** All supported CPUs are little-endian; protocol fields are read with
  `BinaryPrimitives` big-endian readers either way.
