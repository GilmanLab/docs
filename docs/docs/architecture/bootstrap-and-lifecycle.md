---
title: Bootstrap and Cluster Lifecycle
description: Bootstrap flow, CAPI pivot, and cluster lifecycle ownership.
---

# Bootstrap and Cluster Lifecycle

Bootstrap uses the same core tools that should own long-term lifecycle:
Tinkerbell for bare-metal provisioning, CAPI for cluster lifecycle, CAPN for
Incus infrastructure, and the Talos providers for Talos Kubernetes nodes.

The design deliberately avoids a separate hand-built host install path that
would be thrown away after day one.

## Prototype First

Before touching the real `UM760`, prove the risky assumptions locally:

- generate or download a seeded IncusOS USB/IMG image
- write it to a VM's only disk
- boot that disk as the steady-state IncusOS host
- confirm Incus initialization, trusted client certificate access, network
  reachability, and API access
- exercise CAPN plus the Talos providers against Incus

This prototype should be disposable. Its purpose is to learn which parts of the
new path are real before producing exact runbooks.

## Temporary Bootstrap Cluster

The first real bootstrap cluster is a disposable single-node `k0s` cluster on
VyOS, likely as a host-networked container.

It exists only to run:

- Tinkerbell
- Cluster API
- Cluster API Provider Incus
- Talos bootstrap provider
- Talos control-plane provider

VyOS container support must be validated with the required privileges, mounts,
cgroups, and stability before this becomes an operator recipe.

## Host Bootstrap Flow

The intended host bootstrap sequence is:

1. Start the temporary VyOS-hosted `k0s` cluster.
2. Install Tinkerbell and the CAPI providers into that cluster.
3. Generate a seeded IncusOS USB/IMG image for the `UM760`.
4. Use Tinkerbell and HookOS to write that image directly to the internal
   `UM760` disk through `image2disk` or `oci2disk`.
5. Boot the `UM760` into IncusOS as the steady-state host OS.
6. Initialize Incus on the `UM760` with the first-node defaults needed for the
   final cluster.
7. Enable Incus clustering.
8. Import or publish the Talos nocloud image needed by CAPN.
9. Use CAPN and the Talos providers to create the first platform Talos VM on
   the `UM760`.

The same Tinkerbell path provisions the `MS-02` hosts later. Joining nodes must
use IncusOS seed settings appropriate for joining the existing Incus cluster,
not for creating independent local Incus defaults.

## Platform Cluster Bring-Up

The platform cluster starts as one Talos VM on the `UM760`.

Day-0 Talos configuration installs only the substrate needed to make the
cluster reachable and let GitOps take over:

- bootstrap-safe Cilium
- minimal Argo CD on the platform cluster
- an admin-owned root Application pointing at the platform cluster selection in
  `gitops`

After Argo CD is running, the GitOps bootstrap selection installs the full
cluster-core components: full Cilium, full/self-managed Argo CD, and `kro`.

The platform repo owns canonical bootstrap/core artifacts and release history.
The gitops repo owns per-cluster version selection and cluster-local desired
state. Infra and CAPI templates own only immutable day-0 references needed for
fresh installs and reinstalls.

## Bootstrap/Core Artifact Contract

The `platform/bootstrap/` subtree carries both Talos/CAPI day-0 substrate
artifacts and reusable day-1 cluster-core components. The name does not imply
that every component there is consumed directly by Talos.

```text
platform/
└── bootstrap/
    ├── cilium/
    │   ├── Chart.yaml
    │   ├── Chart.lock
    │   ├── values.yaml
    │   ├── bootstrap-values.yaml
    │   ├── templates/
    │   └── render/
    │       ├── bootstrap.yaml
    │       └── full.yaml
    ├── argocd/
    │   ├── Chart.yaml
    │   ├── Chart.lock
    │   ├── values.yaml
    │   ├── bootstrap-values.yaml
    │   └── render/
    │       ├── bootstrap.yaml
    │       └── full.yaml
    └── kro/
        ├── Chart.yaml
        ├── Chart.lock
        ├── values.yaml
        └── render/
            └── full.yaml
```

The contract for each component is:

- `Chart.yaml`: wrapper chart metadata and pinned upstream chart dependency.
- `Chart.lock`: locked dependency resolution for local render parity and chart
  publication.
- `values.yaml`: steady-state defaults for the GitOps-managed install.
- `bootstrap-values.yaml`: day-0 overrides for Talos/CAPI-safe bootstrap
  rendering.
- `templates/`: platform-owned manifests layered on the upstream chart.
- `render/bootstrap.yaml`: immutable raw manifest consumed by Talos/CAPI day-0
  bootstrap.
- `render/full.yaml`: fully rendered steady-state manifest for review and
  validation parity.

`kro` has no Talos/CAPI bootstrap variant, so it does not need
`bootstrap-values.yaml` or `render/bootstrap.yaml`.

The release and selection rules are:

- Change canonical inputs in `platform`.
- Re-render `render/bootstrap.yaml` and `render/full.yaml`.
- Publish the wrapper chart as an OCI artifact under a component-scoped release
  tag.
- Select versions per cluster through `gitops/clusters/<cluster>/bootstrap.yaml`.
- Reference raw Talos/CAPI artifacts by immutable commit SHA, not floating tags.
- Keep the SHA referenced by Talos/CAPI aligned with the released artifact
  selected for that version.

Bootstrap-safe Cilium must preserve the intended steady-state datapath behavior
while disabling secret-producing features that make public immutable raw
manifests unsafe. Argo CD has a minimal bootstrap render for the platform
cluster and a full self-managed render for GitOps. Day-2 changes happen by
updating `platform` inputs and `gitops` selections, not by editing bootstrap
URLs by hand.

## Expanding And Pivoting

After the `MS-02` hosts join the Incus cluster:

1. Add two more Talos VMs on the `MS-02` tier.
2. Run the platform cluster as three dual-role control-plane/worker nodes.
3. Install the CAPI providers into the platform cluster.
4. Use `clusterctl move` to transfer ownership from the temporary bootstrap
   cluster to the platform cluster.
5. Remove the temporary VyOS bootstrap cluster and any temporary PXE behavior
   once the platform cluster and Incus cluster are healthy.

`clusterctl move` is a bootstrap pivot mechanism. It is not a backup or
disaster recovery model.

## Downstream Clusters

The platform cluster is the management cluster for downstream Kubernetes
clusters.

Downstream clusters are Talos-based and created by CAPI through CAPN. They do
not run their own Argo CD instance by default. The platform Argo CD instance
syncs cluster-core and platform API state to them after they exist.

## Prototype Validation Needed

The bootstrap path is not complete until these are proven:

- IncusOS image generation and seeding for first-node and joining-node modes
- Tinkerbell image writing from HookOS to the selected disks
- VyOS-hosted `k0s` stability for the temporary bootstrap stack
- CAPN plus Talos providers creating Talos VMs with the desired boot mode,
  network attachment, storage pool, and endpoint model
- `clusterctl move` from the temporary cluster to the platform cluster

## References

- [IncusOS installation seed](https://linuxcontainers.org/incus-os/docs/main/reference/seed/)
- [Tinkerbell image2disk](https://github.com/tinkerbell/actions/tree/main/image2disk)
- [Tinkerbell oci2disk](https://github.com/tinkerbell/actions/tree/main/oci2disk)
- [CAPN Talos template](https://capn.linuxcontainers.org/reference/templates/talos.html)
- [VyOS containers](https://docs.vyos.io/en/latest/configuration/container/index.html)
