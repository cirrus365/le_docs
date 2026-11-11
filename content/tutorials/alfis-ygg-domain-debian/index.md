---
title: Creating ALFIS .ygg Domains for the Yggdrasil Network on Debian
description: Register a human-friendly .ygg domain for your Yggdrasil address using the ALFIS blockchain DNS - install the ALFIS resolver daemon on Debian, mine a domain with an AAAA record, and resolve it.
weight: 8
categories: [tutorial]
tags: [yggdrasil, debian, alfis, dns, networking, ipv6, blockchain]
---

## What is ALFIS?

[ALFIS](https://github.com/Revertron/Alfis) (Alternative Free Identity System) is a decentralised, blockchain-based DNS for alternative networks. Yggdrasil itself has no naming layer at all - every node is reachable only by its raw `200::/7` address - and the project explicitly leaves DNS out of scope. ALFIS fills that gap with a tiny peer-to-peer synchronised database of domains. Several zones exist, and the `.ygg` zone is reserved for Yggdrasil addresses only.

There is no registrar. You "mine" a block containing your domain, secured by proof-of-work, and it propagates to all ALFIS nodes. Mining needs a machine with a GUI, but the resulting domain can be resolved by anyone - via your own resolver, a public resolver, or the `.ygg.at` clearnet mirror.

## Overview

You will run the ALFIS DNS daemon on a Debian server, mine a `.ygg` domain pointing an `AAAA` record at your Yggdrasil address, and resolve it.

**Prerequisites:** a Debian server with a working [Yggdrasil installation](/tutorials/host-website-yggdrasil-network-debian-nginx/) and its IPv6 address (from `sudo yggdrasilctl getSelf`), plus `sudo` rights. The server needs outbound access to port `4244` for blockchain sync.

## 1. Install the ALFIS DNS daemon

Add the maintainer's APT repository and install the headless (no GUI) daemon:

```bash
gpg --fetch-keys https://deb.revertron.com/key.txt
gpg --export F244E16645D86D62 | sudo tee /usr/local/apt-keys/alfis.gpg > /dev/null
echo 'deb [signed-by=/usr/local/apt-keys/alfis.gpg] https://deb.revertron.com/ debian alfis' | sudo tee /etc/apt/sources.list.d/alfis.list
sudo apt update && sudo apt install -y alfis
```

The configuration lives in `/etc/alfis.conf` and the blockchain database in `/var/lib/alfis`. Review the config and make sure the DNS section and Yggdrasil peers are set:

```bash
sudo nano /etc/alfis.conf
```

```toml
[net]
peers = ["peer-v4.alfis.name:4244", "peer-v6.alfis.name:4244", "peer-ygg.alfis.name:4244"]

[dns]
listen = "127.0.0.1:53"
forwarders = ["1.1.1.1", "8.8.8.8"]
```

### Free port 53 if systemd-resolved is running

If your system runs `systemd-resolved`, its stub listener occupies `127.0.0.53:53` and ALFIS can't bind. Disable the stub and point the system at ALFIS:

```bash
sudo sed -i 's/^#\?DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
sudo systemctl enable --now alfis
```

Then make `127.0.0.1` your resolver. On servers without `systemd-resolved`, edit `/etc/resolv.conf` directly:

```text
nameserver 127.0.0.1
```

> **NetworkManager beware:** on desktop systems it rewrites `/etc/resolv.conf` at will. Use `systemd-resolved` with `DNS=127.0.0.1` in `/etc/systemd/resolved.conf` instead.

## 2. Get a keypair

Domains belong to a cryptographic key, and one key can register up to 10 domains. Generate the keypair once with the desktop GUI app (or the web UI, below):

1. Download the GUI build (no `nogui` suffix) for your OS from the [ALFIS releases](https://github.com/Revertron/Alfis/releases) page. On Linux it needs `libfontconfig` and a desktop session.
2. Unzip it into a directory and run `./alfis`. It looks for a config file named `alfis.toml` in that directory and syncs the blockchain on first start.
3. Open **Manage keys** and generate a keypair, then **Save key** to a `.key` file.

> **Back up the key file.** Whoever holds your key owns your domains, including renewal and transfer. Store it like a private SSH key.

### No desktop? Use the web UI

The headless daemon can be managed from a browser. Add this to `/etc/alfis.conf`:

```toml
[webui]
listen = "127.0.0.1:8082"
password = "your-strong-password"
```

Restart with `sudo systemctl restart alfis`, then tunnel in over SSH and open the UI locally:

```bash
ssh -L 8082:127.0.0.1:8082 user@server
```

`http://127.0.0.1:8082` now offers the same key management and domain mining as the desktop app. Keys created here are saved in the daemon's working directory - add them to `key_files` in `/etc/alfis.conf` so they load on restart, and back them up.

## 3. Mine the domain

1. In **Mine domain**, enter the name you want (without the zone). If it turns red, it's taken or invalid.
2. Add the records in the same dialog: an **`AAAA` record with your Yggdrasil IPv6 address**. You can edit records later, but every change requires mining a new block.
3. Click **Mine domain** and wait. Mining is proof-of-work: anywhere from a few minutes to several hours depending on your CPU and network difficulty. The block is then signed by other nodes and propagates automatically.
4. Renewal: registrations expire after about a year. Mine the domain again with the same key before then - renewing early doesn't extend the clock.

## 4. Resolve it

First sync check, then resolution. `dig` comes from the `dnsutils` package:

```bash
sudo apt install -y dnsutils
dig AAAA yourname.ygg +short
```

Your own daemon answers from the local blockchain copy. Public ALFIS resolvers reachable over Yggdrasil work too - useful before your daemon finishes syncing, or for other people's visitors:

```bash
dig AAAA @302:db60::53 yourname.ygg +short   # Frankfurt
dig AAAA @300:6223::53 yourname.ygg +short   # Bratislava
dig AAAA @302:7991::53 yourname.ygg +short   # Amsterdam
dig AAAA @301:1088::53 yourname.ygg +short   # Buffalo
```

Test the full round trip from any machine running Yggdrasil:

```bash
curl -g http://yourname.ygg/
```

> **Clearnet mirror:** every `.ygg` domain is also resolvable from the regular internet as `yourname.ygg.at`, which simply points at the same Yggdrasil address. Handy for sharing links with people who haven't set up ALFIS yet.

### Browsers and the .ygg suffix

Firefox and Chrome treat unknown TLDs as search terms. In Firefox, set `browser.fixup.domainsuffixwhitelist.ygg` to `true` in `about:config`; Chrome users type the trailing slash. Without this, the browser redirects to a search engine instead of resolving.

## Notes

- **No anonymity.** Your domain records, mining activity and contacts are public in the blockchain, and Yggdrasil peers can see your IP. ALFIS is a naming convenience, not a privacy tool.
- **Self-certification.** ALFIS can publish certificates for browsers to trust without a central CA, but as of now plain HTTP with the network's built-in encryption is the common setup.
- **Mining fairness.** Blocks must be countersigned by other nodes, which can occasionally delay propagation. Patience, not troubleshooting, fixes it.

## Further reading

- [ALFIS on GitHub](https://github.com/Revertron/Alfis)
- [ALFIS releases](https://github.com/Revertron/Alfis/releases)
- [ALFIS Debian repository](https://deb.revertron.com/)
- [Hosting a Website on the Yggdrasil Network with Debian and Nginx](/tutorials/host-website-yggdrasil-network-debian-nginx/)
- [Yggdrasil Network](https://yggdrasil-network.github.io/)
