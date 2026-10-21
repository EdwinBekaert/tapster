# Privacy and data protection

Tapster watches network traffic, and network traffic is full of personal data: a MAC address
belongs to someone's phone, a hostname is often a person's name ("Annas-iPhone"), and DNS
names show what people visit. This page sets out what Tapster does to collect as little as
possible, and what stays the responsibility of whoever deploys it.

This is a design document, not legal advice.

## Minimisation is the design

Tapster's core principle, read the packet where it is and copy out only the values that
identify a device, is data minimisation. Most of what passes a sensor is never stored:

| Data | Seen | Stored |
|---|---|---|
| Packet payloads (web pages, mail, files) | yes | **never** |
| Full packets | yes | **never**, except a [hand-off](reassembly.md) copy held in memory until processed |
| MAC and IP addresses, and when they were active | yes | yes: the inventory needs them |
| Hostnames from DHCP, mDNS, NetBIOS | yes | yes |
| Fingerprints (DHCP, TCP, TLS) | yes | as ids: they describe software, not people |
| Server names from TLS (SNI) and DNS | yes | **per edge, not per visit**: "this device uses this service", with counts, never a list of every lookup with its time |
| DNS names that are not services of the network itself | yes | configurable; by default only names that resolve to addresses inside the network |
| Traffic volumes | yes | per edge and per reporting interval |

## Personal data in the records

- **MAC addresses, IP addresses and hostnames** are treated as personal data throughout. They
  identify a device, and a device usually belongs to a person.
- **Names inside hostnames** are the most direct link to a person. A deployment can have
  hostnames reduced to a pattern ("*-iPhone") or dropped altogether.
- **Fingerprints** on their own identify a model or OS version, not a person.

## Rotating MAC addresses

Phones and laptops rotate their MAC address to stop exactly the kind of tracking that
[identity resolution](identity-model.md#rotating-mac-addresses) does. Tapster resolves rotation
for one purpose: keeping an accurate inventory of the devices on a network someone is
responsible for. So:

- Rotation linking can be switched off per segment, for example on a guest network.
- Links are scored and explainable, never silent.
- Tapster does not follow devices across networks: links exist only within one deployment.

## Retention and erasure

- **Retention is configured per kind of record**, with defaults short enough to need a reason to
  extend: evidence and edge details 90 days, bindings 90 days after they end, devices removed
  after 180 days without activity.
- **Facts expire in the fact log** as well as in the views built from it; a snapshot does not
  keep what the log has dropped.
- **Erasing one device** must also erase it from the append-only fact log. Facts about an
  interface are encrypted with a key per interface; erasure deletes the key, so the facts can no
  longer be read and the next fold leaves them out.

## Exports

- Exports can **pseudonymise** MAC addresses and hostnames with a keyed hash. The key belongs
  to the deployment, so the same device keeps the same pseudonym across exports without the
  real value leaving.
- Nothing is sent to a third party by default. Optional integrations that would (for example an
  online fingerprint service, see [fingerprint sources](fingerprint-sources.md)) are off until
  configured, and send fingerprints, never addresses or names.

## The deployer's part

Tapster is a tool. Whoever runs it decides what is collected and why, and is responsible for:

- having the right to monitor the network: only networks they own or administer;
- a legal basis for processing the personal data, and telling the people concerned;
- local rules on monitoring employees' electronic communications, which in several countries
  (Belgium's collective agreement no. 81, for example) set conditions on purpose, proportionality
  and information;
- choosing retention, rotation linking and export settings that fit their purpose.

The [active probing](roadmap.md#last-active-probing) that comes last has a stronger requirement
still: sending probes needs explicit authorisation for every network probed.
