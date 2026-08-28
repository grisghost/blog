---
title: "Installing AdGuard Home: Docker vs Native — Complete Setup Guide"
description: "A step-by-step guide to installing AdGuard Home using Docker or directly on the system, with proper restart policies to keep it running after every reboot."
date: 2026-08-28 12:00:00 +0200
categories: [Self-Hosting, Networking]
tags: [adguard-home, docker, linux, dns, self-hosting, systemd]
---

## What Is AdGuard Home?

AdGuard Home is a self-hosted DNS server that blocks ads and trackers at the network
level — before they ever reach any device on your network. Unlike browser extensions,
it works for every device connected to your router: phones, smart TVs, consoles, and
everything else.

You can install it in two ways:

- **Docker** — isolated, easy to update, no impact on the host system.
- **Native (system install)** — runs directly on the OS, slightly lighter, integrates
  naturally with systemd.

Both approaches are covered below. Choose whichever fits your setup.

---

## Method 1 — Docker

### Prerequisites

- Docker installed and running
- Port `53` available on the host (not already used by another DNS service)

> On Ubuntu 22.04+, `systemd-resolved` listens on port 53 by default. You'll need to
> disable it first or configure AdGuard to use a different port.
> ```bash
> sudo systemctl disable --now systemd-resolved
> sudo rm /etc/resolv.conf
> echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
> ```
{: .prompt-warning }

### 1. Create the Required Directories

AdGuard Home needs two persistent directories for its configuration and working data:

```bash
mkdir -p ~/adguard/work ~/adguard/conf
```

### 2. Run the Container

```bash
docker run -d \
  --name adguardhome \
  --restart unless-stopped \
  -v ~/adguard/work:/opt/adguardhome/work \
  -v ~/adguard/conf:/opt/adguardhome/conf \
  -p 53:53/tcp \
  -p 53:53/udp \
  -p 3000:3000/tcp \
  adguard/adguardhome
```

| Flag | Purpose |
| :--- | :--- |
| `--restart unless-stopped` | Restart automatically on reboot or crash, unless manually stopped |
| `-v ~/adguard/work` | Persists runtime data across container restarts |
| `-v ~/adguard/conf` | Persists your configuration |
| `-p 53:53` | Exposes the DNS port (TCP + UDP) |
| `-p 3000:3000` | Exposes the initial setup web UI |

### 3. Complete the Setup

Open your browser and go to `http://YOUR_SERVER_IP:3000`. The setup wizard will guide
you through creating an admin account and choosing the DNS listening port (usually `53`).

After setup is complete, the web dashboard moves to port `80` by default (or whichever
port you configure).

### 4. Verify the Restart Policy

```bash
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' adguardhome
```

Expected output:

```
unless-stopped
```

### Updating the Restart Policy on an Existing Container

If you already have a running container without a restart policy:

```bash
docker update --restart unless-stopped adguardhome
```

No restart required — Docker applies the policy immediately.

### Useful Docker Commands

```bash
# Check if the container is running
docker ps

# View live logs
docker logs -f adguardhome

# Stop the container
docker stop adguardhome

# Update to the latest image
docker pull adguard/adguardhome
docker stop adguardhome && docker rm adguardhome
# Re-run the same docker run command from Step 2
```

> Your configuration and data are safe during updates because they live in the mounted
> volumes (`~/adguard/work` and `~/adguard/conf`), not inside the container.
{: .prompt-tip }

---

## Method 2 — Native System Install

Use this if you prefer running AdGuard Home directly on the OS without Docker.

### 1. Download and Install

AdGuard provides an automated install script:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

The script detects your architecture, downloads the correct binary, installs it to
`/opt/AdGuardHome/`, and registers it as a systemd service automatically.

### 2. Complete the Setup

Same as Docker — open `http://YOUR_SERVER_IP:3000` and follow the wizard.

### 3. Enable Auto-Start on Boot

The installer usually handles this, but verify and enable it manually to be sure:

```bash
sudo systemctl enable AdGuardHome
sudo systemctl start AdGuardHome
```

Check the current status:

```bash
sudo systemctl status AdGuardHome
```

You should see `active (running)`.

### 4. Configure a Resilient Restart Policy

By default, systemd may not restart AdGuard Home if it crashes unexpectedly. Fix this
with a drop-in override — this is cleaner than editing the unit file directly because
it survives package updates:

```bash
sudo systemctl edit AdGuardHome
```

This opens a blank editor. Paste the following:

```ini
[Service]
Restart=always
RestartSec=5
```

Save and exit, then apply the changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart AdGuardHome
```

Verify the override is active:

```bash
sudo systemctl show AdGuardHome --property=Restart,RestartUSec
```

Expected output:

```
Restart=always
RestartUSec=5s
```

> `sudo systemctl edit` creates a drop-in file at
> `/etc/systemd/system/AdGuardHome.service.d/override.conf`.
> It does not modify the original service file, so updates won't overwrite your changes.
{: .prompt-tip }

### Useful Native Commands

```bash
# Check status
sudo systemctl status AdGuardHome

# Stop the service
sudo systemctl stop AdGuardHome

# View live logs
sudo journalctl -u AdGuardHome -f

# Update AdGuard Home
sudo /opt/AdGuardHome/AdGuardHome -update
```

---

## Method Comparison

| | Docker | Native |
| :--- | :--- | :--- |
| Isolation | ✅ Fully isolated | ❌ Runs on host |
| Updates | Manual (pull + re-run) | Built-in `-update` flag |
| Resource overhead | Slightly higher | Minimal |
| Port conflicts | Easier to manage | Needs manual resolution |
| Persistence on reboot | `--restart unless-stopped` | `systemctl enable` + override |
| Best for | Servers already running Docker | Minimal setups, dedicated DNS boxes |

---

## Conclusion

Both methods result in a fully functional AdGuard Home instance that survives reboots
and restarts automatically on failure. Docker is the better choice if you're already
managing other containers; the native install is simpler and lighter if AdGuard Home
is the primary workload on the machine.

Once it's running, point your router's DNS to the server's IP address to start
filtering ads and trackers for every device on your network.
