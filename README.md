# ansible-easy-vpn

Ansible playbook that sets up a hardened WireGuard VPN server with a web UI, protected by Authelia 2FA and BunkerWeb WAF, on a single VPS.

## What gets deployed

| Container | Purpose |
|-----------|---------|
| [wg-easy](https://github.com/wg-easy/wg-easy) | WireGuard VPN with web UI |
| [Authelia](https://www.authelia.com) | Single sign-on with TOTP 2FA |
| [BunkerWeb](https://www.bunkerweb.io) + scheduler | Reverse proxy and WAF, Let's Encrypt TLS |
| [Redis](https://redis.io) | Session storage for Authelia |
| [Watchtower](https://containrrr.dev/watchtower) | Automatic Docker image updates |
| [Fail2Ban](https://github.com/fail2ban/fail2ban) | SSH intrusion prevention (optional) |
| AdGuard Home + Unbound + dnscrypt-proxy | DNS filtering with DoH (optional) |

## Requirements

- A fresh VPS running **Debian 11/12/13** or **Ubuntu 20.04+**
- Root or sudo access
- A domain name with DNS A-records pointing to your server's IP:
  - `yourdomain.com`
  - `wg.yourdomain.com`
  - `auth.yourdomain.com`
  - `adguard.yourdomain.com` *(only if enabling AdGuard)*
- Open ports: **80/tcp**, **443/tcp**, **51820/udp** (and your SSH port)

## Quick start

Run the following command as root (or a user with sudo):

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/vraz0r/ansible-easy-vpn/main/bootstrap.sh)
```

The script will install dependencies, prompt you for configuration, generate SSL certificates, and run the playbook.

## Manual setup

```bash
git clone https://github.com/vraz0r/ansible-easy-vpn ~/ansible-easy-vpn
cd ~/ansible-easy-vpn
bash bootstrap.sh
```

## Re-running after changes

To apply configuration changes or update containers:

```bash
cd ~/ansible-easy-vpn
bash bootstrap.sh
```

If `custom.yml` is already filled out, the script will skip the prompts and run the playbook directly.

## Configuration

### custom.yml

Settings that can be overridden in `custom.yml` (never edit `inventory.yml` directly):

| Variable | Default | Description |
|----------|---------|-------------|
| `username` | — | Non-root user to create |
| `root_host` | — | Base domain name |
| `ssh_port` | `22` | SSH port |
| `wireguard_port` | `51820` | WireGuard UDP port |
| `dns_nameservers` | `cloudflare` | Upstream DNS: `cloudflare`, `quad9`, `google` |
| `enable_adguard_unbound_doh` | `false` | Enable AdGuard + Unbound + DoH stack |
| `enable_fail2ban` | `true` | Enable Fail2Ban for SSH |
| `autoupdate_reboot_time` | `03:00` | Time for automatic reboot after updates (24h) |
| `docker_dir` | `/opt/docker` | Persistent storage path for containers |
| `wireguard_subnet` | `10.8.0.x` | WireGuard client subnet |
| `wireguard_client_allowed_ips` | `0.0.0.0/0, ::/0` | Allowed IPs pushed to VPN clients |

To change a setting after initial setup:

```bash
cd ~/ansible-easy-vpn
nano custom.yml
bash bootstrap.sh
```

### secret.yml

Encrypted with `ansible-vault`. Contains passwords and generated secrets.

```bash
cd ~/ansible-easy-vpn
ansible-vault edit secret.yml
```

## Accessing the services

After a successful run you will find:

- **WireGuard UI** — `https://wg.yourdomain.com`
- **Authelia** — `https://auth.yourdomain.com`
- **AdGuard** — `https://adguard.yourdomain.com` *(if enabled)*

All web services are protected by Authelia 2FA. On first login, scan the QR code with a TOTP app (Google Authenticator, Aegis, etc.).

If you did not configure SMTP, retrieve the 2FA setup code by logging in via SSH and running:

```bash
show_2fa
```

## Troubleshooting

### Can connect to VPN but can't access the Internet

The WireGuard port (51820/udp) is likely blocked — either by your VPS provider's firewall or your ISP. Check the provider's control panel, or change the port:

```bash
cd ~/ansible-easy-vpn
echo 'wireguard_port: "12345"' >> custom.yml
bash bootstrap.sh
```

### "Secure connection failed" in the browser

Let's Encrypt failed to issue certificates. Common causes:
- Ports 80/443 are not open
- DNS records don't point to this server yet

Check BunkerWeb and scheduler logs:

```bash
docker logs bunkerweb
docker logs bw-scheduler
```

### "500 Internal Server Error" on the Authelia page

Wrong SMTP credentials or the SMTP port (465) is blocked by your provider. Check Authelia logs:

```bash
docker logs authelia
```

To disable email notifications, remove `email_password` from `secret.yml` and re-run.

### Lost 2FA device

```bash
docker stop authelia && docker rm authelia
sudo rm -rf /opt/docker/authelia
cd ~/ansible-easy-vpn
bash bootstrap.sh
```

### SSH key copy on Windows

```bash
cd ~
scp -P 22 root@YOUR_SERVER_IP:/tmp/id_ssh_ed25519 .ssh/id_vpn_username
ssh -p 22 username@YOUR_SERVER_IP -i .ssh/id_vpn_username
```

## Uninstalling

```bash
docker stop authelia wg-easy adguard-unbound-doh watchtower bunkerweb bw-scheduler redis
docker rm authelia wg-easy adguard-unbound-doh watchtower bunkerweb bw-scheduler redis
sudo rm -rf /opt/docker
docker system prune -a
```

SSH hardening, the non-root user, and unattended upgrades configuration will remain in place.
