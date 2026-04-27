---
title: GitOps and Platform APIs
description: Argo CD, CAPI, kro, Kargo, cluster roles, and repository ownership.
---

# GitOps and Platform APIs

The platform cluster is the management plane for the lab.

It runs the controllers that create clusters, reconcile platform state, publish
reusable platform APIs, and drive application promotion. The workload clusters
run applications and cluster-local platform services, but they do not become
independent management planes by default.

## Cluster Roles

### Platform

The platform cluster owns:

- Argo CD
- Cluster API and providers
- Kargo
- shared `kro` APIs
- platform-only controllers and operational services

It should not host general application workloads by default.

### Nonprod

The `nonprod` cluster hosts:

- `dev` environments
- `staging` environments
- ephemeral environments such as pull requests and load tests
- nonprod shared services that belong in the workload plane

### Prod

The `prod` cluster hosts:

- production application environments
- production shared services
- production policy

## Cluster Lifecycle

CAPI owns workload-cluster lifecycle. CAPN is the infrastructure provider for
Incus, and the Talos providers own Talos bootstrap and control-plane behavior.

The intended responsibilities are:

- install and manage cluster API providers on the platform cluster
- define reusable cluster classes or templates once the prototype proves the
  VM shape
- create and scale `nonprod` and `prod`
- keep cluster lifecycle separate from application promotion

Cluster lifecycle belongs in CAPI. Desired-state reconciliation belongs in Argo
CD. Application promotion belongs in Kargo.

## Argo CD

One Argo CD instance runs on the platform cluster.

It syncs:

- platform control-plane state to the platform cluster
- per-cluster bootstrap/core selections
- cluster-local platform API bundles and `Platform` resources
- workload-cluster shared services and policy
- team/application environment resources

The default shape is:

- dedicated AppProjects
- ApplicationSets for generated fleets of applications
- one admin-owned bootstrap Application per cluster
- Application resources kept in the `argocd` namespace

Avoid production parameter overrides and mutable promotion state inside Argo CD.
Promotion should be represented by Git changes.

One implementation gap remains explicit: the exact flow from CAPI outputs into
Argo CD cluster destinations still needs prototype validation. The platform
cluster will create downstream Talos clusters with CAPI/CAPN, but the mechanism
that turns the resulting kubeconfig, destination identity, and AppProject scope
into Argo CD cluster registration is not yet settled.

## GitOps Repository Layout

The `gitops` repository shape is:

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

Ownership follows the path:

- `platform/`: platform-cluster control-plane state.
- `clusters/*/bootstrap.yaml`: per-cluster version selection for released
  bootstrap/core OCI Helm charts.
- `clusters/*/platform/`: released RGD bundle installation and cluster-local
  `Platform` instances.
- `clusters/*/capsule`, `clusters/*/policies`, and `clusters/*/shared`:
  workload-cluster shared state.
- `teams/`: team-owned application instances.

Argo CD syncs `platform/argocd`, `platform/capi`, and `platform/kargo` to the
platform cluster; cluster bootstrap and platform folders to their destination
clusters; nonprod team environments and ephemeral environments to `nonprod`;
and prod team environments to `prod`.

## Bootstrap And Core Components

Reusable bootstrap/core components are owned in two layers:

- `platform` owns canonical inputs, wrapper charts, rendered day-0 artifacts,
  OCI chart publication, and release history.
- `gitops` owns per-cluster version selection and cluster-local desired state.

Day-0 Talos/CAPI references are narrow and reinstall-focused. They exist to
make a fresh cluster boot. They do not become the long-term day-2 control plane
for Cilium, Argo CD, or `kro`.

The reusable cluster-core layers are:

1. bootstrap-safe Cilium on every cluster
2. minimal Argo CD only on the platform cluster
3. GitOps-managed full Cilium, platform Argo CD self-management, and `kro`
4. released RGD bundles and cluster-local `Platform` resources

## kro APIs

`kro` is the abstraction layer for reusable platform and application APIs.

The split is:

- shared RGD source and release lifecycle live in `platform`
- cluster-local RGD bundle installation lives under each cluster's platform
  state in `gitops`
- application instances live under team-owned environment resources
- Argo CD syncs YAML
- `kro` expands the custom resources into owned Kubernetes objects

Do not model Cilium, Argo CD, or `kro` itself as `kro` APIs. They are
cluster-core primitives, not consumer-facing platform APIs.

## RGD Release And Authoring Contract

The `platform` repo owns shared RGD source and release history. The `gitops`
repo owns which released bundles are installed in each cluster and the
cluster-local resources that consume them.

Release trains:

- `platform-rgds` and `apps-rgds` are independently versioned in `platform`.
- `release-please` manages release PRs, version bumps, tags, and changelog
  updates.
- Publish workflows render final YAML artifacts and push them to OCI registries
  with ORAS.
- Clusters adopt bundle versions by updating the corresponding Argo CD
  Applications in `gitops`.

Authoring model:

- CUE is the build-time authoring and validation language.
- `platform-rgds` starts with one public `Platform` RGD.
- CUE subpackages may model internal capability blocks such as core defaults,
  secrets, networking, and bare-metal integration.
- Those subpackages are authoring boundaries, not separate operator-facing
  APIs.
- CI can import CRDs or equivalent schemas into CUE for structural validation;
  cluster-side `kro` validation remains the final semantic check.

Cluster-local consumption stays intentionally small:

```text
clusters/<cluster>/platform/
├── rgds-platform.yaml
├── rgds-apps.yaml
└── platform.yaml
```

- `rgds-platform.yaml`: install the selected released `platform-rgds` OCI
  artifact.
- `rgds-apps.yaml`: install the selected released `apps-rgds` OCI artifact.
- `platform.yaml`: instantiate the cluster-local `Platform` custom resource.

## Application And Team Model

Workload namespaces use:

```text
team-app-env
```

Examples:

- `teama-appa1-dev`
- `teama-appa1-staging`
- `teama-appa1-prod`
- `teama-appa1-pr-123`

Each workload cluster can use Capsule to enforce team boundaries. The intended
shape is one Capsule tenant per team per workload cluster, while keeping
applications isolated in separate namespaces.

Application environments are concrete instances of shared APIs, not Helm values
or Kustomize overlays. Environment-specific resources should be small and
explicit.

The first developer-facing application API is `App`. Its first pass covers
containers, secrets, configs, and volumes:

- developers author an `App` instance alongside application source
- Kargo materializes the final environment-specific `App` instance into the
  `gitops` repo on `main`
- Argo CD reconciles that materialized instance
- secret references align with External Secrets and environment-owned
  SecretStores or ClusterSecretStores
- plaintext secret values are out of scope
- non-secret config is distinct from secret material
- volumes are declared at the point of use by default
- the API should stay application-centric rather than exposing raw Pod templates

The first schema is still prototype-dependent. The intended shape is concrete
enough to guide implementation:

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
        - path: /etc/orders
          volume:
            config:
              files:
                application.yaml: |
                  http:
                    port: 8080
                  logging:
                    format: json
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

    - name: worker
      image:
        repository: ghcr.io/gilmanlab/teama/orders-worker
        digest: sha256:4444444444444444444444444444444444444444444444444444444444444444
      env:
        - name: LOG_LEVEL
          value: info
      mounts:
        - path: /var/lib/orders
          volume:
            persistent:
              size: 10Gi
```

This example captures the contract, not the final CRD schema:

- containers are the main unit of declaration
- config values are attached where they are consumed
- secret references are attached where they are consumed
- volume definitions are attached where they are mounted
- External Secrets-backed needs are described without hand-authored Kubernetes
  `Secret` objects
- secret store selection defaults from the target environment and is not a
  normal developer-facing input

Generated workload resources stamp stable ownership labels for policy,
selection, and observability:

```text
glab.gilman.io/team
glab.gilman.io/app
glab.gilman.io/env
```

Composition boundaries stay narrow:

- embed related configuration when lifecycle and ownership naturally belong
  with the application instance
- use explicit peer contracts for independently managed or shared capabilities,
  such as a future `Database` resource
- keep environment substitution close to Kargo and the materialized `gitops`
  output
- keep policy and tenancy guardrails in governance layers such as Kyverno and
  Capsule rather than forcing them into `kro`

## Promotion

Kargo runs on the platform cluster.

There should be one promotion project per application pipeline. Durable stages
are `dev`, `staging`, and `prod`; ephemeral environments are created and
destroyed by automation that adds or removes the matching YAML.

The promotion policy is:

- `dev`: automatic promotion is acceptable
- `staging`: automatic promotion is acceptable
- `prod`: promotion requires an explicit approval step

Promotion means editing Git, usually a narrow field such as an image digest.

The working application lifecycle is:

1. A developer authors an `App` instance alongside application source.
2. CI produces an image and the corresponding Git commit.
3. Kargo bundles the image and Git commit into Freight.
4. Kargo promotes that Freight into an environment.
5. During promotion, Kargo combines the source `App` instance with
   environment-specific inputs.
6. Kargo writes the final `App` instance into the destination environment path
   in the `gitops` repo on `main`.
7. Argo CD reconciles that final `App` instance.

For `AppA1`, the long-lived promotion targets are:

- `teams/teama/appa1/envs/dev/app.yaml`
- `teams/teama/appa1/envs/staging/app.yaml`
- `teams/teama/appa1/envs/prod/app.yaml`

Promotion-time composition is intentionally bounded. It may set or combine:

- image repository and digest from Freight
- source Git commit from Freight
- environment-specific non-secret config
- routing, namespace, and cluster placement
- resource sizing or replica count when those values differ by environment
- target-environment secret store defaults

It must not:

- mutate clusters directly
- generate plaintext Kubernetes `Secret` manifests
- change platform-owned defaults that belong in the RGD implementation
- treat mutable Argo CD parameters as the promotion mechanism

The exact composition mechanism needs prototype validation, but the output is
settled: concrete environment-specific YAML in `gitops` on `main`.

## References

- [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/)
- [Cluster API](https://cluster-api.sigs.k8s.io/)
- [Cluster API Provider Incus](https://capn.linuxcontainers.org/)
- [kro](https://kro.run/)
- [Kargo](https://docs.kargo.io/)
