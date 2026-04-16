# Architecture Overview

The canonical version of this document now lives in
[`docs/docs/architecture.md`](docs/docs/architecture.md).

Use the Docusaurus site source under `docs/` for ongoing updates.
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

The current design keeps `Argo CD` scoped to the platform cluster itself.

This means:

- the platform cluster's `Argo CD` manages only platform services and platform-owned infrastructure components
- downstream clusters are not assumed to be centrally registered into platform `Argo CD`
- downstream clusters are expected to manage their own services independently
- whether a downstream cluster uses `Argo CD` or some other delivery model is left to that cluster's own design

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
- backup and disaster recovery boundaries
