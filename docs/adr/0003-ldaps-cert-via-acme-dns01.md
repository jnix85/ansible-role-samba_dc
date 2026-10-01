---
status: accepted
---

# Issue the LDAPS certificate via ACME DNS-01, not self-signed or hand-carried

`samba-tool domain provision` generates a self-signed cert/key/CA under
`/var/lib/samba/private/tls/` on first run, and Samba's `tls enabled`/`tls
keyfile`/`tls certfile`/`tls cafile` smb.conf settings are what LDAPS
(`ldaps://`, and STARTTLS on `ldap://`) actually serves. Left alone, every DC
answers LDAPS with a certificate no client trusts. `roles/samba_dc/tasks/
certificate.yml` replaces it: each DC requests its own certificate from an
ACME server (Let's Encrypt by default) using `community.crypto.acme_certificate`
with a DNS-01 challenge, publishing the `_acme-challenge` TXT record through
`ansible-role-technitium-dns`'s `technitium_dns_record` module (via its new
`acme_dns01_record.yml`, called with `include_role`) — the same
declarative-record-push architecture already used for this role's A/SRV
records (see `docs/adr/0001-external-dns-via-technitium.md`).

This was chosen over HTTP-01 because these DCs have no reason to run a public
web server and HTTP-01 needs port 80 reachable from the internet. It was
chosen over hand-carrying a certificate onto each host (issued elsewhere,
copied in by some other process) because that reintroduces a manual,
easy-to-forget step exactly where this role already avoids one for DNS.
DNS-01 fits this role's existing shape: Technitium already holds an API token
this role uses, and the fleet already has no port-80 exposure story for these
hosts.

Each DC issues its own certificate for its own FQDN — not one shared SAN
certificate for both — so a compromised or expiring DC's cert doesn't
implicate the other, and so nothing ever copies a private key between hosts.

**Consequence, and how it's handled:** DNS-01 needs the challenge TXT record
to be visible to the ACME server's validators, which for Let's Encrypt means
publicly resolvable. `samba_dns_zone` (`directory.clawduino.com`) is
deliberately *not* delegated under the public `clawduino.com` apex (see
`inventory/group_vars/samba_dc_servers/main.yml`) — it's an internal AD zone,
and writing the TXT record there directly would be invisible to Let's
Encrypt. Rather than delegate the whole AD zone publicly just for this, only
the challenge name is delegated: a static `_acme-challenge.<dc fqdn>` CNAME
per DC (in `samba_dns_records`, `roles/samba_dc/defaults/main.yml`) points at
an alias under `dns.clawduino.com` — a zone this same Technitium cluster
hosts and the fleet's real public DNS already delegates to it
(`dns_cluster_zone`) — and `tasks/certificate.yml` writes the TXT value
there instead of under `samba_dns_zone`. `samba_cert_acme_dns_zone` controls
the target zone (`roles/samba_dc/defaults/main.yml`); clearing it falls back
to writing the TXT directly under `samba_dns_zone`, only useful if that zone
becomes publicly delegated itself some day. An internal ACME CA (step-ca,
etc.) that doesn't need public visibility at all is the other option, via
`samba_cert_acme_directory_url`.
