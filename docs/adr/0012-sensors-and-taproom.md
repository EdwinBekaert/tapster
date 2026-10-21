# 0012. Sensors send facts, the taproom folds them

Date: 2026-10-21 · Status: Accepted

## Context
A network has several sensors: embedded routers, collectors, single machines. Each sees part of it; some conclusions (two sensors seeing one VLAN, one device under two MACs) need all of them.

## Decision
- Sensors send facts (observations, evidence, edges) in numbered batches to a central **taproom**.
- The taproom keeps an append-only fact log; devices, the dependency map and the change stream are a fold of it.
- A single-device shell or a lone collector runs a taproom in its own process; embedded sensors always report out.
- A sensor keeps a bounded backlog and never blocks its host when the taproom is down.

## Consequences
- Restart is snapshot plus replay; conclusions can name their facts; a better judge can be replayed over old facts.
- The fact schema is versioned from the first release.

## Alternatives considered
- **Each sensor keeps its own inventory:** no merging across sensors, and inventories on routers.
