# Local PyPI Server Setup

Procedure for standing up the workspace's local PyPI server on the package-server host. It serves Python wheels (currently `cc-client`, later other utility libraries) to target devices and to CI.

**Verified on:** 2026-06-04 — `pihost-002`, Raspberry Pi OS Trixie (Debian 13), Python 3.13.5, pypiserver 2.4.1. Auth enforcement (anonymous upload → `403`) and a full upload→install roundtrip were confirmed (see [Smoke test](#smoke-test)).

Prerequisite: the host has completed [base OS prep](../pi5-setup/basic-setup.md).

## Why `pypiserver` (not `devpi`)

Issue #2 calls for evaluating `pypiserver` vs `devpi`. We use **`pypiserver`**:

- It is a single process that serves a directory of packages as a PEP 503 "simple" index. The on-disk state *is* the package set — nothing else to reason about.
- Minimal dependencies and resource use, which suits a Raspberry Pi.
- `twine upload` and `pip install --index-url` work against it with no special client tooling.
- Basic-auth on uploads via an htpasswd file; anonymous reads on the trusted LAN.

`devpi` adds staging indexes, mirroring/replication, multi-index layouts, and a richer web UI. None of that is needed to serve a handful of first-party wheels to a few devices, and each feature is one more moving part to operate and break. That conflicts with this repo's "boring infrastructure" goal.

## 1. Create the server layout

All server state lives under `/srv/pypi`, owned by the `pi` service account:

```sh
sudo install -d -o pi -g pi /srv/pypi /srv/pypi/packages
```

- `/srv/pypi/packages/` — the wheels. Adding/removing a file here adds/removes a package.
- `/srv/pypi/venv/` — an isolated interpreter for the server (created next).
- `/srv/pypi/.htpasswd` — upload credentials (created in step 3).

## 2. Install pypiserver into a venv

Raspberry Pi OS Trixie marks the system Python as externally managed (PEP 668), so install the server into its own virtualenv rather than system-wide:

```sh
sudo apt-get install -y python3-venv
python3 -m venv /srv/pypi/venv
/srv/pypi/venv/bin/pip install --upgrade pip
/srv/pypi/venv/bin/pip install pypiserver passlib
```

`passlib` lets `pypiserver` read the htpasswd file without pulling in Apache tooling.

## 3. Create upload credentials

Reads are anonymous on the LAN; uploads require auth. Create an htpasswd entry for a single `uploader` account (used by humans and by CI):

```sh
sudo apt-get install -y apache2-utils
htpasswd -c /srv/pypi/.htpasswd uploader      # prompts for a password
sudo chown pi:pi /srv/pypi/.htpasswd
sudo chmod 600 /srv/pypi/.htpasswd
```

> Record the password in the operator's password manager. CI stores it as a repository secret (see [client-config](../client-config/README.md)). Rotating it is a re-run of `htpasswd /srv/pypi/.htpasswd uploader` (without `-c`, which would truncate the file) followed by `systemctl restart pypi-server`.

The upload examples below read the password from `$PYPI_UPLOAD_PASSWORD`. Export it in your shell first so it stays out of process listings and history:

```sh
read -rs PYPI_UPLOAD_PASSWORD && export PYPI_UPLOAD_PASSWORD      # paste the password, press Enter
```

## 4. Install the systemd service

```sh
sudo tee /etc/systemd/system/pypi-server.service >/dev/null <<'UNIT'
[Unit]
Description=Local PyPI server for the Canon Camera workspace
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
Group=pi
ExecStart=/srv/pypi/venv/bin/pypi-server run \
    --port 8080 \
    --passwords /srv/pypi/.htpasswd \
    --authenticate update \
    /srv/pypi/packages
Restart=on-failure

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now pypi-server
```

`--authenticate update` requires auth only for the `update` (upload) action; `list` and `download` stay anonymous.

Confirm it is running and listening:

```sh
systemctl status pypi-server --no-pager
curl -fsS http://localhost:8080/ >/dev/null && echo OK
```

> Port 8080 must be reachable from the LAN. It is opened by the host firewall — see [pi5-setup/firewall.md](../pi5-setup/firewall.md) ([#5](https://github.com/PapaMarky/canon-camera-infra/issues/5)).

## 5. Verify from another machine

From the operator's computer (replace the host if not `pihost-002`):

```sh
curl -fsS http://pihost-002.local:8080/ >/dev/null && echo "index reachable"
```

## Smoke test

A one-time check that auth and the upload→install path both work, using a throwaway package (no first-party wheels needed). Run from any machine with `build` and `twine`:

```sh
python3 -m venv /tmp/cc-smoke && /tmp/cc-smoke/bin/pip install -q build twine
mkdir -p /tmp/smokepkg/src/ccinfra_smoketest && cd /tmp/smokepkg
cat > pyproject.toml <<'EOF'
[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"
[project]
name = "ccinfra-smoketest"
version = "0.0.1"
EOF
: > src/ccinfra_smoketest/__init__.py
/tmp/cc-smoke/bin/python -m build --wheel -o dist .

# Upload with bad credentials must be rejected (expect 403; no credentials gives 401):
/tmp/cc-smoke/bin/twine upload --repository-url http://pihost-002.local:8080/ -u x -p x dist/*.whl

# Upload with the uploader credential must succeed:
/tmp/cc-smoke/bin/twine upload --repository-url http://pihost-002.local:8080/ \
  -u uploader -p "$PYPI_UPLOAD_PASSWORD" dist/*.whl

# Install it back from the index in a clean venv:
python3 -m venv /tmp/cc-consume
/tmp/cc-consume/bin/pip install --index-url http://pihost-002.local:8080/simple/ \
  --trusted-host pihost-002.local ccinfra-smoketest
```

Clean up afterward — delete the test wheel from the server and the temp dirs:

```sh
ssh pi@pihost-002.local 'rm /srv/pypi/packages/ccinfra_smoketest-0.0.1-py3-none-any.whl'
rm -rf /tmp/cc-smoke /tmp/cc-consume /tmp/smokepkg
```

## Operations

`pypiserver` has no database — operating it is reading and writing files under `/srv/pypi/packages/`.

### Upload a wheel

With [twine](https://twine.readthedocs.io/) from a machine that has the built wheel:

```sh
twine upload \
  --repository-url http://pihost-002.local:8080/ \
  --username uploader --password "$PYPI_UPLOAD_PASSWORD" \
  dist/*.whl
```

For repeated manual uploads, add a `[local]` entry to `~/.pypirc` instead of passing flags each time:

```ini
[distutils]
index-servers = local

[local]
repository = http://pihost-002.local:8080/
username = uploader
password = <password>
```

…then `twine upload --repository local dist/*.whl`.

### List what is served

```sh
# On the host:
ls -1 /srv/pypi/packages/

# From any client (human-readable package listing):
curl -fsS http://pihost-002.local:8080/packages/
```

### Remove a wheel

Removal is deletion of the file on the host; the index updates immediately:

```sh
ssh pi@pihost-002.local 'rm /srv/pypi/packages/cc_client-0.1.0-py3-none-any.whl'
```

> There is no garbage-collection daemon. To prune old versions, delete the specific wheel files you no longer want to serve. Keep at least the versions any deployed device still pins.

## Related work

- Client (target-device and CI) configuration to install from this index — [client-config](../client-config/README.md)
- Firewall opening port 8080 on the LAN — **set up**: [pi5-setup/firewall.md](../pi5-setup/firewall.md) ([#5](https://github.com/PapaMarky/canon-camera-infra/issues/5)).
- Host addressing uses mDNS (`pihost-002.local`) **by design**; a static IP / DHCP reservation was **declined** (nothing hardcodes the IP, and the home router doesn't support reservations well) — [#6](https://github.com/PapaMarky/canon-camera-infra/issues/6), closed won't-do.
- cc-client release workflow publishes wheels here — **done**, [PapaMarky/cc-client#13](https://github.com/PapaMarky/cc-client/issues/13).
