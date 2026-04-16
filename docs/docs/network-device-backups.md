---
title: Network Device Backups
description: Future design for RouterOS configuration backup and encrypted Git storage.
---

# Network Device Backups

This document defines the intended backup process for managed network devices.

The first scope is the MikroTik RouterOS devices:

- `CRS309-1G-8S+IN`: lab switch
- `CCR2004`: home router

The `VP6630` runs VyOS and is managed separately through the `infra` repo's
VyOS configuration and Ansible flow. It may be added to the same visibility
surface later, but the first backup process should stay focused on RouterOS.

## Placement

The durable implementation belongs in the platform cluster, not on the `VP6630`.

The backup service is operational plumbing rather than a bootstrap dependency.
Running it in the platform cluster keeps the router focused on routing, DNS, and
PKI duties, while the platform cluster owns automation, GitOps-managed services,
and recovery helpers.

Until the platform cluster exists, ad hoc manual exports are acceptable. Do not
add a long-lived RouterOS backup container to the VyOS router as the default
design.

## Desired Flow

The target flow is:

1. `Oxidized` polls each RouterOS device on a schedule.
2. Oxidized writes the latest fetched config to a private staging volume.
3. A small `backup-sync` job or sidecar reads the staged export.
4. `backup-sync` compares the plaintext export with the decrypted current SOPS
   backup.
5. If the plaintext changed, `backup-sync` writes a structured SOPS-encrypted
   backup into the private `secrets` repo.
6. `backup-sync` commits and pushes only encrypted files.
7. Health checks report the last successful backup time per device.

Oxidized should use file output for the handoff to `backup-sync`, not its native
Git output. Oxidized's Git backend commits plaintext configs, and its encrypted
Git option uses `git-crypt`, not SOPS.

The encrypted Git writer should be deliberately small. It only needs to compare,
encrypt, commit, and push. Oxidized should remain responsible for device polling,
connection handling, and RouterOS model support.

## Secret Boundary

Encrypted backup payloads belong in the private `secrets` repo.

Public repos may contain:

- the platform workload manifests
- the list of expected devices
- config templates
- references to secret names and paths
- the `backup-sync` source code, if a custom tool is needed

Public repos must not contain:

- RouterOS credentials
- Git deploy keys
- SOPS age identities
- encrypted backup payloads
- raw RouterOS exports

The expected private repo layout is:

```text
network/
  mikrotik/
    backup-credentials.sops.yaml
    backup-writer.sops.yaml
    backups/
      ccr2004.sops.yaml
      crs309.sops.yaml
```

`backup-credentials.sops.yaml` should hold the RouterOS backup user material.
`backup-writer.sops.yaml` should hold the Git and SOPS identity material needed
by the platform-cluster writer.

## Backup Format

The reviewable backup artifact should be a SOPS-encrypted YAML envelope, not a
bare `.rsc` file.

The plaintext form before encryption should look like:

```yaml
device: crs309
kind: routeros-export
captured_at: "2026-04-16T00:00:00Z"
routeros_version: "7.16.2"
source: oxidized
export: |
  /interface ethernet
  set [ find default-name=sfp-sfpplus1 ] name=to-vyos
```

This keeps metadata available to tooling while still letting SOPS encrypt the
actual configuration content. The exact schema can grow, but it should remain
simple enough to inspect with `sops -d`.

Do not blindly re-encrypt and commit every poll. SOPS encryption output can
change even when the plaintext has not changed, so `backup-sync` must compare
plaintext before writing a new encrypted file.

## Export Policy

Start with plain text RouterOS exports.

The initial command should prefer a terse export rather than verbose output.
Verbose exports include more default and built-in state, which makes them harder
to review and more fragile as restore input.

Treat text exports as the primary review and change-history artifact. They are
not automatically a complete bare-metal recovery guarantee.

RouterOS text exports do not include every sensitive or device-local artifact,
including system user passwords, installed certificates, SSH keys, Dude data, or
User Manager databases. Future implementation should explicitly decide whether
to add same-device binary backups for disaster recovery. If binary backups are
added, they must also be SOPS-encrypted before commit and their restore path must
be tested.

## Access Model

Create a dedicated RouterOS backup identity per device or per backup domain.

The backup user should have only the policies needed for the selected export
method. Do not reuse the day-to-day administrator identity. If the export process
eventually needs sensitive values, grant that deliberately and document why.

The platform cluster needs network reachability from the backup namespace to:

- `crs309.mgmt.lab.gilman.io` or `10.10.10.2`
- the home `CCR2004` management address

The implementation session must add the minimum firewall and service-access
rules needed for those connections.

## GitOps Shape

The backup stack should be deployed by Argo CD as a platform-owned application.

The public desired state should define:

- namespace
- Oxidized deployment
- `backup-sync` job or sidecar
- persistent or ephemeral staging volume
- network policy, if the cluster network plugin supports it
- health checks and alerting hooks
- secret references, not secret payloads

The private `secrets` repo supplies credentials and stores the encrypted backup
artifacts. The backup writer therefore needs both read access to its deployment
secrets and write access to the backup destination path.

## Restore Expectations

Every backup mechanism must be paired with a restore drill.

The first implementation is not complete until it proves:

- a current export can be decrypted from the `secrets` repo
- the export can be inspected by an operator
- a non-destructive import dry run or lab-device restore test has been performed
- the known gaps in text exports are documented

Do not assume a RouterOS `.rsc` export can be applied blindly to a wiped or
replacement device. RouterOS imports are sensitive to version, hardware, default
objects, interface naming, certificates, keys, and users.

## Future Implementation Checklist

- Choose whether to deploy stock Oxidized plus a custom `backup-sync` container
  or a single custom collector for the first two devices.
- Add RouterOS backup users for the `CRS309` and `CCR2004`.
- Add encrypted RouterOS credentials under `secrets/network/mikrotik/`.
- Add the platform-cluster Git writer credentials under
  `secrets/network/mikrotik/`.
- Create the Argo CD application and manifests for the backup stack.
- Confirm platform-cluster network reachability to both devices.
- Run the first backup and verify only SOPS-encrypted files are committed.
- Test restore behavior against a lab-safe target before relying on the backups
  for disaster recovery.

## References

- [Oxidized](https://github.com/ytti/oxidized)
- [RouterOS Configuration Management](https://help.mikrotik.com/docs/spaces/ROS/pages/328155/Configuration%2BManagement)
- [SOPS](https://github.com/getsops/sops)
