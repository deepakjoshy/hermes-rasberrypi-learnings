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

## Table of contents

1. [What you need](#1-what-you-need)
2. [Assemble the hardware](#2-assemble-the-hardware)
3. [Flash and install the OS](#3-flash-and-install-the-os)
4. [First login and updates](#4-first-login-and-updates)
5. [Give the Pi a stable address](#5-give-the-pi-a-stable-address)
6. [Secure your SSH access](#6-secure-your-ssh-access)
7. [Set up a firewall (ufw)](#7-set-up-a-firewall-ufw)
8. [Block brute-force attacks (fail2ban)](#8-block-brute-force-attacks-fail2ban)
9. [Turn on automatic security updates](#9-turn-on-automatic-security-updates)
10. [Reaching your Pi from outside home](#10-reaching-your-pi-from-outside-home)
11. [Running services with Docker](#11-running-services-with-docker)
12. [Backups and maintenance](#12-backups-and-maintenance)
13. [A tiered approval model for automation](#13-a-tiered-approval-model-for-automation)
14. [Lessons learned](#14-lessons-learned)
15. [Troubleshooting](#15-troubleshooting)
16. [Further reading](#16-further-reading)

---

## 1. What you need

A shopping/checklist before you start:

| Item | Notes |
|---|---|
| A Raspberry Pi | Any model works; a **Pi 4 or Pi 5 with 4GB+ RAM** is comfortable for several Docker containers. This guide was built on a Pi 5 (8GB). |
| Power supply | Use the **official supply for your model** (a Pi 5 wants 5V/5A USB-C). Underpowered supplies cause random crashes and SD-card corruption. |
| Boot storage | A **32GB+ microSD card** (A1/A2-rated) to start. For anything database-heavy running 24/7, plan to move to an **SSD/NVMe** later — see [Lessons learned](#14-lessons-learned). |
| A second computer | To flash the OS and to SSH in from. Windows, macOS, or Linux all work. |
| Ethernet cable (recommended) | Wired is more reliable than Wi-Fi for a server that must stay reachable. |
| Your home router's login | You'll need it later to reserve an IP address for the Pi. |

You do **not** need a monitor, keyboard, or mouse for the Pi — this guide sets
it up "headless" (over the network) from your second computer.

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
   (see [tier 4](#13-a-tiered-approval-model-for-automation)), you'll occasionally
   need physical access — so somewhere reachable, not a five-hour round trip.

Do **not** plug in the power supply yet. First boot happens after flashing.

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

Save (`Ctrl+O`, Enter) and exit (`Ctrl+X`). Then reload SSH:

```bash
sudo systemctl restart ssh
```

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

**Important:** ufw only filters **direct inbound** traffic. Tunnels and VPNs
(next section) make **outbound** connections, so they bypass these inbound
rules entirely — the firewall does not protect you from exposure decisions you
make with a tunnel. Reference: [ufw docs](https://help.ubuntu.com/community/UFW).

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

## 9. Turn on automatic security updates

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
Reference: [unattended-upgrades docs](https://wiki.debian.org/UnattendedUpgrades).

## 10. Reaching your Pi from outside home

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
cloudflared tunnel login
cloudflared tunnel create home
cloudflared tunnel route dns home app.example.com
cloudflared tunnel run home
sudo cloudflared service install   # run it automatically on boot
```

> **Important — a tunnel is still exposure.** Because you didn't open a port, a
> tunnel *feels* private, but the moment you publish a hostname it is just as
> reachable by anyone on the internet as a port-forwarded service. Treat
> publishing each hostname as its own "am I OK exposing this?" decision. Put an
> auth gate (Cloudflare Access or equivalent) in front of anything that
> shouldn't be fully public.

### A reasonable default mix

- Services outsiders need (e.g. a public share) → **tunnel**, ideally behind an
  auth gate.
- Admin tools, dashboards, download clients → **VPN-only or LAN-only**, never
  publicly tunneled.
- SSH → key-only, plus a **VPN-based fallback** path.

## 11. Running services with Docker

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
      - "3001:3001"
    restart: unless-stopped
```

Start it:

```bash
docker compose up -d
```

- `up -d` starts the container in the background.
- The `volumes:` line keeps the app's data in `./data` so it survives updates.
- `restart: unless-stopped` brings it back automatically after a reboot or crash.

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
[step 10](#10-reaching-your-pi-from-outside-home) — don't default everything to
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
learned](#14-lessons-learned)). Decide the path now, e.g. `/mnt/storage/nextcloud`.

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
      - "8080:80"
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
- `./db` and `./html` keep the database and Nextcloud's own app files next to
  the compose file; `/mnt/storage/nextcloud` (the path you planned in step 1)
  is where actual user files live — mapped in separately so you can put it on
  different, larger storage than the app itself.
- The two `MYSQL_PASSWORD` values and the matching one in `db.environment`
  **must be identical** — a common first-run failure is a typo between them.

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
docker exec -it nextcloud php occ config:system:set trusted_domains 1 \
  --value="cloud.example.com"
docker exec -it nextcloud php occ config:system:set trusted_domains 2 \
  --value="<pi-ip>"
```

Each command adds one more entry (index `1`, `2`, `3`, ...) — don't reuse
index `0`, that's reserved for the original setup hostname.

**5. Decide how it's reachable, and if public, set `overwriteprotocol`.** If
you're exposing it via a [tunnel](#10-reaching-your-pi-from-outside-home)
under HTTPS, tell Nextcloud it's being accessed over HTTPS even though the
container itself only speaks plain HTTP internally — otherwise it will
generate broken `http://` links and reject some requests:

```bash
docker exec -it nextcloud php occ config:system:set overwriteprotocol \
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
(`docker exec nextcloud-db mysqldump -u root -p nextcloud > nextcloud-db.sql`),
the `./html` config/app folder, and the actual files in
`/mnt/storage/nextcloud`. See [Backups and maintenance](#12-backups-and-maintenance).

## 12. Backups and maintenance

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
  deletions. Schedule it with `cron` (`crontab -e`) to run nightly.
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

## 13. A tiered approval model for automation

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

## 14. Lessons learned

- **A tunnel is still exposure.** "No port forwarding" doesn't mean private —
  treat every published hostname as a fresh exposure decision.
- **Back up before every config change**, even trivial-seeming ones.
- **microSD is a wear item.** Move to SSD/NVMe boot if you run anything
  write-heavy 24/7.
- **Stage risky changes with a rollback path**, especially anything touching SSH
  or the firewall — that's the class of mistake that turns a five-minute fix
  into re-flashing a card.
- **Test remote access after every reboot or SSH/firewall change**, from a new
  connection, before you close your working session.
- **Decide your automation/agent approval model before you need it**, not after
  something's gone wrong.

## 15. Troubleshooting

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
  fallback](#10-reaching-your-pi-from-outside-home). Last resort: power off, put
  the card in another computer, and fix the file directly.
- **`ping homeserver.local` doesn't resolve** — mDNS may be off on your network.
  Use the Pi's IP address from your router's device list instead.
- **A container won't start** — check its logs:
  ```bash
  docker compose logs -f
  ```
- **Pi feels slow or reboots randomly** — suspect power or heat first. Check the
  temperature and for under-voltage warnings:
  ```bash
  vcgencmd measure_temp
  vcgencmd get_throttled   # 0x0 means no throttling has occurred
  ```

## 16. Further reading

- [Raspberry Pi official documentation](https://www.raspberrypi.com/documentation/)
- [Tailscale docs](https://tailscale.com/kb/)
- [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Docker Engine install (Linux)](https://docs.docker.com/engine/install/)
- [Docker Compose docs](https://docs.docker.com/compose/)
- [DigitalOcean: SSH key-based authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
- [fail2ban](https://github.com/fail2ban/fail2ban)
- [ufw (Uncomplicated Firewall)](https://help.ubuntu.com/community/UFW)
- [unattended-upgrades (Debian wiki)](https://wiki.debian.org/UnattendedUpgrades)
- [r/homelab](https://www.reddit.com/r/homelab/) and r/selfhosted for community
  setups and troubleshooting

---

*This guide was generalized from a real Raspberry Pi 5 setup, written up with
the help of an AI ops agent. If you're setting up something similar and want to
compare notes on any of the choices above, feel free to open an issue.*
