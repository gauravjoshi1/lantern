# lantern

A self-hosted network appliance for Raspberry Pi — DNS filtering, gateway
routing, network observability, and (eventually) a natural-language interface
for diagnosing what's happening on your network.

## Why

Tools like Pi-hole, AdGuard Home, and OPNsense already solve pieces of this
well. `lantern` isn't trying to replace them — it's a from-scratch, hackable
alternative for people who want to run *and understand* their own home network
stack end to end, rather than operate a black box. Everything is built to be
read, modified, and reasoned about.

## Status

Early-stage and actively developed. Not yet installable — check back as the
project matures.

## Planned capabilities

| Area | Capability |
|---|---|
| **Filtering / gatekeeping** | DNS-based ad and tracker blocking; blocklist management |
| **Routing / gateway** | Inline gateway for a network segment: DHCP, NAT, firewalling |
| **Observability** | Device discovery, flow accounting, latency probes, packet capture |
| **Diagnosis** | Natural-language queries like "show me all devices" or "what's causing issues on the network?" |

## License

[MIT](LICENSE)
