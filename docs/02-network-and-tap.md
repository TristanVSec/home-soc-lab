# 02 — Network and tap

## The problem

A single ISP-provided all-in-one gateway, every device hanging off it, and no
place to observe traffic from. No SPAN port, no netflow, no useful syslog.
You can't write detections for traffic you can't see.

## The tap

A managed switch with port mirroring copies frames from selected ports to a
designated port, where a passive listener reads them without being in the
path.

**TP-Link TL-SG108E**, 8-port, ~$25. Unremarkable, which is the point — port
mirroring stopped being an enterprise feature a long time ago.

| Port | Device | Role |
|---|---|---|
| 1 | eth1 | **mirror destination** |
| 2 | ISP gateway | uplink — monitored |
| 3 | desktop | monitored |
| 4 | eth0 | lab management — monitored |

Mirroring the **uplink** is the load-bearing choice: every packet between any
wired device and the internet crosses port 2, so one source captures all
north-south traffic. Ports 3 and 4 add east-west.

### Oversubscription

A mirror destination receives the combined traffic of every source. Three
gigabit sources onto one gigabit destination can overrun it, and the switch
drops frames **silently** — the IDS simply never sees them. Theoretical on a
home link where the WAN is the real bottleneck, and the first thing to check
anywhere larger.

## Known blind spots

- **All wireless.** Wi-Fi clients associate with the gateway's built-in AP
  and are bridged internally, never crossing the switch. Phones, TVs, every
  IoT device — invisible.
- **Upstream of the gateway.** No WAN-side visibility.
- **Encrypted payloads.** Suricata sees TLS metadata — SNI, JA3/JA4, cert
  details — not plaintext. Still a lot of signal, but metadata-level.

Closing the wireless gap needs the gateway in bridge mode plus a separate
router and AP downstream of the switch. Real hardware cost, out of budget
this round. It's the highest-value upgrade available and is written down
rather than implied away.

---

Next: [03 — Log pipeline](03-log-pipeline.md)
