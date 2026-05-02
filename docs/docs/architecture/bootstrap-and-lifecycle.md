---
title: Bootstrap and Cluster Lifecycle
description: Bootstrap flow, CAPI pivot, and cluster lifecycle ownership.
---

# Bootstrap and Cluster Lifecycle

Bootstrap uses the same core tools that should own long-term lifecycle: CAPI
for cluster lifecycle, CAPN for Incus infrastructure, and the Talos providers
for Talos Kubernetes nodes.

Tinkerbell remains relevant for later bare-metal provisioning, but it is no
longer the first-host bootstrap anchor.

The design deliberately avoids a separate hand-built host install path that
would be thrown away after day one.

## Prototype First

Before depending on the real N5 Pro path, prove the risky assumptions in the
smallest useful slice:

- install and seed IncusOS on the N5 Pro
- confirm the `128GB` OS device and mirrored `1TB` NVMe ZFS data pool shape
- confirm Incus initialization, trusted client certificate access, OIDC
  readiness, network reachability, and API access
- run a single-node Talos VM inside Incus with control-plane scheduling enabled
- exercise CAPN plus the Talos providers from that bootstrap cluster

This prototype should be disposable. Its purpose is to learn which parts of the
new path are real before producing exact runbooks.

## Genesis Bootstrap Cluster

The first real bootstrap cluster is a disposable single-node Talos Linux
cluster running as an Incus VM on the N5 Pro NAS.

It exists only to run the controllers needed to create and hand off the real
platform cluster:

- Cluster API
- Cluster API Provider Incus
- Talos bootstrap provider
- Talos control-plane provider

The bootstrap cluster is not the platform cluster. It should be easy to delete
after CAPI ownership moves into the platform cluster.

The old VyOS-hosted `bootstrap-k0s` image path is historical context from the
abandoned UM760-first direction. Do not build new NAS bootstrap work around
that image unless a future cleanup session deliberately reactivates it.

## Host Bootstrap Flow

The intended host bootstrap sequence is:

1. Install IncusOS on the N5 Pro using the `128GB` NVMe device as the OS disk.
2. Create the mirrored ZFS data pool from the two `1TB` WD_Black NVMe devices.
3. Initialize Incus with the first-node defaults needed for the final cluster.
4. Enable Incus clustering so later hosts can join.
5. Import or publish the Talos nocloud image needed by CAPN.
6. Create a disposable single-node Talos VM inside Incus on the N5 Pro.
7. Bootstrap Talos once and enable scheduling on the control-plane node for
   bootstrap workloads.
8. Install CAPI, CAPN, and the Talos providers into the bootstrap cluster.
9. Use those providers to create the first real platform Talos VM.

Tinkerbell can still provision later bare-metal hosts if that remains the best
path. Joining nodes must use IncusOS seed settings appropriate for joining the
existing Incus cluster, not for creating independent local Incus defaults.

The first supported host flow is the N5 Pro. Joiner-specific Tinkerbell
templates, workflows, or replacement tooling for the `MS-02` and `UM760` nodes
come later, after the genesis path is proven.

## Platform Cluster Bring-Up

The platform cluster starts as one Talos VM created from the disposable Talos
bootstrap cluster on the N5 Pro.

Day-0 Talos configuration installs only the substrate needed to make the
cluster reachable and let GitOps take over:

- bootstrap-safe Cilium
- minimal Argo CD on the platform cluster
- the `platform-bootstrap` AppProject
- an admin-owned root Application pointing at the platform cluster selection in
  `gitops`

The `platform-bootstrap` AppProject is part of the day-0 handoff contract. It
must exist before the root Application, because relying on Argo CD's startup
creation of the `default` project caused a first-bootstrap race.

After Argo CD is running, the GitOps bootstrap selection installs the full
cluster-core components: full Cilium, full/self-managed Argo CD, and `kro`.
The platform cluster bootstrap selection uses three separate Applications for
Cilium, Argo CD, and `kro`, all on the `platform-bootstrap` project, so
ownership and failure domains remain visible.

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

`bootstrap/k0s` exists in the repository from the abandoned VyOS-hosted
bootstrap path. It is not part of the active NAS-first bootstrap contract.
Leave deletion, archiving, or repurposing to a focused cleanup session.

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
4. Use `clusterctl move` to transfer ownership from the disposable bootstrap
   cluster to the platform cluster.
5. Remove the disposable Talos bootstrap VM once the platform cluster and Incus
   cluster are healthy.

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

- IncusOS installation and first-node seeding on the N5 Pro
- N5 Pro mirrored NVMe ZFS pool behavior under IncusOS and Incus
- single-node Talos bootstrap VM behavior inside Incus, including
  control-plane workload scheduling
- CAPN plus Talos providers creating Talos VMs with the desired boot mode,
  network attachment, storage pool, and endpoint model
- `clusterctl move` from the disposable Talos bootstrap cluster to the platform
  cluster
- later Tinkerbell or equivalent provisioning for joining bare-metal hosts

## References

- [IncusOS installation seed](https://linuxcontainers.org/incus-os/docs/main/reference/seed/)
- [Tinkerbell image2disk](https://github.com/tinkerbell/actions/tree/main/image2disk)
- [Tinkerbell oci2disk](https://github.com/tinkerbell/actions/tree/main/oci2disk)
- [CAPN Talos template](https://capn.linuxcontainers.org/reference/templates/talos.html)
- [Talos control plane](https://docs.siderolabs.com/talos/v1.12/learn-more/control-plane)
- [Talos control-plane scheduling](https://docs.siderolabs.com/talos/v1.12/deploy-and-manage-workloads/workers-on-controlplane)
