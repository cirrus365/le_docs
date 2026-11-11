---
title: Deploying a Hugo Website on Debian VPS with Caddy
description: How to install and configure Caddy, build and serve a Hugo website, and automate deployments from a public Git repository with a Bash script and cron.
weight: 1
categories: [tutorial]
tags: [hugo, debian, caddy, git, ci-cd]
---

## Overview

Hugo + Caddy auto-deploy on a Debian VPS.

Prerequisites: a Debian VPS with SSH access, a public Git repo containing your Hugo site, and a free hostname from noip.at (e.g. mywebsite.noip.at) pointing to your VPS's public IP. Ports 80 and 443 must be open.

## 1. Prepare the Server

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl debian-keyring debian-archive-keyring apt-transport-https gnupg ufw
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw enable
```

## 2. Install Caddy (Official Repo)

```shell
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

## 3. Install Hugo

`apt install hugo` works but is often outdated. If your theme needs a newer or extended version, download the `hugo_extended_*_linux-amd64.deb` from the Hugo releases page:

```shell
sudo apt install -y ./hugo_extended_*_linux-amd64.deb
hugo version
```

## 4. Clone Your Site

Run this as your normal (non-root) user:

```shell
sudo mkdir -p /var/www/my-personal-blog
sudo chown $USER:$USER /var/www/my-personal-blog
git clone --recurse-submodules https://github.com/YOUR_USER/YOUR_REPO.git /var/www/my-personal-blog
cd /var/www/my-personal-blog && hugo --gc --cleanDestinationDir
```

## 5. Configure Caddy

Edit `/etc/caddy/Caddyfile` with `sudo nano /etc/caddy/Caddyfile`:

```caddyfile
mywebsite.noip.at {
	root * /var/www/my-personal-blog/public
	encode zstd gzip
	file_server
}
```

```shell
sudo systemctl reload caddy
```

Caddy gets and renews the HTTPS certificate automatically. Make sure the `caddy` user can read the files: `chmod o+x /var/www` and keep the files world-readable (the defaults usually work).

## 6. Deploy Script

Create `/usr/local/bin/deploy-blog.sh`. It exits early if the remote has no new commits:

```sh
#!/bin/sh
set -e
cd /var/www/my-personal-blog

# Skip if nothing changed (but still build if public/ doesn't exist yet)
git fetch --quiet origin
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git rev-parse '@{u}')
if [ "$LOCAL" = "$REMOTE" ] && [ -d public ]; then
    echo "$(date '+%F %T') No changes, skipping."
    exit 0
fi

echo "$(date '+%F %T') Changes detected, deploying..."
git pull --ff-only
git submodule update --init --recursive --depth 1
hugo --gc --cleanDestinationDir
echo "$(date '+%F %T') Deploy complete."
```

```shell
sudo chmod +x /usr/local/bin/deploy-blog.sh
/usr/local/bin/deploy-blog.sh   # test run
```

## 7. Schedule with Cron

Run `crontab -e` as the same user that owns the repo:

```shell
*/5 * * * * /usr/bin/flock -n /tmp/deploy-blog.lock /usr/local/bin/deploy-blog.sh >> $HOME/deploy-blog.log 2>&1
```

`flock` prevents overlapping runs. Every 5 minutes, cron checks the repo and only rebuilds when you've pushed new commits.

## 8. Test the Pipeline

Edit a post locally, then `git commit` and `git push`.

Within 5 minutes, `tail -f ~/deploy-blog.log` should show "Changes detected, deploying...".

Visit `https://mywebsite.noip.at` to see the update.

## Troubleshooting

Check Caddy with:

```shell
sudo journalctl -u caddy -f
```

If certificate issuance fails, check that your noip.at DNS record points to the VPS IP and that ports 80 and 443 are open.

`git pull --ff-only` fails if the server's repo has local changes or a diverged history. Never edit files on the server.
