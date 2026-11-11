---
title: Access Home Services with Cloudflare Tunnel on Debian
description: Set up Cloudflare Tunnel on Debian to access a self-hosted web application without port forwarding, and protect it with Cloudflare Access.
weight: 7
categories: [tutorial]
tags: [debian, linux, cloudflare, cloudflared, networking, self-hosting, security]
---

### Access home services with Cloudflare Tunnel on Debian

Cloudflare Tunnel connects a service on your home network to Cloudflare using an outbound connection. You do not need to forward ports on your router or have a static public IP address.

This tutorial uses:

- **Debian** to run the `cloudflared` connector.
- **Cloudflare Tunnel** to carry traffic between Cloudflare and your application.
- **Cloudflare Access** to require authentication before visitors can reach it.

**Important:** a tunnel is not an authentication system by itself. Publishing an application without an Access policy can make it available to anyone who knows its address.

#### 1. Prepare your domain and application

You will need:

- A Debian server with `sudo` access.
- A Cloudflare account and a configured Zero Trust organisation.
- A domain active in your Cloudflare account.
- A running HTTP or HTTPS application.
- Network connectivity from the Debian server to that application.

The connector can run on the same machine as the application or on another machine that can reach it.

The examples use these placeholder values:

| Setting | Example |
|---|---|
| Domain | `example.com` |
| Public application hostname | `app.example.com` |
| Internal application address | `192.168.1.50` |
| Internal application port | `8080` |
| Tunnel name | `home-debian` |

Replace these values with your own.

This guide covers browser-based web applications. It does not configure general-purpose VPN access or direct connections from arbitrary SSH, RDP or other native clients.

#### 2. Add your domain to Cloudflare

If your domain is not already active in Cloudflare:

1. Sign in to the Cloudflare dashboard.
2. Add your domain.
3. Review the imported DNS records.
4. Copy the nameservers Cloudflare assigns to your domain.
5. At your registrar, change the domain's nameservers to those assigned nameservers.
6. Wait for Cloudflare to confirm that the domain is active.

If migrating an existing domain: preserve all required records, including MX records, TXT verification records and existing subdomains.

If the domain already uses DNSSEC, follow the appropriate DNSSEC migration procedure. A stale DS record at the registrar can break DNS resolution.

If you followed an authoritative BIND9 setup: changing the domain's nameservers to Cloudflare moves its public authoritative DNS hosting to Cloudflare. Do not change the delegation without planning that migration.

#### 3. Prepare Debian

Update the package index and install tools for testing HTTP connectivity:

```bash
sudo apt update
sudo apt install ca-certificates curl
```

Check the server's package architecture:

```bash
dpkg --print-architecture
```

Use this architecture when selecting the connector installation instructions.

For example, an amd64 server needs the corresponding x86-64 package, while an arm64 server needs the ARM64 package.

This tutorial installs cloudflared directly on Debian rather than running it in Docker. That also avoids confusing the container's loopback address with the host's loopback address.

#### 4. Test the application locally

From the Debian server, check that the application is reachable:

```bash
curl -I --connect-timeout 10 http://192.168.1.50:8080
```

An HTTP response confirms that the server can reach the application. A redirect or authentication response can be normal.

If the application does not support HEAD requests, try a normal request:

```bash
curl --connect-timeout 10 -o /dev/null -sS -w '%{http_code}\n' http://192.168.1.50:8080
```

Resolve connection errors before continuing.

Check that:

- The application is running.
- Its address and port are correct.
- Its firewall permits connections from the Debian connector.
- It listens on an address the connector can reach.

If the application runs on the same Debian machine, you can use `127.0.0.1` when appropriate. If it runs on another machine, use that machine's reachable address.

For HTTPS applications, test using the hostname that matches the certificate:

```bash
curl -I --connect-timeout 10 https://internal-app.example.com
```

Use your actual internal hostname. Do not disable certificate verification merely to hide a certificate or hostname problem.

#### 5. Define the application in Cloudflare

Open the Cloudflare One dashboard.

Using the guided setup:

1. Open **Get Started**.
2. Choose **Set up secure access to private apps from any browser**.
3. Select **Connect a private web application**.
4. Enter a descriptive application name.
5. Enter the application's internal hostname or IP address.
6. Select its protocol: HTTP or HTTPS.
7. Enter its listening port.

For the example application, enter:

| Setting | Value |
|---|---|
| Application name | `Home application` |
| Internal address | `192.168.1.50` |
| Protocol | `HTTP` |
| Port | `8080` |

Dashboard labels can change. The essential settings are the origin address, origin protocol and origin port.

The origin protocol describes the connector-to-application connection. Selecting HTTP here does not mean visitors must use HTTP for the public address.

#### 6. Choose the public hostname

Select your domain and enter the application's subdomain:

| Setting | Value |
|---|---|
| Domain | `example.com` |
| Subdomain | `app` |
| Resulting hostname | `app.example.com` |

Use an unused hostname. If it already has a DNS record, investigate that record before replacing it.

The guided setup creates the hostname's routing through the tunnel. You do not need to publish your home public IP address in an A record.

#### 7. Require authentication with Cloudflare Access

Add the email addresses of the users who should be allowed to access the application.

The guided setup supports email one-time PIN authentication:

1. A visitor opens the public hostname.
2. Cloudflare asks for their email address.
3. An authorised user receives a sign-in code.
4. Cloudflare permits access after successful authentication.

For a private application, allow only the specific users who need it.

Do not create an unrestricted Allow or Bypass policy simply to make testing easier.

You can later use an identity provider and enforce MFA through an appropriate authentication configuration. Email one-time PIN authentication alone should not be described as MFA.

Keep the application's own authentication enabled, particularly for administrative interfaces.

#### 8. Create the tunnel and install the connector

In the guided setup, create a tunnel named `home-debian`.

Select the Debian/Linux installation option matching your server's architecture.

The dashboard supplies the current installation and connector registration commands. Run those commands on the Debian server, using `sudo` where instructed.

Use the provided instructions rather than assuming cloudflared is available from Debian's default repositories.

For an always-on server, use the dashboard's service installation instructions rather than leaving the connector running only in an interactive terminal.

Treat the tunnel token as a secret:

- Do not commit it to a Git repository.
- Do not include it in screenshots or public documentation.
- Do not paste it into support requests.
- If it is exposed, rotate it and update the affected connectors.

Wait for the dashboard to show that the connector has connected, then complete the guided setup.

#### 9. Review firewall and TLS settings

You do not need inbound router port forwarding for this setup.

Keep your existing firewall protections and preserve any access you need for administering Debian.

If your network restricts outbound connections, follow Cloudflare's current tunnel firewall requirements. The connector must be able to reach Cloudflare, resolve DNS and connect to the application.

Do not assume that allowing ordinary outbound web browsing is sufficient for every tunnel transport.

There are separate connections to consider:

| Connection | Nature |
|---|---|
| Browser to Cloudflare | Public HTTPS |
| Debian connector to Cloudflare | Encrypted tunnel |
| Debian connector to application | Your selected HTTP or HTTPS origin connection |

If the application is on another machine and you select HTTP, that final connection is not encrypted by HTTPS. Use validated HTTPS where the network or application requires it.

For a self-signed HTTPS origin, prefer configuring a trusted certificate or the appropriate trusted CA and server name.

Do not use "No TLS Verify" as the default fix. It disables origin certificate verification and weakens protection against origin impersonation.

Cloudflare terminates public HTTPS in this architecture. It is not a browser-to-origin TLS pass-through connection.

#### 10. Test authorised and unauthorised access

Use the dashboard's Test login option, then test the public hostname from a device outside your home network, such as a phone using mobile data.

Confirm that:

- An unauthenticated visitor sees the Cloudflare Access login screen, not the application.
- An allowed user can sign in.
- A user who is not allowed cannot reach the application.
- The application works after authentication.
- The application's own login remains active where configured.

You can also inspect the initial response:

```bash
curl -I https://app.example.com
```

An Access-protected application commonly redirects an unauthenticated request to a login page. A redirect alone does not prove the policy is correct: perform the browser tests as well.

If the application was previously exposed through port forwarding or another public hostname, remove that alternative route once you have confirmed the tunnel works. Otherwise, visitors may bypass Cloudflare Access.

#### 11. Maintain the connector and application

Keep Debian, cloudflared and the application updated.

Use the update method appropriate to how you installed cloudflared. A package installed through an APT repository and a manually downloaded package do not necessarily receive updates in the same way.

After updates or a reboot, verify that:

- The connector reconnects automatically.
- The tunnel is healthy.
- Access authentication is still required.
- The application works normally.

Review allowed users periodically and remove access when it is no longer needed.

A tunnel reduces direct network exposure; it does not replace application security, backups or operating-system maintenance.

#### Troubleshooting

| Symptom | Check |
|---|---|
| Connector does not appear online | Installation, connector credentials, DNS and outbound connectivity |
| Tunnel is healthy but the application fails | Origin address, port, protocol and local reachability |
| Application works locally but not through the tunnel | Origin TLS validation, hostname handling and application proxy settings |
| Public hostname does not resolve | Domain activation, DNS records and tunnel hostname configuration |
| Visitors reach the application without signing in | Access hostname coverage, policies and alternative public routes |
| Allowed user cannot sign in | Exact email address, authentication method and policy conditions |
| Application fails after a reboot | Connector service startup and application startup |
| HTTPS origin reports certificate errors | Certificate validity, trusted CA and certificate hostname |

#### Further reading

- [Cloudflare Tunnel overview](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Connect and protect a private web application](https://developers.cloudflare.com/cloudflare-one/setup/secure-private-apps/private-web-app/)
