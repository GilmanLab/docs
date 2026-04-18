---
title: Architecture Overview
description: High-level architecture for the GilmanLab homelab rework.
---

# Architecture Overview

This document captures the current high-level architecture for the lab rework.

It is intentionally centered on control flow and system boundaries rather than detailed implementation. Low-level configuration, exact IP plans, and per-service manifests belong elsewhere.

For the physical inventory, see [the hardware reference](./hardware.md).

## Overview

The architecture is built around a dedicated platform cluster running on the `UM760`, with the `MS-02 Ultra` systems acting as Proxmox compute hosts. The platform cluster is the control plane for provisioning, cluster creation, GitOps, and infrastructure automation.

The main design goal is to keep the platform responsibilities concentrated in one isolated Talos-based Kubernetes cluster while treating the Proxmox layer as a unified VM substrate for downstream clusters.

At a high level:

- The `UM760` runs the platform cluster on Talos Linux.
- The platform cluster runs Tinkerbell, CAPI, Argo CD, AWX, and optional TerraKube.
- The `MS-02 Ultra` nodes are provisioned over PXE into Proxmox and then clustered into a single Proxmox control plane.
- AWX and/or TerraKube handle node-level post-provisioning work.
- CAPI creates downstream Talos-based clusters as Proxmox VMs through the clustered Proxmox API surface.
- The `DS923+` provides shared storage to the Proxmox layer.
- The `VP6630` remains the lab router, DNS entrypoint, and network boundary to the home network.

## Design Intent

The current design is driven by a few core ideas:

- Keep the platform control plane separate from the Proxmox compute layer.
- Use Tinkerbell for bare-metal provisioning of the Proxmox nodes.
- Form a Proxmox cluster early so CAPI has one control surface to target.
- Use CAPI as the main cluster lifecycle engine for downstream Kubernetes clusters.
- Prefer reusable Proxmox VM templates over highly customized VM-by-VM definitions.
- Avoid coupling initial Proxmox clustering to shared storage; storage is a later concern.

## Core Components

### Platform Cluster (`UM760`)

The `UM760` hosts the platform cluster as an isolated Talos Linux Kubernetes cluster.

The current design treats this as a single-node cluster.

This cluster is intended to own the following responsibilities:

- `Tinkerbell`: full provisioning stack for PXE-based installation of the Proxmox nodes
- `CAPI`: downstream cluster lifecycle management using Proxmox as infrastructure
- `Argo CD`: GitOps for the platform cluster itself, and potentially for downstream cluster registration and sync
- `AWX`: Ansible orchestration for infrastructure tasks that are still better handled through playbooks
- `TerraKube`: optional Terraform-based automation for future non-node-bootstrap workflows
- network-device backup services for RouterOS configuration history and encrypted recovery artifacts

This machine is not being used as a general-purpose compute node. Its purpose is to act as the lab control plane.

At present there is no separate physical capacity available to make the platform cluster highly available. A fallback idea would be to turn the `UM760` into a Proxmox node and run three platform-cluster VMs on it, but that does not materially improve failure tolerance because all three nodes would still share the same physical host, storage path, and power domain. The added virtualization and bootstrap complexity would outweigh the benefit.

### Proxmox Nodes (`MS-02 Ultra`)

Each `MS-02 Ultra` is intended to be a dedicated Proxmox node.

The current design assumes:

- Each node is provisioned through Tinkerbell over PXE using unattended installation
- Each node receives follow-up configuration through `AWX`
- The three nodes are clustered into a single Proxmox control plane early
- Shared storage is not required for initial Proxmox cluster formation
- Each node should become a "complete" unit that can host VM templates and receive CAPI-created VMs cleanly through the clustered Proxmox API

This means the architecture uses clustered Proxmox management from the start, while deferring shared storage and any dedicated storage backplane work to a later phase.

### Shared Storage (`DS923+`)

The `DS923+` is the shared storage layer for the Proxmox environment.

The current intended uses are:

- `NFS` storage for templates, images, and backups
- Optional NAS-backed disks for workloads that need a more redundant storage path

The default storage model is intentionally local-first:

- active VM disks run from node-local NVMe on the `MS-02 Ultra` hosts
- the `DS923+` is used for templates, images, backups, and optional NAS-backed VM disks exposed over NFS
- `iSCSI` is not part of the default VM execution path

This design leaves room for a mixed storage model, where some storage remains node-local while some is NAS-backed. Shared storage improves migration, HA behavior, and storage flexibility, but it is not a prerequisite for initial Proxmox cluster formation.

The main tradeoff is deliberate: although the NAS has `10GbE` connectivity to the Proxmox hosts, its underlying media is HDD-based. The design therefore prioritizes local NVMe performance for running VMs and uses the NAS primarily for backup, image distribution, and selective redundancy rather than as the primary block storage for all guests.

The architecture does not define a strict workload placement policy for NAS-backed disks. NFS-backed storage is available to VMs that need it, but whether any given node or workload uses that storage is left to the workload's own requirements rather than being fixed globally in advance.

The `DS923+` is also the primary durable backup boundary in the system. Platform-cluster state and Proxmox host configuration are intended to be rebuilt from source-controlled configuration and automation, while NAS-backed artifacts and data are the main state that should be explicitly protected.

### Network Boundary and Switching

The physical network roles remain:

- `CCR2004`: home router
- `VP6630`: lab router, DNS entrypoint, and DMZ boundary to the home network
- `CRS309-1G-8S+IN`: lab switch
- `TL-SG105`: dedicated Intel AMT switch for the `MS-02 Ultra` management links

The current cabling intent is:

- Each `MS-02 Ultra` uses one `25GbE` link to the `CRS309`
- A future second link per node may be dedicated to storage and/or Proxmox clustering traffic
- Intel AMT links terminate on the `TL-SG105`
- The `VP6630` remains the routing boundary and provides optional BGP-based floating IP advertisement for downstream clusters

The baseline network model is intentionally smaller than the previous lab design, but it still preserves a dedicated Layer 2 provisioning domain for Tinkerbell:

- platform / management network
- provisioning network for PXE and DHCP
- workload / VM network
- AMT / OOB network
- optional storage / replication network later

The provisioning network exists because Tinkerbell's DHCP and PXE flow requires Layer 2 access or DHCP relay. The current design assumes a dedicated provisioning segment rather than folding PXE traffic into the general workload path.

### DNS and Naming

The lab uses `lab.gilman.io` as its internal naming root rather than a private-only top-level domain.

This keeps internal naming under a real domain the lab controls while still allowing private DNS views, selective public exposure later, and a clean future path for public certificates on intentionally exposed endpoints.

The current intended DNS design is:

- `VyOS` remains the client-facing resolver for the lab networks.
- A `PowerDNS Authoritative` service runs as a container on the `VP6630`.
- `VyOS` forwards the lab's internal zones to that local authoritative service.
- Internal names are private by default; public DNS is reserved for explicitly exposed entrypoints.
- Internal certificates are expected to use a private CA by default; public CA issuance is reserved for endpoints that benefit from public trust.

Running the authoritative DNS service on the router boundary instead of inside the platform cluster avoids a bootstrap dependency where the platform control plane would need to be healthy before the lab can resolve the names used to reach it.

The namespace is intentionally split by ownership boundary instead of using one flat dynamic zone:

| Zone | Writers | Purpose |
| --- | --- | --- |
| `lab.gilman.io` | manual or GitOps-managed only | Parent zone, delegations, and a small set of static anchor records |
| `mgmt.lab.gilman.io` | manual or GitOps-managed only | Stable management and platform service names |
| `dhcp.lab.gilman.io` | `VyOS` DHCP via `RFC2136` | Dynamic lease-driven hostnames |
| `<cluster>.k8s.lab.gilman.io` | `ExternalDNS` via `RFC2136` | Cluster-scoped workload and ingress names |

This design keeps the management namespace stable while still allowing dynamic DNS for both DHCP clients and Kubernetes workloads. It also keeps update rights narrow: `VyOS` DHCP cannot mutate management records, and each Kubernetes cluster can be constrained to only its own delegated subzone.

### Internal PKI and Trust

The lab's internal PKI is designed around the same bootstrap constraint as internal DNS: the trust anchor for internal services cannot depend on the platform cluster being healthy before it can issue or rotate certificates.

The current intended PKI design is:

- The internal root CA key lives in `AWS KMS`.
- The root CA is treated as operationally offline: no always-on lab service has standing permission to use it for routine issuance.
- A `Smallstep step-ca` service runs as a container on the `VP6630` as the online intermediate CA.
- Internal ACME is provided by that `step-ca` instance for automated certificate issuance and renewal.
- `Vault` remains the expected long-term home for most secret management inside the platform cluster, but it is not the bootstrap owner of the internal CA hierarchy.

This keeps naming and trust in the same edge-adjacent failure domain without forcing the platform cluster to come up first. If the platform cluster is down, the lab can still resolve internal names and issue or renew the certificates needed to restore that control plane.

The intended trust boundary is deliberately split:

| Component | Role | Notes |
| --- | --- | --- |
| Root CA | trust anchor | Stored in `AWS KMS`; used only for intermediate issuance and rotation |
| `step-ca` on `VP6630` | online issuing intermediate | Handles day-to-day certificate issuance for internal services |
| ACME clients | automated consumers | Used by `cert-manager` and other internal services that can rotate through ACME |
| `Vault` | secret management consumer | May issue or store service-specific material later, but does not own bootstrap PKI |

This design accepts that routing, internal DNS, and the online intermediate CA share the `VP6630` failure domain. That is an intentional trade for the homelab: a single edge host keeps the bootstrap path simple, while the root CA remains outside that host's routine operating privileges.

### Network Device Backups

Network-device backup collection belongs in the platform cluster once that
cluster is online.

The first target devices are the MikroTik `CRS309` lab switch and `CCR2004` home
router. The durable flow should use `Oxidized` for RouterOS collection and a
small SOPS-aware writer that commits only encrypted backup artifacts into the
private `secrets` repo.

This is intentionally not a `VP6630` container responsibility. RouterOS backups
are operational recovery support, not a bootstrap dependency like DNS or PKI.
Keeping the backup stack in the platform cluster keeps the router focused on
routing, internal DNS, and certificate issuance while the platform cluster owns
automation and Git-backed operational services.

The design is documented in [Network device backups](./network-device-backups.md).

## Control Flow

### 1. Platform Bootstrap

The first step is bringing up the Talos-based platform cluster on the `UM760`.

The bootstrap path is intentionally independent of `Tinkerbell`, `CAPI`, `AWX`, and `TerraKube`.

The current intended flow is:

1. Generate the Talos machine configuration from source-controlled configuration and scripts.
2. Generate a reproducible Talos ISO for the `UM760`.
3. Write that ISO to a USB installer using standard host tooling.
4. Boot the `UM760` from USB using the normal Talos ISO install path.
5. Bring the node up as the initial platform cluster node.
6. Install `Argo CD`.
7. Let GitOps reconcile the rest of the platform stack.

Talos supports this model directly:

- Talos supports embedding machine configuration directly into the bootable image.
- Talos supports generating customized ISO boot assets through Image Factory or the `imager` tool.
- Talos documents ISO boot on bare metal as the standard installation path.

The important boundary is that Talos provides the image-generation and boot-asset customization tooling, but the act of writing the ISO to a USB stick is still a normal host-side operation rather than a Talos-specific burn command.

Once available, that cluster becomes the system of control for the rest of the lab:

- Argo CD reconciles platform configuration
- Tinkerbell provisions bare metal
- AWX performs node-level infrastructure configuration
- TerraKube remains available for future infrastructure workflows where Terraform is a better fit
- CAPI creates downstream clusters on Proxmox

### 2. Bare-Metal Proxmox Provisioning

The `MS-02 Ultra` nodes are provisioned through Tinkerbell.

The intended path is:

1. Tinkerbell PXE boots a target node
2. The node installs Proxmox using unattended configuration
3. The node comes up with enough baseline config to be remotely managed

This is the primary Tinkerbell use case in the architecture. VM network installs may exist later, but they are not the main design driver.

### 3. Proxmox Cluster Formation

After the nodes are installed, they are joined into a Proxmox cluster.

This happens before any expectation that `CAPI` will treat the Proxmox layer as a shared VM substrate.

The key point is that:

- clustered Proxmox management is required early
- shared storage is not required for this first clustering step
- storage enhancements and dedicated storage traffic can be layered in later

This gives the lab a single Proxmox control surface for scheduling and lifecycle operations without requiring the full storage design to be complete on day one.

### 4. Post-Provision Node Configuration

After a node is installed, additional configuration is applied.

This post-provisioning stage includes items such as:

- networking
- storage configuration
- Proxmox-specific options
- image and template publication
- delivery of custom images such as Packer-built golden images

Current intent:

- Use `AWX` as the primary post-bootstrap configuration plane for Proxmox nodes
- Keep `TerraKube` optional for future workflows that are not part of node bootstrap itself
- Likely TerraKube use cases are complementary infrastructure resources that are better suited to Terraform than Ansible
- Exact TerraKube-managed resource classes are intentionally undecided for now
- Use `AWX` to publish golden images and VM templates into the Proxmox cluster

### 5. Golden Images and Templates

Golden image ownership is split across four layers:

- `Packer` owns image creation
- the `DS923+` is the durable repository for built images and template artifacts
- `AWX` owns publishing or registering those images as Proxmox templates
- `CAPI` consumes only templates that are already present in Proxmox

This keeps responsibilities narrow:

- image creation stays separate from cluster lifecycle
- template publication is an explicit infrastructure operation
- `CAPI` stays focused on machine orchestration rather than image distribution

### 6. Downstream Cluster Creation

Once the Proxmox cluster is ready, CAPI uses Proxmox support to create downstream Kubernetes clusters as VMs.

The downstream cluster assumptions are:

- downstream clusters are Talos-based
- VM creation should rely primarily on reusable templates
- the default storage path for downstream cluster VMs is node-local NVMe
- cluster scaling should be handled through CAPI rather than ad hoc Proxmox operations
- CAPI references templates that have already been published into Proxmox

The desired outcome is that the clustered Proxmox layer exposes a stable substrate, while CAPI owns the actual Kubernetes cluster lifecycle.

After creation, downstream clusters are treated as strongly isolated environments rather than extensions of the platform cluster.

Downstream clusters may make use of BGP-advertised floating IPs through the `VP6630` when they need stable network entrypoints, but this is an available capability rather than a default requirement.

## Role of GitOps

`Argo CD` runs on the platform cluster and manages the desired state of the control plane.

At minimum, that includes:

- platform cluster applications
- platform cluster infrastructure controllers
- provisioning stack configuration
- platform-owned operational services such as network-device backups

The current design keeps `Argo CD` scoped to the platform cluster itself.

This means:

- the platform cluster's `Argo CD` manages only platform services and platform-owned infrastructure components
- downstream clusters are not assumed to be centrally registered into platform `Argo CD`
- downstream clusters are expected to manage their own services independently
- whether a downstream cluster uses `Argo CD` or some other delivery model is left to that cluster's own design

## Disaster Recovery

The lab's recovery model is rebuild-first. Most components are reconstructed from source-controlled configuration, GitOps state, and automation rather than restored from a point-in-time backup. This section defines the backup and restore substrate for the state that cannot be reasonably rebuilt that way.

### Scope

The lab accumulates irreducible state in four tiers:

- **Proxmox VMs** — downstream cluster nodes, utility VMs, and any workload VMs whose guest state cannot be cheaply reconstructed
- **Platform and downstream Kubernetes clusters** — cluster objects (manifests, CRDs), etcd, and persistent volumes holding workload data
- **Arbitrary Linux hosts and one-off container volumes** — the `VP6630` VyOS router's container volumes (`pdns-auth` LMDB, `step-ca` BadgerDB, Tailscale machine key), Talos etcd snapshots pushed out of cluster, and similar filesystem-shaped state that does not live inside a Kubernetes cluster
- **RouterOS configuration history** — covered separately in [Network Device Backups](./network-device-backups.md). Its requirement is reviewable plaintext change history rather than block or filesystem recovery, so it uses a different artifact model

### Tooling

The first three tiers are covered by two complementary tools:

| Tool | Scope |
| --- | --- |
| `Proxmox Backup Server` (PBS) | Proxmox VMs (native block-level, dedup, dirty-bitmap incrementals); arbitrary Linux hosts and container volumes via the standalone `proxmox-backup-client` |
| `Velero` | Kubernetes-native backup for the platform cluster and all CAPI-provisioned downstream clusters: manifests, CRDs, etcd, and persistent volumes via CSI snapshots or file-system backup |

Two tools rather than one is a deliberate choice. PBS is best-in-class for VM-level and filesystem backup but has no native understanding of Kubernetes objects. Velero understands Kubernetes semantics — namespace-scoped restore, CRDs, application-consistent hooks, CSI integration — but does not replace a VM backup system. Each tool covers its slice well, and they do not overlap.

### PBS Placement

PBS runs as a Proxmox VM on the `MS-02 Ultra` tier, not inside the platform cluster and not on the NAS itself.

This placement is driven by several constraints:

- **Upstream install path.** PBS is shipped as a Debian-based appliance. There is no official container image, and the project's design assumes systemd and a local filesystem for the datastore. Running PBS in Kubernetes, including via KubeVirt, forces an off-path install and a container story PBS was not designed for.
- **No circular dependency on the platform cluster.** PBS exists to recover from failures, including platform-cluster failures. Running it inside the cluster it is meant to help restore is a trap.
- **GitOps automation.** Synology VMM on the `DS923+` is functional but has effectively no automation ecosystem. No official Terraform provider, no first-class Ansible coverage, and a semi-documented DSM API. Managing the PBS VM declaratively on Synology would regress against the GitOps posture the rest of the lab holds. Proxmox, by contrast, has a mature Terraform provider, an actively maintained Ansible collection, Packer builders, and a stable API — the PBS VM fits the same declarative pipeline as other infrastructure VMs.
- **Datastore locality.** The PBS datastore lives on NAS-backed storage via NFS from the `DS923+`, consistent with the NAS being the primary durable-data boundary. PBS reads and writes its dedup chunks against that mount; the VM itself stays lightweight and stateless enough to rebuild.

The main tradeoff is timing. PBS depends on the Proxmox layer being up. Until that tier exists, backup for pre-existing state — notably the `VP6630` container volumes — is handled as an interim stopgap rather than through PBS.

### Velero Placement

Velero runs per Kubernetes cluster. The platform cluster gets its own Velero install, and each CAPI-provisioned downstream cluster gets its own.

Every Velero instance writes to a shared S3-compatible object store exposed on the NAS. This keeps backup artifacts consolidated on the same durable substrate as PBS and lets restore operations pull from a single location, while still honoring the per-cluster operational boundary that Velero itself requires.

Velero's backup scope per cluster includes:

- Kubernetes manifests and CRDs
- etcd snapshots (on clusters where Velero's etcd integration applies; Talos clusters additionally retain native `talosctl etcd snapshot` as a direct path, pushed into PBS)
- persistent volumes via CSI snapshots where the cluster's CSI driver supports them, or via Velero's file-system backup path otherwise

Downstream clusters are treated as independent recovery domains. They are not centrally registered into the platform cluster's Argo CD, and their Velero backups are self-contained so that a lost cluster can be reconstituted without standing up the platform cluster first.

### Restore Drills

No backup mechanism is considered complete until its restore path has been exercised at least once. For each of PBS-backed VMs, Velero-backed cluster state, and PBS-backed host-level volumes, the first production use of that backup must be paired with a documented restore drill against a lab-safe target.

Restore expectations differ by tier:

- **VMs** — PBS restore reconstitutes the full VM disk image. Tested via restore to a scratch VM on the Proxmox cluster.
- **Kubernetes clusters** — Velero restore reconstitutes cluster objects and PVC data into an empty cluster. Tested via restore into a throwaway CAPI cluster.
- **Host volumes** — `proxmox-backup-client` restore reconstitutes file trees on a target host. Tested via restore into a scratch directory on a lab VM.

This applies equally to downstream clusters. A downstream cluster whose Velero backup has never been successfully restored is not considered protected.

### Off-Site

A single PBS datastore on the NAS leaves the lab exposed to NAS-level failure. Off-site replication is a planned addition rather than a day-one requirement. PBS supports pull-mode sync between PBS instances; the likely path is a second PBS target either on a different physical location or backed by S3-compatible object storage. Velero's object store can be mirrored or cross-region-replicated through whatever backend is chosen. The exact off-site design is deferred until the primary PBS is in place and the NAS-level failure scenarios have been characterized.
The intended future multi-cluster delivery model is being tracked separately in
[Multi-Cluster GitOps Model](./designs/gitops-multi-cluster.md). Until that
design is implemented, this architecture overview remains intentionally
conservative about downstream-cluster GitOps behavior.

## Why This Layout

This layout separates concerns in a way that matches the intended operating model:

- Talos on the `UM760` keeps the control plane narrow and appliance-like
- Proxmox on the `MS-02 Ultra` nodes provides a clustered VM substrate for downstream clusters
- Tinkerbell handles bare-metal installation
- Early Proxmox clustering gives CAPI a single control surface without waiting for the full shared-storage design
- AWX fills the gap between bare-metal install and a fully configured Proxmox node
- Packer, the NAS, and AWX together provide a clear image pipeline without pushing image management into CAPI
- TerraKube remains available as a future addition for complementary infrastructure automation when that need becomes concrete
- CAPI becomes the main abstraction for downstream cluster creation and scaling
- Argo CD keeps the platform cluster declarative
- BGP floating IPs remain a downstream-cluster capability rather than part of the platform cluster's default exposure model

The design also avoids forcing too much day-one complexity into the Proxmox layer. The nodes can start as individually useful machines before later being combined into a more integrated Proxmox topology.

For the same reason, the platform cluster remains single-node on the `UM760`. This accepts that `UM760` failure is a platform outage, but avoids introducing a false form of HA where multiple control-plane VMs still depend on the same underlying machine.

The same rebuild-first logic applies to recovery boundaries:

- the platform cluster is primarily rebuilt from Talos configuration, GitOps state, and automation
- Proxmox hosts are primarily rebuilt through Tinkerbell and AWX
- downstream clusters are primarily recreated through CAPI
- NAS-backed artifacts, backups, and selected workload data form the primary durable state boundary

## Likely Next Sections

As the design firms up, the next useful additions to this document are likely:

- bootstrap path for the `UM760`
- Proxmox node lifecycle in more detail
- storage model
- network model
- downstream cluster lifecycle
- restore drills and disaster recovery procedures
