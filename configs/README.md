# Sanitized configuration

Every file in this directory is a **sanitized copy** of a config actually
running on the lab node. They are here to be read and adapted, not deployed
verbatim — you will need to substitute real addresses and generate your own
credentials.

## Placeholder conventions

| Placeholder | Stands for |
|---|---|
| `192.0.2.0/24` | The LAN subnet (RFC 5737 documentation range — never routable) |
| `192.0.2.10` | The lab node's LAN address |
| `100.x.y.z` | A mesh-VPN address |
| `labnode` | The lab host's hostname |
| `labuser` | The admin account |
| `eth0` | Management NIC |
| `eth1` | Capture NIC (mirror destination) |
| `<redacted>` | A credential, token, or key that was removed |
| `${VAR}` | An environment variable sourced from a `.env` file that is not committed |

## What was removed and why

- **All credentials.** Grafana admin password, ntfy account passwords, the
  Grafana webhook URL (it embeds the publisher credential). These live in a
  `.env` file that is gitignored; the committed configs reference them as
  environment variables.
- **Real addresses.** LAN subnet, mesh-VPN addresses, the `HOME_NET`
  definition in the Suricata config.
- **Hostnames.** Replaced with `labnode` throughout, including inside Loki's
  configuration.
- **Runtime data.** Loki chunks and index, Grafana's SQLite DB, Grafana plugin
  bundles, and any `eve.json` / pcap output. All gitignored. Raw IDS output in
  particular is a detailed record of real internal addresses, DNS queries,
  TLS destinations, and device fingerprints — it is the last thing that should
  ever be published.

## Files

| File | Status |
|---|---|
| `compose.yml` | sanitized, complete |
| `alloy/config.alloy` | sanitized, complete |
| `loki/config.yml` | sanitized, complete |
| `suricata/suricata.yaml` | partial — vars only |
| `ntfy/server.yml` | pending |
| `grafana/dashboards/*.json` | pending (export) |

## Over-scrubbing note

When sanitizing these by hand, host port bindings and file paths were blanked
entirely rather than placeholdered. That's the safe direction to err, but a
config with empty `ports:` values doesn't run and doesn't teach. The versions
here restore the structure with documented placeholders — substitute your own
addresses and the files are functional.
