# 05 — Alerting

A dashboard you have to remember to check isn't a detection capability.
Closing that gap is cheap.

## ntfy

Self-hosted pub/sub push notifications, in a container, so alert content
never leaves the network.

```yaml
- NTFY_AUTH_DEFAULT_ACCESS=deny-all
- NTFY_AUTH_FILE=/etc/ntfy/auth.db
```

ntfy's default is open topics. `deny-all` inverts that, so every permission
below is additive from zero.

| Account | Permission | Used by |
|---|---|---|
| admin | read + write | phone, subscribing |
| publisher | **write only** | Grafana's contact point |

Grafana holds a credential that can publish and nothing else — it can't read
the topic or administer the server. A credential sitting in a contact-point
config has a nontrivial chance of leaking, and the blast radius should be
"someone can send me spam," not "someone can read my alert history."

Five minutes at build time, annoying to retrofit, and most setups skip it for
a single admin token.

Bound to the mesh-VPN interface only — push reaches the phone because the
phone is on the tailnet, not because anything is exposed. See
[doc 06](06-remote-access.md).

## Grafana → ntfy

Alert rules evaluate LogQL against Loki on a schedule and fire a webhook
contact point pointed at the ntfy topic.

> **TODO:** contact point definition (credentials redacted), the rules and
> thresholds in use, and which classes page immediately vs. fire on a rate
> increase vs. stay dashboard-only. Alert fatigue is the failure mode — an
> alerting system that fires constantly is worse than none, because it comes
> with a false sense of coverage.

Test the path after any ntfy change. A broken alert path is silent by nature.

---

Next: [06 — Remote access](06-remote-access.md)
