---
title: Multi-Cluster GitOps Model
description: Proposed GitOps design for the platform, nonprod, and prod clusters using CAPI, Argo CD, Kargo, kro, and Capsule.
---

# Multi-Cluster GitOps Model

## Status

Proposed.

This document describes the intended GitOps model for the lab once the platform
cluster, workload clusters, and application delivery flows are built out. It is
more specific than the architecture overview, but it is still a design rather
than a description of live state.

Until this design is implemented, the architecture overview remains the source
of truth for the current baseline.

## Scope

This design covers four related concerns:

- platform cluster responsibilities
- downstream cluster creation and management
- long-lived and ephemeral environments
- team and application isolation

It uses the current planning baseline:

- `1` platform cluster on the `UM760`
- `1` nonprod cluster
- `1` prod cluster
- long-lived environments: `dev`, `staging`, `prod`
- ephemeral environments for pull requests, load tests, and similar short-lived
  work

The example team layout used throughout this document is:

- `TeamA`
  - `AppA1`
  - `AppA2`
  - `AppA3`
- `TeamB`
  - `AppB1`
  - `AppB2`

## Goals

- Keep one management cluster responsible for cluster lifecycle and GitOps
  control.
- Keep application delivery Git-native: promotion means changing Git, not
  mutating clusters directly.
- Reuse `kro` APIs for application delivery instead of Helm or Kustomize
  overlays.
- Keep downstream workload clusters strongly isolated from each other.
- Give each team a stable governance boundary without collapsing all of that
  team's applications into a single namespace.

## Non-Goals

- This document does not define the exact `kro` `ResourceGraphDefinition` schema
  for every platform API.
- This document does not define CI pipelines, image signing, or registry
  hardening in detail.
- This document does not define every shared service that should run in
  `nonprod` or `prod`.
- This document does not treat the current design as implemented reality.

## Design Summary

The intended control-plane model is:

- the platform cluster runs `Argo CD`, `CAPI`, and `Kargo`
- `CAPI` creates the `nonprod` and `prod` workload clusters
- `Argo CD` runs only on the platform cluster and syncs plain YAML to all three
  clusters
- `kro` provides the reusable application and platform APIs
- `Capsule` provides the team governance layer in each workload cluster
- `Kargo` promotes applications by editing environment-specific YAML in Git

The high-level split is:

- cluster boundary: `platform`, `nonprod`, `prod`
- team boundary: Capsule tenant per team per workload cluster
- workload boundary: namespace per `team-app-env`

For future multi-node clusters, service and ingress VIPs are intended to use
Cilium LB IPAM plus Cilium BGP peering with the `VP6630`, while the canonical
Kubernetes API endpoint is intended to use Talos VIP. The control-plane
endpoint model is defined in
[Service Exposure and Control Plane Endpoints](./service-exposure-and-control-plane-endpoints.md).

## Cluster Roles

### Platform Cluster

The platform cluster is the only management cluster.

It owns:

- `Argo CD`
- `CAPI`
- `Kargo`
- shared `kro` APIs
- platform-only controllers and operational services

It does not host general application workloads by default.

### Nonprod Cluster

The `nonprod` cluster hosts:

- `dev` environments
- `staging` environments
- ephemeral environments such as `pr-123` or `loadtest-001`
- any nonprod shared services that belong in the workload plane instead of the
  platform plane

### Prod Cluster

The `prod` cluster hosts:

- `prod` application environments
- prod shared services and policy

## Namespace and Team Model

Namespaces use the template:

```text
team-app-env
```

Examples:

- `teama-appa1-dev`
- `teama-appa1-staging`
- `teama-appa1-prod`
- `teama-appa1-pr-123`

This keeps each application instance isolated at the namespace boundary.

Do not use one namespace per team for all of that team's applications. That
would couple unrelated apps at the secrets, RBAC, quota, and blast-radius
layers.

### Capsule

Each workload cluster runs Capsule.

Capsule tenants are cluster-local, so the intended shape is:

- `nonprod`:
  - `Tenant/teama`
  - `Tenant/teamb`
- `prod`:
  - `Tenant/teama`
  - `Tenant/teamb`

This means there is one logical team boundary across the lab, implemented as
one tenant object per team per workload cluster.

`dev` and `staging` do not currently need separate team governance layers.
They share the same team-level Capsule tenant in `nonprod`, while remaining
separate per-app namespaces.

## Environment Model

Environments are not modeled as Helm values files or Kustomize overlays.

Instead, each environment is a concrete instance of the same `kro` API.
Reuse lives in shared `ResourceGraphDefinition`s, while environment-specific
differences live in environment-specific custom resources.

For example, `AppA1` should have separate resources for:

- `teams/teama/appa1/envs/dev/app.yaml`
- `teams/teama/appa1/envs/staging/app.yaml`
- `teams/teama/appa1/envs/prod/app.yaml`

Each file is small because the heavy lifting lives in the shared `kro` API.

Ephemeral environments follow the same pattern under `ephemeral/`, for example:

- `teams/teama/appa1/ephemeral/pr-123/app.yaml`

## GitOps Repository Layout

The intended `gitops` repository shape is:

```text
gitops/
├── platform/
│   ├── argocd/
│   │   ├── bootstrap.yaml
│   │   ├── projects/
│   │   │   ├── platform.yaml
│   │   │   ├── teama.yaml
│   │   │   └── teamb.yaml
│   │   └── applicationsets/
│   │       ├── platform.yaml
│   │       ├── clusters-platform.yaml
│   │       ├── clusters-nonprod.yaml
│   │       ├── clusters-prod.yaml
│   │       ├── teams-nonprod.yaml
│   │       └── teams-prod.yaml
│   ├── capi/
│   │   ├── providers/
│   │   ├── clusterclasses/
│   │   └── clusters/
│   │       ├── nonprod/
│   │       └── prod/
│   ├── kargo/
│   │   └── projects/
│   │       ├── teama-appa1/
│   │       ├── teama-appa2/
│   │       ├── teama-appa3/
│   │       ├── teamb-appb1/
│   │       └── teamb-appb2/
├── clusters/
│   ├── platform/
│   │   ├── bootstrap.yaml
│   │   ├── platform/
│   │   │   ├── rgds-platform.yaml
│   │   │   ├── rgds-apps.yaml
│   │   │   └── platform.yaml
│   │   ├── policies/
│   │   └── shared/
│   ├── nonprod/
│   │   ├── bootstrap.yaml
│   │   ├── platform/
│   │   │   ├── rgds-platform.yaml
│   │   │   ├── rgds-apps.yaml
│   │   │   └── platform.yaml
│   │   ├── capsule/
│   │   │   ├── teama.yaml
│   │   │   └── teamb.yaml
│   │   ├── policies/
│   │   └── shared/
│   └── prod/
│       ├── bootstrap.yaml
│       ├── platform/
│       │   ├── rgds-platform.yaml
│       │   ├── rgds-apps.yaml
│       │   └── platform.yaml
│       ├── capsule/
│       │   ├── teama.yaml
│       │   └── teamb.yaml
│       ├── policies/
│       └── shared/
└── teams/
    ├── teama/
    │   ├── appa1/
    │   │   ├── envs/dev/app.yaml
    │   │   ├── envs/staging/app.yaml
    │   │   ├── envs/prod/app.yaml
    │   │   └── ephemeral/pr-123/app.yaml
    │   ├── appa2/
    │   └── appa3/
    └── teamb/
        ├── appb1/
        └── appb2/
```

The ownership model is:

- `platform/`: platform-cluster control-plane state
- `clusters/*/bootstrap.yaml`: per-cluster version selection for reusable
  bootstrap/core OCI Helm charts released from the `platform` repo
- `clusters/*/platform/`: released RGD bundle installation and cluster-local
  `Platform` instances after the bootstrap/core layer is present
- `clusters/*/capsule`, `clusters/*/policies`, and `clusters/*/shared`:
  workload-cluster shared state
- `teams/`: team-owned application instances

## Argo CD Model

One `Argo CD` instance runs on the platform cluster.

It syncs:

- `platform/argocd`, `platform/capi`, and `platform/kargo` to the platform
  cluster
- `clusters/platform/bootstrap.yaml` to the platform cluster
- `clusters/platform/platform` to the platform cluster
- `clusters/nonprod/bootstrap.yaml` to the `nonprod` cluster
- `clusters/nonprod/platform`, `clusters/nonprod/capsule`,
  `clusters/nonprod/policies`, and `clusters/nonprod/shared` to the `nonprod`
  cluster
- `clusters/prod/bootstrap.yaml` to the `prod` cluster
- `clusters/prod/platform`, `clusters/prod/capsule`,
  `clusters/prod/policies`, and `clusters/prod/shared` to the `prod` cluster
- `teams/*/*/envs/dev`, `teams/*/*/envs/staging`, and
  `teams/*/*/ephemeral/*` to the `nonprod` cluster
- `teams/*/*/envs/prod` to the `prod` cluster

The intended Argo shape is:

- one `AppProject` per team
- `ApplicationSet` for platform-owned fleet generation
- one admin-owned bootstrap `Application` per cluster for
  `clusters/<cluster>/bootstrap.yaml`
- `Application` resources kept in the `argocd` namespace

Each `clusters/<cluster>/bootstrap.yaml` selects the version of the reusable
bootstrap/core components for that cluster by pinning the released OCI chart
versions for the admin-owned Cilium, Argo CD, and `kro` Applications. The full
bootstrap/core delivery sequence is defined in
[Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md).

Once the bootstrap/core layer is in place, `clusters/<cluster>/platform/`
holds the released RGD bundle installation and the cluster-local `Platform`
instance.

Do not rely on application CRs scattered across arbitrary namespaces as the
default control model. The central `argocd` namespace is simpler unless a later
team self-service requirement makes that extra complexity worth it.

## kro Model

`kro` is the abstraction layer for reusable platform and application APIs.

The intended pattern is:

- shared RGD source and release lifecycle live in the `platform` repo
- cluster-local RGD bundle installation and cluster-local platform instances
  live under `clusters/<cluster>/platform/` after the bootstrap/core layer has
  already installed `kro`
- environment-specific application custom resources live under `teams/`
- Argo CD syncs the YAML
- versioned RGD bundles are installed from OCI artifacts
- `kro` expands the custom resources into the Kubernetes objects they own

The platform-side release, CUE authoring, and OCI publication model is defined
in [Platform RGD Delivery Model](./platform-rgd-delivery.md). The preceding
bootstrap/core delivery layer is defined in
[Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md).

An environment-specific application resource should be narrow and explicit. For
example:

```yaml
apiVersion: apps.platform.gilman.io/v1alpha1
kind: AppDeployment
metadata:
  name: appa1
spec:
  team: teama
  app: appa1
  env: dev
  namespace: teama-appa1-dev
  image:
    repository: ghcr.io/gilmanlab/teama/appa1
    digest: sha256:...
  routing:
    host: appa1.dev.apps.lab.gilman.io
```

The shared `AppDeployment` API should stamp stable labels such as:

- `glab.gilman.io/team`
- `glab.gilman.io/app`
- `glab.gilman.io/env`

## CAPI Model

`CAPI` owns workload-cluster lifecycle.

The intended CAPI responsibilities are:

- install and manage cluster API providers
- define reusable cluster classes
- create and scale `nonprod` and `prod`
- keep workload-cluster creation separate from application promotion

This keeps:

- cluster lifecycle in `CAPI`
- desired-state reconciliation in `Argo CD`
- application promotion in `Kargo`

## Kargo Model

`Kargo` runs on the platform cluster.

There should be one Kargo project per application pipeline:

- `teama-appa1`
- `teama-appa2`
- `teama-appa3`
- `teamb-appb1`
- `teamb-appb2`

The intended durable stages are:

- `dev`
- `staging`
- `prod`

Ephemeral environments are intentionally outside the long-lived promotion graph.
They should be created and destroyed by automation that adds or removes the
corresponding YAML from `teams/.../ephemeral/...`.

Promotion means editing the environment-specific resource in Git. A narrow field
such as `spec.image.digest` is the preferred promotion target.

For `AppA1`, the promotion targets are:

- `teams/teama/appa1/envs/dev/app.yaml`
- `teams/teama/appa1/envs/staging/app.yaml`
- `teams/teama/appa1/envs/prod/app.yaml`

The intended policy is:

- `dev`: automatic promotion is acceptable
- `staging`: automatic promotion is acceptable
- `prod`: promotion should require an explicit approval step

## Worked Example: TeamA / AppA1

1. `CAPI` creates the `nonprod` and `prod` workload clusters.
2. `Argo CD` syncs shared control-plane state to the platform cluster.
3. `Argo CD` syncs `clusters/platform/platform/`,
   `clusters/nonprod/platform/`, and `clusters/prod/platform/`, installing
   `kro`, the selected released `platform-rgds` and `apps-rgds` bundles, and
   the cluster-local `Platform` instances.
4. `Argo CD` syncs Capsule tenants `teama` and `teamb` to `nonprod` and `prod`.
5. `teams/teama/appa1/envs/dev/app.yaml` defines an `AppDeployment` with
   namespace `teama-appa1-dev`.
6. CI publishes a new image for `AppA1`.
7. `Kargo` detects the new artifact and updates
   `teams/teama/appa1/envs/dev/app.yaml`.
8. `Argo CD` syncs that file to `nonprod`.
9. `kro` expands the `AppDeployment` into the namespace and workload resources
   for `teama-appa1-dev`.
10. After validation, `Kargo` updates
    `teams/teama/appa1/envs/staging/app.yaml`.
11. `Argo CD` syncs the staging instance to `nonprod` namespace
    `teama-appa1-staging`.
12. After approval, `Kargo` updates
    `teams/teama/appa1/envs/prod/app.yaml`.
13. `Argo CD` syncs the prod instance to the `prod` cluster namespace
    `teama-appa1-prod`.

## Open Questions

- Which `kro` APIs should exist first beyond `AppDeployment` and
  team-namespace bootstrap?
- Should `Argo CD` generate one application per environment directory, one per
  app, or one per team/app/cluster boundary?
- Which shared services belong under `clusters/nonprod/shared` and
  `clusters/prod/shared` versus platform-wide control-plane management?
- What is the exact cluster-registration flow from `CAPI` outputs into Argo CD
  destinations?

## Migration into Architecture Docs

This document should be folded into the architecture overview after the
following are true:

- the `gitops` repo structure exists in a stable form
- the platform cluster is actually running `Argo CD`, `CAPI`, and `Kargo`
- the `nonprod` and `prod` clusters exist under `CAPI`
- at least one real application has exercised the `dev` -> `staging` -> `prod`
  flow

At that point, the architecture overview should be updated so it describes the
steady-state GitOps model directly, and this design document can either be
trimmed or kept as historical design context.
