---
title: Creating a 4096-bit RSA SSH Key and Disabling Password Login on Debian
description: How to generate a named 4096-bit RSA SSH key pair, copy the public key to a Debian host, and disable SSH password authentication so only key-based login works.
weight: 1
categories: [tutorial]
tags: [ssh, rsa, debian, security, linux, openssh]
---

## Overview

You will generate a 4096-bit RSA key pair named `alice`, install the public key on a Debian server, and turn off password authentication so only your key can log in.

**Prerequisites:** a Debian host with SSH access using a password, and a Linux, macOS or Windows (PowerShell or WSL) computer with OpenSSH installed.

> **Important:** Keep your current SSH session open until you've confirmed key login works in a second terminal. Otherwise a mistake could lock you out.

## 1. Generate the key pair

On your local computer:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/alice -C "alice"
```

- `-t rsa -b 4096` sets the key type and size.
- `-f ~/.ssh/alice` names the key pair. The private key is `alice` and the public key is `alice.pub`.
- `-C "alice"` adds a comment to help you identify the key.

When prompted, enter a strong passphrase. It encrypts the private key on disk, so a stolen file is useless without it.

Check that both files exist:

```bash
ls -l ~/.ssh/alice ~/.ssh/alice.pub
```

> **Keep `alice` (the private key) secret.** Never share it, upload it or commit it to a repository. Only `alice.pub` goes on servers. Back up the private key to encrypted storage.

## 2. Copy the public key to the Debian host

Replace `YOUR_USER` and `SERVER_IP` with your own values.

```bash
ssh-copy-id -i ~/.ssh/alice.pub YOUR_USER@SERVER_IP
```

Enter your account password when asked. This appends the key to `~/.ssh/authorized_keys` on the server with the correct permissions.

If `ssh-copy-id` isn't available (for example in Windows PowerShell), do it manually:

```bash
cat ~/.ssh/alice.pub | ssh YOUR_USER@SERVER_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 3. Test key login

In a **new** terminal:

```bash
ssh -i ~/.ssh/alice YOUR_USER@SERVER_IP
```

You should be asked for the key's passphrase, not the account password. Don't continue until this works.

### Optional: add an SSH config entry

On your local computer, add this to `~/.ssh/config`:

```text
Host myserver
    HostName SERVER_IP
    User YOUR_USER
    IdentityFile ~/.ssh/alice
    IdentitiesOnly yes
```

Now you can connect with `ssh myserver`.

## 4. Disable password authentication

On the Debian server, create a drop-in file. It's safer than editing the main config and survives package upgrades:

```bash
sudo nano /etc/ssh/sshd_config.d/99-disable-password-auth.conf
```

Add:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password
```

- `PasswordAuthentication no` disables password logins.
- `KbdInteractiveAuthentication no` closes the keyboard-interactive route, which can also prompt for passwords.
- `PermitRootLogin prohibit-password` allows root to log in with keys only. Use `no` instead if you don't need root login over SSH.

Debian's main config includes files from `sshd_config.d/` first, and the first value set for an option wins. Check that no other file in that directory or `/etc/ssh/sshd_config` sets conflicting values:

```bash
sudo grep -ri "PasswordAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Validate the configuration and reload:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

If `sshd -t` prints nothing, the syntax is valid.

## 5. Verify

Confirm the effective setting:

```bash
sudo sshd -T | grep -Ei "passwordauthentication|kbdinteractiveauthentication"
```

Both should show `no`. Then, from your computer, try to force a password login:

```bash
ssh -o PubkeyAuthentication=no YOUR_USER@SERVER_IP
```

It should fail with `Permission denied (publickey)`. Then confirm key login still works:

```bash
ssh -i ~/.ssh/alice YOUR_USER@SERVER_IP
```

## Troubleshooting

- **Still asked for a password:** the key wasn't installed correctly, or permissions are wrong. On the server, `~/.ssh` should be `700` and `authorized_keys` should be `600`, owned by your user.
- **Password login still works:** another config file is overriding your setting. Re-run the `grep` in step 4.
- **Locked out:** use your VPS provider's web console to log in and fix the config.
- **Debug the connection:** run `ssh -vi ~/.ssh/alice YOUR_USER@SERVER_IP` for verbose output.

## Notes

- 4096-bit RSA is secure and widely compatible. `ed25519` keys are shorter and faster, and are the usual modern choice. Use `ssh-keygen -t ed25519 -f ~/.ssh/alice` if your servers support it.
- Use a separate key pair per device, and remove lost devices' keys from `~/.ssh/authorized_keys` on the server.
- Consider `ssh-agent` so you don't re-enter your passphrase for every connection.
- Pair this with a firewall (`ufw allow OpenSSH`) and automatic security updates.

## Further reading

- [OpenSSH manual pages](https://www.openssh.com/manual.html)
- [Debian Wiki: SSH](https://wiki.debian.org/SSH)