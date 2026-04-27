---
title: RouterOS ACME Certificates
description: How the MikroTik router and switch get WebFig HTTPS certificates from the lab CA.
---

# RouterOS ACME Certificates

:::note

This page documents RouterOS certificates that still use `step-ca`. The
architecture docs define the steady-state PKI path: public HTTP TLS uses Let's
Encrypt with Route 53 DNS-01, and internal runtime PKI uses Vault
intermediates. Keep this page as an operational reference until these RouterOS
consumers are migrated.

:::

The `CCR2004` home router and `CRS309` lab switch use the lab `step-ca`
intermediate for WebFig HTTPS certificates.

The live names are:

| Device | RouterOS identity | HTTPS name | Address |
| --- | --- | --- | --- |
| `CCR2004-16G-2S+` | `Core Router` | `ccr2004.mgmt.lab.gilman.io` | `192.168.1.1` |
| `CRS309-1G-8S+` | `lab-10g-switch` | `crs309.mgmt.lab.gilman.io` | `10.10.10.2` |

Both names are served from the `mgmt.lab.gilman.io` PowerDNS zone. The
`CRS309` address also has a PTR in `10.10.10.in-addr.arpa`.

## CA and DNS Path

`step-ca` runs on the `VP6630` as the online intermediate CA at:

```text
https://ca.mgmt.lab.gilman.io:9000/acme/acme/directory
```

The `stepca` container is configured with `name-server 10.10.10.1` so ACME
HTTP-01 validation resolves internal names through the VyOS recursor.

The MikroTik devices import and trust the lab root CA before using the private
ACME directory:

```routeros
/certificate/import file-name=glab-root-ca.crt name=glab-root-ca trusted=yes
```

The `CRS309` also needs the intermediate imported explicitly so WebFig serves a
complete chain:

```routeros
/certificate/import file-name=glab-intermediate-ca.crt name=glab-intermediate-ca trusted=yes
```

On the `CCR2004`, RouterOS imported the intermediate during ACME issuance, but
it still had to be marked trusted so WebFig would serve the full chain:

```routeros
/certificate/set [find name="glab Intermediate CA"] trusted=yes
```

## Issuance

RouterOS `7.16` and `7.18` use the older ACME command:

```routeros
/certificate/enable-ssl-certificate \
  directory-url=https://ca.mgmt.lab.gilman.io:9000/acme/acme/directory \
  dns-name=ccr2004.mgmt.lab.gilman.io \
  reset-private-key=yes
```

For the switch, replace the DNS name with `crs309.mgmt.lab.gilman.io`.

Current RouterOS documentation describes the newer `/certificate/add-acme`
command. Use that when these devices are upgraded to a release that exposes it.

## WebFig Services

The ACME command assigns the issued certificate to `www-ssl`, but on these
devices it did not enable the service. HTTPS is enabled explicitly:

```routeros
/ip/service/set [find name=www-ssl] \
  disabled=no \
  address=192.168.1.0/24,10.10.0.0/16 \
  certificate=ccr2004.mgmt.lab.gilman.io
```

Use `certificate=crs309.mgmt.lab.gilman.io` on the switch.

Plain HTTP remains enabled only for ACME HTTP-01 validation. It is restricted
to the source address that `step-ca` uses to reach each device:

| Device | `www` allowed source | Reason |
| --- | --- | --- |
| `CCR2004` | `10.0.0.2/32` | VyOS source address toward `192.168.1.1` |
| `CRS309` | `10.10.10.1/32` | VyOS source address toward `10.10.10.2` |

```routeros
/ip/service/set [find name=www] address=10.0.0.2/32
/ip/service/set [find name=www] address=10.10.10.1/32
```

Run the first command on the `CCR2004` and the second on the `CRS309`.

## Verification

From a client that trusts `glab Root CA` and uses the lab resolver or Tailscale
split DNS, these should return HTTP `200` with TLS verification result `0`:

```bash
curl --cacert infra/security/pki/root-ca/root_ca.crt \
  https://ccr2004.mgmt.lab.gilman.io/

curl --cacert infra/security/pki/root-ca/root_ca.crt \
  https://crs309.mgmt.lab.gilman.io/
```

For a client that does not resolve lab DNS locally, use `--resolve` while
testing. Treat this as a fallback; normal access should resolve through the
home router, VyOS, or Tailscale split DNS.

```bash
curl --cacert infra/security/pki/root-ca/root_ca.crt \
  --resolve ccr2004.mgmt.lab.gilman.io:443:192.168.1.1 \
  https://ccr2004.mgmt.lab.gilman.io/

curl --cacert infra/security/pki/root-ca/root_ca.crt \
  --resolve crs309.mgmt.lab.gilman.io:443:10.10.10.2 \
  https://crs309.mgmt.lab.gilman.io/
```
