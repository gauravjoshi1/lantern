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

## 2026-08-23 — EDNS/OPT cache policy

Stage 1's cache strips the OPT record before storing a response and
synthesizes a fresh OPT on every hit from the current requester's own
advertised UDP payload size and DO bit, rather than caching OPT verbatim
(RFC 6891 forbids this) or bypassing caching entirely for any EDNS-bearing
query. Only OPT is excluded and rebuilt per-request; ordinary answer data is
cached and shared across clients as usual. Full RRset caching — the
structural fix real recursive resolvers use, where every response is
synthesized from independently-cached records rather than stored as whole
packets — is explicitly deferred to a later issue: it requires a response
encoder Stage 1 doesn't have, and it would mean reconstructing responses
from parsed fields on every hit, which conflicts with the "Stage 1 DNS
model" decision above (preserve the original packet as closely as
possible).

## 2026-08-23 — UDP/TCP response shaping

When the original downstream client queried over UDP and the real upstream
answer only fits over TCP, Lantern does not perform an internal upstream TCP
fetch on that client's behalf — it returns the already-truncated UDP
response (`TC=1`) and lets the client decide to reconnect over TCP itself,
per RFC 7766's same-transport-response rule. The general upstream TCP
fallback for a truncated upstream UDP response (see "Stage 1 transport"
above) still applies when the downstream client itself queried over TCP and
can use the fuller answer.

## 2026-08-23 — Health check and restart policy

An upstream-reachability failure (the health check's upstream probe) only
ever changes a reported status (logged/exposed as degraded) — it never
triggers a process exit or a restart. Only a genuine liveness failure
(deadlock, a dead listener socket) is wired to the hardware watchdog.
Reserves restart for problems a restart can actually fix — repeatedly
restarting during an upstream/ISP outage can trip `systemd`'s own
crash-loop protection and leave the unit fully stopped, which is worse than
doing nothing.

## 2026-08-23 — Health check bypass mechanism

The health check's probe queries a cryptographically random label under the
reserved `.invalid.` TLD (RFC 6761) — no operator-owned DNS infrastructure
needed, since names under it are guaranteed to never be real. The probe
travels through Lantern's actual pipeline (blocklist check, cache lookup,
upstream exchange, response validation) via an internal trigger, not a
separately-coded shortcut — reusing the real code path is what actually
proves the listener-to-upstream wiring, not just resemblance to it. The
probe is explicitly and narrowly exempted from blocklist matching (scoped to
the internally-generated probe object specifically, not a broader flag a
crafted packet could also trip). Its upstream exchange reuses #8's validated
exchange logic (same anti-spoofing checks) but with its own, more patient
`attemptTimeout`/`maxAttempts`, separate from client-facing values. A valid,
matching `NXDOMAIN` is success.

If Lantern ever implements RFC 6761's suggested local `.invalid` synthesis
for ordinary client traffic, the health-check probe specifically must not
go through that shortcut, or the mechanism silently defeats itself.

Status flips to degraded after 3 consecutive failures, and back to healthy
after 1 success (matching the common liveness/readiness convention). The
check interval backs off, capped, while degraded, and resets to normal
cadence on recovery. None of this ever triggers a process restart (see
"Health check and restart policy" above).

## 2026-08-23 — Blocklist malformed-entry handling

A malformed individual line in the blocklist file is skipped and logged
(with its line number); loading continues. Startup only fails if the
configured file can't be read, or parsing yields zero usable entries. This
replaces the original #10 draft's "any malformed entry fails the whole
load" choice, which traded away too much availability given Lantern is the
only resolver on the network with no fallback if DNS goes down.
