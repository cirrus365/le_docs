---
title: Deploying a Hugo Website as an I2P Eepsite with i2pd on Debian
description: How to serve a Hugo website as an I2P eepsite using i2pd and Nginx so it is not reachable from the regular internet, and automate deployments from a public Git repository with a Bash script and cron.
weight: 3
categories: [tutorial]
tags: [hugo, debian, i2p, i2pd, nginx, git, ci-cd]
---

## Overview

Serve your Hugo site as an I2P eepsite (a `.b32.i2p` address) on a Debian VPS. Nginx listens only on `127.0.0.1`, and the firewall blocks all web ports, so visitors can reach the site only through I2P. A cron job pulls from your public Git repo and rebuilds only when it finds new commits.

**Prerequisites:** a Debian VPS with SSH access, a public Git repo containing your Hugo site, and **i2pd already installed and running**. If you don't have i2pd yet, follow the installation guide on the [official i2pd website](https://i2pd.website/) first, then come back here.

## What is I2P, and why use it instead of Tor?

[I2P](https://geti2p.net/) (the Invisible Internet Project) is an anonymous overlay network, like Tor. The difference is what each one is built for. Tor is mostly a way to reach the regular internet anonymously, and hidden services were added later. I2P is built for services that live *inside* the network, and every participant also routes traffic for others. i2pd is a lightweight C++ implementation of the I2P router.

Reasons you might pick it over Tor:

- **No central directory.** Tor depends on a small set of directory authorities to publish its list of relays. I2P uses a distributed network database, so no single group of servers runs the network.
- **One-way tunnels.** I2P sends traffic out and back through separate tunnels. An observer sees only half of each conversation, which makes traffic correlation harder.
- **Built for internal services.** Since I2P isn't mainly used to reach the clearnet, its users and traffic are centred on in-network sites like yours.
- **Reaching a different audience.** Some people use I2P and not Tor. Running both is a reasonable way to offer your blog to both communities.

The trade-offs: the network is smaller, visitors need an I2P router running (Tor Browser won't work), and a new router can take several minutes to become well connected before it performs well.

## 1. Prepare the server

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ufw
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw enable
```

Don't open ports 80 or 443. You can optionally allow i2pd's router port (the `port =` value in `/etc/i2pd/i2pd.conf`) for TCP and UDP. The router works without it, but it integrates better with the network when peers can reach it directly. That port carries encrypted I2P traffic only and never exposes your website.

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

Create `/etc/nginx/sites-available/eepsite`:

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
sudo ln -s /etc/nginx/sites-available/eepsite /etc/nginx/sites-enabled/eepsite
sudo nginx -t && sudo systemctl reload nginx
ss -tlnp | grep nginx   # should show only 127.0.0.1:8080
```

### Why `server_tokens off`

By default Nginx puts its exact version in the `Server` header (`nginx/1.22.1`) and in its error pages. With tokens off you get a plain `nginx` instead. This makes your server harder to fingerprint and harder to match against known vulnerabilities, and it stops a version string leaking through a 404 page. It doesn't secure anything by itself, but on an anonymous service the goal is to reveal as little about the machine as possible.

## 4. Create the i2pd server tunnel

Open the tunnels config:

```bash
sudo nano /etc/i2pd/tunnels.conf
```

Add:

```ini
[my-blog]
type = http
host = 127.0.0.1
port = 8080
inport = 80
keys = my-blog.dat
```

- `type = http` creates an HTTP server tunnel that forwards visitors to Nginx.
- `inport = 80` lets visitors open the address without adding a port number.
- `keys = my-blog.dat` is the site's private key. i2pd generates it on first start and stores it in its data directory, `/var/lib/i2pd/` on Debian.

```bash
sudo systemctl restart i2pd
```

Get your `.b32.i2p` address from the i2pd web console:

```bash
curl -s http://127.0.0.1:7070/?page=i2p_tunnels | grep -o '[a-z2-7]\{52\}\.b32\.i2p'
```

You can also open the console in a browser through an SSH tunnel (`ssh -L 7070:127.0.0.1:7070 user@vps`, then visit `http://127.0.0.1:7070` and check **I2P tunnels → Server tunnels**). A new destination can take a few minutes to publish before anyone can reach it.

### Back up the site key

Your address comes entirely from `/var/lib/i2pd/my-blog.dat`. If you lose that file, you lose the address for good. If someone copies it, they can impersonate your site. Treat it like a password:

```bash
sudo cp /var/lib/i2pd/my-blog.dat ~/my-blog.dat
sudo chown $USER ~/my-blog.dat
```

Copy it off the VPS to offline storage, such as an encrypted USB stick or a password manager that stores files. Then delete the copy in your home directory, and never commit it to Git.

To restore on a new server, stop i2pd, put the file back and fix ownership:

```bash
sudo systemctl stop i2pd
sudo cp my-blog.dat /var/lib/i2pd/my-blog.dat
sudo chown i2pd:i2pd /var/lib/i2pd/my-blog.dat
sudo chmod 600 /var/lib/i2pd/my-blog.dat
sudo systemctl start i2pd
```

Use the same `[my-blog]` tunnel block on the new server, and shut down i2pd on the old server before you switch over.

## 5. Clone your site

```bash
sudo mkdir -p /var/www/my-personal-blog
sudo chown $USER:$USER /var/www/my-personal-blog
git clone --recurse-submodules https://github.com/YOUR_USER/YOUR_REPO.git /var/www/my-personal-blog
cd /var/www/my-personal-blog && hugo --gc --cleanDestinationDir --baseURL "/" --relativeURLs
```

Relative URLs keep links working on the `.b32.i2p` address without hardcoding it.

## 6. Deploy script

Create `/usr/local/bin/deploy-eepsite.sh`:

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
sudo chmod +x /usr/local/bin/deploy-eepsite.sh
/usr/local/bin/deploy-eepsite.sh
```

## 7. Schedule with cron

Run `crontab -e` as the user that owns the repo:

```cron
*/5 * * * * /usr/bin/flock -n /tmp/deploy-eepsite.lock /usr/local/bin/deploy-eepsite.sh >> $HOME/deploy-eepsite.log 2>&1
```

## 8. Test

1. On your own machine, run an I2P router (i2pd or Java I2P) and set your browser's HTTP proxy to `127.0.0.1:4444`. Open `http://YOUR_ADDRESS.b32.i2p`. The site should load. Give it a few minutes if the router just started.
2. From a normal browser, visit `http://YOUR_VPS_IP`. It should time out or refuse the connection.
3. Push a commit, then run `tail -f ~/deploy-eepsite.log`. Within 5 minutes you should see "Changes detected, deploying...".

## 9. Optional: generate a vanity `.b32.i2p` address

A `.b32.i2p` address is 52 random characters. A vanity address lets you pick a short prefix such as `myblog`. The rest stays random, and anyone can generate the same prefix, so it doesn't make the address more trustworthy. Valid characters are `a`–`z` and `2`–`7`.

The tool is `vain` from [i2pd-tools](https://github.com/PurpleI2P/i2pd-tools). Build it on your own machine, not the VPS:

```bash
sudo apt install -y git build-essential libboost-chrono-dev libboost-date-time-dev \
  libboost-filesystem-dev libboost-program-options-dev libssl-dev zlib1g-dev
git clone --recursive https://github.com/PurpleI2P/i2pd-tools.git
cd i2pd-tools
make
./vain --help
```

Check `./vain --help` for the exact flags in your version, then generate a key with your prefix and write it to `my-blog.dat`. The project's own estimate on a 2.7 GHz CPU is about 6 seconds for 5 characters, a few minutes for 6, and around 30 minutes for 7. Each extra character makes it far slower. Confirm the address with:

```bash
./keyinfo my-blog.dat
```

Copy it to the server, replacing the key i2pd generated:

```bash
scp my-blog.dat user@vps:~/
# on the server:
sudo systemctl stop i2pd
sudo mv ~/my-blog.dat /var/lib/i2pd/my-blog.dat
sudo chown i2pd:i2pd /var/lib/i2pd/my-blog.dat
sudo chmod 600 /var/lib/i2pd/my-blog.dat
sudo systemctl start i2pd
```

You don't need to change `tunnels.conf`. Take a fresh backup (section 4) and delete any leftover copies of the key.

I2P also supports human-readable `.i2p` names such as `myblog.i2p`, registered with address-book services like `reg.i2p`. The `regaddr` tool in i2pd-tools generates the authentication string these services require.

## 10. What this setup does and does not route through I2P

The server tunnel handles **inbound** traffic only. Visitors reach your site through I2P, and i2pd passes their connections to Nginx on `127.0.0.1:8080`. Visitors learn nothing about your server's IP address.

Anything the server starts itself — `git fetch`, `apt`, DNS lookups and NTP — goes out directly over your VPS provider's network. Visitors are protected, but the server is not hidden. Your provider can see the machine, who pays for it, and every clearnet host it contacts, including GitHub every five minutes.

For a blog built from a public repository, that's a deliberate trade-off:

- **The content is public anyway.** Once you publish a post, hiding the fact that your server fetched it hides nothing.
- **I2P isn't built to reach the clearnet.** There are only a handful of outproxies, and they are slow and unreliable. Sending `git pull` through them would mostly cause failed deploys.
- **Your provider already knows the server.** Hiding outbound traffic only hides *where* it goes, which matters when those destinations are sensitive. A public Git repo isn't.

### Why you might want a different Git forge

The choice of forge matters when the repository is more sensitive than the published site. Every `git pull` shows your provider which forge and repository the server tracks. The forge also holds your account, the IP addresses you push from, and the link between you and the content. That's a problem if:

- **You want the source private.** A private repo on a big forge still sits on someone else's servers, tied to your email and billing details.
- **The blog is anonymous.** A forge account that needs an email, phone number or payment card becomes the weak link.
- **You don't trust the forge to keep hosting it.** A commercial forge can remove repos, lock accounts, or be compelled to hand over logs.

[Codeberg](https://codeberg.org/) is a hosted forge run by a non-profit rather than a commercial company. That's a better fit, but it's still someone else's server. [Forgejo](https://forgejo.org/), the software Codeberg runs, is the stronger option. You can self-host it and publish it as its own I2P server tunnel, so the repository is reachable only inside I2P. The deploy script then pulls through i2pd's local HTTP proxy:

```bash
git -c http.proxy=http://127.0.0.1:4444 fetch origin
```

With that setup, no clearnet host ever sees the server's Git traffic. The cost is running a second service and taking care of its backups and uptime. For a public personal blog, GitHub or Codeberg over the clearnet is fine and this setup needs no changes.

## Privacy notes

- Make sure your theme loads no external resources (CDN fonts, analytics, embeds). In most I2P setups those requests fail, and where an outproxy is used they leak outside I2P.
- Keep `server_tokens off`, and leave your real name and hostname out of site metadata if you want anonymity.
- If the address doesn't come up, check `sudo journalctl -u i2pd` and the i2pd web console's tunnel status.