---
title: Hosting a Website on the Yggdrasil Network with Debian and Nginx
description: What the Yggdrasil network is, why you might use it, and how to install Yggdrasil on a Debian VPS, bind Nginx to your Yggdrasil address, and block clearnet access with a firewall.
weight: 5
categories: [tutorial]
tags: [yggdrasil, debian, nginx, networking, privacy, ipv6, ufw]
---

## What is Yggdrasil?

[Yggdrasil](https://yggdrasil-network.github.io/) is an experimental, end-to-end encrypted IPv6 overlay network. Every node gets a stable IPv6 address derived from its cryptographic key. The address falls in the `200::/7` range. Nodes connect to each other (peers) over normal internet connections, TCP/TLS or QUIC. The network then routes traffic between all nodes automatically.

To your applications it looks like an ordinary IPv6 network interface. Nginx, SSH, Git and any other software work without changes.

### Benefits

- **Encrypted by default:** all traffic between nodes is end-to-end encrypted, even if the application protocol (such as plain HTTP) isn't.
- **Stable addresses:** your address comes from your key, not from your provider. If you move to a new VPS, you keep the same address by moving the key.
- **No NAT or port forwarding:** every node is directly addressable, which is useful for home servers.
- **Self-organising routing:** nodes find routes without a central directory or manual configuration.
- **Easy to keep services private:** a service bound only to your Yggdrasil address can't be reached from the regular internet.

### Limitations

- **It is not an anonymity network.** Your peers can see your IP address, and your Yggdrasil address is tied to your key. Unlike Tor or I2P, it doesn't hide who you are. Don't use it expecting anonymity.
- Visitors need to run Yggdrasil too. A normal browser on a normal connection can't reach your site.
- It is still labelled experimental. Expect breaking changes between `0.x` releases.
- Ordinary domain registrars and Let's Encrypt don't work for Yggdrasil addresses. You'll use a raw address such as `http://[200:...]/`, and the network already encrypts the traffic.

## Overview

You will install Yggdrasil on a Debian VPS, serve a website from Nginx on your Yggdrasil address only, and use a firewall so nothing is reachable via the regular internet except SSH.

**Prerequisites:** a Debian VPS with SSH access and a user with `sudo`. This tutorial works for any static site, including Hugo.

## 1. Update the server

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Install Yggdrasil

Distribution packages are often out of date, and a version mismatch can stop you connecting to peers. Install from the official repository instead. These steps follow the [official Debian installation guide](https://yggdrasil-network.github.io/installation-linux-deb.html). Check it for the current signing key if the commands below fail.

```bash
sudo apt install -y dirmngr gnupg
sudo mkdir -p /usr/local/apt-keys
gpg --fetch-keys https://neilalexander.s3.dualstack.eu-west-2.amazonaws.com/deb/key.txt
gpg --export 1C5162E133015D81A811239D1840CDAC6011C5EA | sudo tee /usr/local/apt-keys/yggdrasil-keyring.gpg > /dev/null
echo 'deb [signed-by=/usr/local/apt-keys/yggdrasil-keyring.gpg] http://neilalexander.s3.dualstack.eu-west-2.amazonaws.com/deb/ debian yggdrasil' | sudo tee /etc/apt/sources.list.d/yggdrasil.list
sudo apt update
sudo apt install -y yggdrasil
```

The package generates a configuration, including your private key, at `/etc/yggdrasil.conf`.

## 3. Add peers

Yggdrasil needs at least one peer to join the network. Pick two or three geographically close servers from the [public peers list](https://github.com/yggdrasil-network/public-peers).

```bash
sudo nano /etc/yggdrasil.conf
```

Set `Peers` using the URIs from that list, for example:

```text
Peers: [
  tls://PEER_ONE_ADDRESS:PORT
  tls://PEER_TWO_ADDRESS:PORT
]
```

On a server you may also want to turn off local-network peer discovery and hide node information. Find these options in the file and set them:

```text
MulticastInterfaces: []
NodeInfoPrivacy: true
```

Leave the generated `PrivateKey` untouched.

## 4. Start Yggdrasil and find your address

```bash
sudo systemctl enable --now yggdrasil
sudo yggdrasilctl getSelf
```

The output includes your IPv6 address, which starts with `200:` to `3ff:`. Check that peers are connected and see the interface name:

```bash
sudo yggdrasilctl getPeers
ip -br addr
```

The interface is usually called `ygg0` or `tun0`. Use whichever name you see, and replace `ygg0` below if yours differs.

> **Back up your key.** Your address is derived from the `PrivateKey` in `/etc/yggdrasil.conf`. If you lose it, your address changes for good. Copy the file to encrypted offline storage and never commit it to a repository.

## 5. Install Nginx

```bash
sudo apt install -y nginx
sudo rm -f /etc/nginx/sites-enabled/default
```

Create `/etc/nginx/sites-available/yggdrasil-site`, replacing the address with your own from step 4:

```nginx
server {
    listen [YOUR_YGGDRASIL_ADDRESS]:80;
    server_name _;
    root /var/www/mysite;
    index index.html;
    server_tokens off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Because Nginx binds only to the Yggdrasil address, it doesn't listen on your public IPv4 or IPv6 addresses at all. Create a placeholder page and enable the site:

```bash
sudo mkdir -p /var/www/mysite
echo '<h1>Hello from Yggdrasil</h1>' | sudo tee /var/www/mysite/index.html
sudo ln -s /etc/nginx/sites-available/yggdrasil-site /etc/nginx/sites-enabled/yggdrasil-site
sudo nginx -t
```

### Start Nginx after Yggdrasil

Nginx can't bind to an address that doesn't exist yet. At boot, make sure it starts after Yggdrasil:

```bash
sudo mkdir -p /etc/systemd/system/nginx.service.d
sudo tee /etc/systemd/system/nginx.service.d/yggdrasil.conf > /dev/null <<'EOF'
[Unit]
After=yggdrasil.service
Wants=yggdrasil.service
EOF
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

If Nginx still fails to start at boot because the address isn't ready, check `journalctl -u nginx` and `ip -br addr`.

## 6. Block clearnet access with a firewall

Nginx already ignores the public interfaces. The firewall is a second layer, in case of a configuration mistake later. Enable IPv6 handling in UFW first:

```bash
sudo apt install -y ufw
sudo sed -i 's/^IPV6=.*/IPV6=yes/' /etc/default/ufw
```

Then set the rules:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow in on ygg0 to any port 80 proto tcp
sudo ufw enable
sudo ufw status verbose
```

This allows SSH on the regular network and HTTP only on the Yggdrasil interface. Ports 80 and 443 stay closed on the public interface. You don't need to open any port for Yggdrasil itself unless you want to accept inbound peer connections. Outbound peering works through the default allow-outgoing rule.

> **Optional hardening:** you can also restrict SSH to the Yggdrasil interface with `sudo ufw allow in on ygg0 to any port 22 proto tcp` and removing the `OpenSSH` rule. Make sure you can log in over Yggdrasil first, or you'll lock yourself out. Many VPS providers offer a web console as a fallback.

## 7. Test the setup

### From a machine with Yggdrasil

Install Yggdrasil on your own computer ([installation guides](https://yggdrasil-network.github.io/installation.html)) and add a few public peers. Then test:

```bash
curl -g http://[YOUR_YGGDRASIL_ADDRESS]/
```

You should see the "Hello from Yggdrasil" page. In a browser, open `http://[YOUR_YGGDRASIL_ADDRESS]/`.

### From the regular internet

From a machine without Yggdrasil:

```bash
curl --max-time 10 http://YOUR_VPS_IPV4/
curl -g --max-time 10 http://[YOUR_VPS_PUBLIC_IPV6]/
```

Both should time out or refuse the connection. On the VPS, confirm what Nginx is listening on:

```bash
sudo ss -tlnp | grep nginx
```

The only listener should be on your Yggdrasil address, port 80.

## 8. Deploying your content

Copy or build your site into `/var/www/mysite`. If you use Hugo, the deploy approach from the earlier tutorials works unchanged. Point `root` at your Hugo `public` directory and build with `--baseURL "/" --relativeURLs`.

## Privacy notes

- Yggdrasil encrypts traffic, but it doesn't anonymise it. Peers see your server's IP address, and your VPS provider knows who you are.
- Your server's outbound traffic, such as `apt`, DNS, Git and SSH, still uses the normal network unless you configure it otherwise.
- Anyone on the Yggdrasil network who knows your address can try to reach your services. Binding services to the Yggdrasil address and firewalling the rest is not a substitute for keeping software patched.
- Avoid external fonts, analytics and embeds in your site. They make visitors' browsers contact the regular internet.

## Further reading

- [Yggdrasil documentation](https://yggdrasil-network.github.io/documentation.html)
- [Yggdrasil Debian installation](https://yggdrasil-network.github.io/installation-linux-deb.html)
- [Public peers list](https://github.com/yggdrasil-network/public-peers)
- [Yggdrasil on GitHub](https://github.com/yggdrasil-network/yggdrasil-go)