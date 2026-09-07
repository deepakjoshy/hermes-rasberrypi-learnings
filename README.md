# Raspberry Pi Home Server: A Step-by-Step Setup Guide

A hands-on, beginner-friendly guide to turning a Raspberry Pi into a home
server — file storage, self-hosted apps, and secure remote access. Every step
gives you the **actual command to run** and a plain-English explanation of
**what it does and why**, so you're never copy-pasting blind.

> **About this guide**: it grew out of one person's real Raspberry Pi 5 setup
> (documented with help from an AI ops agent) and was generalized so anyone can
> follow it. The specific tools shown (Cloudflare Tunnel, Tailscale, Docker,
> ufw, fail2ban) are the ones actually used and tested on Debian 12 (Bookworm),
> 64-bit — treat them as recommended defaults, not the only options. Where a
> step involves a real choice, alternatives are noted.

## Who this is for

You have (or are getting) a Raspberry Pi and want it to run useful services for
your home — file sync, a media server, a downloads box, dashboards — with sane
security so it doesn't become a liability the moment it touches the internet.
**You do not need to already know Linux.** You do need patience, a willingness
to read error messages, and the good habit of understanding a command before
you run it. Every command below is explained.

A note on convention: commands you run **on the Pi** are shown in code blocks.
Anywhere you see a placeholder in angle brackets like `<pi-ip>` or `<username>`,
replace it (including the brackets) with your real value.

## How to use this guide

The sections are ordered so you can work straight down the page, but they're
not all equally essential. If you're starting from nothing:

| Sections | What it is | Skip it? |
|---|---|---|
| **1–8** | The core build: hardware, OS, login, stable address, SSH keys, firewall, fail2ban | **No.** This is the minimum for a Pi that's safe to leave running. Budget an unhurried evening. |
| **9–11** | Network awareness and remote access | Read **11** before exposing anything. 9 and 10 can wait a week. |
| **12** | Docker — the foundation for everything you'll actually run | **No**, if you plan to host any app at all. |
| **13–18** | Optional services and tuning: file shares, monitoring, media, local AI, storage, memory limits | Pick only what you want. These are independent of each other. |
| **19–24** | Keeping it alive: backups, config versioning, logs, automation | **19 is not optional.** Do it the same week you put real data on the Pi. |
| **25–28** | Checklist, lessons, troubleshooting, further reading | Reference material — come back when something breaks. |

Two habits worth adopting from section 1, not section 19:

- **Before editing any config file, copy it first.** Every rollback in this
  guide depends on that copy existing.
- **After any change to SSH, the firewall, or the network, open a *second*
  connection to confirm you can still get in** — while the first one is still
  open. Locking yourself out of a headless machine is the one mistake here that
  costs you a re-flash instead of a retry.

## Table of contents

1. [What you need](#1-what-you-need)
2. [Assemble the hardware](#2-assemble-the-hardware)
3. [Flash and install the OS](#3-flash-and-install-the-os)
4. [First login and updates](#4-first-login-and-updates)
5. [Give the Pi a stable address](#5-give-the-pi-a-stable-address)
6. [Secure your SSH access](#6-secure-your-ssh-access)
7. [Set up a firewall (ufw)](#7-set-up-a-firewall-ufw)
8. [Block brute-force attacks (fail2ban)](#8-block-brute-force-attacks-fail2ban)
9. [Watch your LAN for new or unknown devices](#9-watch-your-lan-for-new-or-unknown-devices)
10. [Turn on automatic security updates](#10-turn-on-automatic-security-updates)
11. [Reaching your Pi from outside home](#11-reaching-your-pi-from-outside-home)
12. [Running services with Docker](#12-running-services-with-docker)
13. [File sharing on your LAN with Samba](#13-file-sharing-on-your-lan-with-samba)
14. [Uptime monitoring and alerts](#14-uptime-monitoring-and-alerts)
15. [Download clients and media libraries](#15-download-clients-and-media-libraries)
16. [Running a local AI model with Ollama](#16-running-a-local-ai-model-with-ollama)
17. [Choosing a filesystem for attached storage](#17-choosing-a-filesystem-for-attached-storage)
18. [Memory, swap, and container resource limits](#18-memory-swap-and-container-resource-limits)
19. [Backups and maintenance](#19-backups-and-maintenance)
20. [Versioning your configuration with git](#20-versioning-your-configuration-with-git)
21. [Log management](#21-log-management)
22. [Integrating third-party device and cloud APIs](#22-integrating-third-party-device-and-cloud-apis)
23. [Task automation and scheduled jobs](#23-task-automation-and-scheduled-jobs)
24. [A tiered approval model for automation](#24-a-tiered-approval-model-for-automation)
25. [A checklist to verify your setup](#25-a-checklist-to-verify-your-setup)
26. [Lessons learned](#26-lessons-learned)
27. [Troubleshooting](#27-troubleshooting)
28. [Further reading](#28-further-reading)

---

## 1. What you need

A shopping/checklist before you start:

| Item | Notes |
|---|---|
| A Raspberry Pi | Any model works; a **Pi 4 or Pi 5 with 4GB+ RAM** is comfortable for several Docker containers. This guide was built on a Pi 5 (8GB). |
| Power supply | Use the **official supply for your model** (a Pi 5 wants 5V/5A USB-C). Underpowered supplies cause random crashes and SD-card corruption. |
| Boot storage | A **32GB+ microSD card** (A1/A2-rated) to start. For anything database-heavy running 24/7, plan to move to an **SSD/NVMe** later — see [Lessons learned](#26-lessons-learned). |
| A second computer | To flash the OS and to SSH in from. Windows, macOS, or Linux all work. |
| Ethernet cable (recommended) | Wired is more reliable than Wi-Fi for a server that must stay reachable. |
| Your home router's login | You'll need it later to reserve an IP address for the Pi. |

You do **not** need a monitor, keyboard, or mouse for the Pi — this guide sets
it up "headless" (over the network) from your second computer. That said,
nothing stops you from occasionally plugging in a monitor/keyboard/mouse (or
pairing a Bluetooth set) later for a one-off desktop session — a "headless
server most of the time" and "occasional physical workstation" aren't mutually
exclusive, and several official Pi OS images ship a lightweight desktop
environment by default even on a server-focused install.

If you've never used SSH (the tool for logging into another machine over the
network), skim [DigitalOcean's SSH
primer](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)
first. It's the one prerequisite skill.

## 2. Assemble the hardware

1. **Put the Pi in a case** with a fan or heatsink if your model runs warm (the
   Pi 5 does under load). Overheating causes the Pi to slow itself down
   ("thermal throttling").
2. **Insert the microSD card** — but don't power on yet. You'll flash it from
   your second computer in the next step, so leave it out of the Pi (or take it
   back out) for flashing.
3. **Connect ethernet** from the Pi to your router, if you can. Wi-Fi works too
   and is configured during flashing.
4. **Decide where the Pi will physically live.** For rare recovery situations
   (see [tier 4](#24-a-tiered-approval-model-for-automation)), you'll occasionally
   need physical access — so somewhere reachable, not a five-hour round trip.

Do **not** plug in the power supply yet. First boot happens after flashing.

**A note on overclocking.** Raspberry Pi OS lets you push the CPU past its
stock frequency (e.g. via `raspi-config`'s Performance options, or directly by
setting `arm_freq`/`over_voltage` in `/boot/firmware/config.txt`). It's a real
option once everything else is stable and you want more headroom for several
Docker containers — but treat it as a deliberate, monitored choice, not a
default:

- Always pair a frequency bump with adequate cooling (a case fan, not just a
  passive heatsink, once you're pushing past stock) — overclocking raises heat
  output and a Pi that's already running hot has less thermal margin to spare.
- Check for throttling regularly, not just once after applying the change —
  the two `vcgencmd` commands for this are in
  [Troubleshooting](#27-troubleshooting).
- If you inherit or revisit a Pi and don't remember setting an overclock,
  check `/boot/firmware/config.txt` for `arm_freq`/`over_voltage` lines before
  assuming odd instability is a software problem — it's an easy thing to set
  once and then forget about.

## 3. Flash and install the OS

"Flashing" means writing the operating system onto the microSD card.

1. On your second computer, install the [Raspberry Pi
   Imager](https://www.raspberrypi.com/software/) and open it.
2. Click **Choose Device** and pick your Pi model.
3. Click **Choose OS** → **Raspberry Pi OS (other)** → **Raspberry Pi OS Lite
   (64-bit)**. "Lite" means no desktop — you don't need one for a server, and it
   leaves more resources for your services. (Ubuntu Server for ARM is also a
   fine choice if you prefer it.)
4. Click **Choose Storage** and select your microSD card. **Double-check this** —
   flashing erases the target drive completely.
5. Click **Next**, then **Edit Settings** (this is the important part that makes
   a headless setup possible). Set:
   - **Hostname** — e.g. `homeserver`. You'll reach the Pi at `homeserver.local`.
   - **Username and password** — pick a non-obvious username (not `pi` or
     `admin`) and a strong password.
   - **Wi-Fi** — SSID and password, if you're not using ethernet.
   - **Locale / timezone** — your region.
   - On the **Services** tab: **enable SSH**, and choose **"Allow public-key
     authentication only"** if you already have an SSH key (see
     [step 6](#6-secure-your-ssh-access) — you can also do this later).
6. Click **Save**, then **Write**. Wait for it to finish and verify.
7. Put the card into the Pi and **now** connect power. Give it 1–2 minutes to
   boot.

Official reference: [Raspberry Pi OS installation
docs](https://www.raspberrypi.com/documentation/computers/getting-started.html).

## 4. First login and updates

**Find the Pi on your network.** Try its hostname first:

```bash
ping homeserver.local
```

If that resolves, great. If not, log into your router's admin page and look at
the list of connected devices (often called "DHCP clients" or "attached
devices") to find the Pi's IP address, e.g. `192.168.1.42`.

**Log in over SSH** from your second computer (replace with your values):

```bash
ssh <username>@homeserver.local
# or, using the IP:
ssh <username>@192.168.1.42
```

The first time, you'll see a prompt asking to confirm the server's fingerprint —
type `yes`. Then enter the password you set during flashing.

**Update everything.** A fresh image is usually weeks or months old, so apply
all current updates immediately:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

- `sudo` runs a command with administrator rights.
- `apt update` refreshes the list of available packages.
- `apt full-upgrade -y` installs all available updates (`-y` auto-confirms).
- `reboot` restarts the Pi so any kernel/firmware updates take effect.

Wait a minute, then **confirm you can SSH back in** before doing anything else.
Always verify remote access still works after a reboot.

## 5. Give the Pi a stable address

By default your router hands out IP addresses dynamically, so the Pi's address
can change — which breaks bookmarks, SSH shortcuts, and firewall rules. Pin it
down with a **DHCP reservation**:

1. Find the Pi's MAC address (a permanent hardware ID):
   ```bash
   ip link show
   ```
   Look for the `link/ether` line under your active interface (`eth0` for
   ethernet, `wlan0` for Wi-Fi), e.g. `dc:a6:32:xx:xx:xx`.
2. Log into your router, find **DHCP reservations** (sometimes "static leases"
   or "address reservation"), and bind that MAC to a fixed IP such as
   `192.168.1.42`.

A router reservation is preferred over setting a static IP on the Pi itself,
because it keeps all your address decisions in one place and avoids conflicts.
From now on the Pi always answers at the same address.

**One gotcha:** the reservation only takes effect when the Pi next *renews* its
DHCP lease — it keeps its current address until then. Reboot the Pi (or wait out
the lease) to pick up the reserved IP, then confirm it took:

```bash
hostname -I    # should show the address you reserved
```

## 6. Secure your SSH access

Right now the Pi accepts password logins, which are vulnerable to guessing. The
gold standard is **key-based authentication**: your computer holds a private key,
the Pi holds the matching public key, and only that pair can log in.

> **Safety first:** SSH changes are the classic way to accidentally lock
> yourself out. Do the steps in order, and **keep your current SSH session open**
> while you test a new connection in a *second* terminal. Only close the first
> one after the new method is confirmed working.

**Step 1 — Create a key pair on your second computer** (not on the Pi):

```bash
ssh-keygen -t ed25519 -C "your-email-or-note"
```

Press Enter to accept the default location. A passphrase is optional but
recommended. This creates `~/.ssh/id_ed25519` (private — never share) and
`~/.ssh/id_ed25519.pub` (public — safe to copy).

**Step 2 — Copy your public key to the Pi:**

```bash
ssh-copy-id <username>@homeserver.local
```

Enter your Pi password one last time. This appends your public key to
`~/.ssh/authorized_keys` on the Pi.

**Step 3 — Test it.** Open a **new** terminal and run:

```bash
ssh <username>@homeserver.local
```

If it logs you in **without asking for a password**, keys are working.

**Step 4 — Turn off password logins.** On the Pi, edit the SSH server config:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set these lines (remove any leading `#`):

```text
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

Save (`Ctrl+O`, Enter) and exit (`Ctrl+X`). **Check that the file still parses
before you restart anything** — a typo here makes `sshd` refuse to start, and
then no *new* connection can be made at all:

```bash
sudo sshd -t
```

No output (exit code 0) means the config is valid; any problem is printed with
the offending line number. Only once it is clean, reload SSH:

```bash
sudo systemctl restart ssh
```

Restarting the SSH server does **not** drop sessions that are already open —
each connection is handled by its own process — which is precisely why the
"keep your first session open" rule saves you here.

**Step 5 — Verify.** With your original session still open, start yet another
new SSH connection. It should log in via your key and **refuse** any password
fallback. If something's wrong, you still have the working session to fix it.

Reference: [DigitalOcean's SSH key-auth
guide](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server).

## 7. Set up a firewall (ufw)

`ufw` ("Uncomplicated Firewall") controls which incoming connections the Pi
accepts. The safe pattern is **deny everything inbound, then allow only what you
need.**

```bash
sudo apt install ufw -y
```

**Allow SSH _before_ enabling the firewall** — otherwise you'll lock yourself
out the instant it turns on:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp        # SSH — do this FIRST
```

Now enable it:

```bash
sudo ufw enable
sudo ufw status verbose
```

- `default deny incoming` blocks all unsolicited inbound traffic.
- `default allow outgoing` lets the Pi reach the internet normally.
- `allow 22/tcp` permits SSH.

When you add a service later, open just its port — and scope it to your home
network if it doesn't need to be broadly reachable:

```bash
# Example: allow a web app on port 8096 only from your local network
sudo ufw allow from 192.168.1.0/24 to any port 8096
```

You can scope a rule to a **single client**, not just a whole subnet — handy
for a service that only one other device (e.g. a VPN peer) should reach:

```bash
sudo ufw allow from 100.64.0.0/10 to any port 3001   # Tailscale-only access
```

**Important:** ufw only filters **direct inbound** traffic. Tunnels and VPNs
(next section) make **outbound** connections, so they bypass these inbound
rules entirely — the firewall does not protect you from exposure decisions you
make with a tunnel.

**A second, sharper gotcha — Docker bypasses ufw.** When you publish a
container port (`ports: - "8096:8096"` in Compose), Docker writes its own
`iptables` rules *ahead* of ufw's, so the port becomes reachable on all
interfaces **even if `ufw` says that port is denied**. `sudo ufw status` will
happily show the port blocked while it's wide open. Two reliable fixes: (a)
bind the container to loopback or the LAN address only —
`ports: - "127.0.0.1:8096:8096"` (or your LAN IP) — so it's never published on
the public interface in the first place; or (b) manage the exception in
Docker's own `DOCKER-USER` iptables chain rather than in ufw. The loopback/LAN
bind is the simpler habit and is usually what you want for an admin tool.
Reference: [ufw docs](https://help.ubuntu.com/community/UFW).

## 8. Block brute-force attacks (fail2ban)

Even with key-only SSH, bots will hammer your Pi with login attempts. `fail2ban`
watches the logs and temporarily bans IPs that fail repeatedly.

```bash
sudo apt install fail2ban -y
```

Create a local config (never edit the shipped `jail.conf` directly — updates
overwrite it):

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste a sensible starting policy for SSH:

```text
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 1h
```

This bans an IP for 1 hour after 5 failed attempts within 10 minutes. Enable and
start the service, then check it:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

The status output shows currently banned IPs and totals.

**Verify it's actually reading logs.** fail2ban can start cleanly yet ban
nothing if it's watching the wrong log source. On systems that keep
authentication logs only in the systemd journal (rather than in a
`/var/log/auth.log` file), tell the jail to read the journal by adding
`backend = systemd` under `[sshd]` in `jail.local`, then restart it. A quick
end-to-end test is to ban and unban a documentation-only test address and
confirm both take effect:

```bash
sudo fail2ban-client set sshd banip 203.0.113.10
sudo fail2ban-client set sshd unbanip 203.0.113.10
```

Reference: [fail2ban
docs](https://github.com/fail2ban/fail2ban).

## 9. Watch your LAN for new or unknown devices

fail2ban protects the Pi itself, but a home network has other angles: a new
device joining your Wi-Fi (a guest's phone, a smart-home gadget, or something
you didn't add) is worth knowing about, especially once the Pi is the thing
watching the door.

A lightweight approach is a small script that periodically scans the LAN (e.g.
via `arp-scan` or by reading your router's DHCP client list) and diffs it
against a known-devices list, alerting only on new entries:

```bash
sudo apt install arp-scan -y
sudo arp-scan --localnet
```

Run this on a schedule (see [Task automation](#23-task-automation-and-scheduled-jobs))
and keep a simple text or JSON file of MAC addresses you've already seen — a
device isn't "new" twice. Route the alert to wherever you actually check
notifications (see [Uptime monitoring and alerts](#14-uptime-monitoring-and-alerts))
so it doesn't get lost in a log file nobody reads.

This is a detection tool, not a firewall — it tells you something joined, it
doesn't block it. Pair it with your router's own Wi-Fi password/guest-network
settings for actual access control.

## 10. Turn on automatic security updates

`unattended-upgrades` applies security patches on its own, so a machine that's
always on doesn't quietly fall behind.

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

Choose **Yes** when prompted to enable automatic updates. The behaviour lives in
`/etc/apt/apt.conf.d/50unattended-upgrades` (what to upgrade) and
`/etc/apt/apt.conf.d/20auto-upgrades` (how often).

**Recommended tuning:** let security updates apply automatically, but consider
**excluding things you'd rather upgrade deliberately** — Docker Engine, firmware
— so an unattended update can't break a service while nobody's watching. You can
also schedule an automatic reboot for a quiet hour if a patch needs one.

**Confirm it actually works — don't just assume.** A silent auto-updater that
isn't really running is worse than none, because you'll *believe* you're
patched. Do a dry run:

```bash
sudo unattended-upgrade --dry-run --debug
```

It prints which packages *would* be upgraded and which are held back, without
changing anything. After real runs, the history lives in
`/var/log/unattended-upgrades/` — check it occasionally to confirm patches are
landing. Reference: [unattended-upgrades docs](https://wiki.debian.org/UnattendedUpgrades).

## 11. Reaching your Pi from outside home

To use your services when you're away, you need a way in from the internet.
These options aren't mutually exclusive — choose per service based on how public
it should be. **Start with the least exposed option that meets your need.**

### Option A: Port forwarding (not recommended for beginners)

Forwarding a router port straight to the Pi puts the service directly on the
open internet, with no intermediary and no room for error. Most homelabbers now
use a tunnel or VPN instead. If you don't have a specific reason to port-forward,
don't.

### Option B: Private mesh VPN (e.g. Tailscale) — safest default

[Tailscale](https://tailscale.com/kb/1017/install/) builds an encrypted private
network between your devices. Nothing is exposed to the public internet — only
your own logged-in devices can reach the Pi. This is the right choice for
anything you never want public: dashboards, admin panels, download clients.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the printed link to authenticate. Install Tailscale on your phone and
laptop too, and they'll all see the Pi at a stable private address from anywhere.

```bash
sudo tailscale up --ssh
```

Adding `--ssh` also gives you SSH over the tailnet as a **backup access path**,
independent of your normal SSH — invaluable if you ever misconfigure regular SSH.

### Option C: Reverse tunnel (e.g. Cloudflare Tunnel) — for genuinely public services

A tunnel is for services that people *outside your household* must reach (a
public file-share link, say). A small client on the Pi makes an **outbound**
connection to the provider, which proxies public traffic in — so **no router
ports are opened**. [Cloudflare
Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
(free tier; needs a domain on Cloudflare) is common. After installing
`cloudflared` (see their docs for the current package for 64-bit ARM):

```bash
cloudflared tunnel login                              # browser auth; writes cert.pem
cloudflared tunnel create home                        # prints a tunnel UUID + credentials file
cloudflared tunnel route dns home app.example.com     # points the hostname at the tunnel
```

Creating the tunnel is not enough on its own: `cloudflared` also needs an
**ingress config** telling it which hostname maps to which local service.
Without one it connects to Cloudflare and then serves nothing. Write
`/etc/cloudflared/config.yml`:

```yaml
tunnel: <tunnel-uuid-from-create>
credentials-file: /root/.cloudflared/<tunnel-uuid>.json

ingress:
  - hostname: app.example.com
    service: http://localhost:8080
  - service: http_status:404      # required catch-all; must be the last rule
```

Each `ingress` entry sends one public hostname to one local address. The final
catch-all rule is **mandatory** — `cloudflared` refuses to start without it —
and returning 404 for unmatched hostnames is the sane default.

Validate, test in the foreground, then install it as a boot service:

```bash
cloudflared tunnel ingress validate      # checks the rules parse and the catch-all exists
cloudflared tunnel run home              # foreground; Ctrl+C once you have confirmed it works
sudo cloudflared service install         # reads the config above; starts on boot
sudo systemctl status cloudflared
```

`service install` picks up the credentials from the config file you just wrote,
so write the config *before* running it. Adding a hostname later means editing
`ingress:`, running `cloudflared tunnel route dns` for the new name, and
`sudo systemctl restart cloudflared`.

> **Important — a tunnel is still exposure.** Because you didn't open a port, a
> tunnel *feels* private, but the moment you publish a hostname it is just as
> reachable by anyone on the internet as a port-forwarded service. Treat
> publishing each hostname as its own "am I OK exposing this?" decision. Put an
> auth gate (Cloudflare Access or equivalent) in front of anything that
> shouldn't be fully public.

**What "publish it and secure it later" actually costs.** Automated scanners
find new hostnames within hours, not weeks — certificate transparency logs
publish every TLS certificate issued, so a freshly-published name is
discoverable the moment it gets a certificate, without anyone guessing it.
"Nobody knows the URL" has never been a control. In practice this means the
auth gate has to go up **in the same sitting** as the hostname, not on the
weekend when you get around to it.

The cheap version, if you're not ready to configure a full identity provider:
put HTTP basic auth in front of the hostname at the tunnel/proxy layer, so
credentials are demanded before traffic ever reaches the app. It's crude, but
it's the difference between "one factor" and "none", and it takes minutes.
Upgrade it to a real auth gate (Cloudflare Access, Authelia, or your proxy's
equivalent) when you have time.

If you *have* published something unprotected and want to walk it back, remove
the DNS route first (`cloudflared tunnel route dns` created it), then take the
service down — in that order. Removing the container first leaves a published
hostname pointing at a dead service, which tells a scanner the name is real and
worth revisiting.

### Option D: Remote desktop (VNC) for GUI access

SSH covers a terminal, but occasionally you want an actual graphical screen —
e.g. a browser session for a one-time device-pairing flow. A VNC server (e.g.
[wayvnc](https://github.com/any1/wayvnc) on Wayland, or `x11vnc`/TigerVNC on
X11) exposes a remote desktop over the network. Treat it like any other
service: keep it **LAN/VPN-only by default**, and only put it behind a public
tunnel hostname if you specifically need to reach it from outside and
understand that's a fresh exposure decision (see the warning above).

One subtlety: a headless-server VNC setup often serves its **own virtual
display**, not the physical console (`:0`) — so it won't show you anything that
requires the real local display (some one-time device dialogs, for instance).
If you hit that wall, it usually means a real monitor/keyboard session is the
only way through, not a VNC config problem.

### A reasonable default mix

- Services outsiders need (e.g. a public share) → **tunnel**, ideally behind an
  auth gate.
- Admin tools, dashboards, download clients, remote desktop → **VPN-only or
  LAN-only**, never publicly tunneled.
- SSH → key-only, plus a **VPN-based fallback** path.

## 12. Running services with Docker

Docker runs each app in its own isolated container — easy to install, update,
and remove without cluttering the host. Install it with the official script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker <username>
```

The last line lets you run Docker without `sudo`. **Log out and back in** for it
to take effect, then confirm:

```bash
docker run --rm hello-world
```

If it prints a welcome message, Docker works. Modern Docker includes **Compose**
(`docker compose`), which describes a service in a single file.

**A worked example — a status dashboard (Uptime Kuma).** Create a folder and a
`docker-compose.yml`:

```bash
mkdir -p ~/apps/uptime-kuma && cd ~/apps/uptime-kuma
nano docker-compose.yml
```

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - ./data:/app/data
    ports:
      - "<pi-ip>:3001:3001"
    restart: unless-stopped
```

Start it:

```bash
docker compose up -d
```

- `up -d` starts the container in the background.
- The `volumes:` line keeps the app's data in `./data` so it survives updates.
- `restart: unless-stopped` brings it back automatically after a reboot or crash.
- The `ports:` line deliberately binds to the Pi's own LAN address instead of
  the bare `"3001:3001"` most examples show. The bare form publishes on
  **every** interface, and since [Docker inserts its own iptables rules ahead
  of ufw's](#7-set-up-a-firewall-ufw), your firewall will not stop it. Naming
  the host address (or `127.0.0.1`, for something only the Pi itself needs to
  reach) keeps an admin dashboard off any public interface by construction.
  This assumes the address is stable — see [step 5](#5-give-the-pi-a-stable-address).

Visit `http://<pi-ip>:3001` on your home network. To update later:
`docker compose pull && docker compose up -d`. To remove it entirely:
`docker compose down`.

Common starting points, by need:

| Need | Example self-hosted options |
|---|---|
| File sync/storage | Nextcloud, Syncthing |
| Media server | Jellyfin, Plex |
| Download client | qBittorrent, Transmission |
| Uptime/monitoring | Uptime Kuma, Grafana + Prometheus |
| Reverse proxy | Caddy, Nginx Proxy Manager, Traefik |
| Container management UI | Portainer |

[Portainer](https://docs.portainer.io/) gives you a web UI over Docker if you'd
rather not manage containers from the command line. For each new service,
**decide how it should be reachable** (public tunnel, LAN-only, VPN-only) using
[step 11](#11-reaching-your-pi-from-outside-home) — don't default everything to
"public" just because one service is. Reference: [Docker Engine install
docs](https://docs.docker.com/engine/install/).

### A worked example — file sync and storage with Nextcloud

Nextcloud is heavier than Uptime Kuma (it needs a real database and persists a
lot of user data), so it's worth walking through in full — including the
gotchas that trip people up on a first install.

**1. Plan storage first.** Nextcloud will hold every file you sync to it. If
you're running from a microSD card, put Nextcloud's data directory on an
attached SSD/USB drive instead of the card — both for space and because heavy
file writes wear microSD cards out (see [Lessons
learned](#26-lessons-learned)). Decide the path now, e.g. `/mnt/storage/nextcloud`.

**2. Write the Compose file.** Nextcloud needs two containers: the app itself
and a database (MariaDB here — Nextcloud's own docs recommend it over SQLite
for anything beyond a quick test).

```bash
mkdir -p ~/apps/nextcloud && cd ~/apps/nextcloud
nano docker-compose.yml
```

```yaml
services:
  db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=<choose-a-strong-password>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<choose-a-different-strong-password>
    volumes:
      - ./db:/var/lib/mysql

  app:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "<pi-ip>:8080:80"
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<same-password-as-above>
    volumes:
      - ./html:/var/www/html
      - /mnt/storage/nextcloud:/var/www/html/data
```

- `depends_on` makes Docker start the database before the app.
- The `ports:` line binds to the Pi's LAN address for the same reason as the
  Uptime Kuma example above: a bare `"8080:80"` publishes on **every**
  interface and [Docker's iptables rules sit ahead of ufw's](#7-set-up-a-firewall-ufw),
  so the firewall would not stop it. If you later put Nextcloud behind a
  [tunnel](#11-reaching-your-pi-from-outside-home), the tunnel reaches it from
  the Pi itself — you do not need it published more widely to do that.
- `./db` and `./html` keep the database and Nextcloud's own app files next to
  the compose file; `/mnt/storage/nextcloud` (the path you planned in step 1)
  is where actual user files live — mapped in separately so you can put it on
  different, larger storage than the app itself.
- The two `MYSQL_PASSWORD` values and the matching one in `db.environment`
  **must be identical** — a common first-run failure is a typo between them.
- Those passwords are sitting in plain text in this file. Before you put
  compose files under [version control](#20-versioning-your-configuration-with-git),
  move them into a `.env` file beside the compose file (Compose reads it
  automatically, so `MYSQL_PASSWORD=${NEXTCLOUD_DB_PASSWORD}` just works),
  `chmod 600` it, and add it to `.gitignore`. Doing this now is far easier
  than scrubbing a password out of git history later.

**3. Start it and run the setup wizard.**

```bash
docker compose up -d
docker compose logs -f app   # watch startup; Ctrl+C once it settles
```

Visit `http://<pi-ip>:8080` on your home network. The setup wizard asks for an
admin username/password and the database details — use the same
`nextcloud` / `<same-password-as-above>` / database host `db` you set above.

**4. Fix the "Access through untrusted domain" error.** This is the single
most common Nextcloud gotcha: by default it only accepts requests to the
hostname it saw during setup (usually `<pi-ip>:8080`). The moment you reach it
by a different hostname — your tunnel domain, a Tailscale name, `localhost` —
it refuses the request. Add every hostname you'll use to `trusted_domains`:

```bash
docker exec -u www-data nextcloud php occ config:system:set trusted_domains 1 \
  --value="cloud.example.com"
docker exec -u www-data nextcloud php occ config:system:set trusted_domains 2 \
  --value="<pi-ip>"
```

Each command adds one more entry (index `1`, `2`, `3`, ...) — don't reuse
index `0`, that's reserved for the original setup hostname.

**5. Decide how it's reachable, and if public, set `overwriteprotocol`.** If
you're exposing it via a [tunnel](#11-reaching-your-pi-from-outside-home)
under HTTPS, tell Nextcloud it's being accessed over HTTPS even though the
container itself only speaks plain HTTP internally — otherwise it will
generate broken `http://` links and reject some requests:

```bash
docker exec -u www-data nextcloud php occ config:system:set overwriteprotocol \
  --value="https"
```

**6. Turn on Nextcloud's own maintenance jobs.** Nextcloud expects a recurring
background job for housekeeping (cleaning expired shares, updating previews,
etc.). Point cron at it instead of relying on the slower built-in AJAX trigger:

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * docker exec -u www-data nextcloud php cron.php") | crontab -
```

Then in the Nextcloud admin settings (**Settings → Administration →
Basic settings**), switch "Background jobs" to **Cron**.

**7. Updating.** Nextcloud updates itself in-app for minor versions (via the
web UI's update notification), but for major version jumps, pull the new image
and follow Nextcloud's release notes — major upgrades sometimes require
stepping through versions one at a time rather than skipping ahead:

```bash
docker compose pull app
docker compose up -d app
```

**8. Back up before you touch any of this.** Nextcloud's data lives in three
places, and a backup needs all three or it's not a real backup: the database
(dumped with `mysqldump` — see [Backups and
maintenance](#19-backups-and-maintenance) for the safe way to pass the
password), the `./html` config/app folder, and the actual files in
`/mnt/storage/nextcloud`. Restoring only the files without the database gives
you a Nextcloud that has your data on disk but no idea it exists.

**9. Optional: expose a folder you already keep elsewhere as External Storage.**
If you maintain notes or files outside Nextcloud's own data directory (a git
repo, an Obsidian vault, another app's export folder), you can bind-mount that
host path into the container and register it as **External Storage** (Settings
→ Administration → External Storage) instead of duplicating the data. Two
gotchas: the `files_external` app is disabled by default (enable it with
`docker exec -u www-data nextcloud php occ app:enable files_external` before the
settings page will let you add one), and bind mounts preserve the **host**
file's ownership — if the container's `www-data` user (typically uid `33`)
doesn't already have permission on that host path, grant it explicitly rather
than loosening the whole directory:

```bash
sudo setfacl -R -m u:33:rwX /path/to/your/folder
sudo setfacl -R -d -m u:33:rwX /path/to/your/folder   # so new files inherit it too
```

## 13. File sharing on your LAN with Samba

Nextcloud is great for sync-and-share, but sometimes you just want a folder
that shows up as a normal network drive on Windows/macOS/Linux — no app, no
login page, just drag-and-drop. That's what **Samba** (the SMB/CIFS protocol)
is for.

```bash
sudo apt install samba -y
```

Define a share by adding a stanza to `/etc/samba/smb.conf` (back the file up
first):

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak.$(date +%Y%m%d)
sudo nano /etc/samba/smb.conf
```

```ini
[shared]
   path = /mnt/storage/shared
   browseable = yes
   read only = no
   guest ok = no
   valid users = <username>
```

**Validate the config before restarting.** A typo in `smb.conf` can stop Samba
from starting cleanly; `testparm` parses the file and reports errors without
touching the running service:

```bash
testparm -s
```

If it prints your share stanza back without complaint, the syntax is good.
(`-s` skips the "Press enter to see a dump of your service definitions" prompt
that plain `testparm` stops at — which is also what lets you use it inside a
script.) Then
set a Samba password for your user (separate from their Linux login password)
and restart the service:

```bash
sudo smbpasswd -a <username>
sudo systemctl restart smbd
```

A note on ownership: Samba enforces the *Linux* filesystem permissions on
`path` as well as its own `valid users` list, so if writes fail despite a
correct login, check that your user actually owns (or has group write on) the
underlying directory — `ls -ld /mnt/storage/shared` — before suspecting the
Samba config.

On another machine, connect to `\\<pi-ip>\shared` (Windows) or
`smb://<pi-ip>/shared` (macOS/Linux). Keep Samba **LAN-only** — it has no
business being reachable from the public internet, so don't open its ports (139,
445) on a tunnel or router port-forward; ufw's default-deny-inbound from
[step 7](#7-set-up-a-firewall-ufw) already covers this as long as you don't add
an explicit allow rule for it beyond your local subnet.

## 14. Uptime monitoring and alerts

Once you have more than one service, you want to know the moment one goes down
— not whenever you happen to check. [Uptime Kuma](https://github.com/louislam/uptime-kuma)
(deployed in [step 12](#12-running-services-with-docker)) polls your services
on an interval and can notify you the moment one fails a check.

After it's running, add a **Monitor** per service (HTTP(s), TCP port, ping,
etc.) pointing at its LAN address — monitor from inside the network, not
through your public tunnel, so a tunnel hiccup doesn't look like the service
itself is down. Then configure a **Notification** channel under Settings →
Notifications so a failure actually reaches you instead of sitting unread in a
dashboard: Telegram, Discord, email, and dozens of others are built in.

A few practical choices worth making deliberately:

- **Pick one primary alert channel** you actually check (a messaging app you
  already use beats a new dashboard you'll forget to open).
- **Set a sensible check interval and retry count** — too aggressive and a
  single slow response pages you for nothing; too lax and you find out late.
- **Keep the monitoring dashboard itself off the public internet** (VPN/LAN-only)
  — it's an admin tool, and it also tells an attacker exactly what's running.

**The blind spot: who watches the watcher?** Monitoring self-hosted *on the Pi
it's monitoring* cannot tell you when the whole Pi is down — if the box loses
power, drops off the network, or its disk fills, the monitor goes down with
everything else and sends nothing. To catch a total-host failure you need a
heartbeat checked from **outside** the Pi:

- Have the Pi periodically "check in" to an external service — either a free
  external monitor (e.g. [UptimeRobot](https://uptimerobot.com/) hitting your
  public URL, if you expose one) or a **push/heartbeat** service like
  [Healthchecks.io](https://healthchecks.io/) that alerts you when an expected
  ping *fails to arrive*. Uptime Kuma itself supports a "Push" monitor type for
  the inverse pattern (a script on another machine pings Kuma).
- The key inversion: a normal monitor alerts on a *bad response*; a heartbeat
  alerts on **silence**. Silence is exactly what you get when the Pi is dead, so
  a heartbeat is the only kind of check that survives the failure it's meant to
  report.

## 15. Download clients and media libraries

A download client (e.g. [qBittorrent](https://www.qbittorrent.org/)) and a
media server (e.g. [Jellyfin](https://jellyfin.org/)) are two of the most
common reasons people build a home server in the first place. A few points
that aren't obvious the first time:

- **Run it as its own service, not an ad-hoc terminal process**, so it survives
  reboots and crashes. A systemd **user** service is a good fit for something
  that only needs your own account's permissions:
  ```bash
  mkdir -p ~/.config/systemd/user
  nano ~/.config/systemd/user/qbittorrent.service
  ```
  ```ini
  [Unit]
  Description=qBittorrent-nox

  [Service]
  ExecStart=/usr/bin/qbittorrent-nox --webui-port=8081
  Restart=on-failure

  [Install]
  WantedBy=default.target
  ```
  ```bash
  systemctl --user daemon-reload
  systemctl --user enable --now qbittorrent
  ```
  `daemon-reload` is what makes systemd notice a unit file you just created.
  Pick a web-UI port nothing else is already using: `8080` is a very common
  default and collides with the Nextcloud example in
  [step 12](#12-running-services-with-docker), so check first with
  `sudo ss -tulpn | grep ':8081'` and pick another if it answers.
  **The gotcha that catches everyone with user services:** by default a systemd
  *user* service only runs while you're actually logged in, and it stops the
  moment you close your SSH session — and it won't start at boot. To let it run
  unattended (persist after logout and start on boot), enable "linger" for your
  account once:
  ```bash
  sudo loginctl enable-linger <username>
  ```
  Without this, you'll swear the service is enabled yet find it dead every time
  you reconnect. (If you need it to run fully independently of your user, a
  system-level service or a Docker container is the alternative.)
- **Point downloads at a drive with room to grow**, not the boot SD card — see
  [Choosing a filesystem for attached storage](#17-choosing-a-filesystem-for-attached-storage)
  for the tradeoffs of what that drive should be formatted as.
- **Keep the download client's web UI LAN/VPN-only.** It has no reason to be
  publicly reachable, and admin-tool exposure is exactly the kind of thing
  [step 11](#11-reaching-your-pi-from-outside-home) warns against defaulting to
  public.
- **A completion watcher is a natural automation candidate** — a small script
  on a schedule that checks the client's API for newly finished downloads and
  sends a notification, rather than you polling the UI. See [Task automation
  and scheduled jobs](#23-task-automation-and-scheduled-jobs).

**A worked example — a media server (Plex/Jellyfin) reading an existing library.**
Unlike the download client above, a media server is usually happiest with
**host networking** rather than a mapped port — its local-network discovery
protocol (DLNA) and remote-access features work more reliably that way, and it
sidesteps having to hand-map a dozen individual ports:

```bash
mkdir -p ~/apps/plex && cd ~/apps/plex
nano docker-compose.yml
```

```yaml
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=<Your/Timezone>
      - VERSION=docker
    volumes:
      - ./config:/config
      - /mnt/storage/movies:/movies:ro
    restart: unless-stopped
```

- `network_mode: host` shares the Pi's own network stack directly with the
  container — no `ports:` mapping needed, but it also means the container isn't
  isolated from your LAN the way a normally-networked container is. Keep this
  in mind when deciding what else runs alongside it.
- Mount your media **read-only** (`:ro`) if the server only needs to *serve*
  files, not manage them — one less way a container bug could touch your
  library.
- `PUID`/`PGID` matter here more than in many containers: point them at the
  Linux user that already owns your media files, or Plex's process won't be
  able to read them even though the volume mounted successfully.
- Claim the server via the vendor's web setup (`http://<pi-ip>:32400/web`) on
  first run. Note that with `network_mode: host` the container ignores ufw the
  same way a published port does, so "LAN-only" here is a choice you make in
  the media server's own settings, not something the firewall enforces for you.

## 16. Running a local AI model with Ollama

Not everything you run on a home server needs a cloud API. [Ollama](https://ollama.com/)
lets the Pi itself serve small open-weight language models over a local HTTP API —
useful for offline text tasks, experimenting without per-token cost, or feeding
a local automation script without sending data anywhere.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

The installer creates and starts a systemd service (`ollama.service`) for you —
you do not need to enable it separately, but do confirm it came up and is bound
where you expect:

```bash
systemctl is-enabled ollama       # enabled
ss -tulpn | grep 11434            # expect 127.0.0.1:11434, not 0.0.0.0:11434
```

(As with any `curl | sh` installer, read the script first if you'd rather not
run an unreviewed remote script as root.) The service listens on
`localhost:11434` by default — **not** exposed to your LAN or the internet
unless you deliberately change its bind address, which is the right default
for something with no built-in authentication of its own.

Pull a model and try it:

```bash
ollama pull llama3.2:1b
ollama run llama3.2:1b "Explain what a DHCP reservation does, in one sentence."
```

A few practical notes specific to running this on a Pi rather than a desktop
GPU machine:

- **Model size is the real constraint, not just RAM.** A Pi has no dedicated
  GPU, so everything runs on the CPU — pick small models (parameter counts in
  the 1B-3B range, e.g. `llama3.2:1b`, `qwen3:1.7b`) rather than anything
  marketed for desktop/workstation use. Expect noticeably slower responses
  than a cloud model; this is for lightweight local tasks, not a chat
  replacement for a hosted frontier model.
- **Keep it loopback-only unless you have a specific reason not to.** Ollama's
  API has no authentication at all, so a `0.0.0.0` bind hands anyone who can
  reach the port full use of it — LAN or [VPN](#11-reaching-your-pi-from-outside-home)
  only, never a public tunnel.
- **It's just another API endpoint to your automation.** Anything that already
  talks to a cloud LLM API can usually point at `http://localhost:11434` instead
  for tasks that don't need a bigger model — handy for a scheduled job (see
  [Task automation and scheduled jobs](#23-task-automation-and-scheduled-jobs))
  that would rather not spend cloud API quota on a trivial classification or
  formatting task.
- **Free disk space before pulling models** — even "small" models are 1-2GB+
  each, and they add up on a boot microSD the same way any other write-heavy
  data does (see [Choosing a filesystem for attached storage](#17-choosing-a-filesystem-for-attached-storage)),
  or store the Ollama model directory on external storage via the
  `OLLAMA_MODELS` environment variable if space is tight.

## 17. Choosing a filesystem for attached storage

The moment you attach an external SSD/USB drive for backups or bulk storage,
you have to pick a filesystem — and this choice has real consequences that
only show up once you're relying on the drive.

| Filesystem | Unix permissions/ownership | Symlinks & hardlinks | Cross-platform (Windows/macOS) | Good for |
|---|---|---|---|---|
| ext4 | Full support | Full support | Linux only (needs extra drivers elsewhere) | A drive that only ever touches Linux — best default on the Pi itself |
| exFAT | **Not supported** | **Not supported** | Yes, natively | A drive you also plug into Windows/macOS, and you're OK with the tradeoffs below |
| NTFS | Partial (via `ntfs-3g`/`ntfs3`), can be flaky | Partial | Native on Windows, needs a driver elsewhere | Windows-primary use; watch for "dirty bit" mount failures on Linux |

The exFAT and NTFS tradeoffs are easy to miss until a backup script fails
part-way through:

- **exFAT cannot store Unix ownership, permission bits, symlinks, or
  hardlinks at all.** A backup tool like `rsync` that tries to preserve any of
  these (`-a`/`-p`/`-H`) will error out mid-run. If you need exFAT for
  cross-platform reasons, drop those flags (`--no-owner --no-group`, no `-H`)
  and accept that incremental "hardlink to the previous backup" tricks
  (`--link-dest`) silently fall back to full copies — plan disk space
  accordingly.
- **NTFS on Linux can get stuck with a "dirty bit"** if it wasn't unmounted
  cleanly (e.g. unplugged from Windows without ejecting first), and some
  drivers refuse to mount it read-write until that's cleared — a recurring
  annoyance if the drive moves between operating systems often.
- Whichever you choose, **verify what actually gets backed up matches what you
  intended** — a filesystem that silently drops permissions or symlinks is a
  worse surprise during a restore than during a test.

## 18. Memory, swap, and container resource limits

A Pi's RAM is fixed — you can't add more later — so on a box running several
containers, memory is usually the first resource you run out of. Start by
seeing where you stand:

```bash
free -h                                    # total / used / available RAM and swap
ps -eo pid,comm,rss --sort=-rss | head     # the biggest processes, by resident memory
docker stats --no-stream                   # per-container CPU and memory
```

Read the **available** column of `free -h`, not **free**: Linux deliberately
spends unused RAM on disk cache (the `buff/cache` column), which it hands back
instantly when a program needs it. A small "free" number with a healthy
"available" number is normal and not a problem.

**Swap on a Pi is a cushion, not headroom.** Raspberry Pi OS ships a small
swap *file* (200MB by default) managed by `dphys-swapfile`, configured in
`/etc/dphys-swapfile` via `CONF_SWAPSIZE`, not by a swap partition:

```bash
swapon --show
```

It exists to absorb brief spikes. Enlarging it to paper over a genuine RAM
shortage on a microSD boot device is a bad trade: swapping is thousands of
times slower than RAM, and the constant writes wear the card out (see
[Backups and maintenance](#19-backups-and-maintenance)). The better fixes, in
order, are to run fewer or lighter services, cap the greedy one, or move to a
Pi with more RAM. If you do want more swap without the card wear,
**zram** (a compressed block of swap that lives in RAM itself) is the usual
choice — the `zram-tools` package sets it up — though it buys you capacity by
spending CPU, so it helps with mildly-over-committed memory and not with a
service that genuinely needs more than you have.

**The Pi-specific trap: memory limits silently do nothing by default.**
Docker's `--memory` flag and Compose's `mem_limit` rely on the kernel's *memory
cgroup* controller, and stock Raspberry Pi OS boots with that controller
**disabled**. Docker doesn't fail — it accepts the limit and ignores it. Check
whether yours is affected:

```bash
docker info 2>/dev/null | grep -i "WARNING: No memory limit support"
cat /sys/fs/cgroup/cgroup.controllers      # is "memory" in the list?
```

If the warning prints, or `memory` is absent from the controller list, no
memory limit you set is being enforced (a tell-tale symptom: `docker stats`
reports `0B / 0B` for every container). To enable it, append
`cgroup_memory=1 cgroup_enable=memory` to the kernel command line in
`/boot/firmware/cmdline.txt` (`/boot/cmdline.txt` on Debian 11 and older
images) and reboot.

> **Treat this as a high-consequence edit.** `cmdline.txt` must remain a
> **single line** — options are space-separated, and a stray newline can leave
> the Pi unbootable, which is fixed by putting the card in another computer,
> not over SSH. Back the file up first, change nothing else on the line, and
> ideally do it while you have physical access to the Pi. Reference:
> [k3s requirements](https://docs.k3s.io/installation/requirements#operating-systems),
> which documents the same flags for the same reason.

Once the controller is active, cap the services that can misbehave — a media
scanner, a transcoder, an indexer, a local model — so one runaway container
degrades instead of taking the whole box down with it:

```yaml
services:
  some-app:
    image: example/some-app
    mem_limit: 1g
    restart: unless-stopped
```

Two things to understand before you set a number. A container that hits its
limit gets **killed** by the kernel's OOM killer, not politely slowed — so
pick a ceiling above the app's normal peak, and rely on `restart:
unless-stopped` to bring it back. And on Raspberry Pi OS, *swap* limits stay
unsupported even after the above (`WARNING: No swap limit support` remains),
so `memswap_limit` is not something to depend on.

Finally, know where to look after an unexplained crash. The kernel logs every
OOM kill:

```bash
journalctl -k --no-pager | grep -i "out of memory"
```

An empty result means memory exhaustion is not your culprit — check heat and
power instead (see [Troubleshooting](#27-troubleshooting)).

## 19. Backups and maintenance

A home server is only as safe as its backups. Build these habits early:

- **Back up a config file before you edit it** — cheap insurance:
  ```bash
  sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%Y%m%d)
  ```
- **Back up your actual data off the Pi.** Whatever your file-sync or media app
  holds is only as safe as your last copy stored *somewhere else*. A simple
  approach is `rsync` to another machine or external drive:
  ```bash
  rsync -av --delete ~/apps/ /mnt/backup/apps/
  ```
  `-a` preserves permissions/timestamps, `-v` is verbose, `--delete` mirrors
  deletions. Schedule it with `cron` (`crontab -e`) to run nightly. If the
  destination drive is exFAT, see the caveats in [Choosing a filesystem for
  attached storage](#17-choosing-a-filesystem-for-attached-storage) before you
  reach for `-a`.
- **Never copy a live database file — dump it instead.** Grabbing the files
  under a running database's data directory with `cp` or `rsync` can capture a
  half-written, corrupt snapshot that won't restore. Use the database's own dump
  tool while it's running, which produces a consistent copy:
  ```bash
  # MariaDB/MySQL (e.g. the Nextcloud DB container above)
  set -a; . /path/to/backup.env; set +a   # exports DB_ROOT_PASSWORD, chmod 600
  MYSQL_PWD="$DB_ROOT_PASSWORD" docker exec -e MYSQL_PWD nextcloud-db \
    mysqldump -u root nextcloud > nextcloud-db.sql
  ```
  For PostgreSQL the equivalent is `pg_dump`. Back up the dump file, not the raw
  data folder.

  Note the password handling, which is fiddlier than it looks. Writing
  `-p<password>` on the `mysqldump` command line puts the secret into your shell
  history **and** into the process list, where any user running `ps` can read it
  while the dump runs. Using the `MYSQL_PWD` environment variable avoids the
  history problem — but only if you pass it *by name*. Writing
  `docker exec -e MYSQL_PWD="$DB_ROOT_PASSWORD" ...` expands the value into the
  `docker exec` arguments, so it shows up in `ps` output in full, exactly like
  `-p` would. The bare `-e MYSQL_PWD` form above tells Docker to copy the
  variable in from the surrounding environment instead, so the value never
  appears in any command line. Keep the value itself in a `chmod 600` env file
  outside version control (see
  [step 20](#20-versioning-your-configuration-with-git)). Also **check the dump
  is non-empty before you trust it** — a failed dump still creates a 0-byte file and a
  backup script that doesn't check will happily archive nothing:
  ```bash
  [ -s nextcloud-db.sql ] || { echo "DB dump is empty - aborting"; exit 1; }
  ```
- **Test a restore at least once — an untested backup is not a backup.** The
  only way to know your backup works is to rebuild from it: copy the dump and
  data to a scratch location (or a spare SD card / second Pi), restore, and
  confirm the service comes up with your data intact. Do this *before* you need
  it, not during an emergency.
- **Aim for the [3-2-1 rule](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/):**
  3 copies, on 2 kinds of media, with 1 kept offsite.
- **Watch for storage wear** if running from microSD. Check for disk errors and
  keep an eye on space:
  ```bash
  dmesg | grep -i error
  df -h
  ```
  Don't be surprised if an SD card needs replacing after a year or two of heavy
  24/7 writes — this is why an SSD is worth it for busy setups.
- **Keep a running TODO list** of unfinished items. A home server is rarely
  "done" in one sitting.

## 20. Versioning your configuration with git

Data backups (previous section) protect your *files*. It's also worth keeping
a **version-controlled snapshot of your configuration** — compose files,
systemd units, `/etc/fstab`, firewall rules, package lists — separately, in a
private git repository. The value isn't redundancy with your backup drive;
it's **history and diffs**: you can see exactly what changed, when, and revert
a single bad edit without touching anything else.

A few rules that matter more here than in a typical code repo:

- **Never commit secrets.** Passwords, API tokens, and private keys don't
  belong in git history — once committed, they're there forever even if you
  delete the file later. Keep secrets in files outside the repo (or in a
  `.gitignore`d directory) and reference them by path in the tracked configs.
- **Scan every diff before committing**, ideally automatically — a simple
  grep for common secret patterns (`password=`, `api_key`, PEM headers, etc.)
  run against the changeset, blocking the commit if it trips. Cheap insurance
  against an accidental paste.
- **Keep the repo private**, and if it's pushed to a host like GitHub,
  authenticate non-interactively (a credential helper or deploy key) so an
  automated job can commit and push without a human typing a password.
- This pairs naturally with [scheduled automation](#23-task-automation-and-scheduled-jobs)
  — a nightly job that snapshots changed config, scans it, and commits only if
  there's a real, clean delta.

## 21. Log management

Every service on the Pi writes logs somewhere, and left unmanaged they'll
eventually fill your disk. `logrotate` is the standard Linux tool for keeping
log files bounded — rotating (renaming/compressing) them on a schedule or size
trigger and deleting old ones.

Most distro packages already ship a logrotate config for their own logs, but
anything writing to a custom path (an app under `~/.hermes/logs/`, a script's
own log file, etc.) needs its own rule:

```bash
sudo nano /etc/logrotate.d/my-app
```

```text
/home/<username>/apps/my-app/logs/*.log {
    size 20M
    rotate 7
    compress
    copytruncate
}
```

- `size 20M` rotates once a log file passes 20MB (instead of, or in addition
  to, a time-based trigger like `daily`).
- `rotate 7` keeps 7 old rotations before deleting the oldest.
- `compress` gzips rotated files to save space.
- `copytruncate` copies the log then truncates the original in place — needed
  for a program that keeps a log file open continuously and won't reopen a
  freshly-renamed one on its own.

No new timer is usually needed — most systems already run `logrotate` daily via
a system timer or cron entry; a new config just needs to exist under
`/etc/logrotate.d/` to be picked up on the next run. Test it without waiting:

```bash
sudo logrotate -f /etc/logrotate.d/my-app
```

**Don't forget the systemd journal — it's separate from `logrotate`.** Most
service logs (anything shown by `journalctl`) are managed by `systemd-journald`,
not by logrotate, so a logrotate rule won't touch them. Check how much space
the journal is using and cap it so it can't grow without bound:

```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=200M    # trim now to a 200MB ceiling
sudo journalctl --vacuum-time=2weeks  # or drop anything older than 2 weeks
```

For a permanent cap, set `SystemMaxUse=200M` in
`/etc/systemd/journald.conf` and restart with
`sudo systemctl restart systemd-journald`.

**And don't forget Docker — its container logs are a third, separate pile.**
With the default `json-file` logging driver, everything a container prints to
stdout/stderr is appended to a file under `/var/lib/docker/containers/<id>/`
that **grows without limit** — logrotate doesn't know about it and journald
doesn't own it. A chatty container can quietly eat gigabytes. Check what
yours are using:

```bash
sudo du -ch /var/lib/docker/containers/*/*-json.log | tail -1
```

Cap it globally by creating `/etc/docker/daemon.json` (the file does not exist
by default — create it if it's missing, and back it up first if it isn't):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

Then `sudo systemctl restart docker` to apply. Note two gotchas: restarting the
Docker daemon restarts your containers, so do it at a quiet moment; and the new
limit applies to **newly created** containers only — existing ones keep their
old settings until they're recreated (`docker compose up -d --force-recreate`).
Per-container overrides go in the compose file under `logging:` if one service
needs different retention. Reference: [json-file driver
options](https://docs.docker.com/engine/logging/drivers/json-file/).

**Check whether your logs actually survive a reboot.** The default
`Storage=auto` writes the journal to disk only if `/var/log/journal/` already
exists; if it doesn't, logs live in RAM under `/run/log/journal/` and are wiped
on every reboot. Which one you get depends on the image, so check rather than
assume:

```bash
ls -d /var/log/journal >/dev/null 2>&1 && echo persistent || echo "volatile (RAM only)"
```

If it reports volatile and you want logs to survive a crash-reboot — the case
where they matter most — set `Storage=persistent` in the same file and restart
journald; it will create `/var/log/journal/` for you. Mind the extra SD-card
writes if you're microSD-based, and keep the `SystemMaxUse=` cap above in place
either way.

## 22. Integrating third-party device and cloud APIs

Home servers aren't limited to software you install — a lot of useful
automation comes from **polling an existing device's cloud API** and acting on
the result: a solar inverter, a smart thermostat, a weather station, a robot
vacuum. The pattern is the same regardless of the specific device:

1. **Find the vendor's official API** (a cloud OpenAPI is far more stable than
   scraping a phone app's private endpoints, and less likely to break on an
   app update). Register for API credentials if required.
2. **Poll on an interval that matches how often the data actually changes** —
   there's no benefit to hitting a slow-changing sensor every 10 seconds, and
   an aggressive poll rate can hit a rate limit or even get your credentials
   throttled.
3. **Store just enough state to detect a *change*, not just a value** — e.g.
   "was this already below threshold last run?" — so you alert once when a
   condition starts, not on every single poll while it persists.
4. **Alert through the same channel you already use** (see [Uptime monitoring
   and alerts](#14-uptime-monitoring-and-alerts)) rather than inventing a new
   place to check.
5. **Handle the vendor's regional/datacenter quirks explicitly.** Some cloud
   APIs route by account region (e.g. an India-registered account only working
   against an India-specific endpoint) — a call that mysteriously 404s or
   401s against the "obvious" global endpoint is often exactly this, not a
   credentials problem.

**A related pattern if you run an AI agent for home automation:** don't assume
a bundled capability does everything its name implies. A search backend that
finds pages but cannot fetch their contents typically fails with a generic
error rather than "not supported", so it goes unnoticed until you read the
logs. The fix is the same third-party-API pattern as above — swap in a
purpose-built API and then confirm the previously-failing calls actually
succeed, rather than assuming a config change was the fix.

Run the poller as its own [scheduled job](#23-task-automation-and-scheduled-jobs),
keep its credentials in a permissions-locked env file (not committed to git —
see [step 20](#20-versioning-your-configuration-with-git)), and document any
hard limits or gotchas you discover for the specific API next to the script,
so the next debugging session doesn't start from zero.

## 23. Task automation and scheduled jobs

Most of what keeps a home server healthy without your daily attention is
**scheduled, unattended jobs**: nightly backups, a weekly summary, a poller
checking a device API, a log rotation. Linux gives you two built-in
mechanisms:

- **`cron`** — the classic choice, simplest for "run this script at this
  time/interval." Edit your own crontab with `crontab -e`; each line is
  `<minute> <hour> <day> <month> <weekday> <command>`.
- **systemd timers** — more modern, integrate with `systemctl status`/`journalctl`
  for easier debugging, and can express things like "run 5 minutes after boot"
  that plain cron can't. More setup (a `.service` + a `.timer` unit) for the
  same result.

Either way, a few habits keep scheduled jobs from becoming their own source of
surprises:

- **Make failures loud, successes quiet.** A nightly job that silently fails
  for a month is worse than one that pages you once. Route errors to your
  alert channel; let clean runs produce no notification (or a single quiet
  weekly summary) rather than a message every single night.
- **Add pre-flight checks** before anything destructive or resource-heavy —
  e.g. a backup script should confirm its target drive is actually mounted and
  has free space *before* it starts, not discover that half-way through.
  Abort loudly rather than proceeding on a bad assumption.
- **Log what ran and what changed**, even briefly — when a job runs unattended
  for months, "what actually happened last Tuesday" needs to be answerable
  without guessing.
- **Give any autonomous job (script, or an AI agent driving one) a bounded
  scope and an explicit approval model** for anything beyond its routine
  purpose — see the next section. It's worth deciding this up front rather
  than case-by-case, especially if the job can install software, touch
  networking, or push to a public repo unattended.

## 24. A tiered approval model for automation

If anything other than you personally changes this box — a cron job, a script,
or an AI agent — decide up front **how much autonomy it gets**, rather than
case-by-case under pressure. A scheme that works well in practice:

1. **Read-only / trivially safe** — act freely (log reads, status checks).
2. **Routine, reversible** — act, then report (`apt update`, restarting a
   crashed non-critical service, log rotation).
3. **Real changes** — ask first, every time (installing packages, editing
   `/etc`, firewall rules, systemd units, network config).
4. **High-consequence / hard-to-reverse** — treat as requiring physical presence
   at the machine, not just a remote "yes" (SSH config, disk partitioning,
   bootloader, flushing the firewall). For changes here that could sever remote
   access, use an **automatic rollback timer**: apply the change, schedule a
   revert a few minutes out, and only cancel the revert once you've confirmed
   access still works.

## 25. A checklist to verify your setup

Every section above told you to *do* something. This one tells you how to
**prove it worked** — because the failure mode of a home server is silent: the
backup that never ran, the firewall rule that Docker quietly bypassed, the
auto-updater that stopped a month ago. Run this list after your initial build,
then again every few months.

Each check is a command whose output you can judge on the spot.

**Access and identity**

```bash
hostname -I                       # matches the IP you reserved (step 5)?
sudo ss -tulpn                    # every listening port, and what owns it
```

Look for anything listening on `0.0.0.0` that you did not intend to publish —
that's the single most useful line in this checklist.

**Security**

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
sudo ufw status verbose
sudo fail2ban-client status sshd
```

Expect `passwordauthentication no` and `permitrootlogin no` (step 6), a default
deny policy (step 7), and a jail that shows a non-zero *total* failed count —
zero totals after weeks of uptime usually means fail2ban is reading the wrong
log source, not that nobody tried (step 8).

**Patching**

```bash
systemctl list-timers 'apt-daily*' --all
ls -lt /var/log/unattended-upgrades/ | head
```

You should see **two** timers — `apt-daily.timer` (refreshes the package list)
and `apt-daily-upgrade.timer` (installs the updates) — each with a plausible
`NEXT` and a recent `LAST`. Note the `.timer` suffix and the glob: querying the
`.service` name instead returns "0 timers listed" and looks alarmingly like
nothing is scheduled, when in fact it's the timer unit that holds the schedule.
A genuinely stale `LAST` means you have been unpatched since whenever it
stopped (step 10).

**Storage and data**

```bash
df -h                             # nothing near 100%
findmnt --target /mnt/storage     # external drive actually mounted, not an empty dir
ls -lt /path/to/your/backups | head
```

The mount check matters more than it looks: if an external drive fails to
mount, the mount point still exists as an empty directory on the boot card — so
a backup script writes happily into it, filling your SD card while appearing to
succeed (step 19).

**Health**

```bash
vcgencmd get_throttled            # 0x0 = never throttled since boot
systemctl --failed                # should list zero units
docker ps --format '{{.Names}}\t{{.Status}}'
free -h                           # read the "available" column, not "free"
```

Any container showing `Restarting` is crash-looping, not running — and if one
keeps dying without an obvious log reason, check for an OOM kill (step 18).

**The two checks nothing on this list can do for you**

- **Restore from a backup.** Only an actual restore proves a backup is real.
  Everything above only proves a *file exists*.
- **Confirm your alerts arrive.** Deliberately trip one — stop a monitored
  container, or use your notification channel's test button — and check the
  message reaches the device you actually look at. An alerting path is
  untested until a message has travelled it end to end.

## 26. Lessons learned

- **A tunnel is still exposure.** "No port forwarding" doesn't mean private —
  treat every published hostname as a fresh exposure decision.
- **Back up before every config change**, even trivial-seeming ones.
- **microSD is a wear item.** Move to SSD/NVMe boot if you run anything
  write-heavy 24/7.
- **exFAT and NTFS aren't Linux-native filesystems** — know what they can't
  store (ownership, permissions, symlinks, hardlinks) *before* you point a
  backup tool at one, not after a run fails partway through.
- **Stage risky changes with a rollback path**, especially anything touching SSH
  or the firewall — that's the class of mistake that turns a five-minute fix
  into re-flashing a card.
- **Test remote access after every reboot or SSH/firewall change**, from a new
  connection, before you close your working session.
- **Decide your automation/agent approval model before you need it**, not after
  something's gone wrong.
- **Alerts are only useful if they reach a channel you actually check** — a
  dashboard nobody opens is not monitoring.

## 27. Troubleshooting

- **SSH: "REMOTE HOST IDENTIFICATION HAS CHANGED!"** — appears after you
  re-flash or reinstall the Pi while keeping the same IP/hostname. The Pi has a
  new host key and your computer still remembers the old one. It's not a
  compromise. Remove the stale entry and reconnect:
  ```bash
  ssh-keygen -R <pi-ip-or-hostname>
  ssh <username>@<pi-ip-or-hostname>
  ```
- **Locked out after an SSH config change** — this is why you kept a session
  open. Fix the setting in that session, or use your [Tailscale SSH
  fallback](#11-reaching-your-pi-from-outside-home). Last resort: power off, put
  the card in another computer, and fix the file directly.
- **`ping homeserver.local` doesn't resolve** — mDNS may be off on your network.
  Use the Pi's IP address from your router's device list instead.
- **A container won't start** — check its logs:
  ```bash
  docker compose logs -f
  ```
- **An external drive intermittently fails to auto-mount after an unclean
  disconnect** — common with NTFS-formatted drives that get unplugged without
  "safely eject" first, leaving a "dirty" filesystem flag Linux won't
  auto-mount read-write. A small boot-time check-and-repair script (using
  `ntfsfix -n` to detect the dirty state, then `ntfsfix` to clear it before
  retrying the mount) turns this from a manual fix into something that
  self-heals on every boot — see [Task automation and scheduled
  jobs](#23-task-automation-and-scheduled-jobs) for wiring a script to run at
  boot via systemd. **Remember to retire or rewrite this kind of script if you
  later reformat the drive to a filesystem without a dirty-bit concept** (e.g.
  exFAT — see [Choosing a filesystem for attached
  storage](#17-choosing-a-filesystem-for-attached-storage)); it'll harmlessly
  no-op forever rather than fail loudly, which is easy to forget about.
- **Pi feels slow or reboots randomly** — suspect power or heat first. Check the
  temperature and for under-voltage warnings:
  ```bash
  vcgencmd measure_temp
  vcgencmd get_throttled   # 0x0 means no throttling has occurred
  ```
- **A backup/rsync job errors out partway through on an external drive** —
  suspect the filesystem before the script. See [Choosing a filesystem for
  attached storage](#17-choosing-a-filesystem-for-attached-storage): exFAT in
  particular rejects ownership, permission, and symlink/hardlink operations
  outright.
- **fail2ban is running but never bans anyone** — it may be watching the wrong
  log source; see the journal-backend note in [step 8](#8-block-brute-force-attacks-fail2ban).

## 28. Further reading

- [Raspberry Pi official documentation](https://www.raspberrypi.com/documentation/)
- [Tailscale docs](https://tailscale.com/kb/)
- [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Docker Engine install (Linux)](https://docs.docker.com/engine/install/)
- [Docker Compose docs](https://docs.docker.com/compose/)
- [DigitalOcean: SSH key-based authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
- [fail2ban](https://github.com/fail2ban/fail2ban)
- [ufw (Uncomplicated Firewall)](https://help.ubuntu.com/community/UFW)
- [unattended-upgrades (Debian wiki)](https://wiki.debian.org/UnattendedUpgrades)
- [Samba documentation](https://www.samba.org/samba/docs/)
- [Uptime Kuma](https://github.com/louislam/uptime-kuma)
- [Nextcloud administration manual](https://docs.nextcloud.com/server/latest/admin_manual/)
- [Jellyfin documentation](https://jellyfin.org/docs/)
- [Ollama](https://ollama.com/)
- [logrotate(8) manual](https://linux.die.net/man/8/logrotate)
- [journald.conf(5) — journal size and persistence](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)
- [r/homelab](https://www.reddit.com/r/homelab/) and r/selfhosted for community
  setups and troubleshooting

---

*This guide was generalized from a real Raspberry Pi 5 setup, written up with
the help of an AI ops agent. If you're setting up something similar and want to
compare notes on any of the choices above, feel free to open an issue.*
