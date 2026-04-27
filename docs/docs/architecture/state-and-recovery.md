---
title: State and Recovery
description: Rebuild-first recovery, backup boundaries, and restore expectations.
---

# State and Recovery

The lab recovery model is rebuild-first.

Most infrastructure should be recreated from declarative inputs: IncusOS seeds,
Incus configuration, Talos machine configuration, CAPI objects, GitOps state,
and platform release artifacts. Backups protect the state that cannot be
recreated cleanly from those sources.

## Durable State Boundaries

The lab accumulates state in these tiers:

- IncusOS host configuration, Incus state, and host storage encryption material.
- Talos and Kubernetes control-plane state.
- Kubernetes persistent volumes and application data.
- Runtime identity and secret manager data.
- Router boundary service data such as local DNS zonefile state and tailnet
  identity.
- RouterOS configuration history, covered by
  [Network Device Backups](../network-device-backups.md).

The NAS is the main in-lab durable backup and artifact boundary for state that
must survive host rebuilds.

## IncusOS Hosts

Use IncusOS system backups to protect:

- host configuration
- IncusOS state
- storage-pool encryption keys

These backups are sensitive because they contain encryption key material. Store
them as secret recovery artifacts, not as ordinary logs.

IncusOS system backups do not protect installed application data or Incus
instance data. Those must be covered separately if they matter.

## Talos VMs And Kubernetes

Talos VM disks are not the primary backup object.

Protect cluster state through:

- Talos etcd snapshots
- Kubernetes-native backups such as Velero
- CSI snapshots or file-system backup for persistent volumes where appropriate
- application-level backups for stateful apps

CAPI and GitOps should recreate Talos VM infrastructure when possible.

## Non-Talos Incus VMs

Non-Talos Incus VMs are optional in v1.

If a non-Talos VM holds unique state, use an explicit VM-level backup path such
as Incus snapshots, Incus exports, or copying to another backup Incus server.
Do not build a VM backup platform before a real non-Talos VM requirement exists.

## Router Boundary Data

VyOS-hosted services that are bootstrap dependencies must have direct backup
paths because the platform cluster may be unavailable during recovery.

Examples include:

- local DNS zonefile state
- Tailscale machine identity where applicable
- temporary bootstrap artifacts during an active bootstrap

RouterOS device configuration history is operational evidence and reviewable
change history. It is intentionally handled by the network-device backup flow
rather than by block-level backup.

## No Default VM Backup Platform

There is no default VM backup appliance in v1.

That is intentional. Talos VMs should be recreated through CAPI and Talos
configuration. Non-Talos VMs should justify their own backup requirements when
they appear.

## Restore Drills

A backup mechanism is not complete until its restore path has been exercised.

The first production use of each backup class should include a restore drill
against a lab-safe target:

- IncusOS system backup restored enough to prove host recovery assumptions.
- Talos etcd snapshot used to prove cluster recovery.
- Kubernetes backup restored into a throwaway cluster.
- Application backup restored into a disposable environment.
- Router boundary data restored into a scratch service or equivalent safe
  target.

Exact drill commands belong in justfiles and runbooks once the implementation
exists.

Keycloak's runtime state, backup retention, and rebuild/restore paths are
defined separately in [Keycloak Runtime](./keycloak-runtime.md).

## References

- [IncusOS backup and restore](https://linuxcontainers.org/incus-os/docs/main/reference/system/backup/)
- [Incus instance backups](https://linuxcontainers.org/incus/docs/main/howto/instances_backup/)
- [Talos disaster recovery](https://docs.siderolabs.com/talos/v1.12/build-and-extend-talos/cluster-operations-and-maintenance/disaster-recovery)
- [Velero](https://velero.io/docs/)
