# 08 — Change log

A remediation pass over the running stack. The interesting part isn't that
the configs got fixed — it's that **half of what was actually wrong wasn't in
the configs at all.**

## Found by reading configs

**HOME_NET at the shipped default.** All three RFC1918 ranges on a network
using one /24. Narrowed. Most of the ET ruleset is directional and depends on
that variable being honest.

**Docker socket in the log collector.** `:ro` isn't a boundary (doc 07).
Replaced with a filtering proxy; verified `200` on read, `403` on write.
`NETWORKS=1` was added only after a 403 revealed `discovery.docker` calls
`/networks` for labels — deny-by-default, widened one endpoint at a time
against observed failures.

**Unpinned image, no memory limits, dead ruler block.** Pinned; `mem_limit`
on every service (not `deploy.resources.limits`, which plain
`docker compose up` ignores). Ruler block still there — noted, not silently
claimed fixed.

## Found only by looking at the running system

**The IDS wasn't a managed service.** The systemd unit existed and was
*disabled*. Suricata was running from a hand-typed `sudo suricata -i <nic>`
issued days earlier. It worked, it produced alerts, and it would not have
survived a reboot — with the only symptom being alerts quietly stopping,
indistinguishable from a quiet network. No config file says this.
`systemctl is-enabled` does.

**The capture NIC held an address and the default route.** The
mirror-destination interface had a DHCP lease and, at a lower route metric
than the management NIC, was the host's *primary path to the internet*. The
interface meant to sit silent was carrying the server's own traffic and could
ARP on the segment it was watching. A managed switch passes normal traffic on
a mirror destination port, which is why nothing looked broken.

**Two Suricata instances were splitting the traffic.** After enabling the
service, `ps` showed the old hand-started process still alive alongside
systemd's — both on `cluster-id: 99`. AF_PACKET fanout *distributes* flows
across every socket in a group rather than copying to each, so for ~50
minutes each saw a subset and neither saw all of it.

The cause is worth recording: `pkill -x suricata` matched nothing, because
Suricata renames its main process to `Suricata-Main` in daemon mode. The
`pgrep` used to confirm the kill had the same flaw and reported success.
**A stop confirmed by the same pattern that performed it is not a
confirmation.**

**The drop rate had a different cause than the configs suggested.**

```
NIC level   RX dropped:      40 / 35,528,039     ← fine
Suricata    kernel_drops: 867,309  (2.39%)       ← ring overflow
```

Different layers. The initial hypothesis — NIC offload and checksum
validation — was wrong. The ring was at defaults because the af-packet block
named an interface that didn't exist, so it never applied. Naming the real
interface and raising `ring-size`/`block-size` took drops to **0**, CPU at
10%.

## Verified vs. assumed

| Measured | |
|---|---|
| Drop rate before → after | 2.39% → 0% |
| Proxy read / write | 200 / 403 |
| `signature_id` cardinality | 25 |
| Memory ceilings | confirmed in `docker stats` |

**Not measured:** drop rate under load. Zero drops at ~310 packets/sec is an
idle sample, not a stress test. Honest verification is a saturated transfer
from a monitored host, and it hasn't been run.

## The takeaway

Three of the four worst problems — the unmanaged service, the addressed
capture NIC, the duplicate instances — were invisible in the configuration
and would have survived any amount of config review. They came from
`systemctl is-enabled`, `ip route`, and `ps`.

Configs describe intent. The running system is what's true. Nothing in the
file tells you when they disagree.
