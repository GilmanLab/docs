---
title: Secrets, Identity, DNS, and PKI
description: Bootstrap secrets, runtime secret ownership, AWS anchors, Keycloak, DNS, and internal PKI.
---

# Secrets, Identity, DNS, and PKI

Bootstrap identity must work before any lab cluster is healthy. Runtime secrets
belong inside clusters once they exist. DNS and PKI must also avoid circular
dependencies on the platform cluster they help recover.

## AWS Bootstrap Authority

AWS is the external bootstrap authority.

The AWS structure is a two-account Organization:

- a management account for billing, IAM Identity Center, and organization-level
  configuration
- a `lab` member account for workloads and bootstrap resources

The `lab` account owns the VPC, Route 53 private zone, Tailscale subnet router,
Keycloak host, SOPS KMS key, SSM parameters, and GitHub token broker Lambda.

AWS access uses IAM Identity Center with a local identity-store user and a
hardware security key. Do not make AWS access depend on Keycloak; Keycloak
depends on AWS.

AWS resources are managed with OpenTofu from the `infra` repo under
`infra/aws/`. OpenTofu state lives in an S3 bucket in the `lab` account using
S3 native locking.

The manual AWS bootstrap surface is intentionally small:

1. Create the AWS Organization and the management/member accounts.
2. Enable IAM Identity Center and create the operator user.
3. Create the S3 state bucket.
4. Create the initial IAM role that OpenTofu assumes.

Everything downstream of that, including the VPC, subnet router, private zone,
KMS keys, SSM parameters, instance profiles, and GitHub token broker, belongs in
OpenTofu.

The SOPS KMS key is the customer-managed key in the current `lab` account:

```text
alias/glab-sops
arn:aws:kms:us-west-2:186067932323:key/2aba1d94-6eaf-4d80-8d26-2077f32fd7c5
```

## Secrets Bootstrap

Bootstrap secrets are the minimum material needed to stand up or recover the
lab before Vault and cluster-local controllers are available.

The bootstrap chain is intentionally rooted in an AWS IAM role:

- encrypted files live in the private `secrets` repo
- those files are encrypted with a customer-managed AWS KMS key used as a SOPS
  recipient
- the GitHub App private key is stored in SSM Parameter Store
- bootstrap callers invoke `github-token-broker` to receive a short-lived
  installation token for `GilmanLab/secrets`
- the caller fetches encrypted files with that token and decrypts locally
  through KMS

GitHub access fetches encrypted files. AWS KMS access decrypts them. The GitHub
token broker is only a token broker; it is not a file broker and not a KMS
decryptor.

The GitHub App token path is:

- the GitHub App private signing key is stored in SSM Parameter Store as a
  `SecureString`
- the `github-token-broker` Lambda execution role can read that SSM parameter
- bootstrap principals can invoke the broker, but do not read the App private key
- the broker returns a short-lived installation token for `GilmanLab/secrets`
  with `contents:read`
- callers use that token with `git` or the GitHub Contents API, and do not store
  it on disk

SOPS files use AWS KMS encryption context so IAM can grant decrypt access by
path-oriented scope. SOPS is not a field-level authorization system.

Example scope layout:

```text
network/tailscale/*          Scope=network-tailscale
network/vyos/*               Scope=network-vyos
compute/talos/platform/*     Scope=talos-platform
vault/platform/*             Scope=vault-platform
vault/nonprod/*              Scope=vault-nonprod
vault/prod/*                 Scope=vault-prod
```

Each encrypted file includes context similar to:

```yaml
Repo: GilmanLab/secrets
Scope: network-tailscale
```

A caller scoped to `network/tailscale/*` receives short-lived AWS credentials
for a role that can call `kms:Decrypt` only when the request has both
`Repo=GilmanLab/secrets` and `Scope=network-tailscale`.

PGP and age recipients are removed from current SOPS metadata after the KMS
cutover. That does not remove their ability to decrypt old git revisions, so
any secret that must become AWS-authoritative retroactively is rotated rather
than relying on history rewrite.

Examples of bootstrap material include:

- Talos and cluster bootstrap material
- DNS and PKI bootstrap material
- GitHub App bootstrap material
- initial Vault or Keycloak material where no runtime owner exists yet

SOPS-encrypted bootstrap material lives in the private `secrets` repo. Public
repos may reference paths, identities, and workflows, but must not contain
plaintext secret payloads.

## Runtime Secrets

Each Kubernetes cluster runs its own HashiCorp Vault instance managed by
`bank-vaults`. Vault is the runtime source of truth for secrets in that
cluster, not a shared platform service.

The intended cluster split is:

| Cluster | Vault scope |
| --- | --- |
| `platform` | Platform-cluster services and platform control-plane needs |
| `nonprod` | Non-production workloads |
| `prod` | Production workloads |

Vault instances do not read each other's storage, policies, tokens, or secret
paths. `prod` does not depend on `nonprod`; `nonprod` does not depend on
`prod`; and the platform cluster does not become a universal secret broker for
downstream clusters.

Clusters that host multiple environments segregate secrets by path and policy.
For `nonprod`, the baseline shape is:

```text
dev/*
staging/*
```

Path naming is not the security boundary by itself. Vault auth roles and
policies enforce which workloads, namespaces, and service accounts can read or
write each prefix.

Runtime ownership starts after bootstrap:

```text
SOPS bootstrap material -> initialize/configure Vault -> Vault owns runtime secret distribution
```

Do not create a long-term two-source-of-truth model where SOPS and Vault both
own mutable runtime secret state. If runtime material must be recovered from
bootstrap inputs, document it as a recovery flow.

Vault unseal and recovery material is scoped per cluster even if the lab uses
one customer-managed AWS KMS key for cost control. The key may wrap distinct
per-cluster Vault material, but it is not a shared Vault unseal key.

Isolation is enforced with:

- per-cluster S3 prefixes or buckets for `bank-vaults` storage
- per-cluster IAM roles
- KMS encryption context such as `Purpose=vault-unseal` and `Cluster=nonprod`

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
fixed monthly KMS cost is not worth it at this lab scale unless prototype work
shows that the shared-key policy model is too awkward.

## DNS

The lab domain is `glab.lol`.

The authoritative private zone lives in Route 53 in the `lab` account. A sync
job on the AWS-side subnet router renders the zone to a local zonefile, and the
in-lab DNS path serves that local copy. The read path should not query Route 53
at request time.

Cluster-local DNS automation writes only the delegated cluster zones it owns.
This keeps name resolution available during internet or platform-cluster
outages and prevents workload controllers from mutating management records.

## TLS And PKI

Public HTTP TLS uses Let's Encrypt with Route 53 DNS-01 validation. The
dedicated ACME validation zone is under `acme.glab.lol`; cluster issuers should
write only their scoped validation records.

Cluster responsibilities are split:

- ExternalDNS manages service DNS records.
- cert-manager manages ACME orders, challenges, and certificate renewal.

ExternalDNS does not manage `_acme-challenge` TXT records. Those belong to
cert-manager.

`acme.glab.lol` is a public Route 53 validation zone delegated from Cloudflare.
Clusters use that zone so workload controllers do not need broad Cloudflare DNS
access.

Challenge records use CNAME delegation:

```text
_acme-challenge.<service>.glab.lol
  CNAME _acme-challenge.<service>.<cluster>.acme.glab.lol
```

cert-manager follows the CNAME and writes TXT records into Route 53 using
short-lived AWS credentials. Each cluster role is scoped to its own challenge
names. No wildcard certificate is assumed; individual services receive
individual certificates unless a future workload justifies the broader blast
radius.

Cluster workloads that need AWS credentials, including ExternalDNS and
cert-manager, use the cluster's Kubernetes OIDC issuer to assume IAM roles in an
IRSA-style flow. They do not use Tailscale workload identity federation; that
mechanism is reserved for AWS-hosted clients such as the subnet router.

Internal runtime PKI uses per-cluster Vault intermediates signed by an
AWS-KMS-backed root CA. The hierarchy leaves room for future SPIRE:

```text
AWS KMS root CA pathlen:2
  -> cluster Vault intermediate pathlen:1
    -> SPIRE intermediate pathlen:0
      -> workload SVID leaves
```

Root signing is operationally offline. No always-on lab workload has standing
permission to use the root key. Root signing is used only to mint or rotate
cluster subordinate CAs.

For clusters where Vault directly issues mTLS leaves, the same hierarchy still
works without SPIRE:

```text
AWS KMS root CA pathlen:2
  -> cluster Vault intermediate pathlen:1
    -> workload mTLS leaves
```

Each cluster gets its own subordinate CA generated and held by that cluster's
Vault instance. Vault generates the intermediate private key and CSR; the AWS
KMS root signs the CSR; and the signed intermediate is imported back into
Vault.

The cluster subordinate CA identity includes the cluster name. Example common
names:

```text
glab platform Vault CA
glab nonprod Vault CA
glab prod Vault CA
```

Vault PKI roles issue short-lived certificates for internal use cases such as
service-to-service mTLS, database client authentication, internal controllers,
and future SPIRE upstream authority material.

The current implementation history matters for rebuilds: the existing
`infra/security/pki/root-ca` stack was applied against an earlier AWS account.
The lab root must be recreated in the current `lab` account with the
`pathlen:2` hierarchy, then the earlier account root key and state can be
cleaned up after consumers trust the new chain.

Router-hosted `step-ca` is limited to existing consumers during migration.
Runtime issuance uses cert-manager plus Route 53 for public TLS and Vault for
internal PKI.

The small implementation slices are:

1. Rewrap existing SOPS files with `alias/glab-sops`, add encryption context,
   and remove PGP/age recipients.
2. Rotate bootstrap secrets that previously depended on PGP/age-only history.
3. Create the GitHub App plus SSM bootstrap path for `secrets` repo access.
4. Recreate the internal root CA in the current `lab` account with the new path
   length.
5. Add the shared Vault unseal KMS key and `bank-vaults` storage layout.
6. Stand up Vault in one cluster and prove the SOPS-to-Vault bootstrap handoff.
7. Add cert-manager DNS-01 with Route 53 ACME delegation for one cluster.
8. Migrate internal PKI consumers from `step-ca` to the per-cluster issuers.

Open implementation threads:

- exact KMS encryption-context key names and allowed scope values
- exact GitHub App name, installation ID storage, and SSM parameter paths
- whether Vault unseal material uses one shared S3 bucket with prefixes or
  separate per-cluster buckets
- how trust bundles are distributed to workloads that need to trust internal
  Vault or SPIRE issuers
- whether any future public TLS use case justifies wildcard certificates
- when remaining `step-ca` consumers should be allowed to expire naturally
  versus being actively replaced

## Identity

Keycloak is the central human-facing identity system for lab services, but it
is not a bootstrap dependency for the raw recovery path.

Keycloak runs on a dedicated EC2 instance in the `lab` account, colocated with
Postgres and managed with Docker Compose. It is reached at `id.glab.lol`.
GitHub is the upstream identity provider through OIDC.

Keycloak configuration should be reconciled from git. Runtime state such as
sessions, user credentials, and TOTP enrollment is backed up separately.

See [Keycloak Runtime](./keycloak-runtime.md) for the EC2 runtime shape,
backup contract, rebuild/restore paths, hostname constraint, and break-glass
matrix.

Break-glass paths must still exist for:

- AWS bootstrap access
- Talos API access
- Kubernetes admin kubeconfig generation
- Vault recovery
- router access

The identity system itself should be rebuilt from declarative configuration and
backups rather than treated as an irreplaceable pet.

## References

- [AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)
- [cert-manager Route 53 DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/route53/)
- [cert-manager delegated DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/#delegated-domains-for-dns01)
- [SOPS](https://github.com/getsops/sops)
- [Vault](https://developer.hashicorp.com/vault/docs)
- [Vault PKI secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- [Vault PKI intermediate guidance](https://developer.hashicorp.com/vault/docs/secrets/pki/considerations)
- [Bank-Vaults unseal keys](https://bank-vaults.dev/docs/concepts/unseal-keys/)
- [SPIRE configuration](https://spiffe.io/docs/latest/deploying/configuring/)
- [Keycloak](https://www.keycloak.org/documentation)
- [Smallstep step-ca](https://smallstep.com/docs/step-ca/)
