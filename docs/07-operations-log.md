# 07 — Operations log

Building the stack is following documentation. Running it is where you learn
things.

---

## Memory creep

**Symptom.** `free -h` showing 7.7 GiB of 12 GB used. A reboot dropped it to
~900 MiB and it started climbing again. Caught by comparing a post-reboot
reading against a normal one — there was no graph to look at.

**Investigation.** Container memory accounting didn't show growth
proportional to what the host reported, which pointed away from the stack and
toward process or kernel state.

**Cause.** A recursive extraction job left running against an external USB
SSD that was then unplugged without unmounting. The process was stuck in
uninterruptible sleep (`D` state) on I/O against a device that no longer
existed — unkillable, holding its allocations until reboot.

**Lessons.** `umount` before unplugging isn't pedantry: a process in `D`
state can't be killed by any signal, including `SIGKILL`, because the kernel
won't deliver one until I/O returns, and it never returns. And the diagnosis
was slow because there was no history — a monitoring box with no monitoring
of itself is a real gap.

---

## Template

```
## <description>
**Symptom.** What was observed, and how it was noticed.
**Investigation.** What was checked, and what each check ruled out.
**Cause.** What it actually was.
**Lessons.** What changes as a result.
```

> Entries to add: first real true positive and its triage path; the false
> positives that needed tuning; disk over a full retention cycle vs. estimate.

---

## Current known issues

### The Docker socket

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

`:ro` here is close to cosmetic. It makes the socket *file* read-only, but
the socket is a bidirectional API endpoint — a process that can open it can
still POST to create a container, mount the host root into it, and get root.

Fixed by putting a filtering proxy in front (`CONTAINERS=1`, `NETWORKS=1`,
`POST=0`); Alloy talks TCP to the proxy and never holds the socket. Verified
`GET /containers/json → 200`, `POST /containers/create → 403`.

The proxy now holds the socket, so it's the trusted component — a small
purpose-built filter instead of a log collector parsing untrusted input.
An improvement, not an elimination.

Listed at length because "I mounted it `:ro` so it's fine" is a very common
misreading.

### Still open

| Item | |
|---|---|
| Grafana admin password inline rather than from an env file | medium |
| Alert-only ingest — no DNS/TLS context for triage ([doc 04](04-detection-and-triage.md#ingest)) | medium |
| No host metrics; memory, disk and CPU checked by hand | medium |
| Dead `ruler.alertmanager_url` in the Loki config — nothing listens there | low |
| No healthchecks; `restart: unless-stopped` only catches a container that *exited*, not one that's wedged | low |
| NIC offload (GRO/LRO) not disabled on the capture interface | low |
| Vestigial `/etc/default/suricata` with a stale interface name | low |
| `signature_id` label cardinality — 25 today, watch it | low |

The host-metrics gap is the one that made the memory incident above take
longer than it should have, and it's still open.

---

Next: [08 — Change log](08-change-log.md)
