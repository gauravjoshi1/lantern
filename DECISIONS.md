# Decisions

Decisions already settled for this project, so a new Claude/Codex session
doesn't need to re-derive them or suggest something that contradicts them.
Append-only — add a new entry when something changes rather than editing an
old one.

## 2026-08-22 — DNS parsing

Hand-rolled wire-format parsing, not `miekg/dns`, for the initial resolver
stage. Stage-scoped, not permanent — a library may replace this once the
resolver is further along, so don't treat "no dependencies" as a hard rule
elsewhere in the project.

## 2026-08-22 — Deployment shape

Cross-compile on the dev machine for the Pi's ARM64 target, `scp` the binary
over, run as a systemd unit (`Restart=always`,
`AmbientCapabilities=CAP_NET_BIND_SERVICE`). No Docker/container runtime on
the Pi itself. Logs via `journalctl`.

## 2026-08-22 — Remote access

Tailscale only. Don't suggest port forwarding or exposing a service directly
to the public internet.

## 2026-08-22 — Reliability constraints

Boots from a USB SSD, not an SD card. Runs under a hardware watchdog. Health
checks must perform a real DNS resolution against a test domain — a
process-alive check alone isn't sufficient. The Pi's own `resolv.conf` must
never point at itself (self-referential resolution would deadlock if the
resolver is down).

## 2026-08-22 — Stage 1 DNS model

Stage 1 is a transparent caching DNS forwarder, not a recursive resolver:
queries are forwarded to a configured upstream resolver and cached/filtered
in between, not resolved iteratively against authoritative servers.
Forwarding preserves the original query packet as closely as possible (EDNS,
DNSSEC material, unknown record types) rather than reconstructing it from
only the fields Lantern parses.

## 2026-08-22 — Stage 1 transport

Both UDP and TCP are in scope for Stage 1: a downstream TCP listener (for
clients that retry over TCP) and upstream TC=1 fallback (for truncated UDP
responses from the upstream resolver). Not deferred to a later stage.

## 2026-08-22 — Blocked-domain response

A blocked domain is answered with a synthesized NXDOMAIN response (client
transaction ID and question preserved, QR set, recursion bits handled
consistently with the normal proxy path) rather than REFUSED or a synthetic
null address.

## 2026-08-22 — Cache and query case handling

On a cache hit, the stored response's question section is patched to match
the current client's exact query capitalization rather than returned as
originally cached. This matters for clients using DNS 0x20 case
randomization as an anti-spoofing check.

## 2026-08-22 — Blocklist source scope

Stage 1's blocklist only loads from a local file (one-domain-per-line and
conventional hosts-file formats). Remote fetching, refresh scheduling, and
staleness policy are explicitly out of scope for Stage 1 and belong to a
later issue.

## 2026-08-22 — Health check semantics

The health check must prove current upstream reachability, not just that
Lantern can serve a client from cache — it bypasses the cache and requires a
real round-trip to the upstream resolver to pass. Extends the existing
reliability constraint that health checks must perform a real DNS
resolution.
