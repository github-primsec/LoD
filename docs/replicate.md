# Replication Guide

This guide describes how to replicate the hardened single-host reference architecture on a fresh VPS.

## Prerequisites

- A VPS running **openSUSE MicroOS** (or another transactional Linux distribution with SELinux enforcing)
- A registered domain (`your-domain.com`) with DNS A/AAAA records pointing to the VPS
- WireGuard installed on both the VPS and your administrative workstation
- Podman and `podman-compose` (or `docker-compose` plugin) installed on the VPS

## Step 1: Base OS & SELinux Verification

Ensure SELinux is active and the system is up to date.

```bash
sudo transactional-update up
sudo reboot
```

**Verification:**
```bash
getenforce
# Expected output: Enforcing
```

## Step 2: WireGuard Management Zone

Configure WireGuard on the VPS (e.g., `wg0` with `wg-management-ip`, such as `10.66.66.1/24`) and restrict SSH to the WireGuard interface.

**Verification:**
```bash
sudo wg show
# Expected output: active peer handshake from your workstation
```

## Step 3: Firewall Configuration

Configure `firewalld` so only HTTP (`80/TCP`) and HTTPS (`443/TCP`) are publicly accessible. Management ports (SSH, Grafana, Wazuh) must only be reachable via WireGuard.

```bash
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
```

**Verification:**
```bash
sudo firewall-cmd --zone=public --list-services
# Expected output: http https (plus WireGuard UDP port if applicable)
```

## Step 4: Deploy a Hardened Container Service

Use the minimal example in [`examples/hugo-blog/`](../examples/hugo-blog/) to deploy your first hardened container bound to `127.0.0.1:8080`.

```bash
cd examples/hugo-blog
sudo podman compose up -d --build
```

**Verification:**
```bash
sudo podman ps
curl -s http://127.0.0.1:8080/healthz
# Expected output: ok
```

## Step 5: Reverse Proxy & TLS

Configure Nginx on the host to terminate TLS (via Let's Encrypt / `certbot`) for `your-domain.com` and proxy traffic to `127.0.0.1:8080`.

**Verification:**
```bash
sudo nginx -t
curl -I https://your-domain.com
# Expected output: HTTP/2 200 with security headers (CSP, X-Frame-Options, nosniff)
```

## Step 6: Encrypted Backups (`restic`)

Initialize your primary and secondary `restic` repositories using separate `rclone` configurations, then schedule daily backups and `restic check` via systemd timers.

```bash
restic -r rclone:proton:backups/vps_backup init
restic -r rclone:proton:backups/vps_backup backup /etc /opt/services
```

**Verification:**
```bash
restic -r rclone:proton:backups/vps_backup check
# Expected output: no errors were found
```
