---
title: Bootstrap and Core Delivery Model
description: Proposed design for day-0 substrate and day-1 cluster-core delivery across the platform, nonprod, and prod clusters.
---

# Bootstrap and Core Delivery Model

## Status

Proposed.

This document defines how the lab brings clusters to life before the reusable
`kro` API layer becomes active. It covers the narrow Talos/CAPI bootstrap path,
the reusable cluster-core components that GitOps manages afterward, and the
handoff from bootstrap artifacts to steady-state Argo CD ownership.

## Purpose

The primary purpose of this design is to keep bootstrap delivery,
cluster-core reuse, and platform API ownership clearly separated.

The intended split is:

- the `platform` repo owns canonical bootstrap/core component inputs and
  rendered bootstrap artifacts
- the `gitops` repo owns per-cluster version selection and cluster-local
  desired state after bootstrap
- the `infra` repo and CAPI templates own immutable Talos day-0 references for
  fresh installs and reinstalls

This lets the lab reuse Cilium, Argo CD, and `kro` consistently across the
platform, `nonprod`, and `prod` clusters without copying their canonical install
artifacts into `gitops`.

## Goals

- Keep canonical bootstrap/core component source out of the `gitops` repo.
- Make platform-cluster bootstrap and downstream CAPI cluster creation
  reproducible from versioned artifacts.
- Keep day-0 substrate narrow and explicit.
- Make day-2 ownership belong to Argo CD rather than to Talos/CAPI bootstrap
  references.
- Start `kro` only at the first real platform API boundary.

## Non-Goals

- This document does not define the exact Helm values for Cilium, Argo CD, or
  `kro`.
- This document does not define the long-term service exposure or control-plane
  endpoint strategy for clusters after bootstrap.
- This document does not define the CI workflow implementation for rendering,
  validation, or publishing.
- This document does not define the exact `Platform` schema.
- This document does not change the current architecture assumption that only
  the platform cluster runs Argo CD.

## Design Summary

The intended cluster bring-up model is:

- every cluster boots with a day-0 substrate
- reusable day-1 cluster-core components are then installed by GitOps
- the reusable `kro` platform API begins only after those prerequisites exist

The three layers are:

1. **Day-0 substrate**
   - components required before GitOps or higher-level APIs can act
   - includes Cilium on every cluster
   - includes minimal Argo CD and root-app seeding only on the platform
     cluster
2. **Day-1 cluster-core**
   - reusable cluster components managed by GitOps but not exposed as
     consumer-facing platform APIs
   - includes the full Cilium install and `kro`
   - includes Argo CD self-management on the platform cluster
3. **Platform API**
   - released RGD bundles and the cluster-local `Platform` custom resource
   - starts only after the day-1 cluster-core layer is present

The following are intentionally **not** modeled as `kro` APIs:

- Cilium
- Argo CD
- `kro` itself

They are reusable installable cluster primitives, not consumer-facing platform
APIs.

The long-term service exposure and control-plane endpoint model for those
clusters is defined in
[Service Exposure and Control Plane Endpoints](./service-exposure-and-control-plane-endpoints.md).

## Cluster Flows

### Platform Cluster

The platform cluster bootstrap flow is:

1. Talos installs bootstrap-safe Cilium from an immutable artifact reference.
2. Talos installs minimal Argo CD from an immutable artifact reference.
3. Talos seeds the admin-owned root `Application`.
4. The root app syncs `clusters/platform/bootstrap.yaml` from `gitops`.
5. That per-cluster bootstrap selection installs:
   - full/self-managed Argo CD
   - full Cilium
   - `kro`
6. After `kro` is present, `clusters/platform/platform/` installs the selected
   released RGD bundles and the cluster-local `Platform` custom resource.

### Downstream Clusters

The downstream `nonprod` and `prod` cluster flow is:

1. CAPI/Talos installs bootstrap-safe Cilium from an immutable artifact
   reference.
2. The platform-cluster Argo CD instance registers or reaches the new cluster.
3. Argo CD syncs `clusters/<cluster>/bootstrap.yaml` from `gitops`.
4. That per-cluster bootstrap selection installs:
   - full Cilium
   - `kro`
5. After `kro` is present, `clusters/<cluster>/platform/` installs the selected
   released RGD bundles and the cluster-local `Platform` custom resource.

Downstream clusters do **not** bootstrap their own Argo CD instances under the
current design.

## Ownership and Repository Boundaries

### Platform Repo

The `platform` repo is the source of truth for reusable bootstrap/core
artifacts.

It owns:

- canonical Helm values for reusable cluster primitives
- bootstrap-safe rendered manifests for Talos/CAPI day-0 consumption
- component-scoped Argo application definitions that use the same canonical
  source inputs
- version tags and release history for those artifacts

It does not own per-cluster version selection or cluster-local desired state.

### GitOps Repo

The `gitops` repo is the source of truth for which version of each reusable
bootstrap/core component a cluster should run after GitOps takes over.

It owns:

- `clusters/<cluster>/bootstrap.yaml` for per-cluster bootstrap/core version
  selection
- `clusters/<cluster>/platform/` for released RGD bundle installation and the
  cluster-local `Platform` custom resource
- any other cluster-local desired state after bootstrap

It does not own the canonical rendered source for reusable bootstrap/core
components.

### Infra Repo and CAPI Templates

The `infra` repo and CAPI templates own only the immutable day-0 references
needed to create or reinstall clusters.

They own:

- Talos machine-config references for platform-cluster day-0 artifacts
- CAPI/Talos template references for downstream-cluster day-0 artifacts

They do not own day-2 change management for those components.

## Canonical Artifact Layout

The `platform/bootstrap/` subtree carries both Talos/CAPI day-0 substrate
artifacts and reusable day-1 cluster-core components. The name does not imply
that every component there is consumed directly by Talos.

The intended `platform` repo layout is:

```text
platform/
└── bootstrap/
    ├── cilium/
    │   ├── values/
    │   │   ├── base.yaml
    │   │   ├── bootstrap-overrides.yaml
    │   │   └── full-overrides.yaml
    │   ├── render/
    │   │   ├── bootstrap.yaml
    │   │   └── full.yaml
    │   └── app.yaml
    ├── argocd/
    │   ├── values/
    │   │   ├── base.yaml
    │   │   ├── bootstrap-overrides.yaml
    │   │   └── full-overrides.yaml
    │   ├── render/
    │   │   ├── bootstrap.yaml
    │   │   └── full.yaml
    │   └── app.yaml
    └── kro/
        ├── values/
        │   ├── base.yaml
        │   └── full-overrides.yaml
        ├── render/
        │   └── full.yaml
        └── app.yaml
```

The intended semantics are:

- `values/base.yaml`: shared component baseline
- `values/bootstrap-overrides.yaml`: bootstrap-only overrides needed for
  Talos/CAPI-safe day-0 delivery
- `values/full-overrides.yaml`: steady-state overrides for the GitOps-managed
  install
- `render/bootstrap.yaml`: the immutable raw manifest Talos/CAPI consumes for
  day-0 bring-up
- `render/full.yaml`: the fully rendered steady-state manifest for review and
  validation parity with the Helm-driven install
- `app.yaml`: the component-scoped Argo CD application definition using the
  canonical chart, version, and full values inputs

The per-cluster `clusters/<cluster>/bootstrap.yaml` resources in `gitops`
remain the only cluster-specific version-selection surface. They pin
destination and `targetRevision` while reusing the canonical component source
shape defined in the matching `platform/bootstrap/<component>/app.yaml`.

`kro` has no Talos/CAPI bootstrap variant in the current design, so it does not
need `bootstrap-overrides.yaml` or `render/bootstrap.yaml`.

The intended `gitops` repo surface is:

```text
gitops/
└── clusters/
    ├── platform/
    │   ├── bootstrap.yaml
    │   └── platform/
    │       ├── rgds-platform.yaml
    │       ├── rgds-apps.yaml
    │       └── platform.yaml
    ├── nonprod/
    │   ├── bootstrap.yaml
    │   └── platform/
    │       ├── rgds-platform.yaml
    │       ├── rgds-apps.yaml
    │       └── platform.yaml
    └── prod/
        ├── bootstrap.yaml
        └── platform/
            ├── rgds-platform.yaml
            ├── rgds-apps.yaml
            └── platform.yaml
```

Each `bootstrap.yaml` is admin-owned and selects which released version of the
reusable bootstrap/core components a cluster should adopt.

## Versioning and Promotion Rules

The intended versioning model is:

1. Change the canonical values or source inputs in `platform`.
2. Re-render `render/bootstrap.yaml` and `render/full.yaml` from those pinned
   inputs.
3. Cut a versioned `platform` release tag.
4. Bump each cluster's `clusters/<cluster>/bootstrap.yaml` in `gitops` to the
   selected tag.
5. If a day-0 artifact changed, also bump the immutable bootstrap artifact
   references in:
   - platform-cluster Talos config in `infra`
   - downstream-cluster CAPI templates

The versioning rules are:

- cluster selections happen in `gitops`
- Talos/CAPI raw artifact URLs use immutable commit SHAs
- human-facing release selection happens by tag
- tags must be treated as immutable once published

The SHA referenced by Talos/CAPI must correspond to the released artifact
selected for that version, even if GitOps later advances clusters at different
cadences.

## Bootstrap-Safe Versus Full Installs

### Cilium

Cilium has two delivery shapes:

- **bootstrap-safe**
  - used by Talos/CAPI day-0 bootstrap
  - must preserve the intended steady-state core datapath behavior
  - must disable secret-producing features so the rendered manifest is safe to
    host at a public immutable URL
- **full**
  - used by the GitOps-managed day-1/day-2 install
  - may enable observability and TLS features that create or depend on secret
    material

Bootstrap Cilium is intentionally not a separate product. It is the steady-state
core datapath intent plus a small, explicit set of bootstrap-only exceptions.

### Argo CD

Argo CD also has two delivery shapes on the platform cluster:

- **bootstrap**
  - minimal install sufficient to run the root app
- **full**
  - self-managed steady-state Argo CD installed by GitOps

Downstream clusters do not use an Argo CD bootstrap variant under the current
design.

### kro

`kro` has only a full GitOps-managed install in this design. It is not Talos
day-0 substrate.

## Ownership Handoff

Talos/CAPI bootstrap and Argo CD do not share day-2 ownership equally.

The intended ownership handoff is:

- Talos/CAPI bootstrap gets the cluster alive
- Argo CD becomes the steady-state owner of full Cilium, Argo CD, and `kro`
- day-2 changes are made by updating `platform` inputs and the per-cluster
  selections in `gitops`, not by editing Talos/CAPI day-0 URLs

The Talos/CAPI references remain narrow and reinstall-focused. They exist so a
fresh cluster can boot, not so Talos/CAPI become the long-term control plane
for those components or define the cluster's steady-state external service and
API endpoint model.

## Relationship to Other Designs

This design builds on:

- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md) for cluster topology,
  Argo scope, and application flow
- [Service Exposure and Control Plane Endpoints](./service-exposure-and-control-plane-endpoints.md)
  for the steady-state external service and API endpoint model once bootstrap is
  complete
- [Platform RGD Delivery Model](./platform-rgd-delivery.md) for released RGD
  bundle delivery after `kro` is already present
- [kro Consumption Model](./kro-consumption-model.md) for the ownership split
  between platform-owned APIs, developer release intent, and GitOps
  materialization

This document starts before those other designs. It ends at the point where a
cluster already has its reusable cluster-core components and is ready to consume
released RGD bundles and cluster-local platform APIs.
