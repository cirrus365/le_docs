---
title: Deploying a Hugo Website as a Tor Hidden Service on Debian
description: How to install Tor and Nginx, serve a Hugo website as a .onion service that is not reachable from the regular internet, and automate deployments from a public Git repository with a Bash script and cron.
weight: 2
categories: [tutorial]
tags: [hugo, debian, tor, nginx, git, ci-cd]
---

## Overview

Serve your Hugo site as a `.onion` address on a Debian VPS. Nginx listens only on `127.0.0.1`, and the firewall blocks all web ports, so the site is reachable only through Tor. A cron job pulls from your public Git repo and rebuilds only when new commits exist.

**Prerequisites:** a Debian VPS with SSH access and a public Git repo containing your Hugo site. You don't need a domain name.

## 1. Prepare the server

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ufw
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw enable
```

Don't open ports 80 or 443. Tor makes only outbound connections.

## 2. Install Hugo

```bash
sudo apt install -y ./hugo_extended_*_linux-amd64.deb
hugo version
```

Download the `.deb` from the [Hugo releases page](https://github.com/gohugoio/hugo/releases), or use `sudo apt install hugo` if the packaged version is new enough for your theme.

## 3. Install and configure Nginx

```bash
sudo apt install -y nginx
sudo rm /etc/nginx/sites-enabled/default
```

Removing the default site stops Nginx from listening on public port 80.

Create `/etc/nginx/sites-available/onion`:

```nginx
server {
    listen 127.0.0.1:8080;
    server_name _;
    root /var/www/my-personal-blog/public;
    index index.html;
    server_tokens off;
    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/onion /etc/nginx/sites-enabled/onion
sudo nginx -t && sudo systemctl reload nginx
ss -tlnp | grep nginx   # should show only 127.0.0.1:8080
```

### Why `server_tokens off`

By default Nginx adds its exact version to every response, both in the `Server` header (`nginx/1.22.1`) and in the HTML of its own error pages. Turning tokens off changes that to a plain `nginx`, which helps in two ways:

- **Less fingerprinting.** Anyone hitting the port, or reading an error page, can otherwise tell exactly which Nginx build you run. That makes it trivial to match your server against published vulnerabilities, and it gives an attacker (or a crawler indexing hidden services) a stable signature to recognise your host.
- **Fewer accidental leaks.** Error pages are the usual way version strings escape. A missing file or a mistyped URL is enough to hand the version to a visitor, and a screenshot of a 404 can end up posted somewhere public.

It is not a security control on its own — it only hides a version string, and a determined attacker can still fingerprint Nginx from behaviour — but it removes information you gain nothing by advertising. On a hidden service that is the whole point: publish as little as possible about the machine behind the `.onion` address.

## 4. Install Tor and create the hidden service

```bash
sudo apt install -y tor
sudo nano /etc/tor/torrc
```

Add at the bottom:

```text
HiddenServiceDir /var/lib/tor/hugo_site/
HiddenServicePort 80 127.0.0.1:8080
```

```bash
sudo systemctl restart tor@default
sudo cat /var/lib/tor/hugo_site/hostname
```

The last command prints your `.onion` address.

### Back up the hidden service key

The `.onion` address is derived entirely from the key in `/var/lib/tor/hugo_site/hs_ed25519_secret_key`. There is no account or registrar behind it, so that file is the only thing tying the address to you. Lose it and the address is gone for good, since regenerating it would mean brute-forcing a 256-bit key. Anyone who copies it can stand up an identical service and impersonate you, so treat it like a password rather than a config file. This applies whether you keep the random address Tor just generated or replace it with a vanity one later.

Take the backup from the server and keep it offline:

```bash
sudo tar -C /var/lib/tor -czf hugo_site-keys.tar.gz hugo_site
sudo chown $USER hugo_site-keys.tar.gz
```

Copy `hugo_site-keys.tar.gz` off the VPS to somewhere that is not the server itself — an encrypted USB stick or a password manager that stores files both work. The `hostname` and `hs_ed25519_public_key` files travel with it, but the secret key is the one that matters; the other two can be re-derived. Delete the tarball from the server and from any machine you staged it on afterwards, and never commit it to the Git repo.

Restoring onto a new server is straightforward. Install Tor, stop it, unpack the archive into place and fix the ownership before starting it again:

```bash
sudo systemctl stop tor@default
sudo rm -rf /var/lib/tor/hugo_site
sudo tar -C /var/lib/tor -xzf hugo_site-keys.tar.gz
sudo chown -R debian-tor:debian-tor /var/lib/tor/hugo_site
sudo chmod 700 /var/lib/tor/hugo_site
sudo systemctl start tor@default
```

Point `HiddenServiceDir` at `/var/lib/tor/hugo_site/` in the new server's `torrc` and the address comes back exactly as before. Only run one copy at a time — publishing the same key from two machines splits the service and makes it unreliable — so shut the old server's Tor down before starting the new one.

## 5. Clone your site

```bash
sudo mkdir -p /var/www/my-personal-blog
sudo chown $USER:$USER /var/www/my-personal-blog
git clone --recurse-submodules https://github.com/YOUR_USER/YOUR_REPO.git /var/www/my-personal-blog
cd /var/www/my-personal-blog && hugo --gc --cleanDestinationDir --baseURL "/" --relativeURLs
```

Relative URLs keep links working on the `.onion` address without hardcoding it.

## 6. Deploy script

Create `/usr/local/bin/deploy-onion.sh`:

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
hugo --gc --cleanDestinationDir --baseURL "/" --relativeURLs
echo "$(date '+%F %T') Deploy complete."
```

```bash
sudo chmod +x /usr/local/bin/deploy-onion.sh
/usr/local/bin/deploy-onion.sh
```

## 7. Schedule with cron

Run `crontab -e` as the user that owns the repo:

```cron
*/5 * * * * /usr/bin/flock -n /tmp/deploy-onion.lock /usr/local/bin/deploy-onion.sh >> $HOME/deploy-onion.log 2>&1
```

## 8. Test

1. Open the `.onion` address in [Tor Browser](https://www.torproject.org/download/). The site should load.
2. From a normal browser or machine, visit `http://YOUR_VPS_IP`. It should time out or refuse the connection.
3. Push a commit, then run `tail -f ~/deploy-onion.log`. Within 5 minutes you should see "Changes detected, deploying...".

## 9. Optional: generate a vanity `.onion` address

A normal v3 address is 56 random characters. A vanity address lets you choose a short prefix, such as `myblog`, so the address is recognisable when someone reads it out or pastes it. The rest stays random, and the prefix is only cosmetic — it does not make the address more trustworthy, since anyone can grind the same prefix. Keep it short: each extra character multiplies the work by about 32. Five or six characters is usually minutes to an hour; seven or more can take days. Only letters `a`–`z` and digits `2`–`7` are valid, because onion addresses are base32 and cannot contain `0`, `1`, `8` or `9`.

The standard tool is [mkp224o](https://github.com/cathugger/mkp224o). Generate the keys on your own machine rather than the VPS, so the search doesn't compete with the web server, then copy only the finished key across. The build dependencies and the faster AMD64 implementation come from the project's own instructions:

```bash
sudo apt install -y gcc libc6-dev libsodium-dev make autoconf git
git clone https://github.com/cathugger/mkp224o.git
cd mkp224o
./autogen.sh
./configure --enable-amd64-51-30k
make
./mkp224o -B -s -d ./keys -n 1 myblog
```

`-n 1` stops after the first match, `-s` prints progress, and `-d` keeps the results out of the current directory. When it finishes you get a folder named after the address containing `hostname`, `hs_ed25519_public_key` and `hs_ed25519_secret_key`. Copy that folder to the server and hand it to Tor, replacing the key Tor generated in step 4:

```bash
scp -r ./keys/myblog....onion user@vps:~/
# on the server, as root:
sudo systemctl stop tor@default
sudo rm -rf /var/lib/tor/hugo_site
sudo mv /home/user/myblog....onion /var/lib/tor/hugo_site
sudo chown -R debian-tor:debian-tor /var/lib/tor/hugo_site
sudo chmod 700 /var/lib/tor/hugo_site
sudo systemctl start tor@default
sudo cat /var/lib/tor/hugo_site/hostname
```

The ownership matters: on Debian the Tor daemon runs as `debian-tor`, and it refuses to load a hidden service directory it does not own. No change to `torrc` is needed, since `HiddenServiceDir` still points at the same path. Once the new key is in place, take a fresh backup using the steps in section 4 and delete the key copies from your home directory and from the machine you generated them on.

## 10. What this setup does and does not route through Tor

It is worth being precise about this, because it is the most common misconception about running a hidden service. The `HiddenServicePort` line only handles **inbound** traffic. Visitors connect to your `.onion` address through the Tor network, and the Tor daemon hands those connections to Nginx on `127.0.0.1:8080`. From the visitor's side the connection is end-to-end encrypted inside Tor, and they learn nothing about your server's IP address.

Nothing the server initiates itself goes through Tor. `git fetch`, `apt`, DNS lookups and NTP all leave the machine straight over your VPS provider's network, exactly as they would on any other server. So the honest summary is: visitors are protected, and the server is not hidden. Your provider can see that the machine exists, who pays for it, and every clearnet host it talks to — including GitHub, every five minutes, when cron runs the deploy script.

For a personal blog built from a public repository that is a reasonable and deliberate trade-off, not an oversight:

- **The content is public anyway.** The moment you publish, the posts are world-readable. Hiding the fact that your server fetched them does not conceal anything that was not already published.
- **Clearnet Git hosts are awkward over Tor.** Large forges rate-limit or block traffic from Tor exit nodes because of abuse, so routing `git pull` through Tor tends to mean intermittent deploy failures rather than better privacy.
- **It would not hide you from the adversary who already knows.** Your VPS provider knows the server's IP regardless of where its outbound traffic goes. Torifying outbound traffic only hides the *destinations* from the provider, which matters when those destinations are sensitive, not when they are a public Git repository.

Forcing everything through Tor — transparent proxying, or a firewall that drops any packet not leaving via Tor — is a much larger project than this tutorial, it is easy to get subtly wrong (a single DNS lookup made directly undoes it), and for a static site it mostly buys outages. If you do want to go further, the sensible middle step is to route only Git through the Tor SOCKS port rather than the whole system.

### Why you might want a different Git forge

Where the choice of forge starts to matter is when the repository is the sensitive part rather than the published site. Every `git pull` in this setup tells your VPS provider, and anyone watching its uplink, exactly which forge and which repository the server tracks. On top of that, the forge itself holds the account the commits come from, the IP addresses you push from, and the link between your identity and the content. Three situations make that uncomfortable:

- **You want the source private.** A private repository on a large forge still exists on someone else's servers, tied to an account with your email and billing details. Moving it to a forge you control removes that third party entirely.
- **You do not want your identity attached.** If the blog is anonymous, an account on a forge that demands an email address, a phone number or a payment card is the weak link, not the hidden service.
- **You do not trust the forge to keep hosting it.** A commercial forge can remove a repository, lock an account, or be compelled to hand over logs. That is a real risk for anything politically sensitive, and it is a milder annoyance for everything else when a company's priorities change.

Two options cover most of this. [Codeberg](https://codeberg.org/) is a hosted forge run by a German non-profit rather than a commercial company, so it is not building an advertising profile and it is not beholden to a parent company's terms. It is still someone else's server, and your account and push IPs are still visible to its operators, but the incentives are very different from a venture-funded platform. [Forgejo](https://forgejo.org/), which is the software Codeberg runs, is the stronger option: it is lightweight enough to run on a small VPS, and you can publish it as its own hidden service so the repository is only reachable over Tor at all. At that point the deploy script pulls from a `.onion` address through the local Tor SOCKS port, and no clearnet host ever sees the server's Git traffic. The cost is that you are now maintaining a second service, and you are responsible for its backups and uptime.

Which you pick depends on what you are protecting. For a public personal blog, GitHub or Codeberg over the clearnet is fine and this tutorial's setup needs no changes. If the repository should not be linkable to you, a self-hosted Forgejo instance reachable only over Tor is the option that actually delivers that, and it is the one case where routing the server's Git traffic through Tor is genuinely worth doing.

## Privacy notes

- Make sure your theme loads no external resources (CDN fonts, analytics, embeds). They leak visitor requests outside Tor.
- Keep `server_tokens off` and don't put your real name or hostname in site metadata if you want anonymity.
- Check logs with `sudo journalctl -u tor@default` if the `.onion` address doesn't come up.