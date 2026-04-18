# GilmanLab Docs

This repository is the dedicated documentation home for the GilmanLab
homelab.

The root documents point into the Docusaurus site source under `docs/`, and
the site is the primary surface for architecture notes, hardware references,
runbooks, and future how-to material.

## Quick Start

Prerequisites:

- `moon` 2.x
- Node.js 22, or Moon-managed access to the pinned Node/npm toolchain

Validate the documentation site:

```sh
moon ci --summary minimal
moon run docs:check
```

Start the local docs server:

```sh
moon run docs:start
```

## Current Content

- [`docs/docs/index.md`](docs/docs/index.md): docs landing page
- [`docs/docs/architecture.md`](docs/docs/architecture.md): architecture overview
- [`docs/docs/hardware.md`](docs/docs/hardware.md): hardware inventory
- [`docs/docs/network-device-backups.md`](docs/docs/network-device-backups.md): RouterOS backup design for the future platform cluster

## Support

- Questions and design discussion: GitHub Discussions
- Non-security bugs: GitHub Issues
- Vulnerabilities: follow [SECURITY.md](SECURITY.md)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
