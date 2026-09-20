# Homelab: Network Visibility and Detection Lab

A single-node detection lab: a managed switch mirroring traffic to a
dedicated NIC, Suricata reading the mirror, alerts flowing into Loki/Grafana
with push notifications out the other end.

Recycled hardware, open-source software, and a log of what broke.

## Architecture

```
                    ┌──────────────┐
                    │  ISP gateway │  (all-in-one modem/router/AP)
                    └──────┬───────┘
                      port 2 (uplink)
                    ┌──────┴────────────────────────────┐
                    │   Managed switch (port mirroring) │
                    └──┬──────────────┬──────────────┬──┘
                  port 1          port 3         port 4
              (mirror dest)      (desktop)    (lab primary)
                       │                            │
              ┌────────┴────────────────────────────┴────────┐
              │            labnode  (ThinkPad, Linux)        │
              │   eth1 (promisc, no IP)   eth0 (management)  │
              │         │                        │           │
              │      Suricata                 Docker         │
              │         │                        │           │
              │     eve.json ──► Alloy ──► Loki ──► Grafana   │
              │                                      │       │
              │                                      └► ntfy ─┼──► phone
              └──────────────────────────────────────────────┘
                          Tailscale — no inbound ports
```

The ISP gateway is an all-in-one with no bridge mode, so there's no upstream
chokepoint. This build is scoped to **wired traffic only** and the gap is
documented rather than glossed — see [doc 02](docs/02-network-and-tap.md#known-blind-spots).

## Contents

| Doc | |
|---|---|
| [01 — Hardware and OS](docs/01-hardware-and-os.md) | The node, the second NIC, container layout |
| [02 — Network and tap](docs/02-network-and-tap.md) | Switch, port mirroring, blind spots |
| [03 — Log pipeline](docs/03-log-pipeline.md) | Alloy → Loki → Grafana, labels, retention |
| [04 — Detection and triage](docs/04-detection-and-triage.md) | Suricata, ingest, dashboard |
| [05 — Alerting](docs/05-alerting.md) | ntfy, least-privilege publisher |
| [06 — Remote access](docs/06-remote-access.md) | Tailscale, bindings, key-only SSH |
| [07 — Operations log](docs/07-operations-log.md) | What broke, and known issues |
| [08 — Change log](docs/08-change-log.md) | A remediation pass — and why half the real problems weren't in the configs |

Sanitized config in [`configs/`](configs/).

## State

- [x] Loki / Grafana / Alloy in Docker, pinned versions, memory limits
- [x] Mesh-VPN access, key-only SSH, nothing port-forwarded
- [x] Managed switch with port mirroring to a dedicated, address-less NIC
- [x] Suricata as a managed service, 0% capture drops at idle
- [x] `eve.json` → Loki via Alloy, triage dashboard, ntfy push
- [x] Docker API reached through a filtering proxy, not the raw socket
- [ ] Host metrics — still manual checks
- [ ] Wireless visibility — blocked on gateway architecture (doc 02)
- [ ] Drop rate verified under load, not just at idle

## Sanitization

Every identifier here is a placeholder: `192.0.2.0/24` for the LAN,
`100.x.y.z` for mesh-VPN, `labnode` for the host, `eth0`/`eth1` for the
interfaces, `<redacted>` for anything secret. Runtime data — Loki chunks and
index, Grafana's database and plugins, and any `eve.json` or pcap output — is
gitignored; raw IDS output in particular is a detailed record of internal
addresses, DNS queries and device fingerprints.
