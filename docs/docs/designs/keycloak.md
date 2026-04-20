---
title: Keycloak
description: Proposed design for the lab's central identity system — deployment shape, federation, configuration, backups, and the rebuild-over-restore disaster-recovery model.
---

# Keycloak

## Status

Proposed.

This document defines how the lab runs Keycloak as its central identity
provider. It covers deployment shape, identity federation, declarative
configuration, TLS, backups, the rebuild-over-restore disaster-recovery
model, and the per-service break-glass matrix that lets the lab continue to
operate while Keycloak is down.

This document assumes the [AWS Lab Account](./aws-lab-account.md) design. It
does not re-establish shared context about the AWS Organization, networking,
IAM, or secrets bootstrap.

## Purpose

The primary purpose of this design is to keep the lab's identity system on a
durable off-lab trust anchor, while keeping the lab itself operable when that
trust anchor is unreachable.

The intended split is:

- Keycloak holds the authoritative, human-facing identity of record
- cluster-level and service-level **break-glass paths** exist for every OIDC
  consumer, so identity outages do not cascade into cluster-access outages
- configuration is declaratively sourced from git, so **rebuild is the default
  recovery mode** and restore is a fallback used only for runtime state a
  single-user lab can recreate in seconds

This mirrors a day-job architecture at small scale without overpaying for
HA features a single-user lab cannot justify.

## Goals

- Provide one place to manage human identity across the lab.
- Keep Keycloak outside the lab's physical failure domain while keeping its
  blast radius understood.
- Make Keycloak's configuration surface fully declarative via git, so rebuild
  is a first-class recovery path.
- Ensure every Keycloak-dependent service has a documented break-glass path
  that does not require Keycloak.
- Make the disaster-recovery procedure short enough to execute without a
  runbook open on a phone.

## Non-Goals

- This document does not run Keycloak in a highly-available configuration.
  Single-node is a deliberate choice for a single-user lab and is not a gap.
- This document does not define the per-service Keycloak client configuration
  (redirect URIs, scopes, role mappings, token TTLs). Those live in the
  realm repository.
- This document does not define monitoring, logging, or alerting.
- This document does not define the Keycloak → Identity Center SAML
  federation path. That remains future work.
- This document does not define cluster-level OIDC federation to AWS (used
  by ExternalDNS and similar controllers). That is a separate concern handled
  by the cluster's own OIDC issuer, not by Keycloak.

## Design Summary

Keycloak runs on a single dedicated EC2 instance in the `lab` member account,
colocated with its Postgres database. Access is at **`id.glab.lol`** via a
Route 53 private-zone record and a TLS certificate issued automatically via
**ACME DNS-01**. The instance and database are deployed via **Docker
Compose**.

The only upstream identity source is **GitHub, federated via OIDC**. A single
realm named `lab` holds all users and all OIDC/SAML clients. Sign-in to any
Keycloak-fronted service is: user → service → Keycloak → GitHub.

Keycloak's declarative surface — realms, clients, roles, identity provider
settings, scopes, authentication flows — is reconciled from a git repository
by **`keycloak-config-cli`** running as a scheduled job on the Keycloak host.
Runtime state (user credentials, sessions, TOTP enrollment) is not in git and
is the only part of the system that needs backup-based recovery.

Database dumps and the current TLS cert bundle are backed up nightly to an
S3 bucket in the `lab` account. The lab's Synology NAS pulls those backups
locally on a schedule, so a recent copy of Keycloak's runtime state exists on
a second continent and a second provider.

**Disaster recovery is rebuild-first.** For a single-user lab, a rebuild from
the git-tracked realm + a fresh Postgres is faster than a restore, enforces
discipline that every config-surface change actually lives in git, and
produces a clean outcome. A restore path exists as fallback but is not the
primary recovery mode.

When Keycloak is entirely unavailable, every Keycloak-fronted service has a
local break-glass path documented in this doc. The lab continues to operate;
only the "unified identity" experience degrades.

## Deployment Shape

### Host and Runtime

- **Instance:** `t4g.small` (2 vCPU, 2 GB RAM), Amazon Linux 2023 on ARM, in
  the `lab` account and `172.16.0.0/16` VPC from the AWS design.
- **Runtime:** Docker Compose manages two services:
  - `keycloak` — official upstream Keycloak image, tagged to a specific
    version pinned in `infra/`.
  - `postgres` — official Postgres image, tagged to a specific version pinned
    in `infra/`. Data volume on the instance's EBS root volume.
- **Reverse proxy:** Caddy (or equivalent) runs alongside and terminates TLS,
  proxying to Keycloak on loopback. Caddy performs ACME DNS-01 renewals using
  the instance's IAM role. This is the recommended shape; the exact proxy is
  an implementation detail that does not need to appear in this doc.
- **No EBS snapshots or AMI backups.** State recovery is via the application
  backup path below, not via block-level snapshots.

### Identity Profile for the Host

The EC2 instance's IAM role grants, at minimum:

- Route 53 write access scoped to the `_acme-challenge.id.glab.lol` record
  for DNS-01 validation.
- S3 write access to the Keycloak backup bucket's prefix.
- SSM Parameter Store read for any bootstrap-time secrets held there
  (following the pattern established in the AWS design).

The role carries no other permissions. All lab cluster-access,
secret-decryption, and tailnet-identity paths that this instance depends on
are established in the AWS Lab Account design and are not restated here.

### Sizing and Tuning

2 GB of RAM is tight but workable for a single-user lab because Keycloak 26.x
(Quarkus-based) has a much smaller footprint than earlier WildFly-based
versions. The required tuning is:

- explicit Keycloak JVM max heap (e.g. ~768 MB)
- conservative Postgres `shared_buffers` (~128 MB)
- a swap file on the EBS volume as a safety margin

CPU is burstable but effectively idle for single-user workloads; unlimited
mode is enabled to tolerate rare login bursts at negligible cost.

## Identity Federation

### Realm Structure

A single realm named `lab` holds all lab users and all OIDC/SAML clients. No
separate realms for services vs. humans; a single-user lab does not benefit
from the separation, and multi-realm setups make GitOps reconciliation more
fragile.

### Upstream IdP

The realm has **exactly one identity provider configured: GitHub, via
OIDC**. There is no local username-password fallback. A lab user's identity
is their GitHub identity, federated through Keycloak, presented downstream to
each OIDC client.

The Keycloak admin bootstrap user exists only briefly during initial realm
creation and is disabled once `keycloak-config-cli` has reconciled the
realm from git.

### Intended Early OIDC Clients

Keycloak is expected to front these services first:

- **kubectl → Talos clusters** via each cluster's Kubernetes API-server OIDC
  configuration.
- **Argo CD** (web UI and CLI).
- **Grafana** (when deployed).

Additional clients will be added over time. The authoritative list lives in
the realm repository, not in this document.

## TLS

### Issuance

The cert for `id.glab.lol` is issued via **ACME DNS-01 against Route 53**
using the instance's IAM role. There is no internet-exposed HTTP-01 path,
because the host sits in a private VPC with the subnet router as its only
tailnet attachment.

No wildcard cert is used. Each service in the lab gets its own host-scoped
cert:

- this host: `id.glab.lol`
- future clusters: `nonprod.k8s.glab.lol`, `prod.k8s.glab.lol`
- future services: their own host names

Cluster-fronted services will issue their own certs via cert-manager and
each cluster's OIDC trust relationship with AWS (a separate future design).

### Renewal

Renewal is automatic and handled by whichever TLS-terminating proxy is used
on the host. No human intervention is expected between renewals.

### Restore-time behavior

For same-hostname restore to work without waiting on ACME, the current TLS
cert bundle (cert + private key) is included in the nightly backup payload
alongside the Postgres dump. A restored host can serve HTTPS immediately
with the backed-up cert and let the proxy handle its own renewal on the
normal schedule.

## Configuration as Code

### Source of Truth

Keycloak's declarative surface is reconciled from a git repository. The
**repository is the source of truth**; the running Keycloak's admin surface
is a read-through cache of what's in git.

In scope for git reconciliation:

- realms
- clients
- client scopes
- roles and role mappings
- identity-provider configuration
- authentication flows and required actions
- realm-level settings

Out of scope for git (intentionally runtime state):

- user credentials (password hashes, WebAuthn registrations, TOTP secrets)
- sessions and refresh tokens
- audit and event logs
- ephemeral tokens and one-time codes

### Reconciliation Tool

Reconciliation uses **`keycloak-config-cli`** (adorsys). The tool is mature,
works against the admin API, handles partial updates, and does not require
Kubernetes CRDs or a separate operator. It is the best available GitOps
option as of this writing given the upstream Keycloak Operator still does
not provide first-class CRDs for clients, users, roles, or identity
providers.

### Reconciliation Location

`keycloak-config-cli` runs **on the Keycloak host itself** as a scheduled
job. Pull cadence is a small number of minutes; the exact cadence is an
implementation detail. The job authenticates to Keycloak using a
reconciliation service account stored in SSM Parameter Store.

This location is intentionally simple for now. Pushing reconciliation into
GitHub Actions (so every git push triggers a reconcile) is named as future
work — it would enforce "git push is the only way config changes" more
strictly — but it requires a reachable admin endpoint and an appropriate
trust path, which are better designed once the realm repo has concrete
shape.

### Schema Versioning

Keycloak migrations run forward only. The realm repository pins the
Keycloak version it expects. Upgrades are driven by bumping the pin in
`infra/` and allowing the next reconcile cycle to re-apply cleanly against
the upgraded Keycloak.

## Backups

### What is backed up

- Postgres dump (the full database, including all runtime state).
- Keycloak configuration files that live outside the database: `keycloak.conf`,
  environment overrides, any custom themes or providers.
- The current TLS cert bundle (cert + private key).

Configuration in git is **not** part of the backup — git is already the
durable store for it.

### Where they go

- **Primary destination:** an S3 bucket in the `lab` account. The bucket
  uses server-side encryption with a KMS key; object-lock/versioning is on
  so corruptions cannot silently overwrite known-good backups.
- **Secondary destination:** the lab's Synology NAS, which pulls from the S3
  bucket on its own schedule.

The host writes backups to S3 using the instance's IAM role (no long-lived
credentials). The Synology pulls from S3 using a scoped, read-only access
mechanism chosen when Synology-side automation is implemented — out of scope
for this document.

### Retention

Retention is a rolling window, implemented via S3 lifecycle policies. The
contract:

- **daily** backups retained for **30 days**
- **weekly** backups retained for **12 weeks**
- **monthly** backups retained for **12 months**

"Last backup only" is explicitly rejected. A corruption-style incident
(realm data mangled by a bad change, not a hardware failure) requires
point-in-time restore from days ago.

### Encryption

Backups contain password hashes, signing keys, TOTP secrets, and session
state. They are encrypted twice:

- client-side: the Postgres dump and cert bundle are encrypted before upload
  using a recipient key managed alongside the rest of the lab's bootstrap
  secrets
- server-side: the S3 bucket uses SSE-KMS with a customer-managed key

This ensures that neither a leaked S3 object ACL nor a Synology
compromise yields usable plaintext.

## Disaster Recovery

The lab uses a **rebuild-first** recovery model. Restore exists as a
fallback, not as the primary path.

### Rebuild path (primary)

When Keycloak is unrecoverable or is being moved:

1. Provision a fresh EC2 instance from the `infra/` OpenTofu modules.
2. Run `docker compose up` to start Keycloak and a **fresh** Postgres.
3. Run `keycloak-config-cli` against the new Keycloak, pointing at the realm
   repository. All realms, clients, roles, and GitHub federation come back.
4. Sign in via GitHub. A new user entry is created on first login per the
   identity-provider mapper configuration.
5. Re-enroll WebAuthn / TOTP (30 seconds).
6. Keycloak is operational.

**Target RTO: 15 minutes.** This path requires only the git repository and
AWS access; it does not require any backup store.

### Restore path (fallback)

When a rebuild is unacceptable — for example, if you need to preserve the
exact user-state including federated-identity linkages and audit history —
the restore path is:

1. Provision a fresh EC2 instance.
2. Pull the most recent, or a chosen point-in-time, backup from S3 or
   Synology.
3. Restore the Postgres dump into a fresh Postgres.
4. Place the TLS cert bundle and any config files.
5. Run `docker compose up`. Keycloak boots against the restored database.
6. `keycloak-config-cli` runs its normal reconcile cycle; any configuration
   drift between the backup time and `git HEAD` is corrected forward.

For the single-user lab, the restore path is rarely worth it. For the
day-job mirror architecture, it is the default.

### Hostname preservation

Both paths require that the restored instance serve at **`id.glab.lol`**.
The `issuer` claim on every JWT Keycloak has ever signed is tied to that
URL. Changing the hostname invalidates every existing token and every
client's cached OIDC discovery.

This is normally handled by DNS: the new instance comes up in the same VPC
with the same Route 53 A record pointing to it. During a full internet-loss
DR scenario where Route 53 is unreachable, a local override path exists:
the lab's CoreDNS zonefile can be manually edited to point `id.glab.lol` at
the restored instance's Tailscale address.

### What rebuild does not recover

- Stored user credentials (password hashes, WebAuthn, TOTP). These must be
  re-enrolled by the user on first login. For a single-user lab, 30 seconds.
- Active sessions. All users are forced to re-authenticate.
- Federated-identity linkages established at previous logins. GitHub OIDC
  users are re-linked on their next successful sign-in.
- Audit / event history.

For the lab, none of these matter. For a production deployment of this
design, they would matter, and the restore path becomes primary.

## Break-Glass Matrix

When Keycloak is down, the lab continues to operate via
per-service break-glass paths. These are not emergency workarounds; they are
durable, documented alternate authentication paths that are kept active for
exactly this reason.

| Service            | Break-glass path                                     | Notes |
|--------------------|------------------------------------------------------|-------|
| Talos API          | mTLS via `talosconfig` and machine secrets          | Talos's PKI is independent of Keycloak; `talosconfig` is the ultimate root anchor for the lab. |
| `kubectl` to clusters | Talos-generated admin kubeconfig via `talosctl kubeconfig` | Produced on demand against the cluster's own signing CA; not federated. |
| Argo CD            | Built-in `admin` account and initial admin secret in the cluster | Retained and rotated but never disabled. |
| Vault (per cluster)| Unseal keys and root/recovery keys                  | Kept outside the lab per the Vault design (separate doc). |
| AWS                | IAM Identity Center local user + WebAuthn hardware key | Does not federate to Keycloak by design — see the AWS doc. |
| Grafana            | Local admin account                                  | Kept active alongside OIDC client configuration. |
| GitHub (upstream IdP) | Personal GitHub account, hardware-key MFA        | Keycloak is downstream of GitHub; if GitHub is down, federated login fails, and the above service-local paths are how the lab keeps moving. |

All of these anchors' credentials live outside any Keycloak-dependent store.
`talosconfig`, Argo CD admin secrets, Vault recovery keys, and AWS root
credentials are kept in offline storage (password manager + hardware backup)
whose access does not depend on Keycloak, AWS, or internet connectivity.

## Cost

Monthly, all-upfront amortized over the 3-year EC2 Instance Savings Plan
commitment established in the AWS design:

| Item                           | Monthly |
|--------------------------------|---------|
| t4g.small compute (3-yr SP)    | ~$3.89  |
| S3 backup storage + lifecycle  | ~$0.20  |
| Route 53 record queries        | negligible |
| Data transfer                  | negligible |
| **Approximate total**          | **~$4.10** |

This is additive on top of the baseline AWS footprint laid out in the
AWS Lab Account design.

## Future Work

- **CI-driven reconciliation.** Move `keycloak-config-cli` from an
  on-host cron into a GitHub Actions workflow so `git push` is the only
  trigger that mutates Keycloak's declarative state.
- **Keycloak → IAM Identity Center SAML federation** as a secondary,
  convenience path for AWS console access. The local IAM Identity Center
  user remains primary.
- **Promotion to HA** if / when this design is reused at day-job scale. Two
  Keycloak replicas behind a load balancer with clustered cache and a shared
  database is the standard upgrade path; none of the decisions in this doc
  block it.
- **Automated DR drill.** A periodic exercise — quarterly is appropriate — in
  which the rebuild path is executed against a scratch instance to prove
  the RTO target and keep the muscle alive.
- **Richer per-service break-glass.** Codify the break-glass secrets
  themselves (their storage location, rotation cadence, recovery order) in
  a separate operational runbook.

## References

- [AWS Lab Account](./aws-lab-account.md)
- [Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md)
- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md)
- [keycloak-config-cli (adorsys)](https://github.com/adorsys/keycloak-config-cli)
- [Keycloak Operator: status of first-class CRDs](https://www.keycloak.org/2022/09/operator-crs)
