---
status: accepted
---

# WINS runs on dc1 only, with no replication

Only dc1 (`samba_primary_host`) runs `wins support = yes`; dc2 and all other WINS
clients point at it via `wins server = <dc1 IP>`. There is no replication and no
automated failover — if dc1 is down, legacy NetBIOS name resolution is down too,
while LDAP/Kerberos/DNS keep working via dc2.

This was chosen over running WINS on both DCs because classic Samba WINS (`nmbd`)
has zero database replication support, confirmed against samba.org's own
Samba3-HOWTO (see `docs/research/wins-implementations.md`) — two independently
"active" WINS servers with no replication is worse than one, since clients get
inconsistent answers depending which server they ask (split-brain), not
redundancy. Automated failover was also considered and rejected: without real
quorum/fencing, a network partition (not a genuine dc1 outage) could promote dc2
into dual-active alongside a still-healthy dc1, and even a clean failover loses
whatever dc2 registered while active once dc1 recovers, since WINS has no merge
capability — automated failover would add real operational complexity without
actually removing the underlying inconsistency risk it's meant to solve.

**Open question, deliberately not resolved here:** the Samba AD DC's own `wrepl`
service (a different code path from classic `nmbd`) ships a full implementation of
the MS-WINS replication protocol and may genuinely replicate `wins.tdb` between
dc1 and dc2 — see `docs/research/wins-implementations.md`, Question 2b. This is
unverified for this Samba build and was not tested. If real WINS HA is ever
needed, prototyping `wrepl` (not building a custom sync mechanism) is the next
thing to try, before reconsidering this decision.
