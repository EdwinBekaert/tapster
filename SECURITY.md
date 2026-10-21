# Security

Tapster parses packets that anyone on a network can send, and runs inside the forwarding path
of routers. A bug in how it reads a packet can be a vulnerability.

## Reporting a vulnerability

Report it privately through GitHub: on the repository's **Security** tab, choose **Report a
vulnerability**. Please do not open a public issue.

A useful report says:

- what goes wrong (a crash, an exception at the C boundary, memory growing past its limits, a
  read outside a buffer, a loop that does not end);
- the input that triggers it, ideally as a small capture file;
- the version, platform and consumer (embedded, ERSPAN collector, single device).

## What is in scope

What Tapster itself is responsible for (see
[architecture](docs/architecture.md#who-protects-what)): how it handles malformed or hostile
input, its memory and work limits, and its C API. The environment Tapster is embedded in
(isolation, transport between sensor and taproom, access to outputs) belongs to the software
that wraps it.

## Supported versions

Tapster has no releases yet. Until it does, only the `main` branch is supported.
