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
