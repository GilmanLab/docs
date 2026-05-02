---
title: Hardware Reference
description: Physical inventory, identifiers, and current hardware role notes.
---

# Hardware Reference

This document is a rough inventory of physical lab equipment referenced by the
lab.

It is intentionally descriptive rather than prescriptive:

- It captures hardware carried forward from the old repo plus confirmed newer
  replacements.
- It records identifiers, model references, concrete specs, and network details
  where they were explicitly documented.
- Current architecture roles are recorded only where explicitly decided,
  currently the `VP6630` router role and the N5 Pro NAS genesis role.

## Inventory

### Protectli VP6630

- Quantity: `1`
- Repo identifiers: `VP6630`, `gateway`
- Operating system: `VyOS`
- Hardware details:
  - User-provided product page: `https://protectli.com/product/vp6630/`
  - `2x SFP+` interfaces referenced as `eth0` and `eth1`
  - `4x 2.5GbE` interfaces referenced as `eth2` through `eth5`
- Network details found in repo:
  - Transit address: `10.0.0.2/30`
  - Hostname: `gateway`
  - VLAN gateways:
    - `10.10.10.1` (`LAB_MGMT`, management + platform)
    - `10.10.20.1` (`LAB_PROV`)
    - `10.10.40.1` (`LAB_WORKLOAD`)
    - `10.10.70.1` (`LAB_OOB`)

### Minisforum UM760

- Quantity: `1`
- Repo identifiers: `UM760`, `cp-1`, `platform-cp-1`
- Hardware details referenced in repo:
  - `Ryzen 7`
  - `32GB RAM`
  - Single `2.5GbE RJ45` NIC
  - NVMe install target noted as `>= 256GB`
  - Observed NIC MAC in Talos config: `38:05:25:34:25:d0`
- Network details found in repo:
  - Node address: `10.10.10.10/24`
  - API endpoint references:
    - `https://10.10.10.10:6443`
    - `https://platform.lab.local:6443`

### Minisforum MS-02 Ultra

- Quantity: `3`
- Repo identifiers:
  - `MS-02`
  - `MS-02-Ultra`
  - `ms02-1`, `ms02-2`, `ms02-3`
  - `ms02-node1`, `ms02-node2`, `ms02-node3`
  - `LAB01`, `LAB02`, `LAB03`
- Hardware details:
  - Model confirmed from AMT: `MS-02-Ultra`
  - CPU confirmed from AMT sample: `Intel Core Ultra 9 285HX`
  - Memory confirmed from AMT sample: `64GB RAM` (`2x 32GB Micron SODIMM`)
  - Storage confirmed from AMT sample:
    - `Samsung SSD 990 EVO Plus 2TB`
    - `Patriot M.2 P300 128GB`
  - Repo references:
    - `2x 10/25GbE-capable SFP+` ports
    - `2x 2.5GbE RJ45` ports
    - Intel `vPro/AMT`
- Network details found in repo:
  - OOB static mappings:
    - `LAB01` -> `10.10.70.11` / `38:05:25:35:48:86`
    - `LAB02` -> `10.10.70.12` / `38:05:25:35:4b:04`
    - `LAB03` -> `10.10.70.13` / `38:05:25:35:43:f0`
  - Management addresses referenced in bootstrap docs:
    - `ms02-node1` / `ms02-1` -> `10.10.10.11`
    - `ms02-node2` / `ms02-2` -> `10.10.10.12`
    - `ms02-node3` / `ms02-3` -> `10.10.10.13`

### MINISFORUM N5 Pro NAS

- Quantity: `1`
- Repo identifiers: `NAS`, `N5 Pro`, `nas.lab.local`
- Current role:
  - Replaces the Synology NAS.
  - Runs IncusOS as the genesis Incus cluster node.
  - Hosts the disposable single-node Talos bootstrap cluster inside Incus.
- Hardware details:
  - MINISFORUM N5 Pro 5-bay desktop AI NAS
  - AMD Ryzen AI 9 HX PRO 370, `12c/24t`
  - Crucial `32GB` DDR5 SODIMM kit, `2x16GB`
  - `128GB` NVMe SSD reserved for the OS
  - `2x WD_Black SN7100 1TB` NVMe SSDs in the remaining NVMe slots
  - Planned mirrored ZFS data pool across the two `1TB` NVMe drives
  - `1x10GbE` and `1x5GbE` onboard networking
  - `2xUSB4`, HDMI, and OCuLink available but not currently architecture
    drivers
- Network details found in repo:
  - No fixed IP found
  - Example hostname reference: `nas.lab.local`
- Current network note:
  - Connected to the 10GbE switch on the last port over copper.

### MikroTik CCR2004

- Quantity: `1`
- Repo identifiers: `CCR2004`
- Hardware details referenced in repo:
  - Exact `CCR2004` variant is not specified
- Network details found in repo:
  - Home LAN address: `192.168.1.1`
  - Transit address: `10.0.0.1/30`

### MikroTik CRS309-1G-8S+IN

- Quantity: `1`
- Repo identifiers: `Mikrotik Switch`
- Hardware details:
  - User-confirmed model: `CRS309-1G-8S+IN`
  - Repo descriptions align with an `8x 10G SFP+` switch class
- Network details found in repo:
  - No fixed IP found

### TP-Link TL-SG105

- Quantity: `1`
- Repo identifiers: `TP-Link Dumb Switch`, `unmanaged switch`
- Hardware details:
  - User-provided product link: `https://www.amazon.com/dp/B00A128S24`
  - Model recorded from ASIN mapping: `TL-SG105`
  - `5x 1GbE RJ45` ports
  - Unmanaged desktop switch
- Network details found in repo:
  - No fixed IP found

### UPS

- Quantity: `1`
- Repo identifiers: `UPS`
- Hardware details:
  - User-provided product link: `https://www.amazon.com/dp/B002MZUNXU`
  - Model recorded from ASIN mapping: `APC Smart-UPS SMT1000`
  - Capacity: `1000VA / 700W`
  - Form factor: tower
- Network details found in repo:
  - No fixed IP found

## Verification Notes

The old repo contains a few places where terminology or specs drifted over time.

- `MS-02` is confirmed by AMT screenshots as `MS-02-Ultra`.
- The switch model is `CRS309-1G-8S+IN`; previous `CRS310-8G+2S+IN` references were incorrect.
- The N5 Pro replaces the Synology as the NAS and storage boundary.

## Primary Sources In The Old Repo

- `docs/architecture/07_deployment_view.md`
- `docs/architecture/02_constraints.md`
- `docs/architecture/12_glossary.md`
- `docs/architecture/appendices/B_bootstrap_procedure.md`
- `infra/compute/talos/talconfig.yaml`
- `infra/network/vyos/configs/gateway.conf`
