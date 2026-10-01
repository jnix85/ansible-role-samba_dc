# samba_dc

Provisions and manages a Samba Active Directory domain: dc1 (writable primary DC, all
FSMO roles) and dc2 (secondary DC: RODC or writable replica DC), with WINS for legacy
clients and NTP signing for Kerberos.

## Language

**AD zone**:
The DNS zone for the Active Directory realm (`samba_dns_zone`, e.g.
`directory.clawduino.com`). Hosted as a Primary zone in the fleet's Technitium
DNS cluster, not by Samba itself — its records are pushed there declaratively by
this role (`samba_dns_records`), not maintained via Samba's own dynamic DNS. See
`docs/adr/0001-external-dns-via-technitium.md`.
_Avoid_: "the domain's DNS" (ambiguous with the AD domain itself), "internal DNS"
(there isn't one — that's the point of the ADR).

**WINS server**:
The DC that answers legacy NetBIOS name-resolution queries (`wins support = yes`
in smb.conf). Only dc1 runs as a WINS server; dc2 (and everything else) is a WINS
*client* pointed at it. There is exactly one — Samba's classic WINS has no
replication, so "WINS server" here is always singular, never "servers." See
`docs/adr/0002-wins-single-master.md`.
_Avoid_: "WINS servers" (plural) when describing this deployment — there's only
ever one active at a time by design.

**LDAPS certificate**:
Each DC's own TLS certificate for LDAPS, requested per-DC from an ACME server via
DNS-01 (`roles/samba_dc/tasks/certificate.yml`) — never a self-signed cert (Samba's
provisioning default) and never one certificate shared between dc1/dc2. The DNS-01
challenge is published through Technitium, the same way AD zone records are — see
`docs/adr/0003-ldaps-cert-via-acme-dns01.md`.
_Avoid_: "the domain certificate" (singular/shared) — there are always two,
independently issued and renewed.

**Challenge alias**:
The name (`samba_cert_acme_challenge_alias`, under `dns.clawduino.com`) that each
DC's `_acme-challenge.<dc fqdn>` record CNAMEs to. Exists because `samba_dns_zone`
isn't publicly delegated (see "AD zone" above), so the ACME DNS-01 challenge TXT
value is written at the alias instead of directly under `samba_dns_zone` — see
`docs/adr/0003-ldaps-cert-via-acme-dns01.md`. The CNAME itself is a one-time,
static entry in `samba_dns_records`; only the TXT value at the alias changes on
each issue/renew.
_Avoid_: "the challenge record" without specifying which of the two records
(the static CNAME vs. the TXT that comes and goes) — they live in different
zones and have different lifecycles.

**Secondary DC / RODC**:
The secondary domain controller (dc2). Joins after dc1 is provisioned
(`tasks/join_secondary.yml`, with backwards-compatible `join_rodc.yml` wrapper).
When configured as an RODC (`samba_secondary_role: rodc`), which users/computers it may
cache credentials for ("Allowed RODC Password Replication Group") is left as a manual,
out-of-band decision — see README.md's "Manual steps." When configured as a writable DC
(`samba_secondary_role: dc`), it functions as a full multi-master replica DC.
