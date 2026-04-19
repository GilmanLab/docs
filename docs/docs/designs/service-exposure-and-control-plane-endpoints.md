---
title: Service Exposure and Control Plane Endpoints
description: Proposed design for service VIP exposure and API endpoint HA on future multi-node Talos clusters in the lab.
---

# Service Exposure and Control Plane Endpoints

## Status

Proposed.

This document defines how future multi-node clusters in the lab should expose
application traffic and how their external API endpoints should be formed. It
separates service and ingress VIPs from the Kubernetes API endpoint and from
the Talos API endpoint so those concerns do not blur together in later design
or implementation work.

## Purpose

The primary purpose of this design is to make the cluster entrypoint model
explicit and consistent across the lab.

The intended split is:

- Cilium provides service and ingress VIPs
- the `VP6630` acts as the upstream BGP peer for those VIPs
- Talos provides the external Kubernetes API endpoint via a shared VIP for
  multi-node clusters on shared Layer 2
- Talos API access continues to use direct control-plane endpoints by default

This keeps service exposure, Kubernetes API HA, and Talos API access on
separate mechanisms that match what each layer is actually good at.

## Goals

- Standardize service exposure for future multi-node clusters.
- Standardize the external Kubernetes API endpoint shape for future multi-node
  Talos clusters.
- Keep KubePrism enabled as the internal HA endpoint for host-network
  components.
- Avoid mixing application VIPs with control-plane API endpoint design.

## Non-Goals

- This document does not define concrete Helm values for Cilium.
- This document does not define exact VyOS CLI or HAProxy configuration.
- This document does not define concrete Talos or CAPI manifests.
- This document does not change the current single-node platform cluster into
  an HA cluster today.

## Problem Split

There are three separate networking problems here:

1. **Service and ingress exposure**
   - how traffic from outside a cluster reaches workloads inside it
2. **Kubernetes API endpoint**
   - the canonical `https://...:6443` endpoint for a cluster
3. **Talos API endpoint**
   - how operators reach the Talos API on port `50000`

The lab should not try to solve all three with one mechanism.

KubePrism is related, but it is a fourth, internal concern:

- **Internal API consumers**
  - host-network components such as Cilium or control-plane processes needing a
    resilient in-cluster API path

## Chosen Model

### Service and Ingress VIPs

Future multi-node clusters use:

- Cilium LB IPAM for allocating service VIPs
- Cilium BGP Control Plane for advertising those VIPs
- the `VP6630` as the upstream BGP peer

This applies to stable external entrypoints such as:

- `LoadBalancer` Services
- ingress controller VIPs
- Gateway API data-plane entrypoints

These VIPs are advertised as **service routes**, not PodCIDRs. With the
current Cilium `ipam.mode=kubernetes` assumption, PodCIDR advertisement is not
part of the design.

### Internal API Consumers

Talos clusters keep KubePrism enabled.

KubePrism is the internal HA endpoint for host-network consumers of the
Kubernetes API, including Cilium. It is not the external cluster endpoint used
by operators or external clients.

### Kubernetes API Endpoint

Future multi-node Talos clusters use:

- Talos VIP for the canonical Kubernetes API endpoint

This means each multi-node cluster gets one canonical external endpoint of the
form:

- `https://<cluster-endpoint>:6443`

The endpoint is backed by a Talos virtual IP shared by the control-plane nodes.
This is the default cluster API HA model as long as the control-plane nodes
share a Layer 2 domain.

### Talos API Endpoint

The default Talos API model remains:

- direct control-plane node endpoints

An optional future enhancement is:

- VyOS TCP load balancing for the Talos API

That is intentionally not part of the baseline design for now.

## Supporting Assumptions

The chosen model assumes:

- multi-node Talos control planes share a Layer 2 domain when Talos VIP is
  used
- Cilium peers with the `VP6630` over BGP
- service and ingress VIPs are external traffic entrypoints, not the mechanism
  for Kubernetes API HA
- Talos VIP is never used as the Talos API endpoint
- KubePrism stays enabled and Cilium is configured to use it for internal API
  access

If a future multi-node cluster does not have shared Layer 2 for its control
planes, the Kubernetes API endpoint strategy must be revisited explicitly. That
case is outside this baseline decision.

## Cluster Scope

This decision applies to:

- future multi-node downstream clusters such as `nonprod` and `prod`
- any future multi-node platform cluster, if the platform cluster ever stops
  being single-node

This decision does **not** describe the current live platform cluster, which
remains single-node on the `UM760` and therefore does not yet exercise the HA
endpoint pattern.

## Relationship to Other Designs

This design builds on:

- [Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md) for day-0
  and day-1 cluster bring-up
- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md) for cluster roles and
  GitOps scope

This document defines the intended endpoint model once a cluster is beyond
bootstrap and has become a real multi-node control plane.
