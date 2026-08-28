---
title: "Nginx: Static Sites, Reverse Proxy, and HTTPS with Certbot"
description: "A technical guide to setting up Nginx for serving static websites and reverse proxying applications, secured with a free SSL certificate via Let's Encrypt and Certbot."
date: 2026-08-28 12:00:00 +0200
categories: [Self-Hosting, Networking]
tags: [nginx, linux, reverse-proxy, certbot, ssl, let's-encrypt, self-hosting]
---

## Install Nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable --now nginx
```

Verify it's running:

```bash
sudo systemctl status nginx
```

Open `http://YOUR_SERVER_IP` — you should see the default Nginx welcome page.

---

## Core Concepts

Nginx reads configuration from two directories:

| Path | Purpose |
| :--- | :--- |
| `/etc/nginx/sites-available/` | Where you define server blocks (one file per site) |
| `/etc/nginx/sites-enabled/` | Symlinks to active configs |
| `/etc/nginx/nginx.conf` | Global settings — rarely needs editing |

The workflow is always: write config in `sites-available` → symlink to `sites-enabled` → test → reload.

---

## Method 1 — Static Website

### 1. Place Your Files

```bash
sudo mkdir -p /var/www/example.com
sudo chown -R $USER:$USER /var/www/example.com
```

Drop your `index.html` and assets into `/var/www/example.com`.

### 2. Create the Server Block

```bash
sudo nano /etc/nginx/sites-available/example.com
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 3. Enable and Reload

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> Always run `sudo nginx -t` before reloading. It catches syntax errors without
> touching the live server.
{: .prompt-tip }

---

## Method 2 — Reverse Proxy

Use this when your application (Node.js, Python, Docker container, etc.) is already
listening on a local port and you want Nginx to sit in front of it.

### 1. Create the Server Block

```bash
sudo nano /etc/nginx/sites-available/app.example.com
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Replace `127.0.0.1:3000` with the actual address your app listens on.

### 2. Enable and Reload

```bash
sudo ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> The `Upgrade` and `Connection` headers are needed for WebSocket support. They're
> harmless on non-WebSocket apps, so it's fine to include them by default.
{: .prompt-info }

---

## HTTPS with Let's Encrypt (Certbot)

### 1. Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 2. Obtain and Install the Certificate

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot will:
1. Verify domain ownership via HTTP challenge
2. Obtain the certificate from Let's Encrypt
3. Automatically modify your Nginx config to add HTTPS and redirect HTTP → HTTPS

For the reverse proxy domain:

```bash
sudo certbot --nginx -d app.example.com
```

### 3. Verify Auto-Renewal

Certbot installs a systemd timer that renews certificates automatically before they expire:

```bash
sudo systemctl status certbot.timer
```

Test the renewal process without actually renewing:

```bash
sudo certbot renew --dry-run
```

> Let's Encrypt certificates expire after 90 days. Certbot renews them automatically
> when they have less than 30 days remaining. No manual action needed.
{: .prompt-tip }

---

## What Certbot Adds to Your Config

After running Certbot, your server block will look roughly like this:

```nginx
server {
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

---

## Useful Commands

```bash
# Test configuration syntax
sudo nginx -t

# Reload without downtime
sudo systemctl reload nginx

# Full restart
sudo systemctl restart nginx

# View error logs
sudo tail -f /var/log/nginx/error.log

# View access logs
sudo tail -f /var/log/nginx/access.log

# List active sites
ls -la /etc/nginx/sites-enabled/

# Remove a site
sudo rm /etc/nginx/sites-enabled/example.com
sudo systemctl reload nginx
```

---

## Common Errors

- **`bind() to 0.0.0.0:80 failed`** — another process (Apache, AdGuard, etc.) is
  already using port 80. Find it with `sudo ss -tlnp | grep :80` and stop it.
- **`502 Bad Gateway`** — Nginx can't reach the upstream app. Make sure your application
  is running and listening on the port defined in `proxy_pass`.
- **`could not build server_names_hash`** — your `server_name` value is too long.
  Add `server_names_hash_bucket_size 64;` inside the `http {}` block in
  `/etc/nginx/nginx.conf`.
- **Certificate not renewing** — check `sudo certbot renew --dry-run` and ensure
  port 80 is open and not blocked by a firewall.
