---
title: Platform RGD Delivery Model
description: Proposed design for authoring, releasing, publishing, and consuming platform-side kro APIs.
---

# Platform RGD Delivery Model

## Status

Proposed.

This document defines the platform-side delivery model for shared `kro`
`ResourceGraphDefinition`s. It complements the broader
[kro Consumption Model](./kro-consumption-model.md) by making the platform-owned
API lifecycle concrete without turning the `gitops` repo into the source of
truth for reusable API definitions.

## Purpose

The primary purpose of this design is to keep platform API ownership, release
lifecycle, and cluster consumption clearly separated.

The intended split is:

- the `platform` repo owns authoring, validation, release notes, and
  publication for shared platform APIs
- the `gitops` repo owns cluster-local installation of released API bundles and
  cluster-local instances of those APIs

This lets the lab version and promote platform APIs deliberately, while keeping
cluster bootstrap in Git simple enough to reason about at a glance.

## Goals

- Keep reusable platform API source out of the `gitops` repo.
- Give `platform-rgds` and `apps-rgds` independent release trains.
- Publish released RGD bundles as OCI artifacts that Argo CD can install
  declaratively.
- Keep the cluster-local bootstrap surface small and explicit.
- Use CUE for build-time authoring and validation without exposing CUE as an
  operator-facing runtime interface.

## Non-Goals

- This document does not define the exact `Platform` schema.
- This document does not define the full CI workflow YAML for release or
  publication.
- This document does not define every future platform capability block.

## Design Summary

The intended model is:

- `platform` owns the source for shared `kro` APIs
- `release-please` orchestrates release PRs, tags, and changelog updates
- `platform-rgds` and `apps-rgds` are released independently
- publish workflows render final YAML artifacts from CUE and push them to OCI
  registries with ORAS
- `gitops` installs those released OCI artifacts through Argo CD
- `gitops` also holds the cluster-local `Platform` custom resource that carries
  cluster-specific inputs

## Ownership and Repository Boundaries

### Platform Repo

The `platform` repo is the source of truth for shared RGD definitions.

It owns:

- CUE authoring input for `platform-rgds` and `apps-rgds`
- validation of rendered RGD artifacts before publication
- release configuration and changelog management
- OCI publication of rendered artifacts

It does not own cluster-local desired state.

### GitOps Repo

The `gitops` repo is the source of truth for which released API bundles a
cluster installs and which cluster-local custom resources should exist there.

It owns:

- Argo CD `Application` resources that install `kro` and released RGD bundles
- the cluster-local `Platform` custom resource
- the ordering and composition of those objects during cluster bootstrap

It does not own raw shared RGD source.

## Release Model

The `platform` repo should manage `platform-rgds` and `apps-rgds` as separate
release trains in the same repository.

The intended flow is:

1. `release-please` manages release PRs, version bumps, tags, and changelog
   updates.
2. `platform-rgds` and `apps-rgds` each advance only when their own changes
   require a release.
3. A publish workflow runs after a release is created, renders the final YAML
   artifacts, and pushes them as OCI artifacts via ORAS.
4. Cluster operators choose which released version to install by updating the
   corresponding Argo CD `Application` in `gitops`.

This keeps API release history explicit and lets lower environments validate new
bundle versions before higher environments adopt them.

## Authoring and Build Model

The intended authoring model for `platform-rgds` is:

- one public `Platform` RGD
- CUE as the build-time authoring language
- a root package that defines the final public shape
- CUE subpackages for logically ordered platform capability blocks

The subpackages are intended to correspond to stable blocks of cluster
configuration, such as:

- core platform defaults
- secrets integration
- networking integration
- bare-metal integration such as `tinkerbell`

These subpackages are an authoring and validation boundary, not a separate
operator-facing API surface. The published product remains the rendered RGD
YAML artifact.

CI may import CRDs or equivalent schemas into CUE so the rendered artifact can
be validated structurally before publication. Cluster-side `kro` validation is
still responsible for the final semantic checks when the RGD is created.

## Cluster Consumption Model

The intended cluster-local bootstrap surface in `gitops` is:

```text
clusters/<cluster>/platform/
├── kro.yaml
├── rgds-platform.yaml
├── rgds-apps.yaml
└── platform.yaml
```

Each file has one job:

- `kro.yaml`: install `kro` itself
- `rgds-platform.yaml`: install the selected released `platform-rgds` OCI
  artifact
- `rgds-apps.yaml`: install the selected released `apps-rgds` OCI artifact
- `platform.yaml`: instantiate the cluster-local `Platform` custom resource

An admin-owned Argo CD bootstrap `Application` should point at
`clusters/<cluster>/platform/` and use sync waves so the order is explicit:

1. install `kro`
2. install the released RGD bundles
3. create the cluster-local `Platform` instance

This keeps the cluster bootstrap surface intentionally small and makes the
chosen bundle versions obvious in Git.

## Relationship to Other Designs

This design builds on:

- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md) for cluster topology,
  tenancy, and application flow
- [kro Consumption Model](./kro-consumption-model.md) for the ownership split
  between platform-owned APIs, developer release intent, and GitOps
  materialization

It does not change the application-side model where developers author `App`
instances alongside application source code and Kargo materializes final
environment-specific resources into `gitops`.

## Next Step

The next implementation-oriented design pass should define the initial
`Platform` schema and the first concrete capability block, starting with the
platform-side `tinkerbell` inputs.
