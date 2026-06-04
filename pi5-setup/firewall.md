# Package-Server Firewall

Procedure for putting a default-deny firewall on the package-server host. The trust model is "everything on the home LAN is trusted," so this is cheap insurance against a compromised LAN device rather than a hard requirement — it limits the host's exposed surface to exactly the ports the package servers need.

**Verified on:** 2026-06-04 — `pihost-002`, Raspberry Pi OS Trixie (Debian 13), ufw 0.36.2-9.

Prerequisite: the host has completed [base OS prep](basic-setup.md).

## Why `ufw` (not raw `nftables`)

- `ufw` is a thin, opinionated front-end over the kernel's netfilter: default-deny inbound, allow outbound, and one readable line per opened port (`ufw allow 80/tcp`). The runbook *is* the ruleset.
- A host that just needs ~5 ports open doesn't benefit from hand-writing an `nftables` ruleset; that's more to write, read, and keep correct for no gain here.
- `nftables` is the right tool when you need NAT, multiple tables/chains, or fine-grained policy — none of which this host has. Same "boring infrastructure" reasoning that picked `pypiserver` and `reprepro`.

## Ports

| Port | Proto | For |
|------|-------|-----|
| 22 | tcp | SSH (operator + remote automation) |
| 80 | tcp | apt repository (nginx) — [debian-repo](../debian-repo/setup.md) |
| 8080 | tcp | PyPI server (pypiserver) — [pypi-server](../pypi-server/setup.md) |
| 5353 | udp | mDNS (avahi) — so `pihost-002.local` resolves on the LAN |

ICMP (ping) is allowed by ufw's default `before.rules` (echo-request), so troubleshooting works with no extra rule. Outbound is allowed (so apt/pip, DHCP, NTP, etc. keep working); only inbound is filtered.

## 1. Install ufw

```sh
sudo apt-get install -y ufw
```

## 2. Apply the policy — lockout-safe

Enabling a default-deny firewall over SSH risks locking yourself out. Two safeguards: **allow `22/tcp` before enabling**, and arm a **dead-man timer** that disables ufw after 3 minutes unless you cancel it once you've confirmed a fresh connection still works.

```sh
# Safety net: auto-disable ufw in 180s unless cancelled below (guards against lockout).
sudo systemd-run --on-active=180 --unit=ufw-deadman /usr/sbin/ufw --force disable

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp   comment 'ssh'
sudo ufw allow 80/tcp   comment 'apt repo (nginx)'
sudo ufw allow 8080/tcp comment 'pypi server'
sudo ufw allow 5353/udp comment 'mDNS (avahi)'
sudo ufw --force enable
sudo ufw status verbose
```

Enabling does not drop established connections, so your current session survives.

## 3. Confirm connectivity, then cancel the dead-man

**From the operator's computer, open a _new_ shell** (do not close the existing session) and confirm a fresh login and the services still answer:

```sh
ssh pi@pihost-002.local 'echo OK'
curl -fsS http://pihost-002.local/dists/trixie/InRelease >/dev/null && echo "apt OK"
curl -fsS http://pihost-002.local:8080/ >/dev/null && echo "pypi OK"
ping -c1 pihost-002.local >/dev/null && echo "icmp OK"
```

Once all of that works, cancel the safety timer:

```sh
sudo systemctl stop ufw-deadman.timer 2>/dev/null
sudo systemctl reset-failed 'ufw-deadman.*' 2>/dev/null
```

If a fresh connection ever fails, do nothing — the dead-man timer disables ufw within 3 minutes and access returns. Recover from console access if it has already fired and you need to retry.

ufw enables itself on boot (`systemctl is-enabled ufw` → `enabled`); no extra step.

## Operations

```sh
sudo ufw status verbose         # show the active ruleset
sudo ufw allow <port>/<proto>   # open a port (add a comment with 'comment "..."')
sudo ufw delete allow <port>/<proto>   # close a port
sudo ufw disable                # turn the firewall off (e.g. for debugging)
```

When a new service is added to the host, open its port here and record it in the [Ports](#ports) table.

## Related work

- Base OS prep (SSH hardening, etc.) — [basic-setup.md](basic-setup.md)
- Services whose ports are opened here — [debian-repo](../debian-repo/setup.md) (80), [pypi-server](../pypi-server/setup.md) (8080)
