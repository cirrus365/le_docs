---
title: Set Up Authoritative DNS with BIND9 on Debian
description: Configure primary and secondary authoritative-only DNS servers on Debian with BIND9, zone transfers, registrar glue records and public delegation.
weight: 7
categories: [tutorial]
tags: [debian, linux, bind9, dns, authoritative-dns, nameservers, networking]
---

### Set up authoritative DNS with BIND9 on Debian

This tutorial configures **two authoritative-only DNS servers** for a public domain:

- **Primary:** stores the zone file you edit.
- **Secondary:** automatically retrieves a copy from the primary.

Recursion is disabled: these servers answer questions about your domain, but do not act as general-purpose DNS resolvers.

**Assumptions:** fresh Debian servers using the distribution’s BIND9 packages, `sudo` access, and public IPv4 connectivity. This is an unsigned DNS setup; DNSSEC signing is not covered.

#### 1. Prepare your domain and servers

You will need:

- A domain you control.
- Access to its registrar’s nameserver and glue-record settings.
- Two servers with static public IP addresses, preferably on different networks.
- Inbound **UDP and TCP port 53** permitted by your hosting provider.
- An existing working recursive resolver for each server’s own DNS lookups.

The examples use these **documentation-only values**. Replace them throughout:

| Purpose | Example |
|---|---|
| Domain | `example.com` |
| Primary nameserver | `ns1.example.com` |
| Primary IP | `192.0.2.10` |
| Secondary nameserver | `ns2.example.com` |
| Secondary IP | `198.51.100.20` |
| Website IP | `203.0.113.30` |

**If migrating an existing domain:** copy all existing records, including mail and verification records, before changing its delegation. If the registrar currently publishes a DNSSEC **DS record**, arrange its removal and allow its cached lifetime to expire before switching to these unsigned servers. An obsolete DS record can cause validating resolvers to reject your domain.

Do not change the domain’s nameservers until both servers have passed the tests below.

#### 2. Install BIND9 on both servers

Run on **both servers**:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils
```

Check the installed version:

```bash
named -v
```

Back up the configuration before editing:

```bash
sudo cp -a /etc/bind /etc/bind.backup-$(date +%Y%m%d-%H%M%S)
```

Debian separates BIND configuration into:

- `/etc/bind/named.conf.options` — global settings.
- `/etc/bind/named.conf.local` — your zone definitions.
- `/etc/bind/named.conf.default-zones` — packaged default zones.

Leave the main include structure and default zones intact.

#### 3. Disable recursion on both servers

On **both servers**, open:

```bash
sudo nano /etc/bind/named.conf.options
```

For this fresh-server setup, replace its contents with:

```
options {
    directory "/var/cache/bind";

    // Public authoritative DNS over IPv4.
    listen-on { any; };
    listen-on-v6 { none; };

    // Answer public queries for the zones hosted here.
    allow-query { any; };

    // Do not provide recursive resolution or cached answers.
    recursion no;
    allow-query-cache { none; };

    // Deny zone transfers unless explicitly allowed per zone.
    allow-transfer { none; };

    // Zone changes are made through local files, not DNS UPDATE.
    allow-update { none; };
};
```

This tutorial intentionally uses IPv4 only. Do not publish IPv6 nameserver addresses unless you also configure and test IPv6 DNS connectivity.

**Do not point the servers’ own system resolver at this authoritative-only service** expecting it to resolve arbitrary domains.

#### 4. Define the zone on the primary

On **ns1**, open:

```bash
sudo nano /etc/bind/named.conf.local
```

Add:

```
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";

    // Only the secondary may retrieve the complete zone.
    allow-transfer { 198.51.100.20; };

    // Tell the secondary when the zone changes.
    notify yes;
    also-notify { 198.51.100.20; };
};
```

`master` is BIND’s traditional configuration term for a primary server.

This example authorises transfers by source IP. Use the source address the primary will actually see; NAT or multihomed servers may require adjustments.

#### 5. Create the primary zone file

On **ns1**, create a directory:

```bash
sudo install -d -o root -g bind -m 0750 /etc/bind/zones
```

Create the file:

```bash
sudo nano /etc/bind/zones/db.example.com
```

Add:

```
$ORIGIN example.com.
$TTL 3600

@   IN  SOA ns1.example.com. hostmaster.example.com. (
        1           ; Serial: increase after every change
        3600        ; Refresh: 1 hour
        900         ; Retry: 15 minutes
        1209600     ; Expire: 14 days
        300         ; Negative caching TTL: 5 minutes
)

; Authoritative nameservers.
@       IN  NS      ns1.example.com.
@       IN  NS      ns2.example.com.

; Nameserver addresses.
ns1     IN  A       192.0.2.10
ns2     IN  A       198.51.100.20

; Website records.
@       IN  A       203.0.113.30
www     IN  CNAME   example.com.
```

Set permissions:

```bash
sudo chown root:bind /etc/bind/zones/db.example.com
sudo chmod 0640 /etc/bind/zones/db.example.com
```

Important details:

- `@` represents the zone apex: `example.com`.
- Names ending in a dot are fully qualified.
- `hostmaster.example.com.` represents the administrative email address `hostmaster@example.com`; replace it with an appropriate contact.
- The **serial must increase whenever you change the zone** so the secondary can detect updates.
- The website address does not need to match either nameserver’s address.
- Add your real MX, TXT and other required records before migrating an existing domain.

The zone’s `ns1` and `ns2` address records **do not replace registrar-level glue records**. Those are configured later.

#### 6. Configure the secondary

On **ns2**, open:

```bash
sudo nano /etc/bind/named.conf.local
```

Add:

```
zone "example.com" {
    type slave;
    file "/var/cache/bind/db.example.com";

    masters { 192.0.2.10; };

    allow-notify { 192.0.2.10; };
    allow-transfer { none; };
};
```

`slave` is BIND’s traditional term for a secondary server.

BIND writes the transferred zone under `/var/cache/bind`, where the Debian service can write.

**Do not manually create or edit the secondary’s zone file.** Make changes on the primary and let zone transfers synchronise them.

#### 7. Validate and start both servers

On **ns1**, check the configuration and zone:

```bash
sudo named-checkconf
sudo named-checkzone example.com /etc/bind/zones/db.example.com
```

`named-checkconf` normally prints nothing when it finds no errors. The zone check should finish with `OK`.

On **ns2**, check its configuration:

```bash
sudo named-checkconf
```

Correct any errors before proceeding.

On **both servers**, enable and restart the service:

```bash
sudo systemctl enable named
sudo systemctl restart named
sudo systemctl status named --no-pager
```

Inspect recent logs:

```bash
sudo journalctl -u named -n 50 --no-pager
```

#### 8. Allow DNS through your firewalls

On both servers, allow inbound:

| Protocol | Port | Purpose |
|---|---|---|
| UDP | `53` | DNS queries and notifications |
| TCP | `53` | DNS queries, larger responses and zone transfers |

**TCP is required for public DNS too—not just transfers.** BIND’s transfer ACL restricts who can request a zone transfer while still allowing ordinary TCP queries.

If you already use UFW:

```bash
sudo ufw allow 53/udp
sudo ufw allow 53/tcp
```

Apply equivalent rules in your hosting provider’s firewall or security group. Ensure the secondary can make outbound DNS connections to the primary.

Do not enable an unfamiliar firewall remotely without first preserving SSH access. Do not expose BIND’s RNDC management port, TCP `953`, to the public internet.

#### 9. Test both servers before delegation

First, on each server, check its local authoritative response:

```bash
dig @127.0.0.1 example.com SOA +norecurse
```

Look for:

- `status: NOERROR`
- The `aa` flag, meaning **authoritative answer**.
- Your SOA record.

Then, from **another internet-connected machine**, test both public addresses:

```bash
dig @192.0.2.10 example.com SOA +norecurse
dig @198.51.100.20 example.com SOA +norecurse

dig @192.0.2.10 www.example.com A +norecurse
dig @198.51.100.20 www.example.com A +norecurse
```

Check TCP as well:

```bash
dig @192.0.2.10 example.com SOA +tcp +norecurse
dig @198.51.100.20 example.com SOA +tcp +norecurse
```

Both servers should return authoritative answers and the same SOA serial.

If the secondary has no zone, inspect its logs:

```bash
sudo journalctl -u named -n 100 --no-pager
```

Check TCP connectivity to the primary, its transfer ACL, and the secondary’s actual source IP.

**Check that recursion is unavailable:**

```bash
dig @192.0.2.10 www.example.net A +recurse
dig @198.51.100.20 www.example.net A +recurse
```

These queries must not produce a recursively resolved answer. `REFUSED` is normal with this configuration; the `ra` flag should not be present.

**Check that unauthorised zone transfers are denied**, running from a machine other than the authorised secondary:

```bash
dig @192.0.2.10 example.com AXFR
dig @198.51.100.20 example.com AXFR
```

Both should fail rather than return the entire zone.

#### 10. Register glue and change the delegation

At your domain registrar, register these nameserver hosts:

| Nameserver | Address |
|---|---|
| `ns1.example.com` | `192.0.2.10` |
| `ns2.example.com` | `198.51.100.20` |

Registrars may call this **host registration**, **child nameservers**, **personal nameservers**, or **glue records**.

Glue is needed here because your nameservers sit inside the domain they serve. It supplies the parent zone with their addresses so resolvers can reach them without a circular dependency.

Next, change the domain’s authoritative nameservers to:

```bash
ns1.example.com
ns2.example.com
```

Keep the old DNS service working during the transition: resolvers may continue using cached delegation information.

#### 11. Verify public resolution

After changing delegation, inspect the DNS path:

```bash
dig +trace example.com NS
```

Check resolution through your normal recursive resolver:

```bash
dig example.com NS
dig example.com A
dig www.example.com A
```

Confirm that:

- The parent delegates to your two nameservers.
- The nameserver addresses are correct.
- Both authoritative servers return matching records.
- Normal recursive lookups return the expected website address.

There is no fixed “propagation” interval: visibility depends on registrar processing and cached TTLs.

#### 12. Update records safely

For future changes, edit the zone on **ns1**:

```bash
sudo nano /etc/bind/zones/db.example.com
```

Make the change and increase the serial—for example, from `1` to `2`.

Validate, then reload:

```bash
sudo named-checkzone example.com /etc/bind/zones/db.example.com &&
sudo rndc reload example.com
```

Compare the SOA records:

```bash
dig @192.0.2.10 example.com SOA +short
dig @198.51.100.20 example.com SOA +short
```

The secondary should retrieve the update and show the new serial.

#### Troubleshooting

| Symptom | Check |
|---|---|
| Works locally but times out externally | Host firewall, provider firewall, NAT and listening addresses |
| UDP works but TCP fails | Inbound TCP port `53` |
| Secondary serves an older serial | Serial increment, notifications, transfer ACL and logs |
| Direct queries work but public resolution fails | Registrar delegation, glue and stale DNSSEC DS records |
| Authoritative server returns `SERVFAIL` for your zone | Zone-loading errors, file permissions and configuration |
| Missing `aa` flag | Whether the requested zone is loaded and the query reached the intended server |

For a longer-lived deployment, add **TSIG-authenticated zone transfers**, monitoring and configuration backups. TSIG authenticates transfers but does not encrypt their contents. DNSSEC signing is a separate step, not something enabled merely by a resolver’s `dnssec-validation` setting.

**References:** [Debian primary/secondary BIND setup](https://www.linuxbabe.com/debian/authoritative-dns-server-debian-10-buster-bind9), [Debian/Ubuntu authoritative nameserver configuration](https://www.violetdragonsnetwork.co.uk/how-to-setup-bind9-authoritative-name-servers-on-ubuntu-debian/).