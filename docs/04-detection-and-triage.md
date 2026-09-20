# 04 — Detection and triage

## Suricata

Runs on the host, reading the capture NIC in promiscuous mode with no IP.
Entirely passive — the mirror is a copy, so there's no inline path to drop
anything on. `alert.action` reads `allowed` on every event; that field means
"what the rule said," not "what happened."

Ruleset: Emerging Threats open, via `suricata-update`.

Config: [`configs/suricata/suricata.yaml`](../configs/suricata/suricata.yaml).

### The af-packet block is where the interface comes from

```yaml
af-packet:
  - interface: eth1
    threads: 2
    use-mmap: yes
    tpacket-v3: yes
    ring-size: 32768
    block-size: 1048576
    checksum-checks: no
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
  - interface: default
```

The stock systemd unit passes `--af-packet` and **no `-i` flag**, so this
block is the interface source. A stale name here means the block never
applies and Suricata falls through to `default` with built-in settings —
which is how a tuned-looking config can be doing nothing. See
[doc 08](08-change-log.md).

| Key | Why |
|---|---|
| `ring-size` / `block-size` | raised from defaults; this is what fixed a 2.4% drop rate |
| `threads: 2` | two physical cores — `auto` claims all four and starves the stack |
| `checksum-checks: no` | mirrored frames can arrive with checksums not yet computed |

### HOME_NET

Narrowed from the shipped default (all three RFC1918 ranges) to the actual
monitored subnet. Most of the ET ruleset is directional, so declaring ~18M
addresses internal blunts lateral-movement logic and classifies a spoofed
private-source packet from the internet as local.

The ICS/SCADA variables (DNP3, Modbus, ENIP) ship enabled in every install.
Harmless without the matching rule categories, and a good reminder that the
shipped config is not minimal.

> **TODO:** `eve-log` enabled event types.

## Ingest

```
stage.json    → event_type only
stage.labels  → promote it
stage.match   → drop non-alerts
stage.json    → full extraction, alerts only
stage.labels  → bounded fields only
stage.timestamp
```

Ordering is the point: the cheap single-field extraction runs on every line,
the expensive one only on what survives.

**Current state: alerts only.** That's a real capability loss, not a neutral
optimization — the pivot from an alert to the DNS query before it and the TLS
SNI after it is what turns a signature name into a verdict, and none of that
reaches Loki. The right fix is differentiated retention (alerts 14d, DNS/TLS
24–48h, flow dropped), not ingesting everything. Open.

`src_ip` and `dest_ip` are deliberately not labels — unbounded. Query them
with `| json` instead:

```logql
{job="suricata"} | json | src_ip="192.0.2.50"
```

`signature_id` is the questionable label: bounded by what fires, not by
design. 25 distinct values currently, so fine. If it climbs, demote it to the
body and group on `category`.

**Event time, not ingest time.** Without `stage.timestamp`, a backlog after a
restart lands every replayed event at "now" and the timeline is fiction.

## Dashboard

[`configs/grafana/dashboards/`](../configs/grafana/dashboards/) — in the
order an analyst asks:

1. What fired, how often — once is interesting, every 90 seconds is a noisy rule
2. Severity breakdown
3. Top source/destination in the alerting set, parsed from the body
4. Category grouping and full alert body

Missing: the pivot into surrounding DNS/TLS/flow, for the reason above. The
dashboard is honest about what the data supports.

> **TODO:** panel-by-panel LogQL, sanitized screenshots, and the noisy rules
> actually tuned — suppressed vs. disabled vs. thresholded, with reasoning.

---

Next: [05 — Alerting](05-alerting.md)
