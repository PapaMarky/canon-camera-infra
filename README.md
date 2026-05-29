# canon-camera-infra

Local package infrastructure for the Canon Camera Control workspace. Provides scripts and configuration for hosting an apt repository and a PyPI server on a Pi5, plus the client-side configuration target devices need to consume them.

## Sibling repos this serves

- [base-unit](https://github.com/PapaMarky/base-unit) — publishes `.deb` to the local apt repo
- [cc-client](https://github.com/PapaMarky/cc-client) — publishes `.whl` to the local PyPI server
- [pi-camera-control2](https://github.com/PapaMarky/pi-camera-control2) — publishes `.deb`; target devices install `cc-client` from the local PyPI server during postinst

See [CLAUDE.md](CLAUDE.md) for design notes and conventions, and the workspace [`../CLAUDE.md`](../CLAUDE.md) for shared project conventions.
