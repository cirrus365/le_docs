---
title: Setting Up the Fingerprint Reader on Fedora 43 for a Lenovo Yoga X13
description: How to install fprintd, enroll fingerprints, and work around the Fedora 43 upgrade issues affecting the Synaptics Prometheus MIS touch fingerprint reader.
weight: 3
categories: [tutorial]
tags: [fedora, lenovo, linux]
---

## Overview

Enabling the fingerprint reader on Fedora 43 for a Lenovo Yoga X13 involves installing `fprintd` and configuring authentication, but recent reports indicate compatibility issues post-upgrade. This guide walks through the basic setup, the known Fedora 43 issues, and the fixes that restore working fingerprint login.

## Basic Setup

Install the `fprintd` package:

```shell
sudo dnf install fprintd
```

Enable its service:

```shell
sudo systemctl enable --now fprintd
```

Enroll your fingerprints, touching the sensor multiple times when prompted:

```shell
fprintd-enroll
```

Verify the enrollment:

```shell
fprintd-verify
```

Enable fingerprint authentication with `authselect`:

```shell
sudo authselect enable-feature with-fingerprint
```

Apply the authentication changes:

```shell
sudo authselect apply-changes
```

Finally, reboot.

## Verify Your Sensor

Check that the reader shows up on the USB bus:

```shell
lsusb
```

On a Yoga X13 this reports a Synaptics Prometheus MIS Touch Fingerprint Reader:

```shell
Bus 001 Device 004: ID 06cb:00bd Synaptics, Inc. Prometheus MIS Touch Fingerprint Reader
```

The [Synaptics Prometheus MIS Touch Fingerprint Reader](https://linux-hardware.org/?id=usb%3A06cb-00bd) (USB ID `06cb:00bd`) is detected via `lsusb`, but lacks full native Linux kernel driver support up to recent versions, leading to inconsistent `fprintd` functionality on Fedora 43.

## Fedora 43 Known Issues

{{% callout warning %}}
Users report fingerprint readers failing after upgrading to Fedora 43 on Lenovo devices like the Yoga X13, with errors like `No devices available` during enrollment and missing KDE settings options. The sensor appears in `lsusb`, but `fprintd` cannot access it reliably.
{{% /callout %}}

Some suggest installing a Validity sensor driver from a COPR repo such as `python-validity`, or checking the UEFI BIOS for fingerprint enablement.

## Compatibility Status

This sensor shows mixed results across Linux distros: it works out-of-the-box on some Lenovo ThinkPads (for example T14, P14) with `fprintd` after firmware updates via `fwupd`, but fails frequently on others such as L13 Yoga variants and on post-Fedora 43 upgrades. No dedicated kernel driver exists in standard repos — everything relies on `libfprint`, and enrollment often fails with `No devices available` despite USB detection.

## Setup Attempts

Install `fprintd-pam` if missing:

```shell
sudo dnf install fprintd fprintd-pam
```

Enable fingerprint authentication in one go:

```shell
sudo authselect enable-feature with-fingerprint && sudo authselect apply-changes
```

Try enrolling again, touching the sensor repeatedly:

```shell
fprintd-enroll
```

Update the sensor firmware with `fwupd`:

```shell
sudo fwupdtool refresh && sudo fwupdtool update
```

Or enable the testing repo for Synaptics fixes.

## Troubleshooting and Common Fixes

- Confirm the device with `lsusb` and check the [fprint supported devices list](https://fprint.freedesktop.org/supported-devices.html).
- Verify the UEFI BIOS enables the fingerprint reader under **Security > I/O Port Access**.
- List the devices `fprintd` can see and reboot after changes:

```shell
fprintd-list $(busctl --user get-property org.fprintd /org/fprintd org.fprintd.Manager Devices)
```

- If enrollment fails, try a clean install or wait for kernel/`fprintd` updates, as session login may work partially while `sudo` does not.
- If KDE/GNOME login fails but `sudo` works partially, downgrade the kernel or await `libfprint` 1.94+ updates; a clean Fedora 43 install may help more than an upgrade.

---

Adapted from [Setting up the fingerprint reader on Fedora 43 for a Lenovo Yoga X13](https://www.truefaxmedia.com/post/2026/setting-up-the-fingerprint-reader-on-fedora-43-for-a-lenovo-yoga-x13/) on Truefax Media.
