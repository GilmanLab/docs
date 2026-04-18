---
title: kro Consumption Model
description: Proposed design for how GilmanLab publishes, consumes, and promotes kro-based application and platform APIs.
---

# kro Consumption Model

## Status

Proposed.

This document captures the intended role of `kro` in the lab and the current
working model for how `kro` resources flow from developer-owned source
repositories into environment-specific desired state in the `gitops` repo.

This is an initial design draft. It intentionally does not define concrete
`ResourceGraphDefinition` schemas yet. The next design pass should define the
first public APIs in detail.

## Purpose

The primary purpose of `kro` in this lab is two-fold:

- give software engineers a common abstraction for deploying applications
- give platform engineers a common abstraction for maintaining platform
  capabilities

In practice, these are closely related. Platform engineers and software
engineers are both describing and operating software systems through the same
API layer. The difference is usually ownership and blast radius, not the
fundamental shape of the task.

`kro` is therefore the public API layer for both application delivery and
platform delivery.

## Design Principles

The `kro` API layer should be standardized and opinionated.

The goal is not to recreate the full input surface of the underlying Kubernetes
objects. The goal is to publish a smaller, clearer API that is safer and easier
to reason about.

The design principles are:

- prefer a minimal public interface over a mechanically complete one
- hardcode platform-controlled values when that improves security,
  consistency, or operational correctness
- prefer defaults that "do the right thing" and reduce cognitive load
- keep required inputs minimal
- constrain mutable inputs wherever reasonable
- expose escape hatches only when there is a real need, not as a default design
  posture

The expected result is that a developer should be able to use a platform API
without needing to understand the full Kubernetes object model underneath it.

## Terminology

Public `kro` APIs should prefer common software engineering terms over
specialized Kubernetes terms where doing so improves clarity.

Examples:

- prefer `volume` over `PersistentVolumeClaim` in a public API
- prefer application-level language like `service`, `route`, `database`, or
  `worker` when those terms are understandable without Kubernetes-specific
  context

This is not a strict ban on Kubernetes terminology. Some Kubernetes concepts,
such as `Deployment`, are already self-explanatory to many users. The important
point is that public APIs should be shaped for the intended consumers, not as
thin wrappers around upstream object names.

## API Ownership Model

There are three important ownership layers:

- platform-owned API definitions
- developer-owned application release intent
- environment-owned deployment materialization

### Platform-owned API definitions

The platform layer owns the `ResourceGraphDefinition`s and any shared conventions
that define what public APIs exist and how they behave.

Examples include future APIs such as:

- application delivery APIs
- team namespace bootstrap APIs
- platform capability APIs

This document does not define those schemas yet.

### Developer-owned application release intent

Developers should define instances of platform APIs alongside the application
source code that they belong to.

For example:

```text
project/
├── src/
├── Dockerfile
└── deployment.yaml
```

In this model, `deployment.yaml` is not the `RGD`. It is an instance of a
platform-owned API.

This file should be treated as part of the application's release intent and
versioned alongside the application source and image build inputs.

Important: `App` is only one example of a future public API. This model does
not imply that developers will only ever instantiate a single kind of `kro`
resource.

### Environment-owned deployment materialization

Argo CD still needs a Git source of truth to reconcile from.

In this design, that source of truth remains the `gitops` repo on its mainline
branch. Argo CD does not read directly from developer application repositories.

Therefore, the final environment-specific `kro` resource instances must be
materialized into the `gitops` repo under environment-specific paths on
`main`/`master`.

This means the environment-specific folders in the `gitops` repo are still the
final desired state that Argo CD reconciles.

## Relationship to the GitOps Model

This design builds on the multi-cluster GitOps model described in
[Multi-Cluster GitOps Model](./gitops-multi-cluster.md).

The important clarification is:

- `kro` API definitions are platform-owned
- developer-authored API instances are release inputs
- Kargo materializes environment-specific outputs into the `gitops` repo
- Argo CD reconciles those outputs from `main`

Environment-specific paths remain the expected layout, for example:

```text
gitops/
└── envs/
    ├── dev/
    │   └── orders-api/
    ├── staging/
    │   └── orders-api/
    └── prod/
        └── orders-api/
```

This document does not settle the exact `gitops` folder layout beyond that
principle.

## Promotion Model

`Kargo` should treat a release as more than an image update.

The intended model is:

- CI produces a new image
- CI also produces a Git commit containing the developer-owned application
  release intent
- Kargo bundles the image revision and Git commit into one piece of Freight
- Kargo promotes that Freight through environments
- during promotion, Kargo materializes the final environment-specific desired
  state into the `gitops` repo
- Argo CD reconciles that materialized desired state from `main`

This is different from treating promotion as nothing more than a digest bump.
Behavior-defining configuration that belongs to the application release should
travel with the release artifact.

## Environment-specific Inputs

Environment-specific concerns still exist and must influence the final deployed
resource instances.

A realistic example is an application input such as `THIRD_PARTY_URL`:

- in `dev`, the application may need to call
  `sandbox.api.thirdparty.com`
- in `prod`, the application may need to call `api.thirdparty.com`

This kind of input is not necessarily part of the application release itself.
It may instead be a property of the target environment.

The current design direction is:

- developer-owned release inputs live with the application source
- environment-specific inputs live in the `gitops` repo
- Kargo combines the two during promotion and writes the resulting `kro`
  resource instance into the destination environment folder on `main`

This lets release-coupled configuration move through the promotion pipeline
while still letting environment-specific concerns influence the final deployed
resource.

## Promotion-time Composition

The current working assumption is that any composition of:

- developer-authored release input
- environment-specific overrides or bindings
- final Argo-reconciled output

happens during promotion, inside Kargo.

In other words:

- developers do not manually write environment-specific final manifests into the
  `gitops` repo
- Argo CD should reconcile the final materialized YAML
- Kargo is the place where environment-specific shaping occurs before that YAML
  lands in Git

One pragmatic option is to use Kustomize at promotion time only.

If used, the important boundary is:

- Kustomize is a promotion-time composition tool
- Kustomize is not the public API
- Kustomize is not the developer-facing abstraction
- Argo CD should still reconcile the final materialized YAML written by Kargo

This keeps `kro` as the abstraction layer while allowing a familiar merge and
patch mechanism to help materialize final environment-specific manifests.

The exact composition mechanism remains an open question. Kustomize is one
candidate, not a final commitment in this document.

## Composition Patterns

Cross-resource relationships need explicit design.

The preferred default is to embed related configuration where lifecycle and
ownership naturally belong together.

For example, if an application needs a small amount of secret material or a
simple runtime capability that is specific to that application instance, the
public API should prefer embedding that relationship rather than forcing the
developer to construct multiple loosely related peer resources.

However, embedding will not work in every case.

Some capabilities have independent lifecycle or sharing boundaries. A future
`Database` capability is an example:

- if an application declares that it needs a database, how does the application
  receive connection details?
- if the database is managed independently, what is the contract between the
  application-facing resource and the database-facing resource?
- if multiple consumers share a capability, where should the shared contract
  live?

`kro` does not remove the need to design those contracts carefully. This is an
area that still needs explicit design guidance for the lab.

## Guardrails and Policy

Environment substitution and policy guardrails are related but distinct
problems.

### Environment substitution

This is about supplying environment-sensitive values during promotion or
materialization.

Examples:

- environment-specific endpoints
- environment-specific hostnames
- environment-local service references

This concern belongs close to the promotion/materialization flow and therefore
belongs close to Kargo and the `gitops` repo.

### Policy guardrails

This is about constraining what a final deployed resource is allowed to do.

Examples:

- forbidding privileged or root workloads in prod
- enforcing tenancy boundaries
- constraining namespaces, quotas, or network access

This concern is not necessarily a `kro` concern. It may be better handled by
policy and governance layers such as `Kyverno` or `Capsule`.

This design intentionally does not force those responsibilities into `kro`.

## Current Design Direction

The current working model is:

1. The platform team publishes shared `kro` APIs.
2. Developers instantiate those APIs alongside the application source code.
3. CI produces an image and a corresponding Git commit.
4. Kargo bundles those artifacts into Freight.
5. Kargo promotes Freight between environments.
6. During promotion, Kargo combines release input with environment-specific
   inputs and writes the final resource instances into the environment-specific
   area of the `gitops` repo on `main`.
7. Argo CD reconciles those final resource instances from the `gitops` repo.

This is the design baseline for the next pass.

## Open Questions

- What are the first public `kro` APIs the platform should publish?
- Which relationships should be embedded by default versus modeled as explicit
  peer resources?
- What is the standard contract for peer-style relationships such as an
  application consuming a separately managed database?
- Should Kustomize be the default promotion-time composition mechanism, or only
  one optional implementation technique?
- What exact files live in the environment-specific `gitops` folders during
  promotion and after promotion?
- How should environment-specific inputs be authored, owned, and reviewed in the
  `gitops` repo?

## Next Step

The next design pass should define one or more concrete `ResourceGraphDefinition`
schemas that embody these principles, starting with a minimal application-facing
API and its expected lifecycle.
