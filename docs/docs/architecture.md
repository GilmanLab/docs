---
title: Architecture Overview
description: Start here for the current GilmanLab architecture baseline.
---

# Architecture Overview

This is the current architecture baseline for the GilmanLab homelab.

The architecture centers on IncusOS hosts, an Incus cluster, Talos Linux
virtual machines, and a GitOps/CAPI management model. It is intentionally
practical rather than exhaustive: it records the direction and the important
boundaries, while leaving detailed manifests, exact addresses, and prototype
findings to implementation work.

For the physical inventory, see [Hardware Reference](./hardware.md).

## Reading Order

Read these documents as one architecture set:

1. [Hosts and Substrate](./architecture/hosts-and-substrate.md)
2. [Bootstrap and Cluster Lifecycle](./architecture/bootstrap-and-lifecycle.md)
3. [Networking and Endpoints](./architecture/networking-and-endpoints.md)
4. [GitOps and Platform APIs](./architecture/gitops-and-platform-apis.md)
5. [Secrets, Identity, DNS, and PKI](./architecture/secrets-identity-pki.md)
6. [Keycloak Runtime](./architecture/keycloak-runtime.md)
7. [State and Recovery](./architecture/state-and-recovery.md)

Runbook-style pages remain separate:

- [Network Device Backups](./network-device-backups.md)
- [RouterOS ACME Certificates](./routeros-acme.md)

## Architecture Baseline

The lab shape is:

- The `UM760` and all three `MS-02 Ultra` systems run IncusOS directly on bare
  metal.
- Those four hosts form one Incus cluster.
- Talos Linux runs as Incus VMs and provides the Kubernetes nodes.
- The first platform Talos VM starts on the `UM760`; the final platform cluster
  expands to three dual-role Talos nodes once the `MS-02` hosts join.
- Cluster API Provider Incus, the Talos CAPI providers, and GitOps own normal
  Kubernetes cluster lifecycle.
- VyOS remains the lab network boundary and provides DHCP, DNS, PXE support,
  the real platform Kubernetes API TCP frontend, and BGP peering for Kubernetes
  service VIPs.
- Cilium provides the Kubernetes datapath, LoadBalancer IP allocation, and BGP
  advertisements for service VIPs.
- AWS anchors bootstrap identity, SOPS/KMS access, selected DNS material, and
  GitHub token broker access to the private `secrets` repo.
- Recovery is rebuild-first. Talos VMs and Incus hosts should be recreated from
  declarative inputs when possible; backups protect the state that cannot be
  rebuilt from Git, CAPI, and GitOps.

## Current Status

This is the desired architecture, not a claim that every piece is already live.

The following items are deliberately still prototype-validation work:

- IncusOS `Operation` image generation and seeding for a final disk image.
- Writing the chosen IncusOS image through Tinkerbell `image2disk` or
  `oci2disk`.
- Running the released `bootstrap-k0s` image on VyOS with the required
  privileges, mounts, and host-network behavior.
- CAPN plus the Talos providers creating the desired Talos VM shape.
- Exact VLANs, static addresses, ASNs, DNS records, and service VIP pools.

Those details should be proven in small prototypes before they become runbooks
or exact implementation references.

## Explicit Non-Goals

The v1 architecture does not include:

- Proxmox on the `MS-02` hosts.
- A Proxmox cluster or Proxmox CAPI provider.
- Proxmox Backup Server as the default VM backup system.
- Shared Incus VM storage, Ceph, LINSTOR, or Incus OVN in v1.
- A manual USB path as the preferred host bootstrap workflow.
