# Local Debian Repository Setup

Procedure for standing up the workspace's local apt repository on the package-server host. It serves `.deb` artifacts (`base-unit`, and later `pi-camera-control2`) to target devices.

**Verified on:** 2026-06-04 — `pihost-002`, Raspberry Pi OS Trixie (Debian 13), reprepro 5.3.1, nginx 1.26.3, GnuPG 2.4.7. A full build→sign→include→`apt install`→remove roundtrip was confirmed, including that a missing/incorrect client key makes `apt-get update` fail (see [Smoke test](#smoke-test)).

Prerequisite: the host has completed [base OS prep](../pi5-setup/basic-setup.md).

## Why `reprepro` (not `aptly`)

Issue #1 calls for a local apt repository. We use **`reprepro`**:

- It is a single binary that manages a `pool/` of `.deb` files and the signed `dists/` metadata that apt reads, driven by one config file. The on-disk tree *is* the repository — nothing else to reason about.
- It GPG-signs `Release`/`InRelease` out of the box, which is what lets apt trust the index over plain HTTP.
- Minimal dependencies and resource use, which suits a Raspberry Pi.
- `apt` consumes it with a stock `deb` source line and a `Signed-By` key — no special client tooling.

`aptly` adds snapshots, mirroring/promotion between indexes, a REST API, and a web UI. None of that is needed to serve a handful of first-party `.deb`s to a few devices, and each feature is one more moving part to operate and break. That conflicts with this repo's "boring infrastructure" goal — the same reasoning that picked `pypiserver` over `devpi` for the [PyPI server](../pypi-server/setup.md).

## 1. Create the server layout

All server state lives under `/srv/apt`, owned by the `pi` service account:

```sh
sudo install -d -o pi -g pi /srv/apt /srv/apt/conf /srv/apt/db /srv/apt/public /srv/apt/logs
install -d -m 700 /srv/apt/.gnupg
```

- `/srv/apt/public/` — the published `dists/` + `pool/` tree. This is the only subtree nginx serves.
- `/srv/apt/conf/` — reprepro configuration (`distributions`, `options`).
- `/srv/apt/db/` — reprepro's own database (it creates and owns the files here).
- `/srv/apt/.gnupg/` — a dedicated GnuPG home holding the repository signing key, so the key lives in the service account rather than a personal keyring.

## 2. Install tooling

```sh
sudo apt-get update
sudo apt-get install -y reprepro gnupg nginx dpkg-dev
```

`dpkg-dev` provides `dpkg-deb`, used by the smoke test to build a throwaway package.

## 3. Create the repository signing key

apt verifies the repository by its GPG signature, so the repo needs a signing key. Generate it unattended into the dedicated GnuPG home. The key has no passphrase so reprepro can sign non-interactively (from a service or CI); it is a private key that **never leaves the host and is never committed** — only its public half is distributed to clients.

```sh
export GNUPGHOME=/srv/apt/.gnupg
cat > /tmp/canon-apt-key.params <<'PARAMS'
%no-protection
Key-Type: RSA
Key-Length: 4096
Key-Usage: sign
Name-Real: Canon Camera Workspace Repository Signing Key
Name-Comment: pihost-002 local apt repo
Name-Email: canon-apt@pihost-002.local
Expire-Date: 0
%commit
PARAMS
gpg --batch --gen-key /tmp/canon-apt-key.params
rm /tmp/canon-apt-key.params
```

Capture the key's long id — `conf/distributions` references it below:

```sh
KEYID=$(gpg --list-keys --with-colons canon-apt@pihost-002.local | awk -F: '/^pub:/{print $5}')
echo "$KEYID"
```

Export the **public** key for clients. Commit this file to `client-config/` (public keys are safe to commit); it is installed on each target device:

```sh
gpg --armor --export "$KEYID" > /srv/apt/canon-apt-archive-keyring.asc
```

> Record nothing extra in a password manager — the key has no passphrase by design. To rotate it, generate a new key, repoint `SignWith`, re-run `reprepro export`, and redistribute the new public key. The private key is recoverable only from `/srv/apt/.gnupg`; back that directory up if the repo's continuity matters.

## 4. Configure the repository

`conf/distributions` defines the single `trixie` suite. Architecture-independent (`Architecture: all`) packages — which is what the first-party `.deb`s are — are served under the listed binary architecture automatically.

```sh
tee /srv/apt/conf/distributions >/dev/null <<DIST
Origin: Canon Camera Workspace
Label: canon-camera-infra
Codename: trixie
Architectures: arm64
Components: main
Description: Local apt repository for base-unit and pi-camera-control2
SignWith: $KEYID
DIST
```

`conf/options` keeps the published tree separate from `conf/`/`db/` (so nginx never serves them) and points reprepro at the service keyring:

```sh
tee /srv/apt/conf/options >/dev/null <<'OPTS'
outdir /srv/apt/public
gnupghome /srv/apt/.gnupg
verbose
OPTS
```

Generate the (empty, signed) index:

```sh
reprepro -b /srv/apt export
```

This creates `/srv/apt/public/dists/trixie/` with a signed `Release`/`InRelease`.

## 5. Serve the repository with nginx

apt fetches the static tree over HTTP; the GPG signature — not the transport — is what makes it trustworthy, so plain HTTP is correct here (no TLS to manage).

```sh
sudo tee /etc/nginx/sites-available/canon-apt >/dev/null <<'SITE'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name pihost-002.local;

    root /srv/apt/public;
    autoindex on;
}
SITE

sudo ln -sf /etc/nginx/sites-available/canon-apt /etc/nginx/sites-enabled/canon-apt
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

> Use `restart`, not `reload`, for this first activation. After removing the stock `default` site, a reload (SIGHUP) can keep serving the old default server; a restart cleanly swaps it in. Later config tweaks can use `reload`.

Confirm it is serving the index:

```sh
curl -fsS http://localhost/dists/trixie/InRelease >/dev/null && echo OK
```

> Port 80 must be reachable from the LAN. Opening it is tracked under the firewall work — [#5](https://github.com/PapaMarky/canon-camera-infra/issues/5).

## 6. Verify from another machine

From the operator's computer (replace the host if not `pihost-002`):

```sh
curl -fsS http://pihost-002.local/dists/trixie/InRelease >/dev/null && echo "index reachable"
```

## Smoke test

A one-time end-to-end check — build a throwaway `.deb`, publish it, install it through apt with signature verification, then remove it. Run the build on the host (or anywhere with `dpkg-deb`).

```sh
# Build a minimal throwaway package:
mkdir -p /tmp/canon-smoketest/DEBIAN
cat > /tmp/canon-smoketest/DEBIAN/control <<'CONTROL'
Package: canon-apt-smoketest
Version: 0.0.1
Architecture: all
Section: misc
Priority: optional
Maintainer: Canon Camera Workspace <canon-apt@pihost-002.local>
Description: Throwaway package to verify the local apt repo.
CONTROL
dpkg-deb --root-owner-group --build /tmp/canon-smoketest /tmp/canon-apt-smoketest_0.0.1_all.deb

# Publish it into the repo:
reprepro -b /srv/apt includedeb trixie /tmp/canon-apt-smoketest_0.0.1_all.deb
reprepro -b /srv/apt list trixie

# Configure this machine as a client and install through apt (see client-config):
sudo install -d -m 0755 /etc/apt/keyrings
sudo cp /srv/apt/canon-apt-archive-keyring.asc /etc/apt/keyrings/
sudo tee /etc/apt/sources.list.d/canon-camera.sources >/dev/null <<'SRC'
Types: deb
URIs: http://pihost-002.local
Suites: trixie
Components: main
Architectures: arm64
Signed-By: /etc/apt/keyrings/canon-apt-archive-keyring.asc
SRC
sudo apt-get update
sudo apt-get install -y canon-apt-smoketest
```

`apt-get update` must complete with no signature warning — that is the proof the signing chain works. (Removing or corrupting the keyring makes `apt-get update` fail loudly, which is the intended behavior.)

Clean up afterward — uninstall the test package, remove it from the repo, and drop the test client config:

```sh
sudo apt-get purge -y canon-apt-smoketest
reprepro -b /srv/apt remove trixie canon-apt-smoketest
sudo rm /etc/apt/sources.list.d/canon-camera.sources
rm -rf /tmp/canon-smoketest /tmp/canon-apt-smoketest_0.0.1_all.deb
```

## Operations

reprepro has no daemon — operating the repo is running `reprepro -b /srv/apt <command>` on the host. All commands read `conf/` and update `public/` in place; the index is re-signed automatically.

### Add a `.deb`

```sh
reprepro -b /srv/apt includedeb trixie path/to/package_1.2.3_arm64.deb
```

> reprepro is stricter than `dpkg` about control metadata: the `.deb` must declare a `Section` and `Priority`, or `includedeb` rejects it with `No section given … skipping`. Packages built by `base-unit`/`pi-camera-control2` should set both in their control file.

Sibling repos publish by copying their built `.deb` to the host and running the above:

```sh
scp package_1.2.3_arm64.deb pi@pihost-002.local:/tmp/
ssh pi@pihost-002.local 'reprepro -b /srv/apt includedeb trixie /tmp/package_1.2.3_arm64.deb'
```

> Wiring the `base-unit` and `pi-camera-control2` release workflows to do this automatically is tracked in those repos, not here.

### List what is served

```sh
reprepro -b /srv/apt list trixie
# or from any client:
curl -fsS http://pihost-002.local/dists/trixie/main/binary-arm64/Packages
```

### Remove a package

```sh
reprepro -b /srv/apt remove trixie <package-name>
reprepro -b /srv/apt deleteunreferenced   # prune pool files no longer referenced
```

> reprepro keeps only the latest version of a package per suite by default; `includedeb` of a newer version supersedes the old one, and `deleteunreferenced` removes the orphaned pool file.

## Related work

- Client (target-device) configuration to install from this repo — [client-config](../client-config/README.md)
- Firewall rule to expose port 80 on the LAN — [#5](https://github.com/PapaMarky/canon-camera-infra/issues/5)
- Static IP / DHCP reservation so `pihost-002.local` is stable — [#6](https://github.com/PapaMarky/canon-camera-infra/issues/6)
- Local PyPI server (the sibling deliverable this mirrors) — [pypi-server](../pypi-server/setup.md)
