# Hermes Raspberry Pi Learnings

A running journal of setting up a Raspberry Pi 5 as a home server — hardware,
security hardening, services, and the internet-exposure decisions along the way.
Written up so others building a similar homelab have a real reference, mistakes
and all.

> This isn't a polished tutorial — it's an honest log of an actual setup done by
> someone new to Linux, with an AI agent (Hermes) helping run and document the
> machine. Steps, gotchas, and rollback notes are included so you can adapt them
> to your own Pi.

## The setup

- **Hardware**: Raspberry Pi 5, 8GB RAM, booting from microSD.
  - microSD write amplification is a real constraint — prefer lightweight
    services, watch SD card wear over time, consider migrating to SSD/NVMe boot
    later.
- **Remote access**: an AI agent (Hermes) manages the box via terminal and
  Telegram, following a strict tiered-approval model for anything risky (see
  [Approval model](#approval-model) below).

## Internet exposure

Three hostnames are published via a **Cloudflare Tunnel** (no port-forwarding,
no open router ports — the tunnel makes an outbound connection from the Pi to
Cloudflare):

| Hostname | Points to | Access restriction | Notes |
|---|---|---|---|
| `nextcloud.<domain>` | Nextcloud | public | original service |
| `<agent>.<domain>` | Hermes control endpoint | public | deliberate choice — Cloudflare Access was considered and declined for convenience |
| `vnc.<domain>` | `tcp://localhost:5900` (wayvnc) | public, no Access gate | PC-only; phone VNC goes over Tailscale instead |

**Lesson learned**: a Cloudflare Tunnel *feels* local-only because it's an
outbound connection, not an open inbound port — but the effect for anyone
finding the hostname is identical to a public server. Treat every tunnel
hostname as a fresh "am I okay exposing this" decision, not a rubber stamp
because you already have others.

## Security hardening

- **fail2ban** — sshd jail: 5 failed attempts in 10 minutes → 10 minute ban.
- **ufw** — default-deny inbound. SSH, Samba, Portainer, and VNC (LAN), plus
  Nextcloud, restricted to the local subnet. Tunnel traffic is unaffected by ufw
  since it originates as an outbound connection from the Pi.
- **unattended-upgrades** — security-only updates, auto-reboot at 04:00 IST if
  needed. Docker and RPi firmware repos excluded from auto-upgrade (too risky to
  auto-apply — you don't want a botched firmware update with nobody watching).
- **Tailscale** — installed with `tailscale up --ssh`, giving a second,
  private SSH path over the tailnet in addition to normal SSH. Useful for phone
  access (e.g. VNC over Tailscale) without adding another public hostname.
- **SSH key-only auth** — planned but not yet done at time of writing. This is
  a high-risk change (can lock you out of the box entirely) — see the
  [approval model](#approval-model) for how it's being staged safely.

## Services running

| Service | Access | Notes |
|---|---|---|
| Nextcloud | Public (tunnel) | file sync/storage |
| Hermes (AI ops agent) | Public (tunnel) | manages the box itself |
| wayvnc (VNC) | LAN + tunnel (PC) / Tailscale (phone) | serves its own virtual display, **not** the physical X11 display — can't see local-only dialogs that require a real monitor |
| qBittorrent | LAN only | systemd `--user` service, `Restart=on-failure` |
| Uptime Kuma | Tailscale only | uptime monitoring dashboard, deployed via `docker compose` |
| Tailscale | — | private mesh VPN, also used for phone access |

## Approval model

Because this Pi is internet-facing and the owner is new to Linux, every change
made to it (by the AI agent or otherwise) is bucketed into a tier:

1. **Read-only / trivially safe** — act freely, report if asked (log reads,
   status checks, `df`/`free`/`top`, etc).
2. **Routine, reversible** — act, then report (`apt update`, restarting a
   non-critical crashed service, log rotation).
3. **Real changes** — ask first, every time (installing packages, editing
   `/etc`, firewall rules, systemd units, network config).
4. **High-consequence / hard-to-reverse** — requires being physically at the
   machine, not just a remote "yes" (SSH config, disk partitioning, bootloader,
   flushing the firewall). Changes in this tier that could sever remote access
   get applied with an automatic rollback timer (revert scheduled a few minutes
   out, cancelled only once access is confirmed still working).

This model is the single biggest thing that made handing real `sudo` access to
an AI agent feel safe rather than reckless — happy to talk through it if you're
doing something similar.

## Lessons for your own setup

- **A tunnel is exposure.** Don't let "no port forwarding" fool you into
  thinking something is private.
- **Back up before every config change**, even ones you're confident about —
  `file.conf` → `file.conf.bak.<timestamp>`. Cheap insurance.
- **microSD is a wear item.** Plan for SSD/NVMe boot if the Pi 5 will run
  24/7 with any database-backed service (Nextcloud, Uptime Kuma, etc).
- **Stage risky changes with a rollback path**, especially anything touching
  SSH or the firewall — that's the one mistake that can turn a five-minute fix
  into a trip to re-flash an SD card.

## Status

This is a living document — the setup is ongoing. Current open items:

- [ ] Switch to SSH key-only auth (waiting on keypair generation + a session at
      the physical terminal)
- [ ] Decide whether to keep Tailscale SSH enabled alongside normal SSH
- [ ] Set up Telegram alerts for Uptime Kuma via a dedicated bot

---

*Setup managed with the help of an AI ops agent running on the Pi itself.*
