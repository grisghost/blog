---
title: "Reducing Server Load and Setting Up Swap Memory on Linux"
description: "Practical techniques to reduce CPU and RAM pressure on a Linux server, including creating persistent swap space to handle memory spikes without crashes."
date: 2026-08-28 12:00:00 +0200
categories: [Linux, Server]
tags: [linux, swap, performance, server, memory, ubuntu, oracle-cloud]
---

## Why Servers Run Out of Steam

Low-resource servers — VPS instances, Oracle Cloud Always Free, small Raspberry Pi
nodes — share a common failure pattern: they run fine at idle, then crash or slow to
a crawl the moment a process spikes memory or CPU. Two things cause this:

- **RAM exhaustion** — the kernel starts killing processes (OOM killer) when physical
  memory is full and there's no fallback.
- **Unnecessary CPU load** — background processes, logging, and services eating cycles
  they don't need to.

Both are fixable with straightforward configuration.

---

## Part 1 — Swap Space

Swap is disk space the kernel uses as overflow when RAM fills up. It's slower than RAM,
but it prevents OOM kills and buys the system time to recover from spikes.

### Check Current Swap

```bash
free -h
swapon --show
```

If the swap row is empty or shows `0`, you have none configured.

### Create a Swap File

#### 1. Allocate the File

```bash
sudo fallocate -l 2G /swapfile
```

`fallocate` is instant — it reserves the space without writing zeros. Use `2G` for
servers with 1–2 GB RAM. Adjust as needed:

| RAM | Recommended Swap |
| :--- | :--- |
| 1 GB | 2 GB |
| 2 GB | 2–4 GB |
| 4 GB+ | Equal to RAM or less |

> On some filesystems (like `btrfs`), `fallocate` may fail. Use `dd` instead:
> ```bash
> sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
> ```
{: .prompt-warning }

#### 2. Secure the File

```bash
sudo chmod 600 /swapfile
```

Only root should be able to read or write the swap file. This is a security requirement
— world-readable swap can leak sensitive data from other processes.

#### 3. Format and Enable

```bash
sudo mkswap /swapfile
sudo swapon /swapfile
```

Verify it's active:

```bash
free -h
```

You should see swap space in the output.

### Make It Persistent Across Reboots

Without this step, swap disappears after every reboot:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify the entry was added correctly:

```bash
tail -1 /etc/fstab
```

Expected output:

```
/swapfile none swap sw 0 0
```

> Double-check `/etc/fstab` after editing. A malformed entry can prevent the server
> from booting. Run `sudo mount -a` to catch errors before rebooting.
{: .prompt-danger }

### Tune Swap Aggressiveness (swappiness)

`swappiness` controls how eagerly the kernel moves data to swap. The default is `60`,
which is tuned for desktops. On a server, a lower value keeps more data in RAM and
only swaps under real pressure:

```bash
# Check current value
cat /proc/sys/vm/swappiness

# Apply immediately (lost on reboot)
sudo sysctl vm.swappiness=10

# Make it permanent
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

A value of `10` works well for most server workloads. Use `1` for databases that are
sensitive to swap latency.

---

## Part 2 — Reducing RAM Usage

### Identify What's Using Memory

```bash
# Sorted by memory usage, top 15 processes
ps aux --sort=-%mem | head -15
```

Or with `htop` — press `F6` and sort by `MEM%`.

### Disable Unused Services

List all running services:

```bash
systemctl list-units --type=service --state=running
```

Disable any service you don't need:

```bash
sudo systemctl disable --now snapd
sudo systemctl disable --now ModemManager
sudo systemctl disable --now avahi-daemon
```

Common candidates on a minimal server:

| Service | Safe to disable if... |
| :--- | :--- |
| `snapd` | You don't use snap packages |
| `avahi-daemon` | You don't need mDNS/Bonjour |
| `ModemManager` | No mobile broadband hardware |
| `bluetooth` | No Bluetooth hardware |
| `cups` | No printer attached |

### Reduce Journald Log Retention

The system journal can grow large over time. Cap it:

```bash
sudo journalctl --vacuum-size=200M
sudo journalctl --vacuum-time=7d
```

Make the limits permanent:

```bash
sudo nano /etc/systemd/journald.conf
```

Set:

```ini
[Journal]
SystemMaxUse=200M
MaxRetentionSec=1week
```

Then restart the journal service:

```bash
sudo systemctl restart systemd-journald
```

---

## Part 3 — Reducing CPU Load

### Find CPU-Heavy Processes

```bash
# One-shot snapshot, sorted by CPU
ps aux --sort=-%cpu | head -15

# Live view
top
```

### Adjust Process Priority with nice

Lower the priority of background tasks so they yield CPU to foreground workloads:

```bash
# Run a new process at low priority
nice -n 19 your-command

# Renice an already-running process (use its PID)
sudo renice -n 10 -p PID
```

`nice` values range from `-20` (highest priority) to `19` (lowest). For background
jobs like backups or compression, `10`–`19` is appropriate.

### Limit a Service's CPU with systemd

For any systemd service, you can cap its CPU share without touching the process itself:

```bash
sudo systemctl edit your-service
```

```ini
[Service]
CPUQuota=20%
```

This limits the service to 20% of one CPU core, regardless of load.

---

## Part 4 — Monitor After Changes

After applying these changes, watch the server's behaviour over a few minutes:

```bash
# Live memory and swap usage
watch -n 2 free -h

# Overall system load
vmstat 2 10

# Check if swap is actually being used
swapon --show
```

If swap usage stays near zero under normal load, your RAM headroom is healthy and swap
is doing its job as a safety net — exactly as intended.

---

## Quick Reference

```bash
# Create and enable 2G swap
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Set swappiness
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p

# Check memory and swap
free -h && swapon --show

# Top memory consumers
ps aux --sort=-%mem | head -10

# Top CPU consumers
ps aux --sort=-%cpu | head -10

# Clean journal logs
sudo journalctl --vacuum-size=200M --vacuum-time=7d
```

---

## Conclusion

Swap space is not a replacement for RAM — it's a safety net that prevents crashes when
memory spikes unexpectedly. Combined with disabling unused services, tuning swappiness,
and capping noisy processes, these changes can significantly extend how long a
low-resource server runs stably without intervention.
