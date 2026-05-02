---
title: Keycloak Runtime
description: Keycloak deployment, first-boot configuration, recovery, and break-glass paths.
---

# Keycloak Runtime

Keycloak is the central human-facing identity system for lab services. It is
outside the lab's physical failure domain, but it is not a bootstrap dependency
for raw recovery.

## Deployment Shape

Keycloak runs on one dedicated EC2 instance in the `lab` account.

The runtime contract is:

- instance: `t4g.small`, Flatcar Container Linux on ARM, in the
  `172.16.0.0/16` VPC
- runtime: Ignition-managed systemd units that run Docker containers
- services: upstream Keycloak plus upstream Postgres, both pinned in `infra`
- database: colocated Postgres with data on a dedicated encrypted gp3 data
  volume mounted at `/var/lib/keycloak`
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
- permission to invoke the GitHub token broker Lambda for short-lived
  `GilmanLab/secrets` access
- KMS decrypt only for SOPS material with `Repo=GilmanLab/secrets` and
  `Scope=keycloak`
- SSM managed-instance access for operator inspection and repair

Cluster access, secret decryption, and tailnet access follow the AWS bootstrap
and secrets contracts. Keycloak does not receive broad AWS administrative
permissions.

## Realm And Local Admin

One realm named `lab` holds lab users and OIDC/SAML clients.

The first implemented human admin path is local Keycloak authentication, not a
GitHub identity provider. The `lab` realm has one local admin account,
`admin@glab.lol`, with a generated password and a required WebAuthn enrollment
for YubiKey-backed touch authentication. The temporary master bootstrap admin
exists only for initial realm creation and should be disabled after the browser
login path is proven.

The first service client is `incus`, a public OIDC client for Incus human
authentication. It enables OAuth 2.0 Device Authorization Grant and does not
define Keycloak roles, OpenFGA configuration, or a custom authorization model.
Incus OIDC users are full Incus administrators until OpenFGA becomes worth the
extra moving parts.

Future clients are expected to include:

- Kubernetes API OIDC for Talos clusters
- Argo CD web UI and CLI
- Grafana

The authoritative client list is intentionally deferred until each integration
is implemented.

## Configuration As Code

For the current slice, the Keycloak host imports the minimal `lab` realm from an
`infra/aws/keycloak` template during first boot. That declarative surface
includes:

- realms
- roles and role mappings
- authentication flows and required actions
- realm-level settings
- OIDC clients, starting with the public `incus` client
- the local admin user shell, with its password supplied from SOPS at import
  time

Runtime state is intentionally out of git:

- user credentials
- WebAuthn registrations
- TOTP secrets
- sessions and refresh tokens
- audit and event logs
- ephemeral tokens and one-time codes

`keycloak-config-cli` runs after Keycloak is healthy when the rendered realm
configuration hash differs from the stored
`/var/lib/keycloak/config/lab-realm.sha256` value. The config service fetches
the local admin dotenv payload from `GilmanLab/secrets` through the existing
broker-backed `labctl` path, imports the realm, and writes the new hash only
after a successful import. Scheduled reconciliation and GitHub OIDC are not
part of this slice.

The intended Incus-side OIDC values are:

- `oidc.issuer = https://id.glab.lol/realms/lab`
- `oidc.client.id = incus`
- `oidc.scopes = openid,email,profile`
- `oidc.claim = preferred_username` after token validation confirms the claim
  is present in Incus' access token

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
overwrite known-good backups. The N5 Pro NAS pulls a secondary copy on its own
schedule.

Retention contract:

- daily backups for 30 days
- weekly backups for 12 weeks
- monthly backups for 12 months

Backup payloads are encrypted before upload with a recipient key managed with
the lab's bootstrap secrets, and the S3 bucket also uses SSE-KMS.

## Recovery

The primary path is rebuild-first:

1. Provision a fresh EC2 instance from `infra`.
2. Let Flatcar mount the persistent Keycloak data volume and start the
   systemd-managed Postgres, Keycloak, and Traefik containers.
3. Let the first-boot `keycloak-config-cli` unit import the minimal `lab` realm
   if the import marker is absent.
4. Sign in as the local `lab` realm admin.
5. Enroll or re-enroll the YubiKey WebAuthn credential when prompted.

Target RTO for the single-user lab is 15 minutes. This path requires AWS access
and git; it does not require a backup store.

The restore fallback is:

1. Provision a fresh EC2 instance.
2. Pull a selected point-in-time backup from S3 or the NAS.
3. Restore the Postgres dump.
4. Place the TLS cert bundle and config files.
5. Start the Flatcar systemd units.
6. Let the first-boot import run only if the restored data does not already
   include the import marker and realm state.

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
| GitHub | Personal GitHub account with hardware-key MFA | GitHub is not a Keycloak upstream IdP. The token broker remains a machine bootstrap path for secrets access. |

These anchors live outside Keycloak-dependent storage.
