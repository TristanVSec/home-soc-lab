# 06 — Remote access

**Nothing is exposed to the internet.** No port forwards serve this stack.
Access is over a mesh VPN.

## Tailscale

| Setting | Value | Why |
|---|---|---|
| ACL scope | `autogroup:member` | own devices only, no sharing |
| Funnel | **off** | it publishes a service publicly — the opposite of the goal |
| Tailscale SSH | **off** | own sshd, own keys; one less component holding them, and SSH behaves the same on and off the tailnet |
| Exit node | not enabled | not needed |

## Bindings

Nothing binds to `0.0.0.0`.

| Service | Bound to |
|---|---|
| Grafana | LAN address **and** mesh-VPN address |
| ntfy | mesh-VPN only |
| Loki | loopback only |

Grafana is deliberately reachable from the LAN as well; ntfy and Loki aren't.
Loki's case matters most — `auth_enabled: false` means there's no credential
behind that bind (doc 03).

Naming the interface makes isolation a property of the service, not of the
perimeter. Binding to `0.0.0.0` and trusting NAT means one stray port
forward, one UPnP request, or one guest on the Wi-Fi is the whole control.

Check rather than remember: `docker port <container>` reports what a service
is actually published on, which is not always what you configured.

## SSH

Key-only, password auth disabled, reachable over the tailnet, no port 22
forward. Administered from a phone via iSH with a shell alias.

```sh
alias homelab='ssh labuser@<tailnet-address>'
```

Practical, not a gimmick: an ntfy alert arrives and investigating it is two
taps to a shell. Detect, notify, investigate — the whole loop closes from a
phone on any network.

## Deliberately absent

No reverse proxy publishing services, no dynamic DNS, no VPN server on the
gateway, no port forwards. If something needs reaching, the answer is "put
the client on the tailnet."

---

Next: [07 — Operations log](07-operations-log.md)
