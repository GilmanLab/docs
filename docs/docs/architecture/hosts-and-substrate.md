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
DHCP, DNS entrypoints, PXE reachability when needed, the real platform
Kubernetes API TCP frontend, and BGP peering with Cilium.

VyOS is intentionally part of the network bootstrap path. The lab already
depends on it for network reachability, but it no longer hosts the active
temporary Kubernetes bootstrap cluster.

### N5 Pro NAS

The MINISFORUM N5 Pro runs IncusOS as the first permanent host and the genesis
Incus cluster member.

It replaces the Synology as the NAS and in-lab storage boundary. The `128GB`
NVMe device is reserved for IncusOS. The two `1TB` WD_Black SN7100 NVMe
devices form the initial mirrored ZFS data pool for Incus and NAS duties.

During bootstrap, Incus on the N5 Pro hosts a disposable single-node Talos
cluster. That cluster exists to run the bootstrap controllers before ownership
moves into the real platform cluster.

### UM760

The `UM760` remains available as later IncusOS capacity.

It is no longer the genesis host. Any future `UM760` role should be proven
after the N5 Pro path works, rather than carried forward from the abandoned
UM760-first bootstrap design.

### MS-02 Ultra Hosts

Each `MS-02 Ultra` runs IncusOS directly on bare metal.

The three `MS-02` systems form the main compute tier. After they join the Incus
cluster, the platform Talos cluster expands by placing additional Talos VMs on
this tier.

## Incus Cluster

The intended Incus cluster starts with the N5 Pro and later expands to:

- `ms02-1`
- `ms02-2`
- `ms02-3`
- `um760`, if it remains useful after the NAS-first path is proven

The cluster is intentionally heterogeneous. Incus cluster groups should be used
only as lightweight placement and CPU-boundary labels, for example `amd-nas`,
`amd-um760`, and `intel-ms02`. Kubernetes remains the main workload scheduler.

The N5 Pro is not a disposable bootstrap host. It is the first durable Incus
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

The N5 Pro starts with a mirrored ZFS pool over the two `1TB` NVMe drives. The
small `128GB` NVMe device is for the OS only.

An Incus cluster is a management cluster, not automatically a replicated
storage system. For most Incus storage drivers, volumes remain on the member
where they are created. That is acceptable for Talos VM disks because the
primary recovery model is rebuild-first.

Do not introduce shared VM storage in v1:

- no Ceph
- no LINSTOR
- no Incus OVN/storage architecture just for VM mobility
- no remote NAS-backed default VM disks for every host

The N5 Pro NAS is the durable backup and artifact boundary. Its local Incus
pool may host local VMs, but it is not a remote block-storage platform for
every VM in the lab.

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

- IncusOS installs cleanly on the N5 Pro with the OS/data disk split described
  above
- the first-node Incus seed applies the right default Incus settings
- the mirrored NVMe ZFS pool is exposed to Incus the way the bootstrap VM path
  expects
- joining nodes do not create local networks or storage pools that block cluster
  join
- CAPN can place Talos VMs against the intended Incus profiles and storage pools

## References

- [IncusOS image download](https://linuxcontainers.org/incus-os/docs/main/getting-started/download/)
- [IncusOS installation seed](https://linuxcontainers.org/incus-os/docs/main/reference/seed/)
- [Incus clustering](https://linuxcontainers.org/incus/docs/main/explanation/clustering/)
- [Incus cluster storage](https://linuxcontainers.org/incus/docs/main/howto/cluster_config_storage/)
