# Deployment and outputs

What a user gets from Tapster, and how several sensors on one network become one view.

## Sensors and the taproom

A network usually has more than one place where Tapster runs: a few routers with the embedded
library, an ERSPAN collector in the data centre, a laptop running the single-device shell.
Each of these is a **sensor**. They all report to one **taproom**, the central service that
merges what they saw.

```
 sensor: embedded router ──┐
 sensor: embedded router ──┤   facts (observations, evidence, edges), batched
 sensor: ERSPAN collector ─┼──────────────────────────────────────────────► taproom
 sensor: single device ────┘                                                 │
                                                       fact log ─► fold ─► devices,
                                                                           dependency map,
                                                                           change stream
```

- **Sensors send facts, the taproom draws conclusions.** A sensor knows only its own segments.
  Whether two interfaces are one device, or two sensors see the same broadcast domain, can only
  be decided where all facts meet.
- **A sensor can run without a taproom.** The single-device shell and a lone collector run a
  taproom in the same process. Embedded sensors always report out: their memory is bounded,
  and a router is not where an inventory should live.

## What a sensor keeps and sends

| On the sensor | Kept for | Sent to the taproom |
|---|---|---|
| Observation ring | until the fold drains it | no |
| Local registry (segments, interfaces, bindings) | while active, bounded | as facts, when something changes |
| Edge counters | one reporting interval | totals per interval |
| Evidence log | one reporting interval | yes |

Facts go out in batches, each numbered per sensor. If the taproom is unreachable, the sensor
keeps a bounded backlog; when that is full, the oldest batches are dropped and counted. A
sensor never blocks its host because the taproom is down.

## The fact log

The taproom stores facts in an append-only log. Devices, the dependency map and everything a
user sees are a fold of that log. This gives three things for free:

- **Restart** is loading the latest snapshot and replaying the log after it.
- **Explanations.** Every conclusion can name the facts it came from ("these two interfaces are
  one device because of these four pieces of evidence").
- **Changing the rules.** A better identity judge can be replayed over old facts.

Each fact carries the sensor id, the sensor's batch number and its capture timestamp. The
schema is versioned from the first release; an old taproom refuses a newer major version
instead of misreading it.

## Several sensors, one network

### The same broadcast domain seen twice

Two sensors on the same VLAN (a router and an ERSPAN collector mirroring its uplink) each
create their own segment. The taproom links two segments when they share broadcast traffic:
the same ARP request, DHCP exchange or LLDP announcement, at the same moment, from the same
MAC. Linked segments are treated as one, and their interfaces with the same MAC merge without
needing the identity judge.

### The same packet counted twice

The same flow can pass a router with Tapster inside and be mirrored to a collector. Edge
counters from linked segments are not added up: per reporting interval, the taproom keeps the
highest count any one sensor reported for that edge, and lists which sensors saw it.

### Clocks

Each sensor timestamps with its own clock. Sensors are expected to run NTP (PTP where
available). The taproom estimates each sensor's offset from shared events (the same broadcast
seen by two sensors) and flags a sensor whose clock drifts beyond a configured limit, since
"both MACs active at the same moment" (see [identity model](identity-model.md)) depends on it.

## What a user gets

| Output | Form |
|---|---|
| **Device inventory** | Devices with their interfaces, addresses over time, names, fingerprints and the confidence of each link |
| **Dependency map** | Edges between devices, with service, kind (configured or observed), counts and first and last seen; exported as JSON and GraphML |
| **Explanation** | For any device or edge, the facts behind it |
| **Change stream** | New device, device gone quiet, new dependency, address changed, identity link added or removed; as JSON lines, for alerting and other tools |
| **Sensor health** | Per sensor: drops, evictions, lost mirror packets, backlog, clock offset |

## Records

The records a user sees, as a sketch:

```
Device      id, label, interfaces[ (interfaceId, confidence) ], firstSeen, lastSeen
Interface   id, segmentId, mac, firstSeen, lastSeen
Binding     interfaceId, routingDomain, ip, from, to, source (arp | nd | dhcp)
Edge        client, server, service, kind (configured | observed), packets, bytes,
            firstSeen, lastSeen, sensors[]
Evidence    id, subject, type, value, sensorId, timestamp
```

Ids are stable across restarts and replays: an interface id is derived from its segment and
MAC, a device id from the first interface that formed it.

## Open questions

- **Transport between sensor and taproom.** Requirements: authenticated both ways, encrypted,
  batched, resumable from a batch number. Securing it is the deployment's job (see
  [architecture](architecture.md#who-protects-what)).
- **Storage format of the fact log.** A simple length-prefixed file per day is enough to start;
  a database only when queries need it.
- **Query interface** for the inventory and the map.
