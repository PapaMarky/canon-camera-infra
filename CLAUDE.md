# Canon Camera Infrastructure

Dev/ops infrastructure for the Canon Camera Control workspace. Hosts the local package servers that target devices install from.

See @../CLAUDE.md for shared workspace conventions.

## What this is (and isn't)

- **Not an Application, Utility, or Base Unit.** No runtime code that ships to a device.
- **Is** the scripts and configuration that:
  - Stand up the local apt repository on the Pi5
  - Stand up the local PyPI server on the Pi5
  - Configure target devices to install from those servers
  - Manage packages on those servers (upload, list, remove, garbage-collect)

## Goals

- Reproducible: anyone can rebuild the infrastructure from this repo plus the Pi5 hardware.
- Operable: routine tasks (upload a new wheel, add a `.deb`, rotate keys) are scripted.
- Boring: the servers should be the dullest part of the workspace. Failures are loud; setup is one-shot per machine.

## Components

### Local Debian Repository
Serves `.deb` artifacts for `base-unit` and `pi-camera-control2`. See issue #1.

### Local PyPI Server
Serves Python wheels for `cc-client` (and future utility libraries). See issue #2.

### Client Configuration
`sources.list` snippets and `pip.conf` templates for target devices.

## Project Structure

```
canon-camera-infra/
├── debian-repo/        # apt repo setup runbook (setup.md)
├── pypi-server/        # PyPI server setup runbook (setup.md)
├── pi5-setup/          # Base Pi5 OS prep (common to both components)
└── client-config/      # sources.list, pip.conf templates for consumers
```

> **Delivered as documented runbooks, not scripts.** The package servers are rebuilt
> rarely (once per host, not in a daily/CI loop) and the steps are mostly `apt`/`venv`/
> `systemd`/`reprepro` one-liners, so each component ships as a `setup.md` an operator
> follows once, with copy-paste snippets — rather than a wrapper script to keep in sync.
> (Decision recorded on issue #2; the Debian repo follows the same model.)

## Conventions

- Shell scripts: bash, `set -euo pipefail` at the top, `shellcheck` clean.
- Configuration: host-specific values (Pi5 IP, hostname, user) live in a single config file. Do not hardcode them in scripts.
- Secrets (GPG private keys, server credentials) are NEVER committed. Keep them under `secrets/` (gitignored) and document how to generate them on the Pi5.
- Idempotent where reasonable: re-running a setup script on an already-configured machine should not break it.

## Consumers

| Repo | What it consumes from this infra |
|------|----------------------------------|
| `base-unit` | Publishes `.deb` to the local apt repo on release |
| `cc-client` | Publishes `.whl` to the local PyPI server on release |
| `pi-camera-control2` | Publishes `.deb` to the local apt repo; target devices install `cc-client` from the local PyPI server during postinst |
