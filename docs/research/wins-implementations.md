# WINS server implementations — research notes

> Note on location: no existing convention for research notes exists in this repo or
> in the sibling `ansible-role-technitium-dns` repo. `docs/research/` is a new
> directory created for this task as a reasonable place for such notes going forward.

## Question 1 — Is there a WINS implementation other than Microsoft's and Samba's?

**Short answer: not a viable, actively-maintained one.** Every current, real-world
NBT/WINS server in production is either Microsoft's Windows Server WINS role or
Samba (`nmbd` classic file-server WINS, or the AD DC's built-in NBT/WINS server).

Findings:

- **samba4WINS** — a spin-off from early Samba4 development, described by its own
  packaging as "a full featured replicating WINS server for UNIX."
  Source: FreeBSD ports description, https://www.freshports.org/net/samba4wins/
  Status: **abandoned**. The FreeBSD port had no active maintainer, was deprecated
  for not being adapted to FreeBSD's staging requirements, and was deleted from the
  ports tree on 2014-08-31 (last port update 2014-09-01). No current release,
  no upstream site found still shipping it. Not viable for a 2026 Debian trixie
  deployment.
  Source: https://www.freshports.org/net/samba4wins/ (commit/expiry history)

- No other independent open-source NBNS/WINS server implementation, embedded/router
  firmware WINS daemon, or commercial WINS appliance was found via samba.org,
  wiki.samba.org, or general web search. WINS/NBT is a legacy, Microsoft-defined
  protocol (RFC 1001/1002 define NBNS/NBT name service generically, but WINS itself —
  the centralized name-registration server behavior — is a Microsoft-specific service
  built on top of it); the only parties who ever had a commercial/practical reason to
  implement a WINS *server* (as opposed to just an NBT client) were Microsoft and the
  Samba team, since WINS only matters for legacy NetBIOS-over-TCP/IP name resolution
  inside Windows networks.
  Background: RFC 1001 (NetBIOS Service Protocols — Concepts and Methods),
  RFC 1002 (NetBIOS Service Protocols — Detailed Specifications),
  https://www.rfc-editor.org/rfc/rfc1001, https://www.rfc-editor.org/rfc/rfc1002
  (these define NBNS as a protocol/concept; they do not mandate or describe "WINS"
  by name — WINS is Microsoft's specific centralized NBNS server product built on
  this RFC foundation).

**Conclusion for decision:** There is nothing else worth adopting. The realistic
choice set for a Debian trixie Samba AD DC role is Microsoft WINS (Windows Server
role — out of scope, this is a Linux/Samba stack) or Samba's own WINS support.
Samba is the only viable choice.

## Question 2 — Does Samba have a native mechanism to replicate wins.tdb / its WINS database between two Samba WINS servers?

**This needs to be split into two distinct Samba code paths, because the answer
differs between them and generic secondary sources conflate them.**

### 2a. Classic Samba (`nmbd`, `wins support = yes`, file-server / domain member role)

**Definitively NOT supported.** Primary source — Samba's own Samba3-HOWTO,
Chapter 10 "Network Browsing," section on WINS replication:

> "Samba-3 does not support native WINS replication... There was an approach to
> implement it, called `wrepld`, but it was never ready for action and the
> development is now discontinued... Right now Samba WINS does not support MS-WINS
> replication. This means that when setting up Samba as a WINS server, there must
> only be one `nmbd` configured as a WINS server on the network."

Source: https://www.samba.org/samba/docs/old/Samba3-HOWTO/NetworkBrowsing.html
(official samba.org documentation)

This is also consistent with the `smb.conf` man page's `wins hook` parameter, which
is explicitly a **script/program hook** invoked on database changes — i.e. a
do-it-yourself extension point, not a built-in replication protocol:

> "wins hook" accepts the name of a script or program that passes information on
> modifications to a Samba NBNS server's configuration to other programs — it "can
> be used to replicate NBNS configuration files across servers, to dynamically add
> NetBIOS names to a DNS server, and so on."

Source: `smb.conf(5)` man page, https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html
(parameter `wins hook`)

`smb.conf(5)` also confirms `wins support` and `wins server` are mutually exclusive
(you can be a WINS server, or point at one, never both), reinforcing that Samba's
classic WINS model is single-master with no peer replication:
Source: `smb.conf(5)`, parameter descriptions for `wins support` / `wins server`,
https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html

### 2b. Samba AD DC (the `samba` binary / `source4`, as used for dc1/dc2)

This is the important nuance for this project: **the AD DC code path is not the
same NBT/WINS implementation as classic `nmbd`, and it does ship a WINS replication
service.**

- The `smb.conf(5)` man page's `server services` parameter — which controls which
  internal services the `samba` (AD DC) daemon starts — has a documented default
  that includes a service literally named `wrepl`:

  > default: `server services = s3fs rpc nbt wrepl ldap cldap kdc drepl winbind ntp_signd kcc dnsupdate dns`

  Source: `smb.conf(5)` man page, `server services` parameter,
  https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html

- In the Samba source tree, `source4/wrepl_server/` is a full implementation of the
  MS-WINS **replication** protocol (the same wire protocol Windows Server WINS uses
  for push/pull partner replication), with inbound and outbound handling:
  `wrepl_server.c`, `wrepl_in_connection.c`, `wrepl_in_call.c`,
  `wrepl_apply_records.c`, `wrepl_out_pull.c`, `wrepl_out_push.c`,
  `wrepl_periodic.c`, `wrepl_scavenging.c`.
  Source: https://github.com/samba-team/samba/tree/master/source4/wrepl_server
  (samba-team/samba, the official upstream mirror of the Samba source)

  This means the **AD DC's built-in NBT/WINS server (`nbt` + `wrepl` services) is
  architecturally capable of WINS replication** using the standard MS-WINS
  replication protocol — the same protocol/mechanism a Windows Server WINS role
  uses for push/pull partners. This is a materially different situation from
  classic `nmbd`, which flatly has no replication code at all.

  **Caveat — verify before relying on this in production:** the presence of
  `wrepl_server` and a default `wrepl` service entry confirms the *code path
  exists and runs by default* on an AD DC, but this research did not find current,
  maintained end-user documentation (wiki.samba.org HOWTO, release notes) describing
  how to configure push/pull replication partners for two Samba AD DC WINS servers,
  nor confirmation of its current maturity/support status in Debian trixie's Samba
  4.x build. Historically (mid-2000s Samba4 alpha era), this code was described as
  early/experimental. Before depending on it for dc1/dc2, this should be validated
  hands-on: enable `wins support = yes` on both DCs, confirm `wrepl` is listed in
  `server services` (or add it), and test whether `wins.tdb` entries registered on
  dc1 actually propagate to dc2 without extra configuration. If no supported way to
  configure replication partners is found in practice, treat it as unsupported.

## Decision guidance

1. **Don't look for a third-party WINS server.** samba4WINS is dead (abandoned 2014);
   nothing else exists. Samba is the only real option alongside Microsoft's own role.
2. **Classic `nmbd`-style WINS (`wins support = yes` on a file server / domain
   member) has zero replication support, confirmed by samba.org's own docs.** If you
   run WINS this way on both dc1 and dc2, they are two independent, non-syncing WINS
   databases — clients pointed at both will get inconsistent answers depending which
   one they ask. This is the setup to avoid for a real HA WINS deployment.
3. **The Samba AD DC's built-in `nbt`/`wrepl` services are the one place Samba
   ships real WINS-replication protocol code.** This is the path worth prototyping:
   enabling `wins support = yes` on the `samba` (AD DC) process on dc1 and dc2 and
   testing whether the `wrepl` service actually keeps their WINS databases in sync
   out of the box. This needs hands-on verification against the actual Debian trixie
   Samba package before committing to it in the role.
4. **Fallback if AD DC WINS replication doesn't pan out in testing:** run WINS as
   single-master (one authoritative WINS server, e.g. dc1) with dc2 (and all other
   clients) configured via `wins server = <dc1 IP>`, accepting a single point of
   failure for legacy NetBIOS name resolution — which is likely acceptable given WINS
   is only needed for legacy client compatibility, not for AD DC operation itself
   (AD DC relies on DNS, not WINS/NetBIOS, for its own functioning).

**Decision actually taken for this role (see `docs/adr/0002-wins-single-master.md`):
option 4, single-master WINS on dc1 only.** Option 3 (`wrepl` replication) was
deliberately not built or tested this round — it remains a documented, unverified
future enhancement, not a rejected one.

## Sources

- Samba3-HOWTO, Chapter 10, Network Browsing (WINS replication statement, `wrepld`
  history): https://www.samba.org/samba/docs/old/Samba3-HOWTO/NetworkBrowsing.html
- `smb.conf(5)` man page (current), parameters `wins support`, `wins server`,
  `wins hook`, `server services`:
  https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html
- Samba3-Developers-Guide, Chapter 12, Samba WINS Internals (WINS server groups /
  failover, not replication): https://www.samba.org/samba/docs/old/Samba3-Developers-Guide/wins.html
- Samba source code, `source4/wrepl_server/` (WINS replication protocol
  implementation, AD DC code path):
  https://github.com/samba-team/samba/tree/master/source4/wrepl_server
- samba4WINS abandonment (FreeBSD ports history): https://www.freshports.org/net/samba4wins/
- RFC 1001 — NetBIOS Service Protocols, Concepts and Methods: https://www.rfc-editor.org/rfc/rfc1001
- RFC 1002 — NetBIOS Service Protocols, Detailed Specifications: https://www.rfc-editor.org/rfc/rfc1002
