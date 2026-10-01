# ansible-role-samba_dc

Installs and provisions a Samba Active Directory domain: a primary writable DC
(all FSMO roles) plus a secondary DC (RODC by default, or writable replica DC
via `samba_secondary_role: dc`). Also configures a WINS server/client pair for legacy
clients, and sets up NTP signing so domain-joined Windows clients can sync time
against the DCs (Kerberos requires clocks within a few minutes of each other).
DNS for the AD zone is hosted by the fleet's existing Technitium cluster, not by
Samba — see "DNS" below.

## DNS

dc1/dc2 provision with `samba-tool domain provision --dns-backend=NONE`
(and join with the matching `--dns-backend=NONE`): Samba never runs its
own DNS server or talks to DNS at all. Instead this role computes the
standard AD DNS record set (`samba_dns_records` in
`roles/samba_dc/defaults/main.yml` — A records for dc1/dc2, SRV records
for LDAP/Kerberos/kpasswd/GC and their `_msdcs` equivalents) and pushes
it declaratively into the fleet's Technitium DNS cluster via
`ansible-role-technitium-dns`'s `samba_ad_zone`/`samba_ad_records`
tasks (see `tasks/provision.yml`, `tasks/join_secondary.yml`). In addition,
the role extracts each DC's NTDS Settings `objectGUID` from `sam.ldb` and pushes
the `<guid>._msdcs.<realm>` CNAME replication record into Technitium so AD DRS
replication works without issue. See `docs/adr/0001-external-dns-via-technitium.md`
for why, and `inventory/group_vars/samba_dc_servers/secrets.yml` for the
`technitium_dns_api_token` this needs.

## WINS

Only the primary (`samba_primary_host`) runs as a WINS server
(`wins support = yes`); the RODC is a WINS client pointed at it
(`wins server = <primary IP>`) — Samba's classic WINS has no
replication between servers, so there is exactly one WINS server, ever.
Controlled by `samba_wins_support` / `samba_wins_server` in
`roles/samba_dc/defaults/main.yml`. See
`docs/adr/0002-wins-single-master.md` and
`docs/research/wins-implementations.md` for why, and the open question
around the AD DC's `wrepl` service.

## Certificates

Each DC requests its own LDAPS certificate from an ACME server (Let's
Encrypt by default) via DNS-01, and installs it at the paths
`samba_cert_key_file`/`samba_cert_fullchain_file`/`samba_cert_ca_file` point at
(`roles/samba_dc/defaults/main.yml`), wiring them into smb.conf's `tls
keyfile`/`tls certfile`/`tls cafile`. `tls certfile` is pointed directly at
`samba_cert_fullchain_file` so connecting LDAP clients receive the intermediate
certificates required to validate the trust chain. The DNS-01 challenge is published
through `ansible-role-technitium-dns`, not hand-carried or self-signed — see
`roles/samba_dc/tasks/certificate.yml` and
`docs/adr/0003-ldaps-cert-via-acme-dns01.md`.

DNS-01 needs the challenge TXT record to be publicly resolvable, and
`samba_dns_zone` (`directory.clawduino.com`) isn't publicly delegated in this
fleet (it's an internal AD zone) — so this is handled via CNAME delegation
into `dns.clawduino.com` (which is publicly delegated), automatically: a
static `_acme-challenge.<dc fqdn>` CNAME per DC is pushed alongside the rest
of `samba_dns_records`, and `tasks/certificate.yml` writes the TXT value at
the CNAME's target instead of under `samba_dns_zone` directly. See
`samba_cert_acme_dns_zone`/`samba_cert_acme_challenge_alias` in
`roles/samba_dc/defaults/main.yml`. Nothing manual to set up for this part.

Still required in `group_vars/samba_dc_servers/main.yml`:
`samba_cert_acme_account_email` (a real contact address — the placeholder
there won't work) and `samba_cert_acme_terms_agreed: true` (only after
actually reading Let's Encrypt's Subscriber Agreement). To test without hitting
production rate limits, set `samba_cert_acme_staging: true`. Re-running the play
is what renews the certificate — schedule it.

## Virtual Machine vs. LXC Container

**Samba Active Directory Domain Controllers should always be hosted in Virtual Machines (KVM/QEMU), not LXC containers.**

Key reasons:
- **`security.NTACL` (Extended Attributes):** Unprivileged LXC containers cannot write to the `security.*` xattr namespace due to Linux user namespaces. Domain provisioning, SYSVOL ACLs, and Group Policy updates fail with `Operation not permitted`.
- **Privileged Container Security Risks:** Bypassing xattr restrictions requires running a privileged container where container root is host root. AD Domain Controllers are Tier-0 assets; running one in a privileged container exposes the underlying hypervisor to complete compromise.
- **Clock Discipline & NTP Signing:** Kerberos requires strict clock synchronization (< 5 minute skew) and NTP signing via `ntp_signd`. LXC shares the host kernel clock and cannot adjust hardware clock slewing independently.
- **Kernel & AppArmor Boundaries:** Samba daemons require RPC endpoints, raw sockets, and POSIX ACLs that default LXC profiles often restrict.

## NTP signing

Every DC runs chrony, synced from `samba_ntp_pool`, with
`bindcmdaddress` wired to Samba's `ntp_signd` socket
(`samba_ntp_signd_dir`) so it can sign NTP replies for domain members —
see the Samba wiki's "Setting up NTP signing using Chrony" for the
underlying mechanism.

Extracted from the `proxmox-ansible-roles` monorepo (`roles/samba_dc`,
played by `playbooks/17-samba-dc.yml`) into a standalone role directory.
This is a copy, not a symlink — the original still lives in
`proxmox-ansible-roles` and is the version actually run against production.

## Layout

```
.
├── ansible.cfg
├── requirements.yml
├── playbooks/site.yml          # plays applying the role
├── inventory/
│   ├── hosts.yml                 # samba_dc_primary (dc1) / samba_dc_rodc (dc2)
│   ├── host_vars/{dc1,dc2}.yml
│   └── group_vars/
│       ├── all/                  # domain, vm_fqdn, secrets toggle
│       └── samba_dc_servers/     # role vars + secrets.yml + vault.yml
└── roles/
    └── samba_dc/                 # the role itself
```

## Deliberately no VIP

Unlike every other service in the source monorepo, this pair has no
keepalived VIP: AD clients locate DCs via DNS SRV records / the DC-locator
process and talk to each DC directly — a floating IP in front of
LDAP/Kerberos would break site-affinity and Kerberos SPNs.

## Shared inventory

`domain`, `vm_fqdn`, `secrets_backend`, and the Infisical connection vars
come from `../shared-inventory` (see `ansible.cfg`'s `inventory` list)
instead of being duplicated here — see `../shared-inventory/README.md` for
how that's wired and what does/doesn't belong there. This role does not use
`vips` for a VIP of its own (see "Deliberately no VIP" above), but does read
`vips.dns` — the Technitium cluster's floating IP — for both `resolv.conf`
and the DNS integration described above.

## Dependencies

`ansible-role-technitium-dns` is a real `requirements.yml` dependency
(`technitium_dns`, git-sourced) — see "DNS" above and "Certificates" above.
It's called explicitly via `include_role` from `tasks/provision.yml`/
`join_rodc.yml`/`tasks/certificate.yml`, not a Galaxy `meta/main.yml`
dependency (nothing else in this fleet uses that pattern).

`requirements.yml` also pulls in the `community.crypto` collection — used by
`tasks/certificate.yml` for the ACME certificate (see "Certificates" above).

`playbooks/site.yml` also runs `common` — a sibling role from the source
monorepo not copied here. Supply it yourself, or drop it and pre-bake the
baseline (hostname, `/etc/hosts`, packages, timezone).

dc1/dc2 themselves are expected to be provisioned out-of-band (e.g.
Terraform), not by this role.

## Secrets

`inventory/group_vars/samba_dc_servers/vault.yml` is ansible-vault
encrypted (copied as-is). `secrets_backend` in `group_vars/all/secrets.yml`
toggles between that vault and Infisical (`ansible-role-infisical`) lookups.
Holds `samba_admin_password` and `technitium_dns_api_token` (a pre-issued,
scoped Technitium API token — see "DNS" above).
