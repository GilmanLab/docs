---
title: Hosts and Substrate
description: Physical host roles, IncusOS, Incus clustering, and VM substrate boundaries.
---

# Hosts and Substrate

The bare-metal substrate is IncusOS plus Incus.

IncusOS is the host operating system for every compute node. Incus is the VM
control surface. Talos is the guest operating system for Kubernetes nodes. This
keeps both layers immutable and API-driven without carrying a general-purpose
Linux host management layer.

## Host Roles

### VyOS Router

The `VP6630` runs VyOS and remains the lab network appliance. It owns routing,
DHCP, DNS entrypoints, PXE coordination, bootstrap support, the real platform
Kubernetes API TCP frontend, and BGP peering with Cilium.

VyOS is intentionally part of the bootstrap path. The lab already depends on it
for network reachability, so using it for API fronting and temporary bootstrap
coordination keeps the early system small.

In the current implementation split, VyOS consumes the released
`ghcr.io/gilmanlab/platform/bootstrap-k0s:<version>` image through `infra` and
also serves the IncusOS operation image on `LAB_PROV` during host bootstrap.

### UM760

The `UM760` runs IncusOS as the first permanent host and the first Incus
cluster member.

During bootstrap it hosts the first platform Talos VM. After the `MS-02` hosts
join, it remains useful as bootstrap, recovery, and light-duty capacity.

### MS-02 Ultra Hosts

Each `MS-02 Ultra` runs IncusOS directly on bare metal.

The three `MS-02` systems form the main compute tier. After they join the Incus
cluster, the platform Talos cluster expands by placing additional Talos VMs on
this tier.

## Incus Cluster

The final Incus cluster spans:

- `um760`
- `ms02-1`
- `ms02-2`
- `ms02-3`

The cluster is intentionally heterogeneous. Incus cluster groups should be used
only as lightweight placement and CPU-boundary labels, for example `amd-um760`
and `intel-ms02`. Kubernetes remains the main workload scheduler.

The `UM760` is not a disposable bootstrap host. It is the first durable Incus
cluster member.

## VM Substrate

Talos nodes run as Incus VMs. Those VMs are infrastructure cattle:

- created by CAPI/CAPN where possible
- configured through Talos machine configuration
- reconciled by GitOps after bootstrap
- recreated rather than manually repaired when practical

Non-Talos Incus VMs are allowed later, but they are not a v1 design driver. Any
non-Talos VM that holds unique state must bring its own backup story.

## Storage

Use local ZFS-backed Incus storage on each IncusOS host in v1.

An Incus cluster is a management cluster, not automatically a replicated
storage system. For most Incus storage drivers, volumes remain on the member
where they are created. That is acceptable for Talos VM disks because the
primary recovery model is rebuild-first.

Do not introduce shared VM storage in v1:

- no Ceph
- no LINSTOR
- no Incus OVN/storage architecture just for VM mobility
- no NAS-backed default VM disks

The NAS remains a durable backup and artifact boundary, not the default block
storage path for every VM.

## Network Attachment

Physical switch ports to IncusOS hosts can be trunks.

By default, terminate VLAN handling at the switch/IncusOS/Incus layer and
present Talos VMs with untagged, access-style vNICs on the selected lab VLAN.
Talos guest VLAN tagging is reserved for concrete multi-VLAN guest
requirements.

This keeps DHCP reservations, CAPI templates, and recovery simpler while still
allowing the underlay to remain visible to VyOS.

## Prototype Validation Needed

Before treating the host substrate as implementation reference material, prove:

- the selected IncusOS image mode is a correct final-disk artifact for the
  single-disk `UM760`
- first-node and joining-node IncusOS seeds apply the right default Incus
  settings
- joining nodes do not create local networks or storage pools that block cluster
  join
- CAPN can place Talos VMs against the intended Incus profiles and storage pools

## References

- [IncusOS image download](https://linuxcontainers.org/incus-os/docs/main/getting-started/download/)
- [IncusOS installation seed](https://linuxcontainers.org/incus-os/docs/main/reference/seed/)
- [Incus clustering](https://linuxcontainers.org/incus/docs/main/explanation/clustering/)
- [Incus cluster storage](https://linuxcontainers.org/incus/docs/main/howto/cluster_config_storage/)
