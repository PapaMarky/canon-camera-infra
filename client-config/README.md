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

### CI (publishing wheels)

The `cc-client` release workflow uploads with `twine`. It does **not** use `pip.conf`; it needs the upload endpoint and the `uploader` credential:

- `TWINE_REPOSITORY_URL` = `http://pihost-002.local:8080/`
- `TWINE_USERNAME` = `uploader`
- `TWINE_PASSWORD` = stored as a repository secret (the htpasswd password from PyPI server [setup](../pypi-server/setup.md) step 3)

CI must run on a runner with LAN access to the host. See [PapaMarky/cc-client#13](https://github.com/PapaMarky/cc-client/issues/13).
