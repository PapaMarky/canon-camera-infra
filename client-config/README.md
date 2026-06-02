# Client Configuration

How consumers install packages from the workspace's local servers. This directory holds the templates target devices and CI use.

## pip — install from the local PyPI index

The local PyPI server is served over plain HTTP on the trusted LAN, so pip needs a `trusted-host` entry (which suppresses the "not HTTPS" warning/error). TLS is intentionally omitted to avoid certificate management on an internal-only service; revisit if the index is ever exposed beyond the LAN.

### Target devices

Install [`pip.conf`](pip.conf) to `/etc/pip.conf` (system-wide) so every install on the device can resolve from the local index:

```sh
sudo install -m 644 pip.conf /etc/pip.conf
```

The `pi-camera-control2` `.deb` postinst installs `cc-client` via pip; with this file in place that install finds `cc-client` on the local server with no extra flags.

> **Why `extra-index-url`, and why piwheels is re-listed.** Raspberry Pi OS ships a stock `/etc/pip.conf` containing `extra-index-url = https://www.piwheels.org/simple` — piwheels provides prebuilt ARM wheels, which is what keeps dependency installs on a Pi from compiling from source. This file overwrites that stock config, so it **re-includes piwheels** alongside the local server. The result is three indexes: PyPI (default) and piwheels for *dependencies*, plus the local server for *first-party* wheels (`cc-client`). The local server is an `extra-index-url`, not the sole `index-url`, precisely so dependency resolution keeps working against PyPI/piwheels — only `cc-client` lives on the local server, not its dependency tree.

> **⚠️ Dependency-confusion risk.** With multiple indexes configured via `extra-index-url`, pip does **not** prefer the local server for a given name — it collects candidate versions from *all* indexes and installs the **highest version number**, regardless of source. `cc-client` is a private name that does not exist on public PyPI; if someone registers a package named `cc-client` there with a version higher than ours, pip would install **their** code from PyPI instead of our wheel from the local server. This is the classic "dependency confusion" supply-chain attack. It is an accepted trade-off for keeping piwheels (switching to a sole `index-url` would drop piwheels' prebuilt ARM wheels). **Mitigation:** defensively register an empty `cc-client` placeholder on public PyPI under an account we control, so the namespace is not available to an attacker — tracked in [issue #8](https://github.com/PapaMarky/canon-camera-infra/issues/8).

### CI (publishing wheels)

The `cc-client` release workflow uploads with `twine`. It does **not** use `pip.conf`; it needs the upload endpoint and the `uploader` credential:

- `TWINE_REPOSITORY_URL` = `http://pihost-002.local:8080/`
- `TWINE_USERNAME` = `uploader`
- `TWINE_PASSWORD` = stored as a repository secret (the htpasswd password from PyPI server [setup](../pypi-server/setup.md) step 3)

CI must run on a runner with LAN access to the host. See [PapaMarky/cc-client#13](https://github.com/PapaMarky/cc-client/issues/13).

## apt — install from the local Debian repo

The local apt repository is served over plain HTTP, but — unlike the pip case above — that needs no `trusted-host` workaround. apt verifies the repository by its **GPG signature**, so integrity and authenticity come from the signature, not the transport. TLS is intentionally omitted for the same internal-only reason.

### Target devices

Install the repository's public signing key and the source definition:

```sh
sudo install -d -m 0755 /etc/apt/keyrings
sudo install -m 644 canon-apt-archive-keyring.asc /etc/apt/keyrings/
sudo install -m 644 canon-camera.sources /etc/apt/sources.list.d/canon-camera.sources
sudo apt-get update
```

After this, `apt-get install base-unit` (and other first-party packages) resolves from the local repo. The `pi-camera-control2` `.deb` depends on `base-unit`, so apt pulls it from here automatically.

- [`canon-camera.sources`](canon-camera.sources) — deb822 source, installed to `/etc/apt/sources.list.d/`.
- [`canon-apt-archive-keyring.asc`](canon-apt-archive-keyring.asc) — the repository's **public** signing key (the private half never leaves the host). The `Signed-By` line in the source pins trust to exactly this key, so it applies *only* to this repo — no other apt source is affected.

> **No dependency-confusion analog here.** Unlike pip's `extra-index-url` (which merges candidates across indexes by version), an apt source is scoped to its own suite/components and pinned to a specific signing key. apt will not silently pull a same-named package from a different, untrusted source.

### Publishing `.deb`s

Adding packages to the repo is an operator/CI step run on the host (`reprepro includedeb`), not a client concern. See [debian-repo/setup.md](../debian-repo/setup.md) → Operations.
