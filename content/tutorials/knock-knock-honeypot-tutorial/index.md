---
title: Deploying a Knock-Knock Honeypot with Caddy HTTPS on Debian
description: Deploy a multi-protocol Knock-Knock honeypot and dashboard on a Debian VPS with Redis, MaxMind GeoIP and Caddy HTTPS.
weight: 1
categories: [tutorial]
tags: [debian, linux, security, honeypot, docker, caddy, self-hosting]
---

## Overview

A walkthrough for standing up a multi-protocol honeypot with a live web dashboard,
automatic TLS, and a hardened Debian VPS host.

**Stack:** Knock-Knock (honeypot + dashboard) · Redis (event store) · MaxMind GeoIP
(geolocation) · Caddy (TLS termination and reverse proxy)

**Target host:** Debian VPS · **Domain:** `soldata.noip.at`

{{% callout note %}}
The domain, SSH hostname, source ID and home IP address in this guide are examples. Replace them with your own values.
{{% /callout %}}

### What you are building

Knock-Knock listens on a set of well-known ports and records every connection attempt
against them. A Python monitor process reads those events, enriches them with ASN and
city data from MaxMind, and pushes aggregate statistics into Redis. A FastAPI web
service renders the dashboard from Redis, and Caddy sits in front of it to terminate
TLS and obtain a Let's Encrypt certificate automatically.

| Service | Role | Binding |
| --- | --- | --- |
| `redis` | Event and stats store | `127.0.0.1:6379` |
| `geoipupdate` | Downloads GeoLite2 ASN + City databases weekly | none |
| `honeypot-monitor` | Listens on honeypot ports, enriches, writes stats | host network |
| `web` | FastAPI dashboard | `127.0.0.1:8080` |
| `caddy` | HTTPS, HTTP to HTTPS redirect, reverse proxy | `0.0.0.0:80`, `0.0.0.0:443` |

All services use `network_mode: host` so that source IP addresses are preserved. A
bridged Docker network would rewrite the source address to the Docker gateway and
destroy the fidelity of the data you are collecting. The trade-off is that every
internal service must bind explicitly to loopback, which the configuration below does.

### Ports in use

| Port | Protocol | Purpose |
| --- | --- | --- |
| 80, 443 | TCP | Caddy (ACME challenge + dashboard) |
| 21 | TCP | FTP honeypot |
| 22 | TCP | SSH honeypot |
| 23 | TCP | Telnet honeypot |
| 25, 587 | TCP | SMTP honeypot |
| 445 | TCP | SMB honeypot |
| 3389 | TCP | RDP honeypot |
| 5060, 5061 | TCP/UDP | SIP honeypot |
| 2222 | TCP | Administrative SSH (moved off 22, restricted by source address) |

The HTTP honeypot is deliberately disabled because Caddy owns ports 80 and 443. Port 22
is reserved for the SSH honeypot, which is why administrative SSH moves to 2222 in
section 1.

## 1. Prerequisites

- A Debian VPS with Docker Engine and the Docker Compose plugin installed.
- Roughly 1 GB of RAM is comfortable; the stack idles well under 512 MB.
- A DNS record for `soldata.noip.at` pointing at the VPS public IPv4 address. If an
  `AAAA` record exists, it must point at a reachable IPv6 address or certificate
  issuance will stall.
- A free [MaxMind GeoLite2 account](https://www.maxmind.com/en/geolite2/signup) for an
  Account ID and license key.

### Move real SSH off port 22 first

A TCP port can only be bound by one process. The SSH honeypot needs port 22, so your
administrative `sshd` must move first. Key-based authentication does not change this —
it is a socket binding conflict, not an authentication one. If `sshd` already holds 22
when the monitor starts, the honeypot's SSH listener cannot bind and collects nothing,
while every other protocol keeps working. The failure is silent, so the dashboard looks
healthy with SSH stuck at zero.

{{% callout warning %}}
Work through the following steps in order. Keep your current SSH session open the whole
time and only close it once a new connection on 2222 has authenticated.
{{% /callout %}}

#### 1.1 Confirm who owns port 22

```bash
ss -tlnp | grep ':22 '
```

If this shows `sshd`, the honeypot is not getting the port. If it already shows `python`,
the swap is done and you can skip to section 2.

#### 1.2 Find where the port is actually configured

Do not assume the main config file is authoritative. On Debian,
`/etc/ssh/sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` near the top, and for
most keywords sshd uses the first value it obtains. A drop-in therefore overrides the
main file. Check the effective value rather than the file contents:

```bash
sudo sshd -T | grep -i '^port'
sudo grep -rn -i '^\s*Port' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/ 2>/dev/null
```

If the grep returns nothing at all, there is no `Port` directive anywhere and
`sshd -T` is reporting the compiled-in default of 22. In that case a `sed` substitution
has nothing to match and will silently do nothing — you must add the directive instead.

Also confirm the port is not coming from a socket unit, which ignores `sshd_config`
entirely:

```bash
systemctl status ssh.socket --no-pager
```

If `ssh.socket` is active, skip to section 1.7.

#### 1.3 Add the directive as a drop-in

The drop-in is the cleanest place on Debian. Write both ports for now so you cannot lock
yourself out:

```bash
sudo tee /etc/ssh/sshd_config.d/10-port.conf >/dev/null <<'EOF'
Port 22
Port 2222
EOF
```

If you prefer to keep everything in the main file, insert above any `Match` block
instead. `sed` with the `i` command inserts rather than replaces, and the first `Port`
value wins:

```bash
sudo sed -i '1i Port 2222' /etc/ssh/sshd_config
```

#### 1.4 Validate before restarting

```bash
sudo sshd -t && echo "config OK"
sudo sshd -T | grep -i '^port'
```

{{% callout warning %}}
That should print both `port 22` and `port 2222`. Never restart on a failed `sshd -t` —
that is how you end up needing the provider console.
{{% /callout %}}

#### 1.5 Restart and prove 2222 works

From a second terminal, while the original session stays open:

```bash
sudo systemctl restart ssh
ss -tlnp | grep -E ':(22|2222) '
ssh -p 2222 root@danbo-soldata
```

Both ports should now show `sshd`. Only continue once the 2222 login succeeds. If your
VPS provider has its own firewall or security group in front of UFW, open 2222 there too
— a UFW rule alone will not help if the provider drops the packet first.

#### 1.6 Drop port 22

Reduce the drop-in to a single line and restart:

```bash
sudo tee /etc/ssh/sshd_config.d/10-port.conf >/dev/null <<'EOF'
Port 2222
EOF

sudo sshd -t && sudo systemctl restart ssh
sshd -T | grep -i '^port'
ss -tlnp | grep -E ':(22|2222) '
```

`sshd -T` should now print only `port 2222`, and the `:22` line should be gone. The
`sshd` pid will have changed, which is your confirmation the restart took effect.

#### 1.7 If `ssh.socket` is the active unit

Socket activation takes the port from the unit, not from `sshd_config`, so a `Port`
directive is ignored. Override the socket instead:

```bash
sudo systemctl edit ssh.socket
```

```ini
[Socket]
ListenStream=
ListenStream=2222
```

The empty `ListenStream=` clears the inherited value before the new one is set.

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
ss -tlnp | grep ':2222'
```

#### 1.8 Open 2222 in the firewall

This step is easy to miss and produces a confusing symptom: SSH works from the console
but not over the network, because UFW is still only allowing 22.

```bash
sudo ufw allow from YOUR.HOME.IP.HERE to any port 2222 proto tcp
sudo ufw status numbered
```

Keep 2222 restricted to your own address. Key-only auth stops password guessing, but an
exposed administrative port still generates scan noise and log volume. Leave 22 open —
the honeypot needs it inbound.

#### 1.9 Hand port 22 to the honeypot

```bash
ss -tlnp | grep ':22 '          # should be empty
docker compose up -d --force-recreate honeypot-monitor
docker compose logs -f honeypot-monitor
```

The monitor retries binding, so it may claim 22 on its own once `sshd` releases it. A
forced recreate makes the handover immediate and deterministic.

#### 1.10 Confirm the swap

```bash
ss -tlnp | grep ':22 '          # python / monitor process
ss -tlnp | grep ':2222'         # sshd
```

SSH knocks should start appearing within minutes. Port 22 is the most-scanned port on
the public internet, so it typically overtakes SMB quickly and becomes the top protocol
on the dashboard.

#### 1.11 Rollback

If anything breaks and you lose SSH, use your provider's web console or rescue mode,
revert the drop-in to `Port 22`, and restart `ssh`:

```bash
sudo tee /etc/ssh/sshd_config.d/10-port.conf >/dev/null <<'EOF'
Port 22
EOF
sudo systemctl restart ssh
```

That is the reason for keeping the original session open until the new one is proven.

## 2. Create the project directory

```bash
mkdir -p ~/knock-knock/data
cd ~/knock-knock
```

## 3. Configure Docker Compose

Save this as `~/knock-knock/docker-compose.yml`.

```yaml
name: knock-knock

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    network_mode: host
    command:
      - redis-server
      - --bind
      - "127.0.0.1"
      - --protected-mode
      - "yes"
      - --appendonly
      - "yes"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    logging: *default-logging

  geoipupdate:
    image: ghcr.io/maxmind/geoipupdate:latest
    restart: unless-stopped
    environment:
      GEOIPUPDATE_ACCOUNT_ID: "${MAXMIND_ACCOUNT_ID:?Set MAXMIND_ACCOUNT_ID in .env}"
      GEOIPUPDATE_LICENSE_KEY: "${MAXMIND_LICENSE_KEY:?Set MAXMIND_LICENSE_KEY in .env}"
      GEOIPUPDATE_EDITION_IDS: "GeoLite2-ASN GeoLite2-City"
      GEOIPUPDATE_FREQUENCY: "168"
    volumes:
      - geoip-data:/usr/share/GeoIP
    logging: *default-logging

  honeypot-monitor:
    image: ghcr.io/djkurlander/knock-knock:latest
    restart: unless-stopped
    network_mode: host
    command: ["python", "-u", "monitor.py"]
    environment:
      REDIS_HOST: "127.0.0.1"
      SOURCE_ID: "soldata"
      # HTTP deliberately omitted: Caddy owns ports 80 and 443.
      ENABLED_PROTOCOLS: "SSH,TNET,FTP,RDP,SMB,SIP,SMTP"
      # Aggregate statistics without storing every individual event.
      SAVE_KNOCKS: "false"
    volumes:
      - ./data:/app/data
      - geoip-data:/usr/share/GeoIP:ro
    depends_on:
      redis:
        condition: service_healthy
      geoipupdate:
        condition: service_started
    logging: *default-logging

  web:
    image: ghcr.io/djkurlander/knock-knock:latest
    restart: unless-stopped
    network_mode: host
    command:
      - python
      - -m
      - uvicorn
      - main:app
      - --host
      - "127.0.0.1"
      - --port
      - "8080"
      - --proxy-headers
      - --forwarded-allow-ips
      - "127.0.0.1"
      - --workers
      - "1"
      - --timeout-keep-alive
      - "30"
      - --log-level
      - warning
    environment:
      REDIS_HOST: "127.0.0.1"
      SOURCE_ID: "soldata"
      ENABLE_SSL: "false"
      LOG_VISITORS: "false"
    volumes:
      - ./data:/app/data
    depends_on:
      redis:
        condition: service_healthy
    logging: *default-logging

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      - web
    logging: *default-logging

volumes:
  redis-data:
  geoip-data:
  caddy-data:
  caddy-config:
```

### Notes on the configuration

- `SAVE_KNOCKS: "false"` keeps aggregate counters only. Set it to `"true"` if you want
  every individual event persisted, at the cost of disk growth.
- `ENABLE_SSL: "false"` on the web service is correct: TLS is terminated by Caddy, and
  the hop from Caddy to uvicorn is plain HTTP over loopback.
- `--forwarded-allow-ips 127.0.0.1` lets uvicorn trust the `X-Forwarded-*` headers Caddy
  adds, so the dashboard logs the real client address rather than the proxy.
- The `x-logging` anchor caps every container's log at 10 MB times 3 files. Without it,
  a noisy honeypot can fill the disk.

## 4. Configure MaxMind credentials

Save as `~/knock-knock/.env` and fill in your MaxMind credentials.

```dotenv
MAXMIND_ACCOUNT_ID=REPLACE_WITH_YOUR_ACCOUNT_ID
MAXMIND_LICENSE_KEY=REPLACE_WITH_YOUR_LICENSE_KEY
```

```bash
chmod 600 .env
```

The `${VAR:?message}` syntax in the compose file makes the stack refuse to start with
a clear error if either value is missing, rather than silently running without GeoIP.

## 5. Configure Caddy

Save as `~/knock-knock/Caddyfile`.

```caddyfile
soldata.noip.at {
	encode zstd gzip

	reverse_proxy 127.0.0.1:8080

	log {
		output stdout
		format console
	}
}
```

That is the whole file. Caddy obtains and renews the certificate for
`soldata.noip.at` on its own, and redirects HTTP to HTTPS automatically.

### Optional: password-protect the dashboard

{{% callout warning %}}
The dashboard has no authentication of its own and is publicly readable by default.
{{% /callout %}}

If you would rather not expose it, generate a bcrypt hash and add a `basic_auth` block.

```bash
docker run --rm -it caddy:2-alpine caddy hash-password
```

Then insert this above the `reverse_proxy` line:

```caddyfile
	basic_auth {
		admin $2a$14$paste.your.generated.hash.here
	}
```

Caddy only accepts hashes here, never plaintext, because the Caddyfile effectively acts
as a password database.

## 6. Validate and start

Validate the Caddyfile before starting anything:

```bash
docker compose run --rm --no-deps caddy \
  caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

A successful run ends with `Valid configuration`. Two warnings are expected and benign:

- `Caddyfile input is not formatted` — cosmetic whitespace only. Clear it with
  `caddy fmt --overwrite` against a writable mount if it bothers you.
- `server is listening only on the HTTPS port but has no TLS connection policies` —
  Caddy reporting that it inserted the automatic HTTPS policy itself.

Start the stack:

```bash
docker compose up -d
docker compose ps
docker compose logs -f caddy
```

Watch the Caddy log for the certificate being issued for `soldata.noip.at`. Then check
the monitor:

```bash
docker compose logs honeypot-monitor
```

You should see `✅ GeoIP databases loaded` followed by
`🚀 Knock-Knock Monitor Active...` once the first GeoIP download completes. The initial
download can take a minute or two.

## 7. Firewall

Using UFW on Debian, allow only what the stack needs and restrict administrative SSH.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Caddy
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Honeypot services
sudo ufw allow 21/tcp
sudo ufw allow 22/tcp
sudo ufw allow 23/tcp
sudo ufw allow 25/tcp
sudo ufw allow 587/tcp
sudo ufw allow 445/tcp
sudo ufw allow 3389/tcp
sudo ufw allow 5060/tcp
sudo ufw allow 5060/udp
sudo ufw allow 5061/tcp
sudo ufw allow 5061/udp

# Administrative SSH, restricted to your own address
sudo ufw allow from YOUR.HOME.IP.HERE to any port 2222 proto tcp

sudo ufw enable
sudo ufw status numbered
```

Replace `YOUR.HOME.IP.HERE` with your actual address. If your ISP hands out dynamic
addresses, consider a VPN or a bastion host instead of an IP allowlist.

{{% callout warning %}}
Do not open `6379` or `8080`. Redis and the dashboard are bound to loopback and are
reachable only through Caddy.
{{% /callout %}}

## 8. Verification checklist

Work through these in order; each one catches a different class of mistake.

```bash
# 1. Certificate is valid and the dashboard answers
curl -sI https://soldata.noip.at | head -1

# 2. HTTP redirects to HTTPS
curl -sI http://soldata.noip.at | grep -i location

# 3. Redis is not reachable from outside
ss -tlnp | grep 6379

# 4. The dashboard is not reachable from outside
ss -tlnp | grep 8080

# 5. Honeypot ports are listening
ss -tlnp | grep -E ':(21|22|23|25|587|445|3389|5060|5061)\b'
```

Checks 3 and 4 should show `127.0.0.1` only. If either shows `0.0.0.0` or `*`, the
service is exposed and you should fix the binding before continuing.

Then confirm the honeypot is actually collecting data:

```bash
docker compose exec redis redis-cli keys '*'
```

Within a few minutes of going live you should see keys appear. A fresh VPS with a
public address typically starts receiving SSH and SMB probes almost immediately.

## 9. Day-to-day operation

```bash
# Follow logs for a single service
docker compose logs -f honeypot-monitor

# Restart after a Caddyfile change (graceful, no dropped connections)
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile

# Pull updated images and recreate
docker compose pull
docker compose up -d

# Stop everything, keeping volumes
docker compose down
```

Redis uses append-only persistence, so counters survive a restart. The `data/`
directory is bind-mounted into the monitor and web containers and holds any state the
application writes outside Redis.

## Troubleshooting

**Certificate issuance stalls.** Almost always DNS or port 80. Confirm the A record
resolves to the VPS, that no `AAAA` record points at an unreachable address, and that
port 80 is open inbound. `docker compose logs caddy` names the ACME failure directly.

**Dashboard loads but shows no data.** Check `docker compose logs honeypot-monitor` for
GeoIP errors. If the MaxMind credentials are wrong, the monitor exits and nothing is
written to Redis.

**`username and password cannot be empty or missing` at startup.** You added a
`basic_auth` block but the hash variable did not resolve. Check for a trailing newline
or a `$` sequence mangled by shell expansion.

**Every login returns 401 with a bcrypt error in the logs.** Hash algorithm mismatch:
an argon2id hash was pasted into a block that defaults to bcrypt. Regenerate with
`caddy hash-password` and do not pass `--algorithm`.

**Source IPs all show as the Docker gateway.** A service lost `network_mode: host`.
This is the single most common way to silently ruin honeypot data.

**Disk filling up.** Either `SAVE_KNOCKS` is enabled and events are accumulating, or
container logs are unbounded. The `x-logging` anchor handles the second case; the first
needs a retention policy on `./data`.

## Security notes

A honeypot is an intentionally attractive target, so treat the host as hostile-facing.

- Keep the real SSH daemon on a non-standard port and restricted by source address.
- Never mount the Docker socket into any of these containers.
- Never bind Redis or the dashboard to a public interface; loopback only.
- Keep the dashboard private with `basic_auth`, a VPN, or a firewall allowlist unless
  you have a specific reason to publish it. It displays attempted credentials, which
  are attacker-supplied but still worth not broadcasting.
- Pin image tags once you have a working stack. `latest` on an internet-facing service
  turns a routine restart into an incident.
- Run `docker compose pull && docker compose up -d` on a schedule you control, not
  automatically, so you can watch for regressions.
- Isolate the honeypot from anything you care about. It should be a standalone VPS with
  no credentials, no shared volumes, and no network path to production systems.
