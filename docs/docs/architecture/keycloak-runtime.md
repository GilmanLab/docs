---
title: Keycloak Runtime
description: Keycloak deployment, configuration, backups, recovery, and break-glass paths.
---

# Keycloak Runtime

Keycloak is the central human-facing identity system for lab services. It is
outside the lab's physical failure domain, but it is not a bootstrap dependency
for raw recovery.

## Deployment Shape

Keycloak runs on one dedicated EC2 instance in the `lab` account.

The runtime contract is:

- instance: `t4g.small`, Amazon Linux 2023 on ARM, in the `172.16.0.0/16` VPC
- runtime: Docker Compose
- services: upstream Keycloak plus upstream Postgres, both pinned in `infra`
- database: colocated Postgres with data on the instance EBS root volume
- access name: `id.glab.lol`
- TLS: Traefik ACME DNS-01 through Route 53 using the instance IAM role
- reverse proxy: Traefik on the host, terminating TLS and proxying to Keycloak
  on loopback

State recovery uses application backups. Do not depend on EBS snapshots or AMI
backups as the primary recovery path.

The `t4g.small` shape is tight but workable for a single-user lab. The runtime
must be tuned for the 2 GB memory budget:

- set an explicit Keycloak JVM max heap around 768 MB
- keep Postgres `shared_buffers` conservative, around 128 MB
- provision an EBS-backed swap file as a safety margin
- enable burstable CPU unlimited mode so occasional login bursts do not throttle
  the instance

## IAM Contract

The instance IAM role grants only what the runtime needs:

- Route 53 writes scoped to `_acme-challenge.id.glab.lol`
- S3 write access to the Keycloak backup bucket prefix
- SSM Parameter Store reads for bootstrap-time values such as reconciliation
  credentials

Cluster access, secret decryption, and tailnet access follow the AWS bootstrap
and secrets contracts. Keycloak does not receive broad AWS administrative
permissions.

## Realm And Federation

One realm named `lab` holds lab users and OIDC/SAML clients.

GitHub is the only upstream identity provider and is federated through OIDC.
There is no standing local username-password fallback for normal users. The
bootstrap admin user exists only during initial realm creation and is disabled
after `keycloak-config-cli` reconciles the realm from git.

The first expected clients are:

- Kubernetes API OIDC for Talos clusters
- Argo CD web UI and CLI
- Grafana

The authoritative client list lives in the realm repository.

## Configuration As Code

The realm repository is the source of truth for Keycloak's declarative surface:

- realms
- clients
- client scopes
- roles and role mappings
- identity-provider configuration
- authentication flows and required actions
- realm-level settings

Runtime state is intentionally out of git:

- user credentials
- WebAuthn registrations
- TOTP secrets
- sessions and refresh tokens
- audit and event logs
- ephemeral tokens and one-time codes

`keycloak-config-cli` reconciles from the realm repository on a short schedule
from the Keycloak host. It authenticates with a service account whose secret is
stored in SSM Parameter Store. Keycloak version upgrades are driven by bumping
the pinned runtime version in `infra` and reconciling forward.

## Backups

Backups contain:

- a Postgres dump
- host-local Keycloak configuration files such as `keycloak.conf`
- environment overrides
- custom themes or providers
- the current TLS cert bundle and private key

Git-tracked realm configuration is not part of the backup payload because git
is already the durable store for it.

Backups are written nightly to an S3 bucket in the `lab` account. The bucket
uses SSE-KMS and object lock or versioning so corruptions cannot silently
overwrite known-good backups. The Synology NAS pulls a secondary copy on its
own schedule.

Retention contract:

- daily backups for 30 days
- weekly backups for 12 weeks
- monthly backups for 12 months

Backup payloads are encrypted before upload with a recipient key managed with
the lab's bootstrap secrets, and the S3 bucket also uses SSE-KMS.

## Recovery

The primary path is rebuild-first:

1. Provision a fresh EC2 instance from `infra`.
2. Start Keycloak and fresh Postgres through Docker Compose.
3. Run `keycloak-config-cli` against the new instance using the realm repo.
4. Sign in through GitHub.
5. Re-enroll WebAuthn or TOTP.

Target RTO for the single-user lab is 15 minutes. This path requires AWS access
and git; it does not require a backup store.

The restore fallback is:

1. Provision a fresh EC2 instance.
2. Pull a selected point-in-time backup from S3 or the NAS.
3. Restore the Postgres dump.
4. Place the TLS cert bundle and config files.
5. Start Docker Compose.
6. Let `keycloak-config-cli` reconcile the restored runtime forward to git
   `HEAD`.

Restores are for cases where preserving exact runtime state matters, such as
federated identity linkages, event history, and active user state.

## Hostname Constraint

Recovered Keycloak instances must serve at `id.glab.lol`.

The issuer claim in signed JWTs and client OIDC discovery is tied to that URL.
Changing it invalidates existing tokens and client configuration.

Normally the Route 53 A record points `id.glab.lol` at the replacement
instance. During a full internet-loss recovery where Route 53 is unreachable,
the local CoreDNS zonefile can be edited to point `id.glab.lol` at the restored
instance's reachable address.

## Break-Glass Matrix

When Keycloak is down, the lab keeps operating through service-local
authentication paths.

| Service | Break-glass path | Notes |
| --- | --- | --- |
| Talos API | mTLS via `talosconfig` and machine secrets | Talos PKI is independent of Keycloak. |
| Kubernetes | Talos-generated admin kubeconfig via `talosctl kubeconfig` | Produced on demand from each cluster's signing CA. |
| Argo CD | Built-in `admin` account and initial admin secret | Retained and rotated, not disabled. |
| Vault | Unseal keys and root/recovery keys | Stored outside any Keycloak-dependent path. |
| AWS | IAM Identity Center local user with hardware key | AWS does not federate to Keycloak. |
| Grafana | Local admin account | Kept active alongside OIDC. |
| GitHub | Personal GitHub account with hardware-key MFA | GitHub is upstream of Keycloak. |

These anchors live outside Keycloak-dependent storage.
