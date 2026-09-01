# Raspberry Pi Home Server: A Setup Guide

A step-by-step guide to turning a Raspberry Pi into a home server — file
storage, self-hosted services, and secure remote access — written for
newcomers to Linux and home networking.

> **About this guide**: it grew out of one person's actual Pi 5 setup
> (documented with help from an AI agent). Rather than a log of exactly what
> they did, it's generalized into a guide anyone can follow. Specific tools
> (Cloudflare Tunnel, Tailscale, ufw, etc.) are the ones that were actually
> used and tested — call them recommended defaults, not the only options.
> Where a step involves a real choice, alternatives are noted.

## Who this is for

You have (or are getting) a Raspberry Pi and want it to run useful services
for your home network — file sync, a media server, monitoring dashboards,
torrent downloads, whatever — with sane security so it doesn't become a
liability the moment it touches the internet. You don't need to already know
Linux; you do need patience and a willingness to look things up.

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Hardware setup](#2-hardware-setup)
3. [OS install](#3-os-install)
4. [Initial access and updates](#4-initial-access-and-updates)
5. [Networking and remote access](#5-networking-and-remote-access)
6. [Security hardening](#6-security-hardening)
7. [Running services](#7-running-services)
8. [Backups and maintenance](#8-backups-and-maintenance)
9. [Lessons learned](#9-lessons-learned)
10. [Further reading](#10-further-reading)

---

## 1. Prerequisites

- **A Raspberry Pi.** Any model with enough RAM for what you plan to run
  works; 4GB+ is comfortable if you want to run multiple Docker containers.
  The setup this guide is based on used a **Raspberry Pi 5, 8GB RAM**.
- **Storage.** A microSD card to start is fine and is what most official
  guides assume. If you plan to run anything database-backed (Nextcloud,
  monitoring stacks, etc.) 24/7, consider an SSD/NVMe boot drive instead —
  see [Lessons learned](#9-lessons-learned).
- **A computer to flash the OS from**, and the [Raspberry Pi
  Imager](https://www.raspberrypi.com/software/) installed on it.
- **Basic comfort with a terminal.** You'll be using SSH throughout. If
  you've never used SSH before, [DigitalOcean's SSH
  primer](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)
  is a good starting point.
- **A router you can access** (for checking your local network, and later
  for firewall/exposure decisions).

## 2. Hardware setup

1. Insert the microSD card (or connect the SSD, if your Pi model supports
   USB or NVMe boot) into your flashing computer.
2. Physically set up the Pi: case, power supply rated for your model,
   ethernet cable if you can use one (more reliable than Wi-Fi for a
   server that needs to stay reachable).
3. Note down where the Pi will physically live — for high-risk recovery
   scenarios (see [tier 4](#6-security-hardening) below) you'll want
   physical access to be realistic, not a five-hour round trip.

If you're running anything 24/7 with real write activity (databases,
frequent file syncs), plan for **microSD wear**. SD cards degrade with
sustained writes; watch for corruption over months of uptime, and consider
migrating to SSD/NVMe boot once you know which services you're keeping.

## 3. OS install

1. Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to write
   an OS image to your boot media. **Raspberry Pi OS (64-bit, Lite if you
   don't need a desktop)** is the default choice and what this guide
   assumes, but Ubuntu Server and other ARM distros are also supported on
   most Pi models.
2. In the Imager's advanced options (gear icon / Ctrl+Shift+X), set:
   - hostname
   - a non-default username and password
   - Wi-Fi credentials (if not using ethernet)
   - **SSH enabled**, ideally with your public key pre-loaded rather than a
     password — see [Security hardening](#6-security-hardening) for why.
3. Boot the Pi and find its IP address (check your router's DHCP client
   list, or use `ping <hostname>.local` if mDNS is working).
4. SSH in: `ssh <username>@<pi-ip-or-hostname>`.

Official reference: [Raspberry Pi OS installation
docs](https://www.raspberrypi.com/documentation/computers/getting-started.html).

## 4. Initial access and updates

Once you're in over SSH:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

Confirm you can SSH back in after the reboot before doing anything else.
This is the point to also decide **how you'll manage the box long-term** —
by hand, with config management (Ansible, etc.), or with the help of an
automation/AI agent. If an agent or script is going to have real access to
this machine, decide your approval/change-tiering policy now (see
[Security hardening](#6-security-hardening)) rather than after the first
mistake.

## 5. Networking and remote access

You have several options for reaching services on your Pi from outside your
home network. None of them are mutually exclusive — pick based on how public
you want each service to be.

### Option A: Port forwarding (not recommended for beginners)

Forwarding router ports directly to the Pi is the traditional approach but
puts your Pi's exposed service directly on the open internet with no
intermediary — mistakes here are unforgiving. Most homelabbers now prefer
one of the tunnel/VPN options below instead.

### Option B: Reverse tunnel (e.g. Cloudflare Tunnel)

A tunnel client running on the Pi makes an **outbound** connection to a
provider, which then proxies public traffic to your Pi — no inbound router
ports need to be opened at all. [Cloudflare
Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
(free tier available, requires a domain on Cloudflare) is a common choice.

**Important**: an outbound tunnel *feels* private because you didn't open
any ports, but once a hostname is published, it is just as reachable by
anyone on the internet as a port-forwarded service would be. Treat
publishing each hostname as its own "am I okay exposing this?" decision —
don't treat it as safe by default just because the mechanism feels
different. Cloudflare Access (or an equivalent auth gate) can restrict a
tunnel hostname to authenticated users if you don't want it fully public.

### Option C: Private mesh VPN (e.g. Tailscale)

[Tailscale](https://tailscale.com/kb/1017/install/) (and similar
WireGuard-based mesh VPNs like Headscale or ZeroTier) let your devices reach
the Pi directly over an encrypted private network, with nothing exposed to
the public internet at all. This is the right choice for services you never
want publicly reachable — dashboards, torrent clients, admin panels — while
still being reachable from your phone away from home.

`tailscale up --ssh` additionally gives you SSH over the tailnet as a second
access path, independent of your regular SSH setup — useful as a fallback if
your normal SSH access ever breaks.

### A reasonable default mix

A workable pattern many homelabs converge on:
- Public-facing services people outside your household need (e.g. a file
  sync app) → tunnel, optionally behind an auth gate.
- Admin tools, dashboards, torrent clients → **LAN-only or VPN-only**, never
  publicly tunneled.
- SSH → key-only auth, plus a VPN-based fallback path.

## 6. Security hardening

Baseline hardening steps worth doing on any internet-adjacent Pi:

- **SSH key-only authentication.** Disable password auth entirely once your
  key is confirmed working (`PasswordAuthentication no` in
  `/etc/ssh/sshd_config`). This is a high-risk change — get a second access
  path (VPN SSH, physical console) confirmed working *before* you turn off
  password auth, in case something's misconfigured.
  [DigitalOcean's key-auth
  guide](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server).
- **fail2ban** — bans IPs after repeated failed login attempts. A sensible
  starting point for the sshd jail is 5 failed attempts within 10 minutes →
  a 10 minute ban, tuned up from there. [fail2ban
  docs](https://github.com/fail2ban/fail2ban).
- **A firewall (ufw)** — default-deny inbound, then explicitly allow only
  what you need (SSH, and whichever service ports apply), scoped to your
  local subnet where a service doesn't need to be reachable more broadly.
  Note that tunnel traffic (outbound-initiated) bypasses ufw's inbound
  rules — the firewall doesn't protect you from tunnel/VPN exposure
  decisions, only from direct inbound traffic. [ufw
  docs](https://help.ubuntu.com/community/UFW).
- **Automatic security updates (unattended-upgrades)** — apply security
  patches automatically, scheduled for a low-traffic window, with an
  auto-reboot if required. Consider excluding anything you want to upgrade
  deliberately and under supervision (Docker Engine, firmware) from
  auto-upgrade — a botched unattended firmware update with nobody watching
  is a bad time. [unattended-upgrades
  docs](https://wiki.debian.org/UnattendedUpgrades).

### A tiered approval model (if a script or agent has access)

If anything other than you personally is making changes to the box — a
cron job, an automation script, or an AI agent — it's worth deciding up
front how much autonomy it gets, rather than deciding case-by-case under
pressure. A tiering scheme that works well in practice:

1. **Read-only / trivially safe** — act freely (log reads, status checks).
2. **Routine, reversible** — act, then report (`apt update`, restarting a
   crashed non-critical service, log rotation).
3. **Real changes** — ask first, every time (installing packages, editing
   `/etc`, firewall rules, systemd units, network config).
4. **High-consequence / hard-to-reverse** — treat as requiring physical
   presence at the machine, not just a remote "yes" (SSH config, disk
   partitioning, bootloader, flushing the firewall). For changes in this
   tier that could sever remote access, consider an automatic rollback
   timer: apply the change, schedule a revert a few minutes out, and only
   cancel the revert once access is confirmed still working.

## 7. Running services

Docker is the easiest way to run most self-hosted services with minimal
host pollution — one container per service, easy to update or remove.
[Docker Engine install
docs](https://docs.docker.com/engine/install/) (use the 64-bit ARM
instructions for a Pi).
[Portainer](https://docs.portainer.io/) gives you a web UI over Docker if
you'd rather not manage everything by hand on the CLI.

Common starting points, by need:

| Need | Example self-hosted options |
|---|---|
| File sync/storage | Nextcloud, Syncthing |
| Media server | Jellyfin, Plex |
| Download client | qBittorrent, Transmission |
| Uptime/monitoring | Uptime Kuma, Grafana + Prometheus |
| Reverse proxy | Caddy, Nginx Proxy Manager, Traefik |
| Container management | Portainer |

For anything you want auto-restarting on failure without full Docker
orchestration, a systemd unit with `Restart=on-failure` is a lightweight
option for a single service you run directly on the host rather than in a
container.

Decide per-service how it should be reachable (public tunnel, LAN-only, VPN
only) using the framework in
[Networking and remote access](#5-networking-and-remote-access) — don't
default every new service to "public" just because one already is.

## 8. Backups and maintenance

- **Back up config before every change**, even ones you're confident about
  — a simple `cp file.conf file.conf.bak.$(date +%s)` before editing is
  cheap insurance against a bad edit.
- **Back up actual data**, not just configs — whatever your file
  sync/storage service holds is only as safe as your last backup off the
  Pi itself. The [3-2-1 backup
  rule](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/) (3
  copies, 2 different media, 1 offsite) is a reasonable target.
- **Watch for storage wear** if running from microSD — keep an eye on disk
  errors in `dmesg`/`journalctl`, and don't be surprised if a card needs
  replacing after a year or two of 24/7 database writes.
- **Keep a running list of open/incomplete items** as you go (a TODO section
  in your own notes, a project board, whatever works) — a home server setup
  is rarely "done" in one sitting.

## 9. Lessons learned

- **A tunnel is still exposure.** "No port forwarding" doesn't mean
  private — treat every published hostname as a fresh exposure decision.
- **Back up before every config change**, even trivial-seeming ones.
- **microSD is a wear item.** Plan for SSD/NVMe boot if you're running
  anything write-heavy 24/7.
- **Stage risky changes with a rollback path**, especially anything
  touching SSH or the firewall — that's the class of mistake that turns a
  five-minute fix into re-flashing an SD card.
- **Decide your automation/agent approval model before you need it**, not
  after something's already gone wrong.

## 10. Further reading

- [Raspberry Pi official documentation](https://www.raspberrypi.com/documentation/)
- [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Tailscale docs](https://tailscale.com/kb/)
- [Docker Engine install (Linux)](https://docs.docker.com/engine/install/)
- [DigitalOcean: SSH key-based authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
- [fail2ban](https://github.com/fail2ban/fail2ban)
- [ufw (Uncomplicated Firewall)](https://help.ubuntu.com/community/UFW)
- [unattended-upgrades (Debian wiki)](https://wiki.debian.org/UnattendedUpgrades)
- [r/homelab](https://www.reddit.com/r/homelab/) and r/selfhosted for
  community setups and troubleshooting

---

*This guide was generalized from a real Raspberry Pi 5 setup, written up
with the help of an AI ops agent. If you're setting up something similar
and want to compare notes on any of the choices above, feel free to open an
issue.*
