# 03 — Log pipeline

| Component | Version | Job |
|---|---|---|
| Grafana Alloy | v1.10.2 | tail, label, ship |
| Loki | 3.5.7 | store — label index, compressed chunks |
| Grafana | 12.1.1 | query and dashboards |

Config: [`compose.yml`](../configs/compose.yml). Every image is pinned to an
exact version — an unattended `pull` that moves Loki's schema handling or
Alloy's config syntax takes your visibility offline at the moment you don't
notice.

## Why Loki, not ELK

Elasticsearch full-text indexes every field; the index often exceeds the raw
logs and the JVM wants gigabytes of heap. Loki indexes **only labels** and
compresses the rest into chunks.

The tradeoff is real: label queries are fast, body greps genuinely scan the
chunks. This workload — "alerts of this category in this window" — is a label
query. It's the difference between the stack running on this hardware and
not.

## Label design

Every unique combination of label values is a separate Loki stream. A source
IP in a label means one stream per host pair, and you've rebuilt the problem
you chose Loki to avoid.

| Label | Cardinality | | In the body | |
|---|---|---|---|---|
| `severity` | 4 | | `src_ip` / `dest_ip` | unbounded |
| `category` | ~tens | | `dest_port` | ~65k |
| `action_taken` | 2 | | `signature` | thousands |
| `job` / `host` / `container` | handful | | | |

## Collection

Three paths, in [`config.alloy`](../configs/alloy/config.alloy):

1. **Container logs.** `discovery.docker` polls every 5s, so a new container
   is collected without a config change. Relabeling strips Docker's leading
   slash from names. It reaches the API through a **filtering proxy**, not
   the socket — see [doc 07](07-operations-log.md#the-docker-socket).
2. **Host syslog.** `syslog` and `auth.log`. The latter is every SSH auth
   attempt on the node.
3. **Suricata `eve.json`.** Gets its own pipeline — [doc 04](04-detection-and-triage.md).

Alloy runs unprivileged: `group_add: "1002"` grants read on Suricata's log
directory and nothing else, and every host mount is `:ro`.

## Storage

[`loki/config.yml`](../configs/loki/config.yml). Single-binary, filesystem
store, TSDB index (schema v13), replication factor 1.

```yaml
ingestion_rate_mb: 16              # default 4
ingestion_burst_size_mb: 32        # default 6
reject_old_samples_max_age: 168h
retention_period: 336h             # 14 days
```

Ingestion limits are raised because Suricata on a mirrored uplink is burstier
than application logs. Hitting the limit means 429s and dropped events during
exactly the activity the system exists to catch.

> **`retention_period` does nothing on its own.** Deletion is the compactor's
> job and only happens with `retention_enabled: true`. Set the period, skip
> the compactor, and Loki fills the disk while its config claims 14 days.

### `auth_enabled: false`

No credential of any kind. Anything that reaches port 3100 can read every log
and write forged ones. It's published on loopback only and Alloy reaches it
over the Docker network — **that bind is the entire access control**, which is
worth being clear about rather than assuming the setting does something.

> **TODO:** measured disk over one full retention cycle vs. the estimate.

---

Next: [04 — Detection and triage](04-detection-and-triage.md)
