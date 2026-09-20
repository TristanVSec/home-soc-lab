# 01 — Hardware and OS

| | |
|---|---|
| Machine | Lenovo ThinkPad T450, 12 GB DDR3 |
| eth0 | onboard gigabit — management |
| eth1 | USB 3.0 gigabit — capture only, no IP, promiscuous |
| OS | Linux Mint |

## Why a laptop

The battery is a built-in UPS — worth more than it sounds for a node whose
job is not losing events mid-write. Low idle draw for 24/7 operation, and
quiet enough to live in occupied space.

Real limits: 12 GB ceiling, no GPU, one drive bay.

## The second NIC

A USB adapter serves as the mirror destination. The alternative — moving the
server to Wi-Fi to free the onboard port — was rejected: a monitoring node on
a wireless link adds exactly the loss and jitter that makes IDS output
untrustworthy.

The capture interface carries **no IP address** and is set at the
NetworkManager profile level (`ipv4.method disabled`), not with `ip addr del`,
which the next DHCP lease would undo. See [doc 08](08-change-log.md) — it
wasn't always configured this way, and the consequences were not obvious.

## Base hardening

- Password SSH auth disabled, keys only
- Nothing bound to `0.0.0.0` (doc 06)
- Automatic security updates, lid-close suspend off

## Containers

Everything but Suricata runs from one Compose file at
`/opt/security-stack/compose.yml`. Suricata runs on the host — it needs raw
access to the capture interface, and containerizing it buys only indirection.

**CasaOS** is also installed for non-security containers. It claims port 8080,
which collides with roughly every other self-hosted tool's default. Plan
around it.

```
/opt/security-stack/
├── compose.yml
├── alloy/config.alloy
├── loki/config.yml + runtime data
└── grafana/plugins + runtime data
```

Only config is tracked. Runtime directories are gitignored — they're large,
and Loki's index metadata embeds the instance hostname.

---

Next: [02 — Network and tap](02-network-and-tap.md)
