# 0010. Device identity as scored, explainable links

Date: 2026-10-21 · Status: Accepted

## Context
Phones and laptops rotate their MAC addresses, so one device appears as several interfaces. Resolving this is a v2 goal that must be solved, and the right method is not settled.

## Decision
- Linking interfaces into devices goes through `IIdentityJudge`, which returns same, different or unknown, with a confidence and the evidence used.
- Deterministic rules run first: both MACs active at once means different devices; a matching DUID or client identifier means the same.
- Only ambiguous pairs reach a further judge. Each verdict is stored as evidence, with the judge's name and version.
- A device is a view over interfaces: linking or unlinking rewrites nothing else.
- The evidence log is kept from v1, before any judge exists.

## Consequences
- Judges can be replaced and compared on a labelled set of pairs.
- Every link can be explained and undone.

## Alternatives considered
- **Merge on matching fingerprints:** every device with the same OS version would merge.
- **Hard merges:** a wrong merge could not be undone.
