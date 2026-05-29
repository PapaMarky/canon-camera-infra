# Basic Pi Setup

Procedure for preparing a Raspberry Pi to host the workspace's local Debian repository and local PyPI server.

**Verified on:** 2026-05-29 — Raspberry Pi 5 (16 GB), Raspberry Pi OS Bookworm (Debian 12), host `pihost-002`.

## Prerequisites

- Raspberry Pi 5
- microSD card (or USB-attached SSD) — 32 GB minimum; larger is better as the package repository will grow
- Wired Ethernet — recommended for a server
- A computer with [Raspberry Pi Imager](https://www.raspberrypi.com/software/) installed
- The operator's SSH public key, ready to paste in

## 1. Flash the OS image

Using Raspberry Pi Imager:

1. Choose Device: **Raspberry Pi 5**.
2. Choose OS: **Raspberry Pi OS Lite (64-bit)** (Debian Bookworm).
3. Choose Storage: the target SD card or SSD.
4. Click **Next** and open **Edit Settings**:
   - **General → Hostname:** `pihost-NNN` (replace `NNN` with the next available number; this host is `pihost-002`).
   - **General → Username:** `pi`.
   - **General → Password:** set a strong password (we will use key-based SSH, but the Imager still requires a password).
   - **General → Wi-Fi:** optional. **Prefer Ethernet for a server.** Add Wi-Fi only if Ethernet is not available at the target location.
   - **General → Locale:** set the appropriate time zone and keyboard.
   - **Services → Enable SSH:** select **public-key authentication only** and paste the operator's SSH public key.
5. Write the image.

The Imager applies these settings on first boot, including creating the `pi` user with NOPASSWD sudo (via the `/etc/sudoers.d/010_pi-nopasswd` drop-in) and installing the SSH key into `~pi/.ssh/authorized_keys`.

## 2. First boot and verify connectivity

1. Insert the storage into the Pi.
2. Connect Ethernet (or rely on the configured Wi-Fi).
3. Power on. First boot takes roughly 60–90 seconds (filesystem expansion, network bring-up, hostname publishing).
4. From the operator's computer:

   ```sh
   ssh pi@<hostname>.local
   ```

   If mDNS resolution fails, find the Pi's IP from your router's DHCP table and connect by IP.

## 3. Update the OS to the latest packages

A freshly flashed image is generally weeks-to-months behind current. Update before doing anything else:

```sh
sudo apt-get update
sudo apt-get full-upgrade
sudo apt-get autoremove
sudo apt-get autoclean
```

Expect kernel, libc, systemd, and OpenSSH updates on a stale image.

## 4. Reboot

If `apt-get full-upgrade` touched the kernel, libc, or systemd, reboot:

```sh
sudo reboot
```

> **Note:** Raspberry Pi OS does not install `update-notifier-common`, so the `/var/run/reboot-required` marker is never created. Its absence is not evidence that a reboot is not needed. Treat any of the following package upgrades as requiring a reboot:
>
> - `linux-image-*`
> - `libc6`
> - `systemd`
> - `openssh-server` (a daemon restart often suffices, but a reboot is cleaner)

Reconnect via SSH and confirm:

```sh
uname -a              # running kernel should match the installed package
systemctl --failed    # should report zero failed units
```

## Remaining work

This document covers the **base OS prep** only. The following items will be folded into their own scripted setup steps as the rest of the infrastructure work proceeds:

- Base package install (`git`, `gpg` operator keys, repo-management tooling) — TBD
- SSH hardening (disable password authentication; disable root login) — TBD
- Firewall (`ufw` or `nft`) — TBD
- Static IP / DHCP reservation for the host — TBD
- Local Debian repository setup — [#1](https://github.com/PapaMarky/canon-camera-infra/issues/1)
- Local PyPI server setup — [#2](https://github.com/PapaMarky/canon-camera-infra/issues/2)
