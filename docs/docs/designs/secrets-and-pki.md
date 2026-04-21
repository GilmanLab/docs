---
title: Secrets and PKI
description: Proposed design for bootstrap secrets, per-cluster Vault, and the lab PKI split.
---

# Secrets and PKI

## Status

Proposed.

This document defines the target model for secrets and PKI across the lab. It
connects the AWS bootstrap foundation from [AWS Lab Account](./aws-lab-account.md)
to the future per-cluster Vault model and replaces the older router-hosted
`step-ca` internal PKI design.

## Purpose

The lab needs two different secret systems:

- a **bootstrap** system that works before any cluster or Vault instance exists
- a **runtime** system that belongs to each cluster after that cluster exists

The bootstrap system must be small, externally anchored, and able to recover
the lab from a cold start. The runtime system must be local to the cluster it
serves, because `platform`, `nonprod`, and `prod` are independent recovery and
security domains.

PKI follows the same split. Public HTTP TLS uses public trust through Let's
Encrypt. Internal workload identity and mTLS use lab-managed authorities rooted
in an AWS KMS-held root CA and delegated to cluster-local Vault or future SPIRE
issuers.

## Goals

- Make AWS the authoritative control surface for bootstrap secret access.
- Remove standing PGP and age decryptors from SOPS-encrypted bootstrap
  material.
- Allow short-lived, scope-limited access to selected bootstrap secrets.
- Keep runtime secrets in each cluster's own Vault instance.
- Prevent cross-cluster secret reads between `platform`, `nonprod`, and `prod`.
- Use Let's Encrypt for browser-facing HTTP TLS.
- Use cluster-local subordinate CAs for internal mTLS and future SPIFFE/SPIRE.
- Retire the router-hosted `step-ca` internal issuer from the target design.

## Non-Goals

- This document does not define exact IAM policy JSON, Vault policy HCL, or
  Kubernetes manifests.
- This document does not define a general secret synchronization mechanism
  between SOPS and Vault.
- This document does not make the platform cluster a central secret broker for
  downstream clusters.
- This document does not federate SPIRE trust domains between clusters.
- This document does not cover Keycloak runtime backups or realm
  configuration; those remain in [Keycloak](./keycloak.md).

## Design Summary

Secrets are split into two layers:

| Layer | Authority | Scope | Purpose |
| --- | --- | --- | --- |
| Bootstrap | AWS + GitHub App + SOPS | Pre-cluster and recovery | Retrieve enough material to create or recover clusters, Vault, and base platform services. |
| Runtime | Per-cluster Vault | One cluster | Store and issue secrets for workloads in that cluster. |

PKI is split into two layers:

| Use case | Authority | Issuer |
| --- | --- | --- |
| HTTP TLS | Public WebPKI | Let's Encrypt through cert-manager DNS-01 and Route 53 |
| Internal mTLS / workload identity | Lab internal PKI | Per-cluster Vault subordinate CAs, with future SPIRE authorities |

The root invariant is:

```text
AWS grants the workload identity.
GitHub App grants temporary repository read access.
AWS KMS grants temporary scoped SOPS decrypt access.
Cluster-local Vault grants runtime secret and certificate access.
```

## Bootstrap Secrets

### Source of truth

Bootstrap secret payloads live in the private `GilmanLab/secrets` repository.
Public repositories keep only templates, variable contracts, and lookup logic.

The `secrets` repository holds only material that is needed before a cluster's
Vault is ready, or material needed to recover that Vault. Examples include:

- Talos and cluster bootstrap material
- initial Vault unseal/recovery storage configuration
- GitHub App bootstrap material
- credentials needed to create the first runtime secret sources
- emergency recovery material that cannot live inside the thing it recovers

Runtime application secrets do not belong in SOPS once Vault exists.

### SOPS over AWS KMS

All existing SOPS files are rewrapped with the customer-managed KMS key in the
current `lab` AWS account:

```text
alias/glab-sops
arn:aws:kms:us-west-2:186067932323:key/2aba1d94-6eaf-4d80-8d26-2077f32fd7c5
```

The existing PGP and age recipients are removed from SOPS metadata. After the
cutover, AWS is the only routine decrypt control surface.

### Scoped decryption

SOPS files are encrypted with AWS KMS encryption context so IAM can grant
decrypt access by scope. Scopes are file/path oriented; SOPS is not a
field-level authorization system.

Example scope layout:

```text
network/tailscale/*          Scope=network-tailscale
network/vyos/*               Scope=network-vyos
compute/talos/platform/*     Scope=talos-platform
vault/platform/*             Scope=vault-platform
vault/nonprod/*              Scope=vault-nonprod
vault/prod/*                 Scope=vault-prod
```

Each encrypted file includes KMS context similar to:

```yaml
Repo: GilmanLab/secrets
Scope: network-tailscale
```

A workload that needs only `network/tailscale/*` receives short-lived AWS
credentials for a role that can call `kms:Decrypt` only when the request's
encryption context has `Repo=GilmanLab/secrets` and
`Scope=network-tailscale`.

Because KMS encryption context is authenticated data bound to the encrypted
data key, changing the SOPS file metadata to a different scope does not allow
the ciphertext to decrypt.

### Repository access

Private repository access uses a GitHub App owned by `GilmanLab` and installed
on `GilmanLab/secrets`.

The App private signing key is stored in SSM Parameter Store as a SecureString
in the `lab` account. Bootstrap workloads do not read that key directly. They
invoke the `github-token-broker` Lambda with their AWS identity. The broker is
the only normal principal that can read the App private key, generate a GitHub
App JWT, and exchange it for a short-lived installation token.

Installation tokens are requested with the narrowest useful shape:

```text
repositories = ["secrets"]
permissions  = {"contents": "read"}
ttl          = 1 hour
```

The GitHub token grants access to encrypted files. AWS KMS grants access to
plaintext. A workload may be able to clone the whole private repository while
still being unable to decrypt files outside its KMS context scope.

Callers may use either `git` or the GitHub Contents API with `curl` to fetch
encrypted files. The no-`git` path still uses the GitHub token only for
encrypted repository access; the Lambda does not return SOPS files and does
not decrypt them.

### Historical exposure

Removing PGP and age recipients from current SOPS files does not remove their
ability to decrypt old git revisions. Any bootstrap secret that must become
AWS-authoritative retroactively is rotated after the KMS cutover.

History rewrite is possible but is not the default. For this lab, rotating
affected secrets is the simpler and more auditable path.

## Runtime Secrets

### Per-cluster Vault

Each cluster runs its own HashiCorp Vault instance managed by `bank-vaults`.
Vault is the runtime source of truth for secrets in that cluster.

The intended cluster split is:

| Cluster | Vault scope |
| --- | --- |
| `platform` | Platform-cluster services and platform control-plane needs |
| `nonprod` | Non-production workloads |
| `prod` | Production workloads |

Vault instances do not read each other's storage, policies, tokens, or secret
paths. `prod` does not depend on `nonprod`; `nonprod` does not depend on
`prod`; `platform` does not become a universal secret broker.

### Environment separation

Clusters that host multiple environments segregate secrets by path and policy.
For `nonprod`, the baseline shape is:

```text
dev/*
staging/*
```

Path naming is not the security boundary by itself. Vault auth roles and
policies enforce which workloads, namespaces, and service accounts can read or
write each prefix.

### Bootstrap-to-runtime handoff

SOPS may seed initial Vault configuration and initial secret material during
cluster bootstrap. Once the cluster is operating, runtime mutation belongs in
Vault.

There is no bidirectional SOPS-to-Vault synchronization loop. That would create
two sources of truth. The direction is:

```text
SOPS bootstrap material -> initialize/configure Vault -> Vault owns runtime
```

If a runtime secret must be recovered from bootstrap material, the recovery
process is explicit and documented for that secret class.

### Vault unseal material

For cost management, Vault unseal and root/recovery material may share one
customer-managed AWS KMS key across clusters. The KMS key wraps distinct
per-cluster Vault material; it is not a shared Vault unseal key.

Isolation is enforced with:

- per-cluster S3 prefixes or buckets for bank-vaults storage
- per-cluster IAM roles
- KMS encryption context such as `Purpose=vault-unseal` and
  `Cluster=nonprod`

Example:

```text
KMS key: alias/glab-vault-unseal

S3:
  s3://glab-vault-unseal/platform/*
  s3://glab-vault-unseal/nonprod/*
  s3://glab-vault-unseal/prod/*

KMS context:
  Purpose = vault-unseal
  Cluster = platform | nonprod | prod
```

One KMS key per cluster would provide cleaner blast-radius isolation, but the
fixed monthly KMS cost is not worth it at this lab scale.

## Public HTTP TLS

HTTP TLS certificates are always issued by Let's Encrypt through ACME DNS-01
against Route 53.

Cluster responsibilities:

- ExternalDNS manages service DNS records.
- cert-manager manages ACME orders, challenges, and certificate renewal.

ExternalDNS does not manage `_acme-challenge` TXT records. Those belong to
cert-manager.

The AWS account already contains a public Route 53 ACME validation zone,
`acme.glab.lol`, delegated from Cloudflare. The cluster TLS design uses that
zone rather than granting cluster workloads broad Cloudflare DNS access.

The challenge delegation convention is:

```text
_acme-challenge.<service>.glab.lol
  CNAME _acme-challenge.<service>.<cluster>.acme.glab.lol
```

cert-manager is configured to follow CNAMEs and write TXT records into the
Route 53 ACME zone using short-lived AWS credentials. Each cluster's AWS role
is scoped to its own challenge names.

No wildcard certificate is assumed. Individual services receive individual
certificates unless a future workload proves a wildcard is worth the broader
blast radius.

## Internal PKI

### Root CA

The internal PKI root is an AWS KMS asymmetric signing key. The private key
never leaves KMS.

The existing `infra/security/pki/root-ca` stack was applied against an old AWS
account and is not the target root. The target implementation recreates the
root CA in the current `lab` account, then cleans up the old-account root key
and state.

During recreation, the root certificate's path length is increased from the
current `pathlen:1` model. The recommended target is `pathlen:2`:

```text
Root CA pathlen:2
  -> cluster Vault intermediate pathlen:1
    -> SPIRE intermediate pathlen:0
      -> workload SVID leaves
```

This keeps the future SPIRE path open without requiring another root rotation.
For clusters where Vault directly issues mTLS leaves, the same hierarchy still
works:

```text
Root CA pathlen:2
  -> cluster Vault intermediate pathlen:1
    -> workload mTLS leaves
```

Root signing is operationally offline. No always-on lab workload has standing
permission to use the root key. Root signing is used only to mint or rotate
cluster subordinate CAs.

### Cluster subordinate CAs

Each cluster gets its own subordinate CA, generated and held by that cluster's
Vault instance. Vault generates the intermediate private key and CSR; the AWS
KMS root signs the CSR; the signed intermediate is imported back into Vault.

The cluster subordinate CA identity includes the cluster name. Example common
names:

```text
glab platform Vault CA
glab nonprod Vault CA
glab prod Vault CA
```

Vault PKI roles issue short-lived certificates for internal use cases such as:

- service-to-service mTLS
- database client authentication
- internal controllers that need X.509 credentials
- future SPIRE upstream authority material

### SPIFFE and SPIRE

SPIRE is a future addition, not a baseline dependency.

The first SPIRE deployment in each cluster uses an independent trust domain and
does not federate with other clusters. That keeps `platform`, `nonprod`, and
`prod` aligned with the Vault isolation model.

When SPIRE is introduced, the preferred shape is:

```text
cluster Vault CA -> SPIRE intermediate -> workload SVIDs
```

The SPIRE intermediate is local to the cluster. Federation is deferred until a
real cross-cluster workload requires it.

## Retiring step-ca

The older architecture placed `Smallstep step-ca` on `VP6630` as the online
internal intermediate CA. That was the right bootstrap-oriented first shape,
but it is not the target model for this design.

In this design:

- public HTTP TLS moves to Let's Encrypt and Route 53 DNS-01
- internal runtime issuance moves to per-cluster Vault
- future workload identity moves to per-cluster SPIRE
- `step-ca` is removed once its remaining consumers have migrated

The root CA migration is the natural time to make this break. Old chains can
expire or be replaced as consumers move to the new issuers.

## Implementation Slices

This design should be implemented in small slices:

1. Rewrap existing SOPS files with `alias/glab-sops`, add encryption context,
   and remove PGP/age recipients.
2. Rotate bootstrap secrets that previously depended on PGP/age-only history.
3. Create the GitHub App + SSM bootstrap path for `secrets` repo access.
4. Recreate the internal root CA in the current `lab` AWS account with the new
   path length.
5. Clean up the old-account PKI root after the new root is usable.
6. Add the shared Vault unseal KMS key and bank-vaults storage layout.
7. Stand up Vault in one cluster and prove SOPS -> Vault bootstrap handoff.
8. Add cert-manager DNS-01 with Route 53 ACME delegation for one cluster.
9. Migrate internal PKI consumers away from `step-ca`.

## Open Threads

- Exact KMS encryption context keys and scope names.
- Exact GitHub App name, installation ID storage, and SSM parameter path.
- Whether Vault unseal material uses one shared S3 bucket with prefixes or
  separate per-cluster buckets.
- Whether the new root CA should use `pathlen:2` exactly or a larger value.
- How trust bundles are distributed to workloads that need to trust internal
  Vault or SPIRE issuers.
- Whether public TLS certificates should ever use wildcards.
- When old `step-ca` certificates are allowed to expire versus actively
  replaced.

## References

- [AWS Lab Account](./aws-lab-account.md)
- [Keycloak](./keycloak.md)
- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md)
- [cert-manager Route 53 DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/route53/)
- [cert-manager delegated DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/#delegated-domains-for-dns01)
- [Vault PKI secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- [Vault PKI intermediate guidance](https://developer.hashicorp.com/vault/docs/secrets/pki/considerations)
- [Bank-Vaults unseal keys](https://bank-vaults.dev/docs/concepts/unseal-keys/)
- [SPIRE configuration](https://spiffe.io/docs/latest/deploying/configuring/)
