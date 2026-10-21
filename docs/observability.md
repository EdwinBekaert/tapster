# Observability

Throughout the design, Tapster "drops and counts" instead of waiting or growing. This page
lists those counts and where they can be read.

## Counters

Each tap keeps its own counters, written only by its own thread, so counting costs a plain
increment. A reader sums them across taps when asked.

| Area | Counters |
|---|---|
| **Frames** | seen, bytes seen, malformed per layer (Ethernet, VLAN, IP, TCP, UDP, each protocol view), truncated |
| **Observations** | written, dropped because the ring was full |
| **Reassembly** | slots opened, completed, expired, evicted, abandoned (too many gaps), refused (over the size limit); streams opened and finished; hand-offs |
| **New-flow table** | entries added, evicted; orphan segments held, matched, dropped |
| **ERSPAN shell** | packets per session, lost (sequence gaps), reordered, released early (window full), duplicates removed |
| **Fold** | observations folded, conflicts found |
| **Sensor to taproom** | batches sent, acknowledged, backlog size, batches dropped |
| **Errors** | exceptions caught at the C boundary (should always be zero) |

Two gauges complete them: slots and flow-table entries in use, and the estimated clock offset
of each sensor at the taproom.

## Where to read them

| Consumer | How |
|---|---|
| **Embedded** | `tapster_get_counters` fills a struct the caller provides, in the C API's usual style: no allocation, nothing to free. The host decides how to publish them. |
| **ERSPAN collector** and **single device** | A small HTTP endpoint in the Prometheus text format, written without a dependency, and the same numbers in the sensor's health report to the taproom. |
| **Taproom** | Sensor health per sensor (see [deployment and outputs](deployment-and-outputs.md#what-a-user-gets)). |

## What to watch

| Counter rising | Means |
|---|---|
| observations dropped | the fold cannot keep up: a bigger ring, or a faster fold |
| slots evicted | too many split messages at once: a bigger pool, or a flood |
| ERSPAN packets lost | the mirror or the network to the collector drops packets |
| malformed frames on one layer | a protocol variant the view does not handle yet: worth a capture |
| exceptions at the C boundary | a bug, always |
