---
title: "How to Keep Your Oracle Cloud Instance Alive Using stress-ng"
description: "A practical guide to creating a systemd service that applies a light load on CPU and RAM to prevent Oracle Cloud from reclaiming your free instance due to inactivity."
date: 2026-08-28 12:00:00 +0200
categories: [Linux, Server]
tags: [oracle-cloud, stress-ng, systemd, linux, vps]
---

## The Problem

Oracle Cloud monitors resource usage on Always Free instances. If your server sits idle
for too long — meaning CPU usage stays near zero — Oracle may flag it for reclamation
or shut it down entirely.

The fix is straightforward: keep the instance visibly active by running a lightweight,
continuous load in the background using `stress-ng`.

## Requirements

- An Ubuntu instance on Oracle Cloud (Always Free or paid)
- `sudo` access

## Step 1: Install stress-ng

`stress-ng` is a tool designed to stress-test system components by generating synthetic
load on the CPU, memory, and other subsystems.

```bash
sudo apt update && sudo apt install stress-ng -y
```

## Step 2: Create a systemd Service

Rather than running the command manually every time, we'll create a `systemd` service
that starts automatically on boot and stays running indefinitely.

```bash
sudo nano /etc/systemd/system/stress-cpu.service
```

Paste the following:

```ini
[Unit]
Description=CPU and RAM Keepalive Service for Oracle
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/stress-ng --cpu 2 --cpu-load 15 --vm 1 --vm-bytes 20%
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Option Breakdown

| Option | Description |
| :--- | :--- |
| `--cpu 2` | Spawn two CPU worker processes |
| `--cpu-load 15` | Target 15% CPU load per worker |
| `--vm 1` | Spawn one memory worker process |
| `--vm-bytes 20%` | Allocate 20% of total RAM |
| `Restart=always` | Automatically restart if the service crashes |
| `RestartSec=3` | Wait 3 seconds before restarting |

> 15% CPU load is enough to register as meaningful activity without affecting the
> performance of anything else running on the server.
{: .prompt-tip }

## Step 3: Enable and Start the Service

Reload `systemd` to pick up the new service file, then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable stress-cpu
sudo systemctl start stress-cpu
```

- `daemon-reload` — tells systemd to re-read its configuration and recognize the new service.
- `enable` — marks the service to start automatically on every boot.
- `start` — starts it immediately without requiring a reboot.

## Step 4: Verify It's Running

```bash
sudo systemctl status stress-cpu
```

You should see output similar to this:

```
● stress-cpu.service - CPU and RAM Keepalive Service for Oracle
     Loaded: loaded (/etc/systemd/system/stress-cpu.service; enabled)
     Active: active (running) since ...
```

> If the status shows `failed` instead of `active (running)`, verify the path to
> `stress-ng` by running `which stress-ng` and make sure it matches the path in
> `ExecStart`.
{: .prompt-warning }

## Monitoring the Load with htop

Once the service is running, open `htop` to confirm the load is being applied:

```bash
htop
```

You should notice:

- CPU bars showing steady activity around 15%
- A slight increase in RAM usage from the `--vm` worker

This confirms the service is working as intended.

## Conclusion

With a single service file and a few commands, your Oracle Cloud instance will maintain
a consistent level of activity that keeps it off Oracle's idle radar. The load is light
enough to have no real impact on your workloads, and persistent enough to register as
genuine resource usage — exactly what's needed to protect an Always Free instance.
