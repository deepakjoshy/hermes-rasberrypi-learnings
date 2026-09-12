# Raspberry Pi Home Server: A Step-by-Step Setup Guide

A hands-on, beginner-friendly guide to turning a Raspberry Pi into a home
server — file storage, self-hosted apps, and secure remote access. Every step
gives you the **actual command to run** and a plain-English explanation of
**what it does and why**, so you're never copy-pasting blind.

> **About this guide**: it grew out of one person's real Raspberry Pi 5 setup
> and was generalized so anyone can follow it. The specific tools shown
> (Cloudflare Tunnel, Tailscale, Docker, ufw, fail2ban) are the ones actually
> used and tested on Debian 12 (Bookworm), 64-bit — treat them as recommended
> defaults, not the only options. Where a step involves a real choice,
> alternatives are noted.
>
> That server is administered day to day by an **AI ops agent**
> ([Hermes](https://github.com/NousResearch/hermes-agent)) running on the Pi
> itself — it maintains this document, and most of the gotchas below were found
> by it while fixing something real. That setup is the subject of
> [step 25](#25-running-an-ai-ops-agent-on-the-server). It is genuinely optional,
> though: sections 1–8 are the server, and everything here works whether or not
> you ever run an agent.

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
| **13–19** | Optional services and tuning: file shares, monitoring, media, local AI, storage, memory limits, reverse proxy | Pick only what you want. These are independent of each other. |
| **20–26** | Keeping it alive: backups, config versioning, logs, automation, AI ops agent | **20 is not optional.** Do it the same week you put real data on the Pi. |
| **27–30** | Checklist, lessons, troubleshooting, further reading | Reference material — come back when something breaks. |

Two habits worth adopting from section 1, not section 20:

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
19. [Putting several services behind one reverse proxy](#19-putting-several-services-behind-one-reverse-proxy)
20. [Backups and maintenance](#20-backups-and-maintenance)
21. [Versioning your configuration with git](#21-versioning-your-configuration-with-git)
22. [Log management](#22-log-management)
23. [Integrating third-party device and cloud APIs](#23-integrating-third-party-device-and-cloud-apis)
24. [Task automation and scheduled jobs](#24-task-automation-and-scheduled-jobs)
25. [Running an AI ops agent on the server](#25-running-an-ai-ops-agent-on-the-server)
26. [A tiered approval model for automation](#26-a-tiered-approval-model-for-automation)
27. [A checklist to verify your setup](#27-a-checklist-to-verify-your-setup)
28. [Lessons learned](#28-lessons-learned)
29. [Troubleshooting](#29-troubleshooting)
30. [Further reading](#30-further-reading)

---

## 1. What you need

A shopping/checklist before you start:

| Item | Notes |
|---|---|
| A Raspberry Pi | Any model works; a **Pi 4 or Pi 5 with 4GB+ RAM** is comfortable for several Docker containers. This guide was built on a Pi 5 (8GB). |
| Power supply | Use the **official supply for your model** (a Pi 5 wants 5V/5A USB-C). Underpowered supplies cause random crashes and SD-card corruption. |
| Boot storage | A **32GB+ microSD card** (A1/A2-rated) to start. For anything database-heavy running 24/7, plan to move to an **SSD/NVMe** later — see [Lessons learned](#28-lessons-learned). |
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
   (see [tier 4](#26-a-tiered-approval-model-for-automation)), you'll occasionally
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
  [Troubleshooting](#29-troubleshooting).
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
ip -brief -4 addr show eth0    # or wlan0 on Wi-Fi
```

Use the interface form rather than `hostname -I`. `hostname -I` prints **every**
address the machine holds, on every interface, space-separated — once you add
Docker (step 12) or a VPN (step 11) that is a line of six or eight addresses
including `172.17.0.1` and a `100.x` tailnet address, and picking your LAN one
out of it is guesswork. Naming the interface asks the question you actually
meant. It exits non-zero and says `Device "eth0" does not exist` if you name the
wrong one, which is also more useful than silence.

## 6. Secure your SSH access

Right now the Pi accepts password logins, which are vulnerable to guessing. The
gold standard is **key-based authentication**: your computer holds a private key,
the Pi holds the matching public key, and only that pair can log in.

> **Safety first:** SSH changes are the classic way to accidentally lock
> yourself out. Do the steps in order, and **keep your current SSH session open**
> while you test a new connection in a *second* terminal. Only close the first
> one after the new method is confirmed working. If the Pi is somewhere
> inconvenient to reach physically, add a
> [rollback timer](#the-rollback-timer-concretely) as well.

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
the offending line number.

**Check that nothing is overriding you.** Near the top of the Debian/Raspberry
Pi OS `sshd_config` is a line `Include /etc/ssh/sshd_config.d/*.conf`, and sshd
uses the **first** value it obtains for any keyword — so a drop-in file included
there wins over the same setting written further down in the main file. Imager's
advanced options, cloud-init, and some hardening scripts all drop files in that
directory. Rather than reading both files and reasoning about order, ask sshd
what it concluded:

```bash
ls /etc/ssh/sshd_config.d/
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|permitrootlogin'
```

`sshd -T` prints the *effective* configuration after all includes are resolved.
If it disagrees with what you just typed, the answer is in that directory. (See
[step 27](#27-a-checklist-to-verify-your-setup) for two ways this command can
still mislead you.)

Only once it is clean, reload SSH:

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
Note the precise scope: Docker creates these rules for **bridge** networks
only. A container using `network_mode: host` (or ipvlan/macvlan) gets no Docker
firewall rules at all and is filtered by ufw exactly like any other process on
the host — see the media-server example in
[step 15](#15-download-clients-and-media-libraries).
References: [ufw docs](https://help.ubuntu.com/community/UFW),
[Docker packet filtering](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

**And the mirror image — containers are not on your LAN.** When a container
reaches a service *outside* itself (on the host, or on another machine), the
connection arrives from that container's **Docker bridge subnet** — e.g.
`172.19.0.0/16` — never from the host's LAN address. A rule scoped only to your
LAN will therefore silently drop it. This bites hardest with a containerized
monitor (see [step 14](#14-uptime-monitoring-and-alerts)) checking a
bare-metal service: the check fails forever, the dashboard shows a confident
false DOWN, and the app logs show nothing at all, because the packet died in
the firewall before it arrived. Find the subnet and add a matching rule
alongside the LAN one:

```bash
docker network ls
docker network inspect <network> --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
sudo ufw allow from 172.19.0.0/16 to any port 8081 comment 'app-from-monitor'
```

To confirm the firewall is what's dropping something, read the kernel's own
rejection log — ufw logs there, and `/var/log/ufw.log` only exists on distros
whose rsyslog config creates it (Raspberry Pi OS often has no such file):

```bash
sudo journalctl -k | grep 'UFW BLOCK' | tail
```

The `SRC=` field in a blocked line tells you exactly which address to allow.

**The third gotcha — your LAN-scoped rules are probably IPv4-only.** Most home
ISPs now hand out a *routable* IPv6 prefix, and there is **no NAT in IPv6** — so
where an IPv4 service is incidentally shielded by your router doing address
translation, the same service on IPv6 has your firewall and nothing else in
front of it. Check whether that applies to you:

```bash
ip -6 addr show scope global    # a 2xxx:/3xxx: address here means globally routable
ping6 -c2 2606:4700:4700::1111  # and working v6 internet
```

If you have one, note how `ufw` treats these two rules differently:

```bash
sudo ufw allow 22/tcp                              # both IPv4 and IPv6
sudo ufw allow from 192.168.1.0/24 to any port 445 # IPv4 ONLY — the CIDR is v4
```

Any rule scoped to an **IPv4** address or subnet can only ever match IPv4
traffic. There is no v6 counterpart unless you write one with an IPv6 range.
The trap is that `sudo ufw status` renders both kinds identically, with no
column telling you which protocol a rule covers, so a screenful of confident
`ALLOW IN 192.168.1.0/24` lines can be leaving IPv6 completely unaddressed.
Read the generated rule files instead, which cannot hide it:

```bash
sudo grep '^-A ufw-user-input'  /etc/ufw/user.rules    # your IPv4 rules
sudo grep '^-A ufw6-user-input' /etc/ufw/user6.rules   # your IPv6 rules
```

On a machine where every rule was written with a LAN CIDR, the second command
returns **nothing at all**.

Two possible readings of that, and they point opposite ways:

- **If your default policy is `deny (incoming)`, you are fine** — unmatched
  IPv6 hits the default DROP, so the services are closed rather than open. This
  is the common case and the reason it so rarely bites.
- **But nothing you allowed on the LAN works over IPv6 either**, which is the
  more likely thing to actually confuse you: a share or dashboard that works
  from one device and not another, where the difference turns out to be that
  the working client resolved an IPv4 address and the failing one preferred
  IPv6.

Also confirm what your services are even bound to — `[::]` means "all IPv6
addresses", the routable one included. Match on the **local address column
only**, not on the whole line:

```bash
sudo ss -tulpnH | awk '$5 ~ /^\[::\]:/ {i=index($0,"users:"); print $1, $5, (i?substr($0,i):"-")}'
```

The obvious `sudo ss -tulpn | grep '\[::\]'` looks equivalent and is not: a
listening socket prints `[::]:*` in its *peer* address column, so the grep also
matches sockets bound to `[::1]` (loopback) or a link-local address and hands
you a list roughly twice as long as the real one. (Verified on a Pi running
Docker and Samba: 16 matching lines, of which only 7 were genuine wildcard
binds.) `-H` drops the header so the column numbers are stable, `$5` is the
local address, and everything from `users:` onward is the owning process — taken
whole rather than as `$7`, because a process name with a space in it would
otherwise be cut off mid-name (see
[step 27](#27-a-checklist-to-verify-your-setup)).

If you want a rule to cover both protocols, either drop the address scope
(`sudo ufw allow 445/tcp` — but then it is open to the whole internet, so only
for something genuinely public), or add an explicit second rule for your IPv6
prefix. If you would rather not reason about two protocols at all, the honest
alternative is to disable IPv6 deliberately rather than by accident — but
decide it, don't drift into it.

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
sudo fail2ban-client status sshd | grep 'Banned IP list'   # should list it
sudo fail2ban-client set sshd unbanip 203.0.113.10
sudo fail2ban-client status sshd | grep 'Banned IP list'   # should be empty again
```

Both `set` commands print `1` (the number of addresses affected); the two
`status` checks are what actually prove the ban was applied and then removed.

That only proves fail2ban can *act*. To prove it is **reading** the right
source, ask it directly rather than waiting for real traffic to show up:

```bash
sudo fail2ban-client get sshd logpath
sudo fail2ban-client status sshd | grep 'Journal matches'
```

On a jail using the systemd backend, the first prints
`No file is currently monitored` — which looks alarming but is correct, since
there is no file — and the second prints the journal filter it is actually
applying, e.g. `Journal matches: _SYSTEMD_UNIT=sshd.service + _COMM=sshd`.
On a file backend it is the other way round: `logpath` lists the file being
tailed and the `grep` returns nothing. The genuine failure to look for is
`logpath` naming a file that **does not exist** on your system — classically
`/var/log/auth.log` on an image with no rsyslog installed. That jail starts
cleanly, reports healthy, and reads nothing at all.

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
sudo arp-scan --interface=eth0 --localnet    # name your LAN interface explicitly
```

**Name the interface.** `--localnet` derives the range to scan from an
interface's own address and netmask, and if you don't say which interface,
`arp-scan` picks the lowest-numbered configured, up, non-loopback one. On a Pi
running Docker or a VPN that is very often `docker0` (`172.17.0.0/16`) or a
tunnel interface rather than your LAN — so the scan succeeds, reports a handful
of containers, and never looks at your network at all. Check what you have with
`ip -brief addr` and pass the right name (`eth0` wired, `wlan0` Wi-Fi).

Run this on a schedule (see [Task automation](#24-task-automation-and-scheduled-jobs))
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
sudo unattended-upgrade --dry-run -v
```

It prints which packages *would* be upgraded, without changing anything — look
for the `Packages that will be upgraded:` line, and for `Allowed origins are:`
just above it, which tells you which repositories it is willing to touch at all.
(Many examples use `--debug` instead. That works, but it buries the same answer
under ~170 lines of apt pinning detail; `-v` is the readable form, and `--debug`
is for when `-v` says nothing and you need to know why.) After real runs, the history lives in
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
credentials-file: /home/<username>/.cloudflared/<tunnel-uuid>.json

ingress:
  - hostname: app.example.com
    service: http://localhost:8080
  - service: http_status:404      # required catch-all; must be the last rule
```

Each `ingress` entry sends one public hostname to one local address. The final
catch-all rule is **mandatory** — `cloudflared` refuses to start without it —
and returning 404 for unmatched hostnames is the sane default.

**Use the credentials path that `create` actually printed**, rather than the
one in any tutorial (including this one). `cloudflared tunnel create` writes the
credentials file *next to the origin certificate* that `cloudflared tunnel
login` produced, and prints the path it chose (`Tunnel credentials written to
...`). Run `login` as your normal user and that is
`~/.cloudflared/<uuid>.json`; run it under `sudo` and it is
`/root/.cloudflared/<uuid>.json`. Pasting the `/root/...` form when your file is
really in your home directory gives you a tunnel that starts and then fails to
authenticate. Whichever path you use must be readable by **root**, since the
service installed below runs as root.

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

**Publishing only a slice of an app.** Sometimes you want one narrow path
public — a status page at `/status/mypage`, say — while the same app's
dashboard, login and API stay private. You don't need a reverse proxy for this:
an ingress rule can match on `path:` (a Go regular expression) as well as
hostname. Put the narrow rule *above* a hostname-specific 404, so anything that
doesn't match falls through and is refused:

```yaml
ingress:
  - hostname: status.example.com
    path: ^/(status/|assets/|api/status-page/)
    service: http://localhost:3001
  - hostname: status.example.com
    service: http_status:404      # everything else on this hostname
  - service: http_status:404      # global catch-all; must be last
```

Rules are evaluated **top to bottom, first match wins**, so ordering is the
whole mechanism. Note that a status page usually needs its static assets and a
data endpoint too, not just the page URL — check your app's network requests and
widen the regex until the page renders, no further. Test the logic before
restarting anything; `cloudflared` will tell you which rule a URL hits:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://status.example.com/status/mypage   # should hit the app
cloudflared tunnel ingress rule https://status.example.com/dashboard       # should hit the 404
```

Confirm from a machine *off your network* (or with `curl -I`) that the private
paths really return 404 before you consider it done.

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
| Reverse proxy | Caddy, Nginx Proxy Manager, Traefik (see [step 19](#19-putting-several-services-behind-one-reverse-proxy)) |
| Container management UI | Portainer |
| Workflow automation | n8n (see worked example below) |

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
learned](#28-lessons-learned)). Decide the path now, e.g. `/mnt/storage/nextcloud`.

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
  Uptime Kuma example above — a bare `"8080:80"` would bypass ufw. If you later
  put Nextcloud behind a [tunnel](#11-reaching-your-pi-from-outside-home), the
  tunnel reaches it from the Pi itself, so it still does not need publishing
  any wider.
- `./db` and `./html` keep the database and Nextcloud's own app files next to
  the compose file; `/mnt/storage/nextcloud` (the path you planned in step 1)
  is where actual user files live — mapped in separately so you can put it on
  different, larger storage than the app itself.
- The two `MYSQL_PASSWORD` values and the matching one in `db.environment`
  **must be identical** — a common first-run failure is a typo between them.
- Those passwords are sitting in plain text in this file. Before you put
  compose files under [version control](#21-versioning-your-configuration-with-git),
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
maintenance](#20-backups-and-maintenance) for the safe way to pass the
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

### A worked example — workflow automation with n8n

[n8n](https://n8n.io/) is a self-hosted workflow automation tool — think
"connect service A to service B on a trigger" without writing a full app. It's
a reasonable next step once Docker feels comfortable, but it deserves a
different mental model than Uptime Kuma or Nextcloud: **a workflow you build
in it can call out to anything and store credentials for anything it talks
to.** Treat its exposure decision and its login as seriously as you'd treat
SSH, not as casually as a dashboard.

**1. Write the Compose file.**

```bash
mkdir -p ~/apps/n8n && cd ~/apps/n8n
nano docker-compose.yml
```

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "<pi-ip>:5678:5678"
    environment:
      - N8N_HOST=<pi-ip>
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_WEBHOOK_URL=http://<pi-ip>:5678/
      - GENERIC_TIMEZONE=Asia/Kolkata
      - TZ=Asia/Kolkata
    volumes:
      - ./data:/home/node/.n8n
```

- Binding `ports:` to `<pi-ip>` instead of a bare `"5678:5678"` keeps this off
  every interface by default, same reasoning as the Uptime Kuma example above.
- `N8N_HOST`/`N8N_WEBHOOK_URL` tell n8n what address to put in the webhook
  URLs it generates — set these to whatever hostname you'll actually reach it
  by, or webhooks it hands you will point at the wrong place.
- The data volume holds n8n's SQLite database: all your workflows, execution
  history, and encrypted credentials. Back it up like you would any other
  stateful app (see [step 20](#20-backups-and-maintenance)).

Start it, then open `http://<pi-ip>:5678` and create the owner account on
first load — don't skip this or leave the instance in its unauthenticated
setup state if it's reachable from anywhere but your own machine.

**2. Reaching other host services from inside the container.** A common first
workflow calls something running directly on the Pi — a local LLM via Ollama,
for example. The usual Docker advice of using `host.docker.internal` **does
not resolve by default on Linux** (it's a Docker Desktop convenience for
Mac/Windows). Two ways to fix it:

- Add `extra_hosts: ["host.docker.internal:host-gateway"]` under the service
  in the compose file, then use `host.docker.internal` in n8n's credentials —
  the portable fix, survives the container's IP changing.
- Or find the bridge network's gateway IP directly (`docker network inspect
  <network-name> | grep Gateway`, commonly `172.x.x.1`) and use that.

Either way, if the host service is firewalled with ufw, container traffic
arrives from the Docker bridge subnet, **not** your LAN CIDR — it needs its
own `ufw allow` rule for that subnet, or the container's requests will hang
looking like a firewall problem rather than a config one.

**3. If you expose it beyond your LAN, raise the bar.** n8n's own login is
the only thing standing between the internet and a tool that can execute
arbitrary workflow code and holds every credential you've given it. If you
put it behind a [tunnel](#11-reaching-your-pi-from-outside-home), strongly
consider an additional gate in front of it (e.g. Cloudflare Access) rather
than relying on the app password alone — and if you decide against that
extra gate, that's a real risk being accepted, not a formality to skip past.

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
that plain `testparm` stops at, which is also what lets you use it in a script.)
Then set a Samba password for your user — separate from their Linux login
password — and have `smbd` pick up the new share:

```bash
sudo smbpasswd -a <username>
sudo systemctl reload smbd
```

Prefer `reload` over `restart` here: it signals `smbd` to re-read `smb.conf`
without tearing down existing connections, so nobody's in-progress file copy
dies because you added a share. (`smbpasswd` needs no restart at all — the
password database is read live.)

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
itself is down. If a monitor stays stubbornly DOWN while the service is
plainly fine from your laptop, suspect the firewall before the app — a
containerized monitor reaches LAN services from its Docker bridge subnet, not
the host's LAN address (see [step 7](#7-set-up-a-firewall-ufw)). Then configure
a **Notification** channel under Settings →
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
everything else and sends nothing. Catching that needs a check that runs
somewhere else entirely — so think of monitoring as **two layers**, and expect
to run both:

| Layer | Runs | Sees | Blind to |
|---|---|---|---|
| Internal (e.g. Uptime Kuma) | On the Pi | Individual services failing; per-service detail; LAN-only endpoints | The Pi itself dying |
| External (hosted or push) | Off the Pi | Total host failure — power cut, network drop, crash | Anything not reachable from outside |

For the external layer, either:

- A **free hosted monitor** polling a URL you already expose publicly (e.g.
  [HetrixTools](https://hetrixtools.com/), [UptimeRobot](https://uptimerobot.com/)).
  Check which alert channels the free tier actually includes before committing —
  they differ, and a monitor that can only email you is a monitor you'll miss.
- A **push/heartbeat** service like [Healthchecks.io](https://healthchecks.io/),
  where a cron job on the Pi pings a URL on a schedule and the service alerts you
  when a ping *fails to arrive*. This is the better option if you expose nothing
  publicly. Uptime Kuma supports the same "Push" monitor type.

The key inversion: a normal monitor alerts on a *bad response*; a heartbeat
alerts on **silence**. Silence is exactly what a dead Pi produces, so a
heartbeat is the only check that survives the failure it exists to report.

### A check that cannot fail is not a check

Monitoring, backup verification, and health scripts share a failure mode that is
worse than having no check at all: a check that reports **pass** unconditionally,
because it is structurally incapable of reporting anything else. It runs on
schedule, it never complains, and it earns trust it hasn't got. Three ways this
happens in practice:

- **The check measures the wrong thing.** A "failures since yesterday" counter
  that tests a *log file's* modification time instead of the dates on the lines
  inside it will match the whole file forever once anything is ever written to
  it — so it reports either "everything failed" or "nothing failed" permanently,
  regardless of the contents. Any check whose input is a proxy for the real
  signal deserves a second look at what it is actually reading.
- **The failure branch is unreachable.** Counting matches with `grep -c` looks
  harmless, but `grep` exits **1** when it finds nothing — so under `set -e` the
  script dies before it can report zero, and in `n=$(grep -c ... || true)` style
  code the "no problems" and "grep broke" cases become indistinguishable. Test
  the empty case explicitly:

  ```bash
  n=$(grep -c 'ERROR' /var/log/myapp.log || true)   # 0 and exit 1 on no match
  [ -n "$n" ] || { echo "check itself failed"; exit 1; }
  ```

- **The success signal was never produced.** If a job's "did it work" evidence is
  an artefact it creates itself — a `latest` pointer, a marker file, a status
  line — then a run where creating that artefact *silently failed* looks
  identical to a run that succeeded before the artefact existed. Have the job
  check its own output exists and is non-empty before declaring success.

There is also the check that stops running entirely. A cron job that is never
triggered emits no errors, so nothing looks wrong; the only way to notice is a
**staleness assertion** — fail if the newest artefact is older than the interval
you expect:

```bash
# alert if the most recent backup snapshot is more than 36 hours old
find /mnt/backup -maxdepth 1 -name 'snapshot-*' -mmin -2160 | grep -q . \
  || echo "WARNING: no backup snapshot in the last 36h"
```

The single habit that catches all of these: **break the thing on purpose and
confirm the alarm fires.** Stop a monitored container, feed the parser a log line
you know is an error, rename the backup directory. A check you have never seen
fail is an assumption, not a check — the same reasoning as the untested-restore
and untested-alert-path items in
[the verification checklist](#27-a-checklist-to-verify-your-setup).

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
  **Change the web UI password before anything else.** qBittorrent up to and
  including 4.5.x ships a *hardcoded* default login (`admin` / `adminadmin`) —
  it is in the published source, so it is not a secret from anyone. Newer
  releases (4.6.0+) dropped it in favour of printing a random one-off password
  to the log on first start, which you then have to read and replace:
  ```bash
  journalctl --user -u qbittorrent | grep -i "temporary password"
  ```
  Either way, set your own under **Tools → Options → Web UI** on first login.
  Combined with the LAN-only rule below, that is the difference between a
  download client and an open remote-code-execution endpoint.
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
  and scheduled jobs](#24-task-automation-and-scheduled-jobs).

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
  first run. Host networking is the one case where the [Docker/ufw
  bypass](#7-set-up-a-firewall-ufw) does **not** apply: Docker writes firewall
  rules only for *bridge* networks, so a host-networked container listens like
  an ordinary host process and your ufw rules do govern it. That cuts both
  ways — nothing is published for you either, so add the allow rules yourself:
  ```bash
  sudo ufw allow from 192.168.1.0/24 to any port 32400 proto tcp comment 'Plex-LAN'
  ```
  Reference: [Docker — packet filtering and
  firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

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
ss -tulpn | grep 11434            # expect 127.0.0.1:11434 and nothing else
```

Read that second line carefully rather than skimming it for the string
`0.0.0.0`. `ss` renders a wildcard bind as **`*:11434`**, not as
`0.0.0.0:11434` — so a service listening on every interface looks nothing like
the address you were told to watch out for, and a quick glance can read as a
pass. (Verified on Debian 12 with a deliberately wildcard-bound Ollama: the
output line is `tcp LISTEN 0 4096 *:11434 *:*`.) Anything other than a literal
`127.0.0.1` here means the API is reachable beyond the Pi itself.

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
  [Task automation and scheduled jobs](#24-task-automation-and-scheduled-jobs))
  that would rather not spend cloud API quota on a trivial classification or
  formatting task.
- **Free disk space before pulling models** — even "small" models are 1-2GB+
  each, and they add up on a boot microSD the same way any other write-heavy
  data does (see [Choosing a filesystem for attached storage](#17-choosing-a-filesystem-for-attached-storage)),
  or store the Ollama model directory on external storage if space is tight.
  Note that `OLLAMA_MODELS` is read by the **server**, not by your shell:
  exporting it in your terminal changes nothing, because the daemon installed
  above runs under its own `ollama` user from a systemd unit with its own
  environment. Set it with a drop-in and make sure that user can write the new
  path:
  ```bash
  sudo systemctl edit ollama        # add: [Service] then Environment="OLLAMA_MODELS=/mnt/storage/ollama"
  sudo install -d -o ollama -g ollama /mnt/storage/ollama
  sudo systemctl restart ollama
  systemctl show ollama -p Environment    # confirm the variable is actually there
  ```
  Skipping the ownership step gives you a service that starts and then fails to
  pull anything.

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

**Swap on a Pi is a cushion, not headroom.** Raspberry Pi OS swaps to a
*file* managed by `dphys-swapfile`, configured in `/etc/dphys-swapfile`, not to
a swap partition. Don't assume a size — read the one you actually have:

```bash
swapon --show                              # actual size in use right now
grep -E '^CONF_(SWAPSIZE|MAXSWAP)' /etc/dphys-swapfile
```

The number varies by image and by how the file was sized, so figures quoted in
older guides are unreliable. What is fixed is the *logic*: `CONF_SWAPSIZE` sets
an absolute size in MB, and if it is left **empty** the size is computed as
`CONF_SWAPFACTOR` (default 2) times your RAM — then clamped by `CONF_MAXSWAP`
(default 2048 MB) and by `CONF_MAXDISK_PCT` (default 50% of the free space on
the filesystem holding the file). So on an 8GB Pi the "computed" answer is not
16GB; it is whichever of those two ceilings bites first. Changes take effect
via `sudo dphys-swapfile swapoff && sudo dphys-swapfile setup && sudo
dphys-swapfile swapon`, not by editing the file alone.

It exists to absorb brief spikes. Enlarging it to paper over a genuine RAM
shortage on a microSD boot device is a bad trade: swapping is thousands of
times slower than RAM, and the constant writes wear the card out (see
[Backups and maintenance](#20-backups-and-maintenance)). The better fixes, in
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
docker info 2>&1 | grep -i "no memory limit support"
cat /sys/fs/cgroup/cgroup.controllers      # is "memory" in the list?
```

Note the `2>&1`: `docker info` prints its warnings to **stderr**, so the
`2>/dev/null` that many examples use throws away the exact line you are looking
for and makes an affected Pi look healthy.

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
journalctl _TRANSPORT=kernel --no-pager | grep -i "out of memory"
```

**Don't reach for `journalctl -k` here**, which is the obvious spelling and the
wrong one: `-k` implies `-b`, so it shows kernel messages from the *current boot
only*. An OOM kill bad enough to have rebooted the Pi — exactly the crash you
are investigating — happened on the previous boot and is therefore invisible,
and the command returns nothing at all while looking like a clean result.
`_TRANSPORT=kernel` selects the same kernel messages without the implicit boot
filter, so it searches the whole retained journal (which needs a persistent
journal — see [step 22](#22-log-management)). Add `-b -1` if you specifically
want the boot before this one.

An empty result from *that* command means memory exhaustion is not your culprit
— check heat and power instead (see [Troubleshooting](#29-troubleshooting)).

## 19. Putting several services behind one reverse proxy

Once you're running three or four apps, you're juggling
`http://192.168.1.42:8080`, `:8096`, `:3001` — port numbers nobody remembers,
no HTTPS, and a new firewall decision for every app. A **reverse proxy** is a
single service that listens on ports 80/443, and forwards each incoming
hostname to the right app behind it. You get memorable names
(`jellyfin.home.example.com`), one place to terminate TLS, and one place to
decide what's reachable.

Common choices: [Caddy](https://caddyserver.com/docs/) (smallest config,
automatic HTTPS), [Traefik](https://doc.traefik.io/traefik/) (configured from
container labels), and [Nginx Proxy Manager](https://nginxproxymanager.com/)
(web UI, if you'd rather not edit config files). Caddy is the easiest to read,
so it's the example here.

**A minimal Caddy setup.** Put the proxy on a shared Docker network so it can
reach the other containers by name:

```bash
docker network create proxy      # do this once; add other apps to it too
mkdir -p ~/apps/caddy && cd ~/apps/caddy
```

`docker-compose.yml`:

```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data:/data          # keeps issued certificates across restarts
    networks: [proxy]
networks:
  proxy:
    external: true
```

`Caddyfile` — one block per app, using the container's name and *internal*
port:

```text
jellyfin.home.example.com {
    reverse_proxy jellyfin:8096
}

uptime.home.example.com {
    reverse_proxy uptime-kuma:3001
}
```

Then `docker compose up -d`. To put an app behind the proxy, each app's compose
file needs **both** halves — the service joins the network, *and* the network is
declared as external so Compose attaches to the existing one instead of trying
to create its own:

```yaml
services:
  jellyfin:
    # ...
    networks: [proxy]
networks:
  proxy:
    external: true
```

Adding only `networks: [proxy]` to the service fails immediately with
`service "jellyfin" refers to undefined network proxy: invalid compose project`.
Once both are present, Caddy can resolve the app by its container name.

Points that are easy to get wrong:

- **Create the `Caddyfile` before the first `docker compose up`.** A bind mount
  whose host path does not exist yet is created by Docker as an **empty
  directory**, not a file — so Caddy starts, finds a directory where its config
  should be, and serves nothing. If you see that, `docker compose down`, remove
  the stray directory, write the real file, and start again. This applies to
  every single-file bind mount, not just Caddy's.
- **The proxy only helps if the apps stop publishing their own ports.** Once
  Caddy can reach a container over the shared network, remove that container's
  `ports:` mapping — otherwise the app is still directly reachable on its old
  port and bypasses everything you just set up (including, as
  [step 12](#12-running-services-with-docker) warns, your firewall rules).
- **Names have to resolve to the Pi.** A hostname like
  `jellyfin.home.example.com` means nothing until something answers for it —
  either a wildcard/`A` record in a domain you control pointing at the Pi's LAN
  IP, or a local DNS server (a Pi-hole or your router, if it supports custom
  entries) mapping the names internally. Editing each client's `hosts` file
  works but doesn't scale past a couple of devices.
- **Automatic HTTPS needs a public challenge or a DNS challenge.** Caddy gets
  free certificates automatically, but the default HTTP challenge requires the
  hostname to be publicly reachable on port 80 — which a LAN-only service is
  not. For internal names, either use the **DNS-01 challenge** with a domain
  you own (which needs a Caddy build that includes your DNS provider's plugin —
  the stock image does not ship them), or let Caddy issue its own internal CA
  certificates and install that CA on your devices. Both are more work than
  the one-line config above suggests; budget for it.
- **This is not a substitute for [step 11](#11-reaching-your-pi-from-outside-home).**
  A reverse proxy on your LAN doesn't make anything reachable from outside, and
  putting one on the public internet is a separate, deliberate exposure
  decision — with its own attack surface, since it now speaks to every app you
  put behind it.

## 20. Backups and maintenance

A home server is only as safe as its backups. Build these habits early:

- **Back up a config file before you edit it** — cheap insurance:
  ```bash
  sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%Y%m%d)
  ```
- **Back up your actual data off the Pi.** Whatever your file-sync or media app
  holds is only as safe as your last copy stored *somewhere else*. A simple
  approach is `rsync` to another machine or external drive:
  ```bash
  mountpoint -q /mnt/backup || { echo "backup drive not mounted - aborting"; exit 1; }
  rsync -av --delete ~/apps/ /mnt/backup/apps/
  ```
  `-a` preserves permissions/timestamps, `-v` is verbose, `--delete` mirrors
  deletions. Schedule it with `cron` (`crontab -e`) to run nightly — noting the
  crontab quoting and `PATH` traps in [step 24](#24-task-automation-and-scheduled-jobs).
  If the destination drive is exFAT, see the caveats in [Choosing a filesystem
  for attached storage](#17-choosing-a-filesystem-for-attached-storage) before
  you reach for `-a`.

  The `mountpoint` guard on the first line is not optional padding. If the
  external drive fails to mount, `/mnt/backup` still exists as an empty
  directory *on the boot card* — so `rsync` writes the whole backup onto the
  microSD it was supposed to protect, fills it, and exits 0. `mountpoint -q`
  returns non-zero for a plain directory, so the job stops instead. Put the
  same guard in front of every scheduled job that writes to attached storage
  (see [step 27](#27-a-checklist-to-verify-your-setup)).

  **Check `rsync`'s exit code, don't just check that it ran.** A run that copies
  most files but fails on some — one unreadable file, one attribute the
  destination filesystem can't store — still transfers everything else and then
  exits **23** ("some files/attrs were not transferred"); **24** means files
  vanished mid-run. Only `0` is a clean backup, so a scheduled job should test
  it and say so:
  ```bash
  rsync -av --delete ~/apps/ /mnt/backup/apps/ || { echo "rsync failed (exit $?)"; exit 1; }
  ```
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
  [step 21](#21-versioning-your-configuration-with-git)). Also **check the dump
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
  sudo dmesg -T --level=err,warn | tail -30
  df -h
  ```
  `-T` prints real dates instead of seconds-since-boot (otherwise you cannot
  tell a fault from this morning from one from six months ago), and
  `--level=err,warn` selects by the kernel's own severity field rather than
  grepping for the word "error" — which misses the I/O and `mmc0:` messages a
  failing card actually emits. Note this is the kernel ring buffer, so it only
  covers the current boot. For longer history use
  `journalctl _TRANSPORT=kernel -p warning --since "1 week ago"` (needs a
  persistent journal — see [step 22](#22-log-management)). Not `journalctl -k`
  with a `--since`: `-k` implies `-b`, so it silently clamps the answer to the
  current boot no matter how far back you ask — the reboot you were trying to
  explain is on the other side of that boundary. (Verified on a Pi: the two
  forms returned 2,776 and 31,766 lines for the same one-week window.)
  Don't be surprised if an SD card needs replacing after a year or two of heavy
  24/7 writes — this is why an SSD is worth it for busy setups.
- **Keep a running TODO list** of unfinished items. A home server is rarely
  "done" in one sitting.

## 21. Versioning your configuration with git

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
- This pairs naturally with [scheduled automation](#24-task-automation-and-scheduled-jobs)
  — a nightly job that snapshots changed config, scans it, and commits only if
  there's a real, clean delta.

## 22. Log management

Every service on the Pi writes logs somewhere, and left unmanaged they'll
eventually fill your disk. `logrotate` is the standard Linux tool for keeping
log files bounded — rotating (renaming/compressing) them on a schedule or size
trigger and deleting old ones.

Most distro packages already ship a logrotate config for their own logs, but
anything writing to a custom path (an [AI ops agent](#25-running-an-ai-ops-agent-on-the-server)
under `~/.hermes/logs/`, a script's own log file, etc.) needs its own rule:

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

**One exception to this guide's back-up-before-you-edit habit:** do not leave
the backup copy *inside* `/etc/logrotate.d/`. logrotate reads every file in that
directory except a fixed list of taboo suffixes (`.dpkg-old`, `.rpmsave`,
`.ucf-new`, a trailing `~`, and a few more) — and `.bak` is **not** on that
list. A `my-app.bak.20260101` sitting beside `my-app` is parsed as a second,
live config, which then trips `duplicate log entry for <path>` and makes the
whole run exit non-zero. Keep backups of these files somewhere outside the
directory (or end the name with `~`).

No new timer is usually needed — most systems already run `logrotate` daily via
a system timer or cron entry; a new config just needs to exist under
`/etc/logrotate.d/` to be picked up on the next run. To check your rule without
waiting for that run, do it in two steps:

```bash
sudo logrotate -d /etc/logrotate.d/my-app    # dry run: says what it WOULD do
sudo logrotate -f /etc/logrotate.d/my-app    # force: actually rotates, now
```

`-d` is the safe one to reach for first — it parses the config, reports errors,
and changes nothing on disk. `-f` is **not** a test: it performs a real
rotation immediately, ignoring your `size`/`daily` conditions, and burns one of
your `rotate N` slots. That is fine for a fresh rule, but running it against a
system config out of curiosity will genuinely rotate live logs. Note also that
`-d` implies debug output and skips the state file, so it does not tell you
whether the *schedule* would have fired — only whether the rule is valid.

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
doesn't own it. A chatty container can quietly eat gigabytes. Check what yours
are using:

```bash
sudo find /var/lib/docker/containers -name '*-json.log' -exec du -ch {} + | tail -1
```

The `find` form is not fussiness. `/var/lib/docker/containers` is readable only
by root (`drwx--x---`), and a shell expands `*/*-json.log` **before** `sudo`
runs — as you, without permission — so the obvious
`sudo du -ch /var/lib/docker/containers/*/*-json.log` reports
`No such file or directory` and a reassuring `0 total` even when the logs are
gigabytes. Quoting the pattern hands the matching to `find`, which is already
running as root.

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

## 23. Integrating third-party device and cloud APIs

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

Run the poller as its own [scheduled job](#24-task-automation-and-scheduled-jobs),
keep its credentials in a permissions-locked env file (not committed to git —
see [step 21](#21-versioning-your-configuration-with-git)), and document any
hard limits or gotchas you discover for the specific API next to the script,
so the next debugging session doesn't start from zero.

## 24. Task automation and scheduled jobs

Most of what keeps a home server healthy without your daily attention is
**scheduled, unattended jobs**: nightly backups, a weekly summary, a poller
checking a device API, a log rotation. Linux gives you two built-in
mechanisms:

- **`cron`** — the classic choice, simplest for "run this script at this
  time/interval." Edit your own crontab with `crontab -e`; each line is
  `<minute> <hour> <day> <month> <weekday> <command>`.

  Two things about crontab lines bite almost everyone once:

  - **A bare `%` is not a percent sign.** In a crontab, an unescaped `%` is
    turned into a newline and everything after the first one is fed to the
    command as standard input. So the timestamped-backup idiom used elsewhere
    in this guide, `cp file file.bak.$(date +%Y%m%d)`, is *silently truncated*
    when pasted straight into a crontab — you get `date +` and an error. Escape
    each one (`\%Y\%m\%d`), or better, put the command in a script file and
    schedule the script. See `man 5 crontab`.
  - **cron gives you a minimal environment**, not your interactive shell's. It
    runs commands under `/bin/sh` and does not read your `.bashrc`/`.profile`,
    so aliases, virtualenv activation, and anything found via a customised
    `PATH` are all absent. Use **absolute paths** for every binary and file in a
    cron job (`/usr/bin/rsync`, not `rsync`), or set `PATH=` explicitly at the
    top of the crontab.
- **systemd timers** — more modern, integrate with `systemctl status`/`journalctl`
  for easier debugging, and can express things like "run 5 minutes after boot"
  that plain cron can't. More setup (a `.service` + a `.timer` unit) for the
  same result.

Either way, a few habits keep scheduled jobs from becoming their own source of
surprises:

- **Make failures loud, successes quiet.** A nightly job that silently fails
  for a month is worse than one that pages you once. Route errors to your
  alert channel; let clean runs produce no notification (or a single quiet
  weekly summary) rather than a message every single night. This assumes the
  job can actually detect its own failure — see [A check that cannot fail is
  not a check](#a-check-that-cannot-fail-is-not-a-check).
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

## 25. Running an AI ops agent on the server

Everything so far assumed *you* are the one typing commands. A newer option is
to run an **AI ops agent** on the Pi itself: a persistent process with terminal
access that you talk to in plain English, which can investigate problems, run
scheduled jobs, and explain what it finds. This guide was itself written with
one, so it's worth being concrete about what that's actually good for — and
where it is a liability.

**Where an agent genuinely helps**

- **Diagnosis over memorization.** "Why is this container restarting?" is a
  question you can ask instead of remembering which of `docker logs`,
  `journalctl`, and `dmesg` to reach for. This is the single biggest win for a
  beginner.
- **Scheduled jobs that summarize rather than dump.** A cron job that emails you
  200 lines of log is noise; one that reads the logs and messages you only when
  something genuinely changed is useful (see
  [Task automation](#24-task-automation-and-scheduled-jobs)).
- **A written record.** An agent that maintains its own notes file about your
  machine gives you a changelog you'd never keep by hand.

**Where it is a liability — read this part twice**

- **It has your shell.** An agent with terminal access can do anything you can,
  including destroy the machine. This is not hypothetical: the whole reason
  [the tiered approval model](#26-a-tiered-approval-model-for-automation)
  exists is to bound that.
- **It is confidently wrong sometimes.** It will state things that sound
  authoritative and are false. Ask it to *show you the command output* it based
  a claim on — and treat "I verified it" as a claim needing evidence too.
- **Anything it can reach, it can leak.** A prompt is not a security boundary.
  Keep credentials in permission-locked files, and assume anything the agent can
  read could end up in a reply.
- **It runs on someone's model.** Unless you point it at a
  [local model](#16-running-a-local-ai-model-with-ollama), your commands and
  file contents go to a cloud provider. Decide if you're OK with that *before*
  pointing it at private data.

### Installing one (Hermes as the worked example)

[Hermes](https://github.com/NousResearch/hermes-agent) is the agent used to
build this guide, so it's the concrete example here — the same reasoning applies
to any comparable tool. On a Pi running 64-bit Raspberry Pi OS:

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

As with any `curl | bash` installer, read the script first if you'd rather not
run an unreviewed remote script — the same caution as the Docker and Ollama
installers earlier. It pulls in Python 3.11, Node.js, and its own dependencies,
then walks you through provider and messaging setup:

```bash
hermes setup          # interactive: model provider, API key, messaging platform
hermes                # start an interactive session
```

Useful subcommands once it's running:

| Command | What it does |
|---|---|
| `hermes setup` | Interactive configuration wizard |
| `hermes config` | View and edit configuration directly |
| `hermes cron` | Manage scheduled jobs |
| `hermes gateway` | Messaging gateway (Telegram, Discord, and others) |
| `hermes tools` | Choose which tools are enabled per platform |
| `hermes logs` | View and filter its logs |
| `hermes send` | Send a message from a script or cron job |

Configuration lives in `~/.hermes/config.yaml`, secrets in `~/.hermes/.env`
(keep it `chmod 600` and out of git — see
[step 21](#21-versioning-your-configuration-with-git)), and logs in
`~/.hermes/logs/`, which is exactly the kind of custom path that needs its own
logrotate rule ([step 22](#22-log-management)).

### What actually makes it useful: the layer on top

A vanilla install gives you a capable assistant with a shell. It does **not**
give you something that knows your machine, holds a consistent policy, or does
anything while you sleep. That comes from four things you add yourself, and they
matter far more than the install command:

**1. A constitution — the operating contract it reads every session.**

The highest-value file you will write. A plain-markdown document defining who
the agent is, who it works for, and precisely what it may do unprompted. Keep
it in the agent's home directory and have it loaded at the start of every
session. Worth pinning down explicitly:

- **The single overriding rule.** On a headless box that is almost always *"do
  not lock the owner out of this machine."* Every judgment call gets measured
  against it, and a safely reversible change beats an elegant one that could
  strand you.
- **The permission tiers** — see
  [the next section](#26-a-tiered-approval-model-for-automation).
- **Communication style**, including how much to explain. Tuning this for a
  Linux beginner ("define jargon on first use, lead with the answer, one problem
  at a time") is what turns the agent into a teacher rather than a black box.
- **What counts as approval.** State plainly that **silence is never consent**
  and that an unanswered message times out into "no". Without this an agent will
  happily read a lack of objection as a green light.
- **Instructions come only from you.** Log files, error messages, web pages, and
  container images are *data*, not commands. An agent that will run whatever a
  README tells it to is a serious liability — this rule is the mitigation.

**2. A memory file — what it knows about *your* box.**

A living document the agent maintains itself: what's installed and why, every
config change with its rollback command, decisions you've already made (so they
don't get relitigated monthly), and open threads. This is what makes a cold
session start informed instead of re-discovering your setup every time.

Two rules keep it honest — have the agent verify claims against the live system
rather than trusting its own past notes, and record *rollback commands beside
every change*, not just what was done. Version it with git alongside your other
config ([step 21](#21-versioning-your-configuration-with-git)) so it can't
silently regress.

**3. Scheduled jobs — the part that runs without you.**

This is where an always-on server earns its keep, and it goes well beyond the
sysadmin basics. Jobs fall into roughly three classes:

| Class | Runs as | Examples |
|---|---|---|
| **Deterministic scripts** | Plain shell/Python, no model | Nightly backup, new-device LAN scan, download-completion watcher, config-repo sync, device API pollers |
| **Agent jobs** | Model reasons over live data | Weekly state audit that diffs the machine against last week, doc maintenance, digests that summarize instead of dumping |
| **External-world jobs** | Model + web access | Price/deal watches, news digests, anything needing judgment about relevance |

Two hard-won rules. **Anything expressible as a script should be a script** —
it's faster, free, and can't hallucinate; save the model for jobs that need
judgment. And **make failures loud, successes quiet**: a job that messages you
nightly gets ignored within a week, so have them emit nothing unless something
genuinely changed.

A worked example of the second class: a weekly audit that snapshots containers,
systemd units, listening ports, and installed packages to a file, diffs it
against last week's, and has the agent investigate real changes and fold them
into the memory file. That is the automated answer to "the notes drift out of
date."

**4. Skills — reusable procedures it loads on demand.**

A skill is a markdown file describing how to do one recurring task properly,
loaded only when relevant. This solves the problem that a constitution can't:
you don't want every niche procedure in the system prompt on every call, but you
do want the agent to get it right when it comes up.

Write one whenever you work out a non-obvious procedure — the gotchas of a
specific vendor API, how a recurring report should be formatted, the safe
sequence for a fiddly migration. Each is cheap and pays off every later time
that task recurs. Crucially, when the agent gets something wrong and you correct
it, **fold the correction back into the skill** so it isn't relearned next time.

### Back up the agent's state directory, not just your data

Everything in the previous section — the constitution, the memory file, the job
definitions, the skills you wrote after each correction — lives in the agent's
own home directory, typically `~/.hermes/`:

```
~/.hermes/
  config.yaml      # providers, models, gateways, tool permissions
  .env             # API keys and tokens (chmod 600)
  memories/        # what it knows about your box
  skills/          # procedures you taught it
  cron/            # scheduled job definitions
  logs/
```

It is worth being blunt about why this matters: **reinstalling the software does
not get any of this back.** The install command takes minutes; the contents of
that directory are months of accumulated corrections, decisions you already made
once and don't want to relitigate, and procedures you worked out the hard way.
Lose it and you are not doing a fresh install, you are starting the tuning over
from zero — and you won't remember most of what was in there, because the whole
point of writing it down was to stop carrying it in your head.

So fold it into the routine you already have from
[step 20](#20-backups-and-maintenance), alongside your app data:

```bash
mountpoint -q /mnt/backup || { echo "backup drive not mounted - aborting"; exit 1; }
rsync -av --delete --exclude '.env' --exclude 'logs/' \
  ~/.hermes/ /mnt/backup/hermes-state/
```

Two decisions to make deliberately:

- **Secrets.** The `--exclude '.env'` above assumes the backup destination is
  less trusted than the Pi — a shared NAS, a drive that travels, a cloud target.
  That's the safer default, but it means a restore needs you to re-add API keys
  by hand, so write that down somewhere. If you do include secrets, the
  destination has to be treated as being as sensitive as the Pi itself.
- **The rest of the directory is not innocuous either.** The memory file is a
  written description of your machine: what's installed, which ports are open,
  where the config lives, which decisions were made and why. That is genuinely
  useful to you and genuinely useful to anyone else who gets a copy. Back it up
  somewhere you'd be comfortable storing your config repo
  ([step 21](#21-versioning-your-configuration-with-git)) — the same standard,
  for the same reason.

The memory file and skills are plain markdown, so the neatest arrangement is to
version them in that private config repo and let the backup job cover the rest.
You get diffs on the parts that change meaningfully and a flat copy of
everything else.

### Delegating work to subagents

Some agents can spawn a **subagent**: a separate task with its own context that
does a chunk of work and reports back a summary, rather than doing it inline in
your conversation. It's a real capability, and it's also one people reach for
too early, so it's worth naming when it actually pays.

It's worth it when:

- **The work is genuinely parallel.** Three independent questions — "what
  changed in the container set", "is the backup drive healthy", "any new devices
  on the LAN" — have no dependency on each other. Running them as separate tasks
  finishes in roughly the time of the slowest one instead of the sum.
- **The intermediate output is enormous and disposable.** Grepping a week of
  journald to answer one question produces thousands of lines you never want to
  read. A subagent can wade through it in its own context and hand back the
  three-line answer, leaving your main session uncluttered — which matters
  because a context stuffed with log noise makes the agent measurably worse at
  the thing you actually asked.
- **You want a genuinely independent second look.** A review task that hasn't
  seen the reasoning that produced the work is more likely to catch a bad
  assumption than the same session grading its own homework.

It's overkill when the task is one tool call. Spawning a task to run `df -h` is
strictly slower and more expensive than running `df -h`, and it adds a layer
where information gets summarized — which is exactly where detail goes missing.
The failure mode is worth knowing: **a subagent only reports back what it
decided was relevant.** If you need the exact command output, ask for the output,
not a summary of it. That is the same "show me what you based that on" rule from
the top of this section, and it applies harder across a delegation boundary.

A practical middle ground on a Pi: keep delegation for the two cases above —
parallel independent work, and noisy searches — and do everything else inline.
The complexity is only worth carrying where it buys you wall-clock time or a
clean context.

### Voice messages: transcription that runs on the Pi

If you drive the agent from a messaging app, **voice notes are the feature that
makes it genuinely usable from a phone** — dictating "why is Nextcloud throwing
502s" while walking is far easier than thumb-typing it. That requires
speech-to-text, and this is one place where the local option is genuinely
competitive.

Most agents default to a local [Whisper](https://github.com/openai/whisper)
implementation — commonly [faster-whisper](https://github.com/SYSTRAN/faster-whisper),
a reimplementation that is several times quicker on CPU than the original:

```bash
pip install faster-whisper
```

Models download automatically on first use. **The default is usually the `base`
model, and upgrading it is the single highest-value tweak here** — technical
vocabulary is exactly where the small models fail, and a homelab voice note is
nothing *but* technical vocabulary.

Measured on a Pi 5 (8GB, CPU-only, `int8`), transcribing a 13.8-second dictated
sentence containing "IPv4", "Docker bridge subnet", and "UFW":

| Model | Disk | Cold load | Decode 13.8s | Result |
|---|---|---|---|---|
| `tiny` | ~75 MB | 5.7s | 3.4s | heard "UFW **blocklines**" |
| `base` | ~142 MB | 2.3s | 6.3s | heard "UFW **blocklines**" |
| `small` | ~464 MB | 6.7s | 19.3s | heard "UFW **block lines**" ✅ |

All three got the general sense right. Only `small` correctly split the term it
had never seen — and on a homelab that difference is the whole point, because
the words you dictate are `ufw`, `fstab`, `journalctl`, and container names.

Note the honest tradeoff: `small` decodes **slower than real time** on a Pi
(19.3s of compute for 13.8s of audio). For a voice note that is completely fine
— you send it and the reply arrives moments later — but it is not suitable for
live streaming transcription, and it costs ~500MB of RAM while running. On a
4GB Pi already running several containers, stay on `base`.

Point the config at the bigger model (exact key varies by agent):

```yaml
stt:
  enabled: true
  language: en          # pinning the language is faster and more accurate
  local:
    model: small        # upgraded from the default "base"
```

Two settings worth getting right:

- **Pin the language** rather than leaving auto-detect on. Detection costs time
  and occasionally guesses wrong on a short clip, producing confident nonsense.
- **Prefer local over a cloud STT provider** unless you have a reason not to.
  Voice notes are the most personal thing you'll send the agent, and local
  transcription means the audio never leaves the Pi — no API key, no per-minute
  cost, and it keeps working when your internet doesn't. Cloud options (Groq,
  OpenAI, ElevenLabs) are faster and better at accents; that is the real reason
  to choose one, not convenience.

If accuracy still disappoints, the fix is usually **not** a bigger model:
record somewhere quieter, closer to the mic. Whisper degrades much harder on
background noise than on vocabulary.

### Cost, privacy, and a local fallback

A cloud-model agent running scheduled jobs draws quota on a schedule whether or
not anything interesting happened. Worth doing deliberately:

- **Match the model to the job.** Reserve a frontier model for genuinely hard
  reasoning; a small fast model handles routine summarizing fine.
- **Point trivial work at a local model.** Anything that doesn't need a big
  model can go to [Ollama](#16-running-a-local-ai-model-with-ollama) on the Pi
  itself — free, private, and it keeps working when your quota doesn't. Most
  agents accept an OpenAI-compatible endpoint, so `http://localhost:11434/v1`
  usually just works, including as an automatic **fallback** when the primary
  provider is unavailable.
- **Pin the model per job**, rather than letting jobs inherit whatever your
  global default happens to be — see the next subsection.

### Pin the provider and model on every scheduled job

This one deserves its own heading because it is easy to get wrong, and the
failure is silent.

If your agent's scheduler lets a job run under the **global default model**, then
that job's behavior is defined by a setting that lives somewhere else and that
you will eventually change for an unrelated reason — you switched providers to
cut cost, a new model shipped and you tried it, your key for the old one lapsed.
Nothing errors. The job still runs on schedule. It just starts producing
different output: a digest that used to be four tight lines is now twelve, a
summariser that reliably found the one real change now buries it, or a job tuned
against a model with web access quietly loses it.

The fix is to write the provider and model into the job definition itself,
always, even when it matches the current default:

```yaml
# a scheduled job, with the model pinned explicitly
name: weekly-state-audit
schedule: "0 8 * * 1"
provider: anthropic
model: claude-sonnet-4
prompt: |
  Diff this week's system snapshot against last week's and report only
  real changes.
```

Two things follow from this:

- **Pinning makes the cost/quality choice visible per job.** A nightly log
  summariser and a weekly audit that has to reason about diffs genuinely want
  different models, and that decision belongs next to the job, not in a global.
  It also pairs with pointing trivial jobs at a
  [local model](#16-running-a-local-ai-model-with-ollama).
- **Changing the global default becomes safe again.** You can experiment with
  your interactive default without touching anything that runs unattended,
  which is the whole reason to bother.

Treat it as a rule rather than a judgment call: pin every job at creation time.
This is the kind of thing you get bitten by exactly once — you spend an evening
debugging a job whose output changed, find nothing wrong with the job, and
eventually realise you changed a setting two weeks earlier in a different file.
After that you never leave one unpinned again.

### More than one messaging channel is more than one exposure decision

Once the agent is reachable from your phone, it's tempting to make it reachable
from *everywhere* — Telegram and WhatsApp and Discord and email — because each
one is a small config block and each one is genuinely convenient in a different
place. Most agents support several gateways at once, and the setup is easy:

```bash
hermes gateway   # configure a messaging platform
```

The ease is the trap. Every inbound channel you add is **a fresh exposure
decision in exactly the sense of [step 11](#11-reaching-your-pi-from-outside-home)**,
and a higher-stakes one than publishing a normal service, because what's behind
this door has a shell on your server. Specifically, each additional gateway is:

- **Another place a message can arrive from.** Access control lives in the
  gateway's own config — an allowlist of user or chat IDs. If a platform's
  identifier is easier to guess or spoof than the others, your agent is now only
  as protected as the weakest one you've enabled.
- **Another long-lived credential to protect.** A bot token is a bearer token:
  whoever holds it can send messages as that bot, and on some platforms read the
  history too. It belongs in `~/.hermes/.env` at `chmod 600`, out of git
  ([step 21](#21-versioning-your-configuration-with-git)), and — as above — out
  of a backup destination you don't fully trust.
- **Another thing that can be misconfigured.** Webhook-based gateways need a
  publicly reachable URL, which quietly turns "an agent on my LAN" into "an
  endpoint on the internet." If you go that route, terminate it behind the
  tunnel and auth gate from [step 11](#11-reaching-your-pi-from-outside-home)
  rather than opening a port, and verify the gateway itself rejects unknown
  senders rather than assuming an obscure URL is protection.

The practical advice is not "don't" — a second channel is legitimately useful,
especially if one platform is where family already messages you. It's **add them
one at a time, and for each one confirm the allowlist works by messaging the bot
from an account that shouldn't have access.** An allowlist you haven't tested
from the outside is an assumption, not a control. And prefer the channel you
already use for alerts ([step 14](#14-uptime-monitoring-and-alerts)) as the
primary one, so the approval path and the alert path stay the same place.

### Setting it up sanely

- **Connect it to the alert channel you already use** rather than a new one
  ([step 14](#14-uptime-monitoring-and-alerts)). A messaging platform doubles as
  the approval channel, so the agent can ask permission when you're not at a
  terminal.
- **Keep secrets in permission-locked files**, referenced by path, never pasted
  into a config the agent quotes back. `chmod 600`, outside git
  ([step 21](#21-versioning-your-configuration-with-git)).
- **Rotate its logs.** `~/.hermes/logs/` is exactly the custom path
  [step 22](#22-log-management) is about.
- **Back up its state directory** with everything else
  ([step 20](#20-backups-and-maintenance)) — the tuning in there is the part you
  can't reinstall.
- **Don't let it be your only way in.** If the agent is how you administer the
  box, a broken agent is a lockout. Keep SSH working independently
  ([step 6](#6-secure-your-ssh-access)) and a
  [VPN fallback](#11-reaching-your-pi-from-outside-home).
- **Expect to correct it.** The setup above is not one-time configuration; it's
  a feedback loop. Every wrong assumption it makes is a line to add to the
  constitution, the memory file, or a skill.

> **If you expose its web or chat interface, that is a fresh exposure decision**
> under [step 11](#11-reaching-your-pi-from-outside-home) — and a higher-stakes
> one than usual, because the thing behind the hostname has a shell on your
> server. Put it behind a VPN or an auth gate; never publish it bare.

## 26. A tiered approval model for automation

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
   access, use an **automatic rollback timer** — see below.

### The rollback timer, concretely

The idea is simple: before applying a change that could cut your own connection,
schedule the undo *first*, then apply the change. If you can still get in, you
cancel the undo. If you can't, the machine fixes itself while you go and make
tea, instead of you driving home to fetch the SD card.

Many write-ups reach for `at` for this, but `at` is not installed by default on
Raspberry Pi OS or a minimal Debian, so a copy-pasted `echo ... | at now + 5
minutes` fails with `command not found` at exactly the wrong moment. `systemd-run`
is already present on any systemd machine and needs no package:

```bash
# 1. Schedule the undo BEFORE touching anything.
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.rollback
sudo systemd-run --on-active=5min --timer-property=AccuracySec=1s \
  --unit=ssh-rollback \
  /bin/bash -c '/bin/cp /etc/ssh/sshd_config.rollback /etc/ssh/sshd_config && /bin/systemctl restart ssh'

# 2. Now make the change and apply it.
sudo nano /etc/ssh/sshd_config
sudo sshd -t && sudo systemctl restart ssh

# 3. From a SECOND terminal, prove you can still log in.
#    Only then, cancel the pending revert:
sudo systemctl stop ssh-rollback.timer
```

A few details that matter:

- `--unit=` gives the job a predictable name. Without it `systemd-run` invents a
  random one (`run-r4f8....timer`), and you have to go hunting for it under
  pressure — which defeats the point.
- Check it is really pending with `systemctl list-timers ssh-rollback --no-pager`;
  it should show a `NEXT` a few minutes out. After you stop it, the same command
  should list **zero** timers, and the revert never runs. (Verified on Debian 12:
  scheduling, firing, and cancelling all behave exactly as described here.)
- **`--timer-property=AccuracySec=1s` is doing real work.** By default systemd
  gives a timer a one-minute accuracy window and may fire it anywhere inside
  that window, so a nominal `--on-active=5min` can actually run at nearly six
  minutes. (Measured on a Pi: a default-accuracy `--on-active=5sec` job fired 23
  seconds late.) For a rollback you want the deadline you asked for — and if you
  *test* the pattern with a short delay first, without this property you will
  conclude it is broken when it has merely not fired yet.
- **The revert command must be able to run without you.** It executes as root
  with no shell profile and no terminal, so use absolute paths and don't make it
  depend on anything interactive.
- **Name the restore file exactly**, not with a wildcard. The timestamped
  `.bak.$(date ...)` convention used elsewhere in this guide is right for an
  archive, but a `cp /etc/ssh/sshd_config.bak.* ...` in the revert command breaks
  the moment a second backup exists — `cp` then sees several sources and one
  non-directory target, and fails. A single fixed `.rollback` copy has exactly
  one meaning.
- The same pattern covers a firewall change (`ufw reset` or restoring
  `/etc/ufw/user.rules`) or a network reconfiguration. Adjust the delay to how
  long you realistically need to test — five minutes is enough to open one new
  SSH session, not enough to get distracted.
- This is a safety net, not a substitute for the older rule in this guide: keep
  your existing session open, and test from a *new* one.

## 27. A checklist to verify your setup

Every section above told you to *do* something. This one tells you how to
**prove it worked** — because the failure mode of a home server is silent: the
backup that never ran, the firewall rule that Docker quietly bypassed, the
auto-updater that stopped a month ago. Run this list after your initial build,
then again every few months.

Each check is a command whose output you can judge on the spot.

**Access and identity**

```bash
ip -brief -4 addr show eth0       # matches the IP you reserved (step 5)?
sudo ss -tulpn                    # every listening port, and what owns it
```

Look for anything listening on a wildcard address that you did not intend to
publish — that's the single most useful line in this checklist. Note that `ss`
writes a wildcard bind several ways depending on the socket: `0.0.0.0`, a bare
`*`, and `[::]` (all IPv6 addresses, which on Linux usually accepts IPv4 too)
all mean "every interface". Grepping only for `0.0.0.0` misses the other two,
so ask for all three at once — and only in the *local* address column, since
the peer column of a listening socket is a wildcard on every socket and will
match anything:

```bash
sudo ss -tulpnH | awk '$5 ~ /^(0\.0\.0\.0|\*|\[::\]):/ {i=index($0,"users:"); print $1, $5, (i?substr($0,i):"-")}'
```

The `index`/`substr` part looks like fussiness and isn't: several real daemons
register a process name containing a space (`Plex Media Serv`,
`Plex Tuner Serv`), so a plain `print $7` chops the name at the first space and
prints `users:(("Plex` — dropping the PID and the rest of the identity, which is
the whole reason you ran the command. Taking everything from `users:` onward
keeps the owner intact whatever it's called. (Verified on a Pi running Docker,
Samba and Plex: `$7` truncated 6 of 20 lines.)

Read that list against the services you meant to publish. On a Pi running
Docker, Samba and a media server it is long and most of it is expected — the
point is to spot the one entry you cannot account for. Then confirm your
firewall actually has IPv6 rules rather than only IPv4 ones (step 7):

```bash
sudo grep -c '^-A ufw6-user-input' /etc/ufw/user6.rules   # IPv6 allow rules
sudo grep -c '^-A ufw-user-input'  /etc/ufw/user.rules    # IPv4 allow rules
```

Compare the two numbers. A large IPv4 count next to a `0` for IPv6 is the
normal-looking-but-wrong state: `IPV6=yes` in `/etc/default/ufw` makes ufw
*filter* IPv6, but every `ufw allow from 192.168.1.0/24 ...` rule you wrote is
IPv4-only, so none of them has an IPv6 twin. Whether that is a problem depends
on which side of default-deny you land on — see step 7.

**Security**

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
sudo ufw status verbose
sudo fail2ban-client status sshd
```

Expect `passwordauthentication no` and `permitrootlogin no` (step 6), a default
deny policy (step 7), and a jail that reports `Status for the jail: sshd`
rather than an error.

Two ways this particular check can reassure you wrongly:

- **A bare `sshd -T` does not evaluate `Match` blocks.** It prints the global
  defaults only, so a config that says `PasswordAuthentication no` at the top
  and re-enables it inside a `Match Address 192.168.1.0/24` block still reports
  a clean `passwordauthentication no`. To see what a *specific* client actually
  gets, supply the connection context with `-C`:
  ```bash
  sudo sshd -T -C user=<username>,host=example,addr=<client-ip> | grep -i passwordauth
  ```
  Run it once with an address inside any `Match` you have and once with an
  outside address; if the two answers differ, that difference is your real
  policy. (Verified on Debian 12: the same file reports `no` bare and `yes`
  with a matching `-C addr=`.)
- **`permitrootlogin without-password` is not `no`.** That is how `sshd -T`
  renders `prohibit-password`, the Raspberry Pi OS default — it means root may
  still log in *with a key*, just not a password. It is a reasonable setting,
  but if you intended to bar root entirely, this output is not the confirmation
  it looks like; set `PermitRootLogin no` and re-check.

Resist reading the *counters* as a health check. A `Total failed: 0` is often
taken as proof that fail2ban is watching the wrong log, but on a Pi that is
LAN-only — or whose SSH port is firewalled to a subnet or a VPN range (step 7)
— zero is simply the truth: the bots never reach sshd to fail against it. The
counter only becomes evidence once the port is genuinely reachable from the
internet. To check the log source itself, use the direct `get sshd logpath`
test in step 8 instead; it answers in one command and doesn't depend on being
attacked.

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
findmnt -M /mnt/storage || echo "NOT MOUNTED"   # drive mounted, not an empty dir
ls -lt /path/to/your/backups | head
```

The mount check matters more than it looks: if an external drive fails to
mount, the mount point still exists as an empty directory on the boot card — so
a backup script writes happily into it, filling your SD card while appearing to
succeed (step 20). Use `-M` (exact mountpoint), **not** `--target`: `--target`
walks *up* to whichever filesystem contains the path, so on an unmounted
`/mnt/storage` it cheerfully prints the root filesystem and exits 0 — the false
pass you were trying to catch.

**Health**

```bash
vcgencmd get_throttled            # throttled=0x0 = clean; anything else, decode it (step 29)
systemctl --failed                # should list zero units
docker ps -a --format '{{.Names}}\t{{.Status}}'
free -h                           # read the "available" column, not "free"
```

Note the `-a` on `docker ps`. Without it the command lists only *running*
containers, so a service that has stopped dead — the exact failure this
checklist exists to catch — is not reported as broken; it simply isn't in the
output, and a short, clean-looking list reads like a pass. (Verified: a
container that exits is absent from `docker ps` and shown as
`Exited (3)` by `docker ps -a`.) Read the list against the set of services you
expect to be running, not just the statuses printed.

Any container showing `Restarting` is crash-looping, not running — and if one
keeps dying without an obvious log reason, check for an OOM kill (step 18).

**The two checks nothing on this list can do for you**

- **Restore from a backup.** Only an actual restore proves a backup is real.
  Everything above only proves a *file exists*.
- **Confirm your alerts arrive.** Deliberately trip one — stop a monitored
  container, or use your notification channel's test button — and check the
  message reaches the device you actually look at. An alerting path is
  untested until a message has travelled it end to end.

## 28. Lessons learned

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

## 29. Troubleshooting

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
- **A container won't start** — establish whether it exited or is crash-looping
  before reading logs, and read the *last* output rather than following a
  stream that may already be over:
  ```bash
  docker compose ps -a                        # -a also lists exited containers
  docker compose logs --tail=50 <service>
  ```
  A `Restarting` status means it starts and dies repeatedly — check for an OOM
  kill ([step 18](#18-memory-swap-and-container-resource-limits)) or a
  permissions problem on a mounted volume before suspecting the image itself.
- **An external drive intermittently fails to auto-mount after an unclean
  disconnect** — common with NTFS-formatted drives that get unplugged without
  "safely eject" first, leaving a "dirty" filesystem flag Linux won't
  auto-mount read-write. A small boot-time check-and-repair script (using
  `ntfsfix -n` to detect the dirty state, then `ntfsfix` to clear it before
  retrying the mount) turns this from a manual fix into something that
  self-heals on every boot — see [Task automation and scheduled
  jobs](#24-task-automation-and-scheduled-jobs) for wiring a script to run at
  boot via systemd. **Remember to retire or rewrite this kind of script if you
  later reformat the drive to a filesystem without a dirty-bit concept** (e.g.
  exFAT — see [Choosing a filesystem for attached
  storage](#17-choosing-a-filesystem-for-attached-storage)); it'll harmlessly
  no-op forever rather than fail loudly, which is easy to forget about.
- **Pi feels slow or reboots randomly** — suspect power or heat first. Check the
  temperature and for under-voltage warnings:
  ```bash
  vcgencmd measure_temp
  vcgencmd get_throttled   # prints e.g. throttled=0x0
  ```
  `throttled=0x0` is the all-clear. Anything else is a **bit field**, and the
  half that matters is which half of it is set — the low bits describe what is
  happening *right now*, the high bits are sticky flags that latch on the first
  occurrence and stay set until reboot:

  | Bit | Meaning now | Sticky bit | Meaning since boot |
  |---|---|---|---|
  | 0 (`0x1`) | Under-voltage | 16 (`0x10000`) | Under-voltage has occurred |
  | 1 (`0x2`) | ARM frequency capped | 17 (`0x20000`) | Frequency capping has occurred |
  | 2 (`0x4`) | Currently throttled | 18 (`0x40000`) | Throttling has occurred |
  | 3 (`0x8`) | Soft temperature limit active | 19 (`0x80000`) | Soft limit has been hit |

  So `throttled=0x50000` means "under-voltage and throttling happened at some
  point since boot, and neither is happening now" — worth investigating, not an
  emergency. A value with any of the low four bits set means it is happening as
  you read it. Under-voltage bits point at the power supply or cable; the
  temperature bits point at cooling. Reference:
  [`vcgencmd` documentation](https://www.raspberrypi.com/documentation/computers/os.html#get_throttled).
- **A backup/rsync job errors out partway through on an external drive** —
  suspect the filesystem before the script. See [Choosing a filesystem for
  attached storage](#17-choosing-a-filesystem-for-attached-storage): exFAT in
  particular rejects ownership, permission, and symlink/hardlink operations
  outright.
- **fail2ban is running but never bans anyone** — first rule out the boring
  explanation: if SSH is only reachable from your LAN or a VPN, there is
  nothing to ban. If the port really is public, confirm the jail is reading a
  log source that exists, using `fail2ban-client get sshd logpath` as described
  in [step 8](#8-block-brute-force-attacks-fail2ban).

## 30. Further reading

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
- [Hermes agent](https://github.com/NousResearch/hermes-agent)
- [Caddy documentation](https://caddyserver.com/docs/)
- [logrotate(8) manual](https://linux.die.net/man/8/logrotate)
- [journald.conf(5) — journal size and persistence](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)
- [r/homelab](https://www.reddit.com/r/homelab/) and r/selfhosted for community
  setups and troubleshooting

---

*This guide was generalized from a real Raspberry Pi 5 setup, written up with
the help of an AI ops agent. If you're setting up something similar and want to
compare notes on any of the choices above, feel free to open an issue.*
