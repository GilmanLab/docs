---
title: AWS Lab Account
description: Proposed design for a dedicated AWS Organization and member account that anchors lab DNS, bootstrap identity, and offsite compute for identity systems.
---

# AWS Lab Account

## Status

Proposed.

This document defines how the lab uses a dedicated AWS Organization and member
account as its durable, out-of-lab trust anchor. It covers account and identity
structure, the VPC and its Tailscale site-to-site link to the lab, the Route 53
private zone and the in-lab mirror that consumes it, and the secrets bootstrap
path that lets AWS be the single identity required to retrieve and decrypt all
other bootstrap material.

Detailed Keycloak design is out of scope for this document and is covered
separately.

## Purpose

The primary purpose of this design is to keep lab identity, lab DNS, and
lab secrets from having a circular dependency on the lab itself.

The intended split is:

- AWS owns the durable trust anchor — the account, the identity system, the
  KMS key, the private zone of record
- the lab consumes what AWS provides without requiring continuous connectivity
  to operate steady-state
- a small number of AWS-resident components (a Tailscale subnet router, a
  Keycloak host) provide the minimum bridge between the two sides

This keeps the lab's hardest-to-bootstrap layers (identity, DNS, and secret
decryption keys) outside the lab, while keeping day-to-day serving and latency
local.

## Goals

- Provide a durable, off-lab trust anchor that survives total lab failure.
- Break the chicken-and-egg between lab DNS, lab identity, and lab secrets.
- Keep AWS itself identity-independent from anything the lab hosts, so AWS
  remains reachable even when Keycloak, the platform cluster, or the lab
  network is unavailable.
- Eliminate long-lived static credentials on AWS-hosted bootstrap nodes.
- Allow the lab to continue serving DNS and operating existing workloads
  during a total internet outage.
- Mirror the durable-trust-anchor shape of a future day-job architecture so the
  lab exercises the same patterns at small scale.

## Non-Goals

- This document does not define the Keycloak deployment, its database, or its
  disaster-recovery plan. Those belong to a separate Keycloak design doc.
- This document does not define monitoring, logging, or alerting for the AWS
  account.
- This document does not define exact IAM policy JSON, SCPs, Tailscale ACL
  rules, CoreDNS configuration, or OpenTofu module layout. The doc names
  contracts, not implementations.
- This document does not define OIDC trust between cluster workloads and AWS.
  That is a separate future design connected to ExternalDNS and similar cluster
  components.
- This document does not federate IAM Identity Center to Keycloak. Doing so
  would reintroduce the circular dependency this design is built to avoid.

## Design Summary

The lab uses a dedicated AWS Organization with two accounts:

- a **management account** that holds the Organization, billing, and IAM
  Identity Center, and runs no workloads
- a **member account** that holds all lab-owned AWS resources: the VPC, the
  Tailscale subnet router, the Keycloak host, the Route 53 private zone, the
  KMS key used for SOPS, and the SSM parameters used for bootstrap

Identity into both accounts comes from **IAM Identity Center with its built-in
identity store**. There is no external identity provider. A single human user
signs in with a hardware security key and assumes time-limited permission sets
into the member account. Root credentials on both accounts are break-glass only
and stored offline.

The member account peers with the lab via a pair of **Tailscale subnet
routers**, one in AWS and one on VyOS. Devices on either side can reach devices
on the other side by their real IPs without being Tailscale nodes themselves.
The AWS-side subnet router authenticates to the tailnet via **Tailscale
workload identity federation**, using its attached IAM role — no pre-shared
auth key.

The lab's authoritative DNS lives in a **Route 53 private zone** for `glab.lol`
bound to the lab's VPC. A sync job on the subnet router renders that zone to a
local zonefile, serves it over the tailnet, and an in-lab fetcher pulls the
file to disk. CoreDNS in the lab serves from the on-disk zonefile. The read
path never reaches AWS at query time, so DNS serving survives internet outages
and cold starts.

Bootstrap secrets are gated end-to-end by the AWS-resident IAM role:

- encrypted secrets live in the existing `secrets/` repo on GitHub, encrypted
  with a **KMS customer-managed key** used as a SOPS recipient
- that repo is cloned via a **GitHub App** whose private key is stored in SSM
  Parameter Store (SecureString)
- both KMS decrypt and SSM read are granted to the bootstrap instance via its
  IAM role

The result is that an AWS-resident bootstrap node holds zero persistent
secrets on disk: every identity it uses — Tailscale, GitHub, the SOPS
decryption key, AWS itself — traces back to its IAM role.

## Account Structure

The lab uses a two-account AWS Organization:

| Account     | Purpose                                                                     |
|-------------|-----------------------------------------------------------------------------|
| `lab-mgmt`  | Organization management account. Holds billing, IAM Identity Center, org-level config. No workloads. |
| `lab`       | Lab workload member account. Holds VPC, EC2, Route 53, KMS, SSM, and all other lab resources. |

The split follows AWS's own recommendation that the management account should
not run workloads. It also leaves room to add additional member accounts later
(for example, a prod-mirror account that stages day-job patterns) without
restructuring.

Region: **`us-west-2`**.

All resources in this design live in `us-west-2` in the `lab` account unless
explicitly stated otherwise.

## Identity

### Primary path

IAM Identity Center is enabled in the `lab-mgmt` account and uses its
**built-in identity store**. There is no external IdP wired in. The identity
store holds one human user with **WebAuthn MFA enforced** via a hardware
security key.

Access to the `lab` account is granted via permission sets assigned from
Identity Center. Daily operator access — console and CLI — is short-lived:

- console access through the Identity Center access portal
- CLI access via `aws sso login`, which produces short-lived role credentials

No long-lived IAM user access keys exist in either account for human use.

### Break-glass

Root user credentials exist on both accounts and are used for emergency
recovery only (loss of Identity Center access, billing-only actions not
permitted to Identity Center). Both root accounts:

- use strong unique passwords
- have hardware-key MFA enabled
- are stored offline (outside any system whose recovery depends on AWS or
  Keycloak being reachable)

### Why the identity store is local

Federating IAM Identity Center to Keycloak would make AWS access depend on
Keycloak. Keycloak depends on AWS for its compute, its DNS, and its secrets
bootstrap. Coupling the two defeats the entire reason for placing identity on
a durable off-lab trust anchor.

A future addition of Keycloak SAML federation as a **secondary, convenience**
path for Identity Center is possible and explicitly deferred. The local
identity store always remains the primary admin path.

## Network

### VPC

- **CIDR:** `172.16.0.0/16`
- **Subnets:** one public subnet, single AZ
- **Internet gateway:** attached; the subnet router carries outbound traffic
  via an Elastic IP attached to its ENI
- **NAT gateway:** none. With a single public-subnet instance there is no
  workload needing egress through a private subnet; skipping NAT removes the
  largest ongoing fixed cost that would otherwise apply (~$32/mo)

`172.16.0.0/16` is deliberately far from both the lab's `10.10.0.0/16` and
Tailscale's `100.64.0.0/10` CGNAT range, so no address-space collisions can
occur when routes are advertised across the tailnet.

### Site-to-site with the lab

The lab and the VPC connect via **Tailscale subnet routers on both sides**:

- **AWS side:** the subnet router EC2 instance advertises `172.16.0.0/16` and
  accepts `10.10.0.0/16`.
- **Lab side:** VyOS runs Tailscale and advertises `10.10.0.0/16` while
  accepting `172.16.0.0/16`.

Both sides run with `--snat-subnet-routes=false` so traffic preserves real
source IPs. The VPC route table directs `10.10.0.0/16` to the subnet router's
ENI, and the ENI has source/destination check disabled so it can forward.
Security groups allow `10.10.0.0/16` as a source on the ENI.

From either side, a host can address the other side by its real IP without
being a Tailscale node itself. Lab DNS clients reach `172.16.0.0/16`
transparently; VPC workloads (Keycloak) can reach lab workloads when needed.

MSS clamping is configured on VyOS to avoid black-holed large packets through
the WireGuard-based tunnel's smaller MTU. Tailscale ACLs permit traffic
between the two advertised CIDRs.

### Tailscale node identity

The AWS-side subnet router authenticates to the tailnet via **Tailscale
workload identity federation**, using its attached IAM role. No pre-shared
auth key is stored on the instance. Tailscale ACL tags are derived from IAM
claims (role ARN, account ID), so policy can be written against the
IAM identity rather than per-device labels.

The VyOS node uses a traditional Tailscale auth key, because workload identity
federation only supports cloud-hosted clients. That key is managed out of band
and lives on a single on-prem device; it is not checked into any repo.

## DNS

### Authoritative zone

The canonical lab domain is **`glab.lol`**. A Route 53 **private hosted zone**
for `glab.lol` lives in the `lab` account, bound to the VPC. All lab DNS
records are managed there.

Private was chosen over public intentionally: the day-job architecture this
lab mirrors requires record names themselves to be non-public. A public zone
would be operationally simpler but would not exercise the same pattern.

### Lab read path

CoreDNS in the lab serves `glab.lol` from a **local zonefile on disk**. The
file is kept up to date by a sync pipeline that runs entirely outside the lab:

1. A job on the AWS-side subnet router reads the Route 53 zone using its IAM
   role and renders it to a standard zonefile. Refresh cadence is ≤1 minute.
2. The subnet router serves the rendered file over the tailnet.
3. An in-lab fetcher periodically pulls the file and writes it to the
   filesystem CoreDNS reads from.

CoreDNS never queries Route 53 at request time. The fetch path is
asynchronous and decoupled from serving.

### Failure characteristics

- **Steady-state AWS or internet outage:** fetches fail; CoreDNS continues to
  serve from the last-fetched zonefile. The zone data becomes progressively
  stale in proportion to the outage length, but queries continue to resolve.
- **Cold start during an outage:** CoreDNS loads the last zonefile from local
  disk and resumes serving. The sync job is not on the critical path.
- **Full lab internet loss:** the tailnet path to the subnet router is itself
  unreachable, which stops syncs but not serving. The zonefile on disk is the
  resilience layer.
- **Stale-vs-unavailable tradeoff:** this design accepts staleness as the
  price of availability during outages. Zone changes during an outage simply
  do not propagate until connectivity returns.

### Why the mirror exists

Steady-state DNS resilience can be provided by CoreDNS itself — the `route53`
plugin reads zones into memory, and the `cache` plugin with `serve_stale`
enabled keeps answering through upstream outages. The mirror layer's specific
job is **cold-start and bootstrap resilience**: if CoreDNS restarts (node
reboot, container replaced) while Route 53 is unreachable, it has no in-memory
zone to fall back on. A zonefile on disk removes that failure mode.

## Secrets Bootstrap

### Contract

An AWS-resident bootstrap instance must be able to, starting from only its
IAM role:

1. Reach the tailnet.
2. Clone the private `secrets/` repo from GitHub.
3. Decrypt SOPS-encrypted files in that repo.

At no point may the instance hold a durable, plaintext credential for any of
the three systems (Tailscale, GitHub, SOPS). All identity traces back to the
instance's attached IAM role.

### KMS as a SOPS recipient

The existing `secrets/` repo continues to hold SOPS-encrypted files. A single
**customer-managed KMS key** in the `lab` account is added as an additional
SOPS recipient. Any principal granted `kms:Decrypt` on that key — human
(via Identity Center permission set) or machine (via instance profile) — can
decrypt.

This is non-breaking for existing workflows: SOPS supports multiple
recipients, so the KMS key can be added alongside the existing age key.
Retiring the age key is possible later but not required.

The details of the SOPS-over-KMS workflow, key rotation, and human vs.
automation paths live in a separate secrets design doc. This document only
establishes that the KMS key lives in the `lab` account and is the anchor
for machine decryption.

### GitHub App for repo access

Cloning private repos from an AWS-resident bootstrap instance uses a
**GitHub App** owned by the `GilmanLab` organization and installed on the
`secrets` repo (plus any other private repos bootstrap needs to reach).

- The App's **private signing key** is stored in an SSM Parameter Store
  `SecureString` in the `lab` account.
- The instance's IAM role grants `ssm:GetParameter` + `kms:Decrypt` on that
  specific parameter path only.
- On bootstrap, the instance fetches the key, generates a JWT, exchanges it
  for a short-lived installation token (1-hour TTL), and clones.

The only durable non-AWS secret anywhere in the chain is the App's private
signing key itself, and that key is at rest in AWS, gated by IAM. Installation
tokens are never stored on disk.

### The single-anchor property

Taken together, the chain on a single EC2 bootstrap instance is:

| Step | Identity used                                                  |
|------|----------------------------------------------------------------|
| Join tailnet | IAM role (via workload identity federation) |
| Read GitHub App key | IAM role (via instance profile → SSM + KMS) |
| Mint installation token | App private key (short-lived, in memory) |
| Clone `secrets/` repo | Installation token (short-lived, in memory) |
| Decrypt SOPS files | IAM role (via instance profile → KMS) |

Every persistent identity is the IAM role. Lose AWS, lose bootstrap. Gain AWS,
everything else unlocks in order. This is the design outcome the account
structure is in service of.

## Compute and Cost Model

### Instances

| Name           | Type       | Purpose                                                     |
|----------------|------------|-------------------------------------------------------------|
| subnet router  | `t4g.nano` | Tailscale site-to-site, Route 53 zonefile rendering         |
| Keycloak host  | `t4g.small`| Keycloak + colocated Postgres. Detailed design in Keycloak doc. |

Both run Amazon Linux 2023 on ARM (`t4g` / Graviton). Tailscale and Keycloak
both ship native ARM builds.

The two are kept **as separate instances** rather than colocated. A colocated
box would save ~$1.75/mo but would collapse the subnet router and identity
failure domains into one. The premium for separation is cheap insurance, and
separation is also more faithful to the day-job architecture this design
mirrors.

EC2 instances do **not** run EBS snapshot or AMI backup jobs. The lab's
philosophy is rebuild-over-restore: every instance's durable state is either
in an external store (Route 53, KMS, SSM, S3, GitHub) or is designed to be
reconstructed from those sources.

### Savings Plan commitment

Both instances are long-lived infrastructure and not expected to change
instance family over their lifetime. The commitment shape is:

- **3-year EC2 Instance Savings Plans, all-upfront**, covering the `t4g`
  family in `us-west-2`
- expected effective discount ~72% vs. on-demand
- one purchase sized to cover both instances; additional commitment can be
  layered later

### Cost envelope

Approximate, all-upfront amortized:

| Item                          | Monthly | 3-year |
|-------------------------------|---------|--------|
| Subnet router (t4g.nano)      | ~$1.75  | ~$63   |
| Keycloak host (t4g.small)     | ~$3.89  | ~$140  |
| KMS customer-managed key      | ~$1.00  | ~$36   |
| SSM Parameter Store (standard)| ~$0     | ~$0    |
| Route 53 private zone + queries | ~$0.50  | ~$18   |
| Data transfer                 | negligible | negligible |
| **Total (approximate)**       | **~$7.15** | **~$260** |

EBS, Elastic IPs attached to running instances, and Route 53 API calls for the
1-minute zonefile sync all fall into noise-level cost at lab scale.

## Infrastructure as Code

- All AWS resources are managed with **OpenTofu** from the `infra/` repo
  under `infra/aws/`.
- OpenTofu state is stored in an **S3 bucket in the `lab` account**, using
  S3's native locking.
- The OpenTofu entrypoint assumes a permission set role via Identity Center
  for human-operator runs. Future CI-triggered runs will use a separate
  identity (out of scope for this document).

### Manual bootstrap surface

A small amount of setup exists outside of OpenTofu, because it must exist
before OpenTofu can run:

1. Creation of the AWS Organization and the two accounts.
2. Enablement of IAM Identity Center and the single operator user.
3. Creation of the S3 state bucket and the minimum IAM role OpenTofu will
   assume.

Everything downstream of that — the VPC, the subnet router, the private zone,
the KMS key, the SSM parameters, the instance profiles — is declared in
OpenTofu.

## Failure Domains

What fails together and what does not:

| Failure                        | Lab DNS | Lab serving | AWS console access | Bootstrap of new lab instances |
|--------------------------------|---------|-------------|--------------------|------------------------------|
| Lab internet outage            | ✓ (cached zonefile) | ✓ | ✗ (can't reach AWS) | ✗ |
| Subnet router EC2 down         | ✓ (cached zonefile) | ✓ | ✓ | ✗ (tailnet → AWS bridge down) |
| `lab` account compromise       | ✓ (cached zonefile, short-term) | ✓ | partial | ✗ |
| `lab-mgmt` account lost        | ✓ (cached zonefile, short-term) | ✓ | ✗ | ✗ |
| Keycloak host down             | ✓ | ✓ (except OIDC-gated services) | ✓ | ✓ |
| AWS region outage              | ✓ (cached zonefile) | ✓ | ✗ | ✗ |
| Full lab power/hardware loss   | ✗ | ✗ | ✓ | depends on external rebuild |

The dominant pattern: **lab-side serving is robust to any offsite failure**
thanks to the zonefile-on-disk DNS path and locally-resident workloads.
Offsite failure primarily costs the ability to make changes, not the ability
to keep running.

## Future Work

The following are known next steps that are intentionally out of scope here:

- **Keycloak design doc.** Deployment shape, Postgres colocation, backup to
  object storage with Synology sync, rebuild-over-restore DR procedure, and
  GitOps via `keycloak-config-cli`.
- **Cluster workload OIDC to AWS.** ExternalDNS-style workloads on the Talos
  cluster will need AWS credentials; the cluster's own OIDC issuer (IRSA-style
  federation) is the expected mechanism, not Tailscale-based federation.
- **GitHub Actions OIDC.** Trusting GitHub Actions as an OIDC identity
  provider in the `lab` account so CI can apply OpenTofu without long-lived
  keys.
- **Keycloak SAML federation to IAM Identity Center** as a secondary,
  convenience access path alongside the local identity store.
- **Additional member accounts** under the same Organization as the lab's
  day-job mirroring grows (prod-mirror, dev, etc.).
- **Secrets design doc** covering the SOPS-over-KMS workflow, rotation,
  Vault relationship, and promotion model across bootstrap vs. per-cluster
  secrets.

## References

- [Keycloak](./keycloak.md)
- [Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md)
- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md)
- [Service Exposure and Control Plane Endpoints](./service-exposure-and-control-plane-endpoints.md)
- [Tailscale Workload Identity Federation](https://tailscale.com/kb/1581/workload-identity-federation)
- [AWS IAM Identity Center external IdP options](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
