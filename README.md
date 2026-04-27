# GilmanLab Docs

This repository is the dedicated documentation home for the GilmanLab homelab.

The Docusaurus site source lives under `docs/`. The architecture set is the
canonical starting point for the current lab design, with runbooks and
implementation details added separately as prototypes become real workflows.

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
- [`docs/docs/architecture.md`](docs/docs/architecture.md): architecture entrypoint and reading order
- [`docs/docs/architecture/`](docs/docs/architecture): focused architecture documents, including Keycloak runtime and recovery
- [`docs/docs/hardware.md`](docs/docs/hardware.md): hardware inventory
- [`docs/docs/network-device-backups.md`](docs/docs/network-device-backups.md): RouterOS backup design for the future platform cluster
- [`docs/docs/routeros-acme.md`](docs/docs/routeros-acme.md): RouterOS ACME certificate notes

## Support

- Questions and design discussion: GitHub Discussions
- Non-security bugs: GitHub Issues
- Vulnerabilities: follow [SECURITY.md](SECURITY.md)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
