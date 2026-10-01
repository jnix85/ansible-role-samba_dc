---
status: accepted
---

# Use Technitium as the AD zone's authoritative DNS, not Samba's internal DNS

dc1/dc2 provision with `samba-tool domain provision --dns-backend=NONE` instead of
`SAMBA_INTERNAL`. Samba never runs its own DNS server or talks to DNS at all;
`ansible-role-samba_dc` computes the standard AD DNS record set (A records per DC,
SRV records for LDAP/Kerberos/kpasswd/GC and their `_msdcs` equivalents — see
`samba_dns_records` in `roles/samba_dc/defaults/main.yml`) and pushes it
declaratively into the fleet's existing Technitium DNS cluster through
`ansible-role-technitium-dns`'s `technitium_dns_record` module, authenticated with
a normal API token.

This was chosen over Samba's default AD-integrated DNS (the obvious, zero-config
path every Samba AD DC guide assumes) because the fleet already has one
authoritative, HA DNS system (Technitium, behind `vips.dns`) and running a second
one for just the AD zone would mean two DNS sources of truth. It was chosen over
Samba talking to Technitium via TSIG-authenticated dynamic DNS (RFC 2136,
`samba_dnsupdate`/`nsupdate`) — the standard way to point an AD DC at *any*
external DNS server — because that mechanism's exact behavior against a non-BIND
server like Technitium was unverified during this work, and because pushing the
(small, well-known, mostly-static) record set from Ansible avoids depending on a
live update protocol at all: idempotent, inspectable, and consistent with how
every other zone in this fleet's Technitium cluster is already managed.

**Consequence and Resolution:** NTDS-GUID-keyed `_msdcs` CNAME records (used for AD replication
topology) are dynamically queried from `sam.ldb` after provisioning and joining, and pushed
declaratively into Technitium DNS as `<guid>._msdcs.<samba_dns_zone>` CNAME records pointing to each DC's
FQDN. This ensures DRS multi-DC replication succeeds seamlessly even with external DNS.
