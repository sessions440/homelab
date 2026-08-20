# VPS Relay for CGNAT Traversal

**Status:** Planned — VPS not yet provisioned
**Last updated:** 2026-08-20

---

## Context / Goal

Starlink residential internet uses CGNAT: no public IP is assigned to the home
router, so inbound connections from the internet are categorically impossible,
regardless of port-forwarding configuration. This is a fixed architectural
constraint, not a one-off Minecraft problem — any future service that needs to
accept connections from outside the LAN will need the same kind of solution.

This doc covers the **generic mechanics of a self-hosted VPS relay**:
provisioning, security baseline, and FRP setup. Service-specific tunnel
configuration (which local port maps to what, DNS records, etc.) lives in that
service's own doc. See `docs/services/minecraft.md` for the first consumer of
this pattern.

---

## Decision Summary

Two questions were worked through (full reasoning in chat history; condensed
here):

**1. FRP vs. hand-rolled WireGuard + iptables DNAT, to run the relay itself:**
Chose **FRP** (Fast Reverse Proxy). Both route traffic between a home LXC and
a public VPS; FRP is purpose-built for exactly this pattern and handles the
tunnel plus port-forwarding as a single config block per exposed service,
versus hand-writing WireGuard peer config, iptables DNAT rules, and dealing
with connection-tracking debugging. Meaningfully less to get wrong for someone
newer to networking, and trivially extensible — adding a second tunneled
service later is a few lines of config, not new firewall rules.

**Caveat:** unlike WireGuard, FRP's transport is not encrypted by default —
`transport.tls` must be explicitly enabled (done in the config below). Low
stakes here since none of the traffic through this relay is expected to be
sensitive, but worth keeping on as a default habit.

**2. Self-hosted VPS+FRP vs. playit.gg** (a turnkey third-party relay service
purpose-built for game servers):
Chose **VPS+FRP**. playit.gg is legitimate and would work fine for Minecraft
specifically — zero player-side install, generous free tier, open-source
host-side agent. Genuinely a defensible alternative choice. VPS+FRP won on:

- **Reusability:** this VPS becomes general-purpose infrastructure for *any*
  future TCP/UDP service, not scoped to game hosting. Concretely relevant if a
  future service (e.g. a mail relay) needs a real IP with a controllable
  reverse-DNS record — a third-party relay can't provide that.
- **Zero recurring cost:** Oracle Cloud's Always Free tier is free
  indefinitely, not a trial.
- **Upskilling:** provisioning, hardening, and running a small internet-facing
  box is a transferable skill for the rest of this project.
- Latency was assessed as roughly a wash between the two — both depend on
  regional routing choices.

`playit.gg` remains a documented fallback if VPS+FRP proves too much ongoing
overhead. See `docs/services/minecraft.md` for the concrete comparison as it
applied there.

---

## Architecture

```
[External clients] --TCP/UDP--> [VPS: frps, public IP] --encrypted tunnel--> [Home LXC: frpc] --> [local service]
```

The VPS never sees or stores anything beyond relaying bytes — the actual
service runs entirely on the home LXC, same as everything else in this
homelab.

---

## Part 1 — VPS Provisioning (manual, human only)

This section cannot be delegated to a coding agent — it requires the Oracle
web console and can't be reached via SSH until the instance exists.

### Oracle Cloud account

1. Create an account at https://www.oracle.com/cloud/free/. A mobile number
   and credit card are required for identity verification — **the card is not
   charged** unless you explicitly upgrade to a paid tier.
2. When choosing a home region, check whether `ca-toronto-1` is offered and
   has Always Free capacity — best latency to you and likely to most players.
   Confirm Always Free resources are actually available before committing;
   availability varies by region.

### Instance creation

1. Compute → Instances → Create Instance.
2. **Shape:** the **AMD Micro** shape (`VM.Standard.E2.1.Micro`), not the ARM
   Ampere shape. Both are Always-Free-eligible, but the ARM shape is popular
   enough that "out of host capacity" errors are common in busy regions. The
   AMD micro instance is more reliably available and is more than sufficient
   for a relay workload — this box does nothing but forward bytes.
3. **Image:** Debian (latest stable), matching this homelab's OS convention.
   If Debian isn't offered as an Always-Free-eligible image in your region,
   Ubuntu LTS is the documented fallback — note which one you actually used
   in the Status section below once done.
4. **SSH keys:** generate a **new, dedicated keypair** for this box — do not
   reuse `ai_homelab` or `human_homelab`. Suggested name: `vps_relay` (human
   key). See "SSH access model" below for the agent-side key. Rationale: this
   is the homelab's only genuinely internet-facing SSH endpoint; a compromise
   here shouldn't be a credential that also works anywhere else. Paste the
   public key into the instance creation form.
5. Create the instance. Note the assigned public IP — see "Is the VPS IP
   sensitive?" below for how to treat it.

### SSH access model

Recommend a **second** dedicated key, e.g. `ai_vps_relay`, authorized on this
box for the coding agent, separate from `ai_homelab` (scoped to LAN service
LXCs). This box lives in a different trust domain than the rest of the
homelab — worth keeping its credential separate so revoking agent access to
one doesn't require touching the other.

Unlike the Proxmox host (hard off-limits for the agent because a compromise
there cascades to everything), this VPS is architecturally closer to a
service LXC: compromising it only exposes itself and whatever tunnels are
actively configured through it. Treating it like a service LXC — agent gets
SSH access via a dedicated key, same as Caddy/git/Vaultwarden/Minecraft — is
consistent with existing conventions. This is the first time agent access has
extended to a box with a public IP; confirm you're comfortable with that
framing before authorizing the key.

### Is the VPS public IP sensitive?

No — treat it differently from the domain name / Cloudflare token, which are
kept out of agent-touched files because the services they protect are *meant
to stay unreachable from outside* ("security through inaccessibility"). This
VPS is the opposite: its whole purpose is to be a publicly reachable,
publicly connectable endpoint. Its IP will be publicly resolvable via DNS the
moment a service's A record points at it. No special handling needed — write
it in configs/docs normally.

---

## Part 2 — Security baseline (manual, before handing off to the agent)

Two separate firewall layers exist for any cloud VPS — a common first-timer
trip-up is only configuring one:

1. **Oracle's cloud-level firewall** (Security List or Network Security
   Group, in the OCI console): must explicitly allow inbound TCP on whatever
   port(s) will be relayed (25565 for Minecraft) plus port 22 for SSH.
2. **The OS-level firewall inside the instance** (iptables/ufw): must
   independently allow the same ports. Traffic can be blocked by either layer
   even if the other is wide open.

Also, before handing off to the agent:

- Confirm password authentication is disabled for SSH
  (`PasswordAuthentication no` in `sshd_config`) — should be default on
  Oracle's images, but verify rather than assume.
- Consider `fail2ban` on sshd given this box is genuinely internet-facing,
  unlike the LAN-only convention everywhere else in this homelab.
- Enable unattended security upgrades (`unattended-upgrades` package on
  Debian/Ubuntu) — this box needs tighter patch discipline than LAN-only
  infrastructure, since exposure level should track patch urgency.

Once SSH access works with the `ai_vps_relay` key and both firewall layers
are open for the relevant ports, the rest of this doc (and any service's
specific FRP config) can be handed to the coding agent.

---

## Part 3 — FRP installation (agent-executable)

> Written as instructions for a coding agent with SSH access to both the VPS
> and the relevant home LXC. Concrete port/service values for a given service
> (e.g. Minecraft) live in that service's own doc; this section stays
> generic.

### On the VPS (`frps` — the server side)

```bash
# Check the current release before downloading — do not assume a version
# number from this doc is still current:
# https://github.com/fatedier/frp/releases
FRP_VERSION="<check releases page>"
wget https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_amd64.tar.gz
tar xzf frp_${FRP_VERSION}_linux_amd64.tar.gz
sudo mv frp_${FRP_VERSION}_linux_amd64/frps /usr/local/bin/
sudo mkdir -p /etc/frp
```

`/etc/frp/frps.toml`:
```toml
bindPort = 7000

# Auth token — required, prevents unauthorized clients from using this relay
auth.method = "token"
auth.token = "REPLACE_WITH_GENERATED_TOKEN"

transport.tls.force = true
```

Generate a token with `openssl rand -hex 32` and use the same value in the
client config below. Never write this token as a literal value in any file
destined for the git repo — same handling as the Cloudflare API token
pattern already used for Caddy.

`/etc/systemd/system/frps.service`:
```ini
[Unit]
Description=FRP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
sudo systemctl status frps
```

### On the home LXC (`frpc` — the client side)

Same download pattern, but extract `frpc` instead of `frps`.

`/etc/frp/frpc.toml` — base config (service-specific `[[proxies]]` blocks
live in the consuming service's own doc):
```toml
serverAddr = "<VPS_PUBLIC_IP>"
serverPort = 7000

auth.method = "token"
auth.token = "REPLACE_WITH_SAME_TOKEN_AS_SERVER"

transport.tls.enable = true
```

`/etc/systemd/system/frpc.service`:
```ini
[Unit]
Description=FRP Client
After=network.target

[Service]
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## Part 4 — Adding a future tunneled service

This is the reusability payoff the VPS+FRP decision was partly based on. To
expose a new service later:

1. Add a new `[[proxies]]` block to the existing `frpc.toml` on whichever LXC
   runs the new service (or a new `frpc` install on a new LXC, pointed at the
   same `frps`).
2. If it's a new port, open it in both firewall layers on the VPS (Oracle
   console + OS-level).
3. Restart `frpc`.
4. Add a DNS record if the service needs a hostname.

No changes to `frps` itself are needed for most additions.

---

## Alternative considered: WireGuard + iptables DNAT (not chosen)

Documented for completeness, and in case FRP ever proves unsuitable.

Same relay topology, but the tunnel is a standard WireGuard point-to-point
link between the VPS and the home LXC, and forwarding is done with manual
`iptables` DNAT rules on the VPS rather than FRP's built-in proxying.

```bash
# On the VPS, once the WireGuard interface (wg0) is up and the home LXC is
# reachable at its WG-internal IP:
iptables -t nat -A PREROUTING -p tcp --dport 25565 -j DNAT --to-destination <LXC_WG_IP>:25565
iptables -A FORWARD -p tcp -d <LXC_WG_IP> --dport 25565 -j ACCEPT
sysctl -w net.ipv4.ip_forward=1
```

Trade-offs vs. FRP: WireGuard is encrypted by default (no TLS toggle to
remember), and it's a tool already used elsewhere in this homelab for remote
LAN access. Against that: every new exposed port needs a new manual DNAT rule,
and NAT/connection-tracking issues are a genuinely fiddly thing to debug for
someone newer to networking. FRP was judged the better fit given that;
revisit this section if that calculus changes.

---

## Cost

$0/mo on Oracle's Always Free tier, as long as usage stays within the AMD
Micro shape's always-free allocation. Credit card on file is for identity
verification only and is not charged absent an explicit upgrade.

---

## Status

- **Planned.** Not yet provisioned. Next step: manual Oracle account +
  instance creation (Part 1), then hand off Parts 2–3 to the coding agent.
