---
title: Networking and Endpoints
description: Lab network ownership, Kubernetes API endpoints, Cilium service VIPs, DNS, and VLAN boundaries.
---

# Networking and Endpoints

The network design keeps the underlay visible to VyOS and avoids adding an
Incus SDN layer in v1.

VyOS remains the routing and naming boundary. Kubernetes service exposure is
handled by Cilium. The platform Kubernetes API endpoint is handled by VyOS
HAProxy, not by the same Cilium service VIP path that depends on the cluster
already being healthy.

## Ownership Split

| Layer | Owner | Purpose |
| --- | --- | --- |
| Physical network | Switches and VyOS | VLANs, trunks, routing, DHCP/PXE reachability |
| Host substrate | IncusOS and Incus | VM attachment to lab VLANs |
| Kubernetes nodes | Talos | Kubernetes control plane and worker runtime |
| Service VIPs | Cilium plus VyOS | LoadBalancer IP allocation and BGP advertisement |
| Platform API frontend | VyOS HAProxy | TCP passthrough to Talos control-plane VMs |

## Kubernetes API Endpoint

The real platform cluster uses a VyOS-owned TCP frontend for the Kubernetes API:

- configure CAPN with an external load balancer model
- create a stable DNS name such as `kube.platform.<domain>`
- point that name at a VyOS-owned listener
- run VyOS HAProxy in TCP mode on `:6443`
- configure backends as the reserved IPs of the platform Talos control-plane
  VMs on `:6443`
- do not terminate Kubernetes API TLS on VyOS
- start with basic TCP health checks

This makes the platform API reachable without depending on Cilium, Talos VIP
leadership, or a CAPN-managed development HAProxy container.

CAPN's built-in HAProxy load balancer remains acceptable for local prototypes.
Its own Talos template documentation describes that container as a development
or evaluation path and warns that the template is not currently tested in CI.

## Talos API Endpoint

`talosctl` should target individual Talos control-plane node IPs on the Talos
API port, `50000`.

Do not use the Kubernetes API DNS name, the VyOS HAProxy frontend, or a virtual
IP as the Talos API recovery endpoint. Talos API access must still work when
the Kubernetes API or etcd is unhealthy.

## KubePrism

Keep KubePrism enabled on Talos clusters.

KubePrism is the internal highly available Kubernetes API endpoint for
host-network consumers such as control-plane components and CNI components. It
does not replace the external platform API frontend.

## Talos VIP

Talos VIP is not the default platform API endpoint.

Talos VIP can simplify access for some clusters, but it depends on etcd
election state and comes alive only after bootstrap. That makes it a weak
recovery endpoint for the platform cluster. Future workload clusters may use it
only after the tradeoff is explicit for that cluster.

## Kubernetes Service Exposure

Use Cilium LB IPAM and Cilium BGP Control Plane for Kubernetes
`LoadBalancer` services.

Expected behavior:

- Cilium LB IPAM allocates service IPs from configured pools.
- Cilium BGP Control Plane advertises selected service VIPs to VyOS.
- Those advertisements are service routes, not PodCIDRs. With the current
  `ipam.mode=kubernetes` assumption, PodCIDR advertisement is not part of the
  design.
- VyOS learns exact routes to those VIPs and forwards traffic toward eligible
  cluster nodes.
- `externalTrafficPolicy`, node selectors, and advertisement policy can be
  refined after real workloads exist.

Cilium is not responsible for the same cluster's initial Kubernetes API
bootstrap endpoint because it depends on the Kubernetes API to reconcile.

## VLAN Boundary

The default VM attachment model is:

```text
IncusOS bridge/VLAN     = VM attachment and node underlay
VyOS DHCP/DNS/HAProxy   = addressing and platform API frontend
Talos/Kubernetes        = compute and control plane
Cilium+BGP to VyOS      = service LoadBalancer VIP advertisement
```

Physical switch ports can be trunks. Talos VMs should normally receive
untagged, access-style vNICs on their intended VLAN. Guest-side VLAN tagging is
reserved for cases where the VM genuinely needs multiple VLANs.

Avoid in v1:

- Incus NAT networks for real Talos nodes
- `macvlan` as the default VM attachment mode
- Incus OVN
- service VIP VLAN trunking into Talos guests

## DNS

The lab domain is `glab.lol`.

The authoritative private zone lives in Route 53 in the `lab` AWS account. A
sync path renders that zone to a local zonefile served inside the lab, so query
serving does not depend on reaching AWS at request time.

## AWS Link And DNS Mirror

The AWS `lab` VPC uses `172.16.0.0/16`, intentionally separate from the lab
`10.10.0.0/16` space and Tailscale's `100.64.0.0/10` range.

The AWS network shape is intentionally small:

- one public subnet in a single AZ
- an attached internet gateway
- no NAT gateway
- a `t4g.nano` Amazon Linux 2023 subnet-router instance
- an Elastic IP attached to the subnet router ENI for outbound traffic
- a separate Keycloak host, not colocated with the subnet router

The lab and AWS connect through Tailscale subnet routers on both sides:

- the AWS-side subnet router advertises `172.16.0.0/16` and accepts
  `10.10.0.0/16`
- VyOS advertises `10.10.0.0/16` and accepts `172.16.0.0/16`
- both sides preserve source IPs with `--snat-subnet-routes=false`
- the VPC route table sends `10.10.0.0/16` to the subnet router ENI
- source/destination check is disabled on that ENI
- security groups allow `10.10.0.0/16` as a source
- VyOS clamps MSS to avoid black-holed large packets through the tailnet path

The AWS-side subnet router authenticates to Tailscale through workload identity
federation using its IAM role. The VyOS node uses a traditional Tailscale auth
key because workload identity federation is cloud-client only.

Tailscale ACL tags for the AWS subnet router should derive from IAM claims such
as the role ARN and AWS account ID, so tailnet policy follows the AWS identity
rather than a hand-maintained device label.

DNS serving uses a local zonefile for cold-start resilience:

1. A job on the AWS-side subnet router reads the Route 53 private zone using
   its IAM role and renders a standard zonefile at least once per minute.
2. The subnet router serves that rendered file over the tailnet.
3. An in-lab fetcher periodically writes the file to the path CoreDNS serves.

CoreDNS does not query Route 53 at request time. The mirror exists because
CoreDNS cache and `serve_stale` behavior help during steady-state upstream
outages, but they do not solve a cold start where CoreDNS has no in-memory
zone. The zonefile on disk makes restart and bootstrap behavior deterministic.

The intended split is:

| Zone | Writers | Purpose |
| --- | --- | --- |
| `glab.lol` | Route 53 private zone automation | parent zone and static anchors |
| `mgmt.glab.lol` | manual or GitOps-managed | stable management and platform names |
| `dhcp.glab.lol` | DHCP/DNS automation | lease-driven hostnames |
| `<cluster>.k8s.glab.lol` | ExternalDNS | cluster-scoped workload names |
| `acme.glab.lol` | Route 53 DNS-01 automation | public certificate validation targets |

Exact records, forwarding rules, local zonefile mechanics, and address
assignments are implementation details and should be filled in after prototype
validation.

## References

- [VyOS HAProxy](https://docs.vyos.io/en/latest/configuration/loadbalancing/haproxy.html)
- [VyOS containers](https://docs.vyos.io/en/latest/configuration/container/index.html)
- [Cilium LB IPAM](https://docs.cilium.io/en/stable/network/lb-ipam/)
- [Cilium BGP Control Plane resources](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-configuration/)
- [Talos KubePrism](https://docs.siderolabs.com/kubernetes-guides/advanced-guides/kubeprism)
- [Talos virtual shared IP](https://docs.siderolabs.com/talos/v1.12/networking/advanced/vip)
