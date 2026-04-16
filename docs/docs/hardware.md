---
title: Hardware Reference
description: Physical inventory and identifiers carried forward from the old lab.
---

# Hardware Reference

This document is a rough inventory of physical lab equipment referenced in the
old lab repository.

It is intentionally descriptive rather than prescriptive:

- It captures what hardware appears to exist in the old repo.
- It records identifiers, model references, concrete specs, and network details where they were explicitly documented.
- It does not imply current or future architecture.
- The only operational detail retained here is that the `VP6630` is the lab router running VyOS.

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
    - `10.10.10.1` (`LAB_MGMT`)
    - `10.10.20.1` (`LAB_PROV`)
    - `10.10.30.1` (`LAB_PLATFORM`)
    - `10.10.40.1` (`LAB_CLUSTER`)
    - `10.10.50.1` (`LAB_SERVICE`)
    - `10.10.60.1` (`LAB_STORAGE`)
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
  - Node address: `10.10.30.10/24`
  - API endpoint references:
    - `https://10.10.30.10:6443`
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

### Synology DiskStation DS923+

- Quantity: `1`
- Repo identifiers: `Synology NAS`, `NAS`, `nas.lab.local`
- Hardware details referenced in repo:
  - `DiskStation DS923+`
  - One ADR references `32GB RAM` shared with DSM
  - User-confirmed `10GbE` PCIe add-in NIC installed
- Network details found in repo:
  - No fixed IP found
  - Example hostname reference: `nas.lab.local`
  - NFS paths referenced in repo:
    - `/volume1/images`
    - `/volume1/backups`
    - `/volume1/media`
    - `/volume1/iso`

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
- The `DS923+` uses an installed `10GbE` PCIe NIC rather than native `10GbE`.

## Primary Sources In The Old Repo

- `docs/architecture/07_deployment_view.md`
- `docs/architecture/02_constraints.md`
- `docs/architecture/12_glossary.md`
- `docs/architecture/appendices/B_bootstrap_procedure.md`
- `infra/compute/talos/talconfig.yaml`
- `infra/network/vyos/configs/gateway.conf`
