# Testing

Five kinds of tests, from fast and small to slow and large. All of them run on every platform
in the [support matrix](platforms.md).

| Kind | What it checks | Speed |
|---|---|---|
| **Unit tests** | One view or building block, on bytes built in the test | milliseconds |
| **Capture tests** | The whole core over recorded captures, against expected results | seconds |
| **Fuzzing** | Malformed, truncated, split, reordered input never breaks the packet path | runs continuously |
| **Benchmarks** | Speed per packet, and zero allocated bytes | minutes |
| **Replay at line rate** | A shell under real load: throughput, drops, memory | minutes, Linux only |

## Unit tests

- Frames are built in the test with a small frame builder (`Frame.Ethernet(...).Vlan(20).Ipv4(...)
  .Udp(68, 67).Dhcp(...)`), so each test shows exactly the bytes it is about.
- Unit tests start no processes, open no sockets and read no capture files.
- Each view has tests for its fields, every `TryGet…` failure (too short, wrong type, bad
  length), and its depth and length limits.
- Struct sizes are asserted, like the layout checks of a C header.

## Capture corpus

Recorded captures, each with the results Tapster is expected to produce from it.

```
tests/captures/
  dhcp/relayed-discover-ack.pcapng
  dhcp/relayed-discover-ack.expected.json
  erspan/type3-reordered.pcapng
  erspan/type3-reordered.expected.json
  ...
  SOURCES.md
```

- **Where captures come from:** public sample captures from other repositories and websites,
  captures generated with the frame builder, and captures recorded on test networks for the
  project.
- **`SOURCES.md`** lists each capture's origin (the URL it was downloaded from, or how it was
  made), so any file can be traced and replaced.
- **`.expected.json`** holds the observations, bindings and edges the capture must produce.
  A test runs the core over the capture and compares. When a change alters the result on
  purpose, the expected file is regenerated and the difference reviewed in the pull request.
- **What the corpus must cover**, at least:
  - double VLAN tagging, and VLANs stripped into the context;
  - ERSPAN types I, II and III, GRE with each flag combination, VXLAN and Geneve;
  - IPv6 with extension headers, IPv4 fragments;
  - DHCP direct and relayed, with option 82 and option overload; DHCPv6;
  - TLS ClientHellos in one segment and in several, including post-quantum key shares;
  - out-of-order segments, missing SYN, duplicates and truncated frames;
  - a device rotating its MAC address, and two devices with the same hostname;
  - ARP spoofing and MAC cloning, with the conflicts Tapster must report.

## Fuzzing

- A fuzzer drives every view and the whole `Observe` path with generated input, including
  sequences of segments that are split, reordered, duplicated and cut short.
- A failure is any exception, any read outside the buffer, any loop past its limit, or memory
  growing beyond the configured pools.
- Inputs that once found a bug are kept in the corpus as regression cases.

## Benchmarks

- BenchmarkDotNet runs per view, per building block and for the whole packet path, over
  frames from the corpus.
- Every benchmark of the packet path asserts **zero allocated bytes** after warm-up.
- Results are kept per platform, so a regression shows as a change against the last run.

## Replay at line rate

On Linux, `tcpreplay` sends corpus captures through a network interface at line rate into the
ERSPAN collector shell. The run reports packets per second per core, dropped observations,
evictions and memory, from Tapster's own [counters](observability.md).
