---
title: GnuPG Guide - Create a Key, Sign, Encrypt, Verify, Decrypt and Use Keyservers
description: How to generate an OpenPGP key pair with GnuPG, back it up, publish it to keys.openpgp.org, sign and encrypt messages, and verify and decrypt messages from other people.
weight: 2
categories: [tutorial]
tags: [gnupg, gpg, openpgp, encryption, security, linux, keyserver]
---

## Overview

GnuPG (`gpg`) is the free implementation of OpenPGP. With a key pair you can **sign** data to prove it came from you and hasn't been altered, and **encrypt** data so only the intended recipient can read it.

- Your **public key** is shared with everyone.
- Your **private (secret) key** never leaves your control.

For this tutorial, your identity is `Alice <alice@example.org>`. Replace it with your own name and email address.

## 1. Check your GnuPG version

```bash
gpg --version
```

Install or update it if needed:

- **Debian/Ubuntu:** `sudo apt install gnupg`
- **Fedora:** `sudo dnf install gnupg2`
- **macOS (Homebrew):** `brew install gnupg pinentry-mac`
- **Windows:** [Gpg4win](https://gpg4win.org/)

The 2.5 series is the current stable branch, and the 2.4 series reached end of life on 30 June 2026. Debian and Ubuntu often ship older versions with security fixes backported, and this guide works on all of them. See the [GnuPG download page](https://gnupg.org/download/) for current releases.

> **Note:** On older guides you may see `gpg2`. On current systems the command is simply `gpg`.

## 2. Generate a key pair

The quickest way uses the default algorithms and a two-year expiry:

```bash
gpg --quick-generate-key "Alice <alice@example.org>" default default 2y
```

Enter a strong passphrase when prompted. Store it in a password manager, because without it your private key is useless.

The defaults in GnuPG 2.4 and later are an Ed25519 key for signing and certifying, plus a Curve25519 encryption subkey. Check what you got:

```bash
gpg --list-keys alice@example.org
gpg --list-secret-keys --keyid-format long
```

Output looks like this:

```text
pub   ed25519 2026-11-11 [SC] [expires: 2028-11-10]
      0123456789ABCDEF0123456789ABCDEF01234567
uid           [ultimate] Alice <alice@example.org>
sub   cv25519 2026-11-11 [E] [expires: 2028-11-10]
```

The 40-character string is your **fingerprint**. It identifies your key uniquely.

### Optional: interactive generation

To pick the algorithm yourself, use:

```bash
gpg --full-gen-key
```

RSA 3072 is a safe, widely compatible choice, and 4096 is also fine. GnuPG 2.5 can also create experimental post-quantum ECC+Kyber keys (menu option 16). Other OpenPGP software may not support them yet, so don't use them if you need broad compatibility.

> **Why an expiry date?** You can extend it at any time, even after it expires. An expiry limits the damage if you lose access to the key and can't revoke it.

## 3. Create a revocation certificate

Do this **now**, while you still have the key. It lets you declare the key dead if you lose it or it's compromised.

```bash
gpg --output alice-revoke.asc --gen-revoke alice@example.org
```

Choose reason `0` (no reason specified) and confirm. Store `alice-revoke.asc` offline, away from your private key. Anyone with this file can revoke your key.

## 4. Back up your keys

```bash
gpg --armor --export alice@example.org > alice-public.asc
gpg --armor --export-secret-keys alice@example.org > alice-private.asc
```

Copy `alice-private.asc` to encrypted offline storage, then delete the plain copy from your working directory. Never email it or commit it to a repository.

To restore on another machine:

```bash
gpg --import alice-private.asc
```

## 5. Publish your public key

Share the fingerprint through a channel people trust, such as your website over HTTPS, so others can check it. Then publish the key itself.

### Set a keyserver

Upstream GnuPG has had no built-in default keyserver since 2.5.3, and some distributions such as Debian set one for you. Set it explicitly in `~/.gnupg/dirmngr.conf`:

```text
keyserver hkps://keys.openpgp.org
```

```bash
gpgconf --reload dirmngr
```

If an old `dirmngr.conf` contains a line starting with `hkp-cacert` (from the old SKS network), delete it. It breaks normal certificate checks.

### Upload to keys.openpgp.org

```bash
gpg --export alice@example.org | curl -T - https://keys.openpgp.org
```

The server replies with a link. Open it and click **Send Verification Email**, then click the link in the email. Only after you confirm your address can people find your key by email.

> **`--send-keys` can't verify your email.** `gpg --send-keys` uploads the key, but the server can't start email confirmation that way. Use it later to push updates such as a new expiry date, a subkey or a revocation, once your address is verified.

```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys 0123456789ABCDEF0123456789ABCDEF01234567
```

### Other options

- **[keyserver.ubuntu.com](https://keyserver.ubuntu.com/):** a synchronizing keyserver. It doesn't verify email addresses, and uploaded keys can't be deleted, so be sure you're ready.
- **Web Key Directory (WKD):** if you control your email domain, you can serve your key from it so `gpg --locate-keys` finds it automatically.

## 6. Get someone else's public key

Always prefer the full fingerprint when you have it:

```bash
gpg --keyserver hkps://keys.openpgp.org --recv-keys BOB_FULL_FINGERPRINT
```

Or search by exact email address:

```bash
gpg --auto-key-locate clear,wkd,keyserver --locate-keys bob@example.org
```

Or import a key file Bob sent you:

```bash
gpg --import bob-public.asc
```

### Verify the fingerprint

A keyserver is a directory, not a notary. Anyone can upload a key with someone else's name. Compare the fingerprint with Bob using another channel, such as a phone call, in person, or his HTTPS website:

```bash
gpg --fingerprint bob@example.org
```

### Mark the key as trusted

Once the fingerprint matches, certify Bob's key locally:

```bash
gpg --lsign-key bob@example.org
```

`--lsign-key` makes a local-only signature. Use `--sign-key` only if you want a signature that can be exported and shared.

### Keep keys up to date

```bash
gpg --refresh-keys
```

This fetches new subkeys, expiry extensions and revocations for every key you hold.

## 7. Sign a message

Create a file to work with:

```bash
echo "Hello from Alice" > message.txt
```

### Clear-signed message (readable text with a signature)

```bash
gpg --local-user alice@example.org --clear-sign message.txt
```

This creates `message.txt.asc`, containing the readable text plus the signature. It suits emails and forum posts.

### Detached signature (for files)

```bash
gpg --local-user alice@example.org --armor --detach-sign file.tar.gz
```

This creates `file.tar.gz.asc` next to the original file, so you can publish the file and signature separately.

## 8. Verify a signature

You need the signer's public key in your keyring (step 6).

```bash
gpg --verify message.txt.asc
```

For a detached signature, give both files:

```bash
gpg --verify file.tar.gz.asc file.tar.gz
```

Look for `Good signature from "Alice <alice@example.org>"`. Also check the key fingerprint printed below it. A warning that the key is not certified means GnuPG hasn't verified who owns it, so check the fingerprint and run `gpg --lsign-key` as in step 6. `BAD signature` means the data or signature was altered, so don't trust the file.

## 9. Encrypt a message

You need the **recipient's public key**. To send an encrypted message to Bob:

```bash
gpg --armor --encrypt --recipient bob@example.org message.txt
```

This creates `message.txt.asc`, which only Bob's private key can decrypt. Send him that file.

### Encrypt and sign together

```bash
gpg --armor --sign --encrypt --local-user alice@example.org --recipient bob@example.org message.txt
```

Bob can then both decrypt the message and check that it came from you.

### Encrypt to multiple recipients, including yourself

```bash
gpg --armor --encrypt --recipient bob@example.org --recipient alice@example.org message.txt
```

You can't decrypt a message you encrypted only to someone else, so add yourself if you want to read it later.

### Encrypt with a passphrase only

For a quick file that needs no key pair:

```bash
gpg --armor --symmetric message.txt
```

## 10. Decrypt a message

Decrypt a message sent to you, using your private key and passphrase:

```bash
gpg --decrypt message.txt.asc
```

The text prints to the terminal. To save it to a file:

```bash
gpg --output message-decrypted.txt --decrypt message.txt.asc
```

If the message was also signed, GnuPG verifies the signature during decryption and prints `Good signature from ...`, as long as you have the sender's public key.

### Try it yourself

You can test the whole cycle alone by encrypting to your own key:

```bash
echo "Test message" > test.txt
gpg --armor --encrypt --recipient alice@example.org test.txt
gpg --decrypt test.txt.asc
```

## 11. Maintain your key

### Extend the expiry date

Extend the primary key and all subkeys by another two years from today:

```bash
gpg --quick-set-expire 0123456789ABCDEF0123456789ABCDEF01234567 2y
gpg --quick-set-expire 0123456789ABCDEF0123456789ABCDEF01234567 2y '*'
```

The first command changes the primary key and the second changes all subkeys. Then re-upload:

```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys 0123456789ABCDEF0123456789ABCDEF01234567
```

### Revoke a key

Import your saved revocation certificate and publish it:

```bash
gpg --import alice-revoke.asc
gpg --keyserver hkps://keys.openpgp.org --send-keys 0123456789ABCDEF0123456789ABCDEF01234567
```

## Command quick reference

| Task                  | Command                                                       |
| --------------------- | ------------------------------------------------------------- |
| Generate a key        | `gpg --quick-generate-key "Name <email>" default default 2y`  |
| List public keys      | `gpg --list-keys`                                             |
| List private keys     | `gpg --list-secret-keys --keyid-format long`                  |
| Show fingerprint      | `gpg --fingerprint email`                                     |
| Export public key     | `gpg --armor --export email`                                  |
| Import a key          | `gpg --import file.asc`                                       |
| Receive from keyserver| `gpg --recv-keys FINGERPRINT`                                 |
| Find by email         | `gpg --locate-keys email`                                     |
| Sign (clear text)     | `gpg --clear-sign file`                                       |
| Sign (detached)       | `gpg --armor --detach-sign file`                              |
| Verify                | `gpg --verify file.asc [file]`                                |
| Encrypt               | `gpg --armor --encrypt --recipient email file`                |
| Decrypt               | `gpg --decrypt file.asc`                                      |
| Refresh keys          | `gpg --refresh-keys`                                          |

## Troubleshooting

- **No public key when encrypting:** import the recipient's key first (step 6).
- **`gpg: key ...: no user ID`:** the key on keys.openpgp.org has no verified email address. Some distributions, including Debian, patch GnuPG to handle this, but upstream doesn't. Ask the owner to verify their address, or get the key from them directly.
- **Passphrase prompt doesn't appear over SSH or in a script:** run `export GPG_TTY=$(tty)` in your shell, and add it to `~/.bashrc`.
- **Old agent after upgrade:** run `gpgconf --kill all` and try again.
- **No keyserver available:** make sure `keyserver hkps://keys.openpgp.org` is in `~/.gnupg/dirmngr.conf`, and remove any old SKS-era entries.

## Security notes

- Never share, upload or commit your private key or revocation certificate.
- A keyserver doesn't prove identity. Always compare the full 40-character fingerprint, never a short 8-character ID.
- Encrypting hides the content, but not the fact that you communicated, and not the subject line if you use email.
- Keep GnuPG updated, and consider a hardware key such as a YubiKey or Nitrokey to hold your private key.
- Keyservers can't delete data on synchronizing networks, so only upload keys and identities you're comfortable keeping public forever.

## Further reading

- [GnuPG documentation](https://gnupg.org/documentation/)
- [GnuPG download and release status](https://gnupg.org/download/)
- [keys.openpgp.org: using GnuPG](https://keys.openpgp.org/about)
- [Keyoxide: Using OpenPGP with GnuPG](https://docs.keyoxide.org/gnupg/)