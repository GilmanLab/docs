---
title: App RGD Design
description: Proposed first-pass design for the application-facing kro API.
---

# App RGD Design

## Status

Proposed.

This document captures the intended shape of the first application-facing `kro`
API, referred to here as `App`.

This is not the final schema. The goal of this pass is to make the API shape
clear enough that the next pass can define the actual `ResourceGraphDefinition`
without reopening the high-level model.

## Summary

`App` should be the primary developer-facing API for deployable application
workloads.

The first pass should focus on four concerns:

- containers
- secrets
- configs
- volumes

The current design direction is:

- developers author an `App` instance alongside application source code
- the public API stays small and application-centric
- secrets, configs, and volumes are defined at the point where they are used or
  mounted
- secret store selection defaults from the target environment and is not a
  normal developer-facing input
- Kargo materializes the final environment-specific `App` instance into the
  `gitops` repo on `main`
- Argo CD reconciles that final `App` instance

## Illustrative Example

The example below is illustrative only. It exists to make the intended shape
concrete before the actual schema is written.

```yaml
apiVersion: apps.platform.gilman.io/v1alpha1
kind: App
metadata:
  name: orders-api
spec:
  team: teama
  app: orders-api

  containers:
    - name: api
      image:
        repository: ghcr.io/gilmanlab/teama/orders-api
        digest: sha256:2222222222222222222222222222222222222222222222222222222222222222
      ports:
        - name: http
          port: 8080
      env:
        - name: MAX_PRICE
          value: "20"
        - name: LOG_LEVEL
          value: info
        - name: DB_USERNAME
          secret:
            remoteKey: kv/orders-api
            property: username
        - name: DB_PASSWORD
          secret:
            remoteKey: kv/orders-api
            property: password
        - name: THIRD_PARTY_API_KEY
          secret:
            remoteKey: kv/shared/third-party
            property: apiKey
      mounts:
        - path: /var/lib/orders
          volume:
            persistent:
              size: 10Gi
        - path: /etc/orders
          volume:
            config:
              files:
                application.yaml: |
                  http:
                    port: 8080
                  logging:
                    format: json
                features.yaml: |
                  enableDiscountsV2: true
        - path: /var/run/secrets/orders
          volume:
            secret:
              files:
                db-username:
                  remoteKey: kv/orders-api
                  property: username
                db-password:
                  remoteKey: kv/orders-api
                  property: password
                third-party-api-key:
                  remoteKey: kv/shared/third-party
                  property: apiKey

    - name: worker
      image:
        repository: ghcr.io/gilmanlab/teama/orders-worker
        digest: sha256:4444444444444444444444444444444444444444444444444444444444444444
      env:
        - name: LOG_LEVEL
          value: info
        - name: DB_USERNAME
          secret:
            remoteKey: kv/orders-api
            property: username
        - name: DB_PASSWORD
          secret:
            remoteKey: kv/orders-api
            property: password
      mounts:
        - path: /var/lib/orders
          volume:
            persistent:
              size: 10Gi

    - name: metrics-proxy
      image:
        repository: ghcr.io/gilmanlab/platform/metrics-proxy
        digest: sha256:3333333333333333333333333333333333333333333333333333333333333333
      ports:
        - name: metrics
          port: 9090
```

This example shows the intended first-pass shape:

- containers are the main unit of declaration
- config values are attached where they are consumed
- secret references are attached where they are consumed
- volume definitions are attached where they are mounted
- the API describes External Secrets-backed needs without exposing handwritten
  Kubernetes `Secret` objects

It also intentionally leaves some things open:

- the exact inline shape for config files versus simple env vars
- the exact secret reference shape
- how much container surface is exposed in v1
- how environment-specific values such as `THIRD_PARTY_URL` are merged during
  promotion/materialization

## Design Rules

### Containers

- `App` should support more than one container.
- The single-main-container case should still be the easiest path.
- The public API should model deployable application containers, not raw Pod
  templates.
- Security and runtime defaults should be platform-owned wherever practical.

### Secrets

- Secrets must align with `ExternalSecret` plus `SecretStore` or
  `ClusterSecretStore`.
- Plaintext secret values are out of scope.
- Hand-authored Kubernetes `Secret` manifests are not the primary path.
- Secret store selection should default from the target environment rather than
  being a normal developer-facing field.

### Configs

- Non-secret config should be distinct from secrets.
- Release-coupled config lives with the developer-authored `App` instance.
- Environment-specific config is added during promotion/materialization.
- The public API should not force developers to author raw `ConfigMap`
  manifests.

### Volumes

- The public API should use the term `volume`, not `PersistentVolumeClaim`.
- Volume definitions should default to point-of-use declaration at the mount
  site.
- True shared-lifecycle named volumes may exist later, but they should not
  shape the v1 API around a less common reuse case.

## Why This Shape

This design optimizes for local reasoning.

The expected common case is:

- one container needs one config value
- one container needs one secret value
- one mount path needs one volume definition

That means the public API should default to point-of-use declarations instead of
top-level registries and references.

For the same reason, the API should not expose more of Kubernetes than it needs
to. The goal is a small, opinionated application API, not a renamed PodSpec.

## Relationship to GitOps and Promotion

The current working lifecycle is:

1. A developer authors an `App` instance alongside application source code.
2. CI produces an image and the corresponding Git commit.
3. Kargo bundles the image and commit into Freight.
4. Kargo promotes that Freight into an environment.
5. During promotion, Kargo combines the source `App` instance with any
   environment-specific inputs or overrides.
6. Kargo writes the final `App` instance into the destination environment path
   in the `gitops` repo on `main`.
7. Argo CD reconciles that final `App` instance.

This means the `App` API is developer-facing, but the final reconciled instance
is still environment-specific GitOps state.

## Out of Scope for This Pass

This document does not define:

- the final `App` schema
- the exact `ExternalSecret` generation model
- the exact promotion-time composition mechanism
- policy and governance rules that are better enforced by `Kyverno`,
  `Capsule`, or similar systems

## Open Questions

- What is the smallest useful first schema for the common backend API case?
- How much container surface should v1 expose for commands, args, probes,
  resources, and ports?
- What is the minimum honest secret-reference shape for the External Secrets
  model?
- Should config files and simple env values use one unified shape or distinct
  ones?

## Next Step

The next pass should define the actual first schema draft for `App`.
