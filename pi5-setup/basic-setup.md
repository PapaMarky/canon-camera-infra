# Basic Pi Setup

Procedure for preparing a Raspberry Pi to host the workspace's local Debian repository and local PyPI server.

**Verified on:** 2026-06-04 — Raspberry Pi 5 (16 GB), Raspberry Pi OS Trixie (Debian 13), host `pihost-002`.

## Prerequisites

- Raspberry Pi 5
- microSD card (or USB-attached SSD) — 32 GB minimum; larger is better as the package repository will grow
- Wired Ethernet — recommended for a server
- A computer with [Raspberry Pi Imager](https://www.raspberrypi.com/software/) installed
- The operator's SSH public key, ready to paste in

## 1. Flash the OS image

> The Raspberry Pi Imager UI evolves between versions; field and button labels may shift. The settings below describe the configuration intent — locate the equivalent control in whichever Imager version is current.

Using Raspberry Pi Imager:

1. Choose Device: **Raspberry Pi 5**.
2. Choose OS: **Raspberry Pi OS Lite (64-bit)** (Debian Trixie).
3. Choose Storage: the target SD card or SSD.
4. Click **Next** and open **Edit Settings**:
   - **General → Hostname:** `pihost-NNN` (replace `NNN` with the next available number; this host is `pihost-002`).
   - **General → Username:** `pi`.
   - **General → Password:** set a strong password (we will use key-based SSH, but the Imager still requires a password).
   - **General → Wi-Fi:** configure the home Wi-Fi network as a backup path. Ethernet is the primary link for a server, but having Wi-Fi configured means the host remains reachable if the cable is unplugged or the switch port goes down.
   - **General → Locale:** set the appropriate time zone and keyboard.
   - **Services → Enable SSH:** select **public-key authentication only** and paste the operator's SSH public key.
5. Write the image.

The Imager applies these settings on first boot: it creates the `pi` user and installs the SSH key into `~pi/.ssh/authorized_keys`.

> **Trixie note:** unlike older Bookworm images, the Trixie Imager does **not** grant the first user passwordless sudo — `sudo` prompts for a password. Step 2 adds the `NOPASSWD` drop-in so the rest of this runbook (and remote automation over SSH) can run `sudo` non-interactively.

## 2. First boot and verify connectivity

1. Insert the storage into the Pi.
2. Connect Ethernet to the LAN. If Wi-Fi was also configured, both interfaces will come up; NetworkManager prefers the wired link automatically via a lower routing metric, and `<hostname>.local` resolves to the wired IP.
3. Power on. First boot takes roughly 60–90 seconds (filesystem expansion, network bring-up, hostname publishing).
4. From the operator's computer:

   ```sh
   ssh pi@<hostname>.local
   ```

   If mDNS resolution fails, find the Pi's IP from your router's DHCP table and connect by IP.

5. Grant the `pi` user passwordless sudo. The Trixie Imager does not configure this, but the remaining steps (and remote automation over SSH) run `sudo` non-interactively, so add the drop-in — it prompts for your password once:

   ```sh
   echo 'pi ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/010_pi-nopasswd >/dev/null
   sudo chmod 440 /etc/sudoers.d/010_pi-nopasswd
   sudo -n true && echo "passwordless sudo OK"
   ```

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

## 5. Install base packages

Install the minimum tooling needed to operate the host. Component-specific tooling (apt-repo manager, GPG signing keys, PyPI server, etc.) is installed by the Debian repo and PyPI server runbooks ([debian-repo/setup.md](../debian-repo/setup.md), [pypi-server/setup.md](../pypi-server/setup.md)); this step is only the host-level basics.

```sh
sudo apt-get install -y git
```

## 6. Harden SSH

Disable password authentication and root login. SSH access on a server should be key-only.

Drop the hardening config into `/etc/ssh/sshd_config.d/` so the main `sshd_config` stays unmodified (Trixie's stock `sshd_config` already includes drop-ins from that directory). Trixie also ships a `50-cloud-init.conf` drop-in that already sets `PasswordAuthentication no`; the file below sorts ahead of it and additionally enforces `PermitRootLogin no`:

```sh
sudo tee /etc/ssh/sshd_config.d/10-canon-hardening.conf >/dev/null <<'CONF'
# Canon workspace package-server hardening.
# Key-based auth only; no root login.
PasswordAuthentication no
PermitRootLogin no
CONF
sudo chmod 644 /etc/ssh/sshd_config.d/10-canon-hardening.conf
```

Validate the config, confirm the effective settings, then reload `sshd`:

```sh
sudo sshd -t                                                          # syntax check
sudo sshd -T | grep -iE '^(passwordauthentication|permitrootlogin) '  # effective values
sudo systemctl reload ssh
```

**Verify from the operator's computer in a new shell — without closing the existing session** — that key-based SSH still works:

```sh
ssh pi@<hostname>.local 'echo OK'
```

If the new connection fails, recover from the existing session:

```sh
sudo rm /etc/ssh/sshd_config.d/10-canon-hardening.conf
sudo systemctl reload ssh
```

## Related work

This document covers the **base OS prep**. The rest of the package-server stack is tracked separately:

- Local Debian repository — **set up**: [debian-repo/setup.md](../debian-repo/setup.md) ([#1](https://github.com/PapaMarky/canon-camera-infra/issues/1), closed).
- Local PyPI server — **set up**: [pypi-server/setup.md](../pypi-server/setup.md) ([#2](https://github.com/PapaMarky/canon-camera-infra/issues/2), closed).
- Firewall — **set up** with ufw: [firewall.md](firewall.md) ([#5](https://github.com/PapaMarky/canon-camera-infra/issues/5), closed).
- Host addressing uses mDNS (`pihost-002.local`) **by design**; a static IP / DHCP reservation was **declined** (nothing hardcodes the IP, and the home router doesn't support reservations well) — [#6](https://github.com/PapaMarky/canon-camera-infra/issues/6), closed won't-do.
