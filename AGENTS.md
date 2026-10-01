# Project Context: ansible-role-samba_dc

**Type:** Ansible Role / Infrastructure Automation  
**Target:** Debian 12 (Bookworm), Debian 13 (Trixie), Ubuntu  
**Service:** Samba Active Directory Domain Controller (dc1 primary writable + dc2 secondary RODC or writable DC)

---

## Overview

Provisions a Samba Active Directory domain with a writable DC (all FSMO roles) and a secondary DC (configured as an RODC by default, or writable DC via `samba_secondary_role: dc`).
Configures WINS and signed NTP for domain-joined Windows and Linux clients.
AD DNS zone and replication records are hosted by the Technitium cluster.

---

## Repository Layout

```
.
├── AGENTS.md                         # AI source of truth
├── CLAUDE.md -> AGENTS.md            # Symlink for Claude Code
├── README.md                         # Human documentation
├── ansible.cfg                       # Inventory and SSH settings (wires shared-inventory)
├── requirements.yml                  # Collections
├── inventory/
│   ├── hosts.yml                     # samba_dc_servers group (dc1, dc2)
│   └── group_vars/                   # all/ and samba_dc_servers/
├── playbooks/
│   └── site.yml                      # Main playbook
└── roles/
    └── samba_dc/                     # Role implementation
```

---

## Common Commands

```bash
# Syntax check
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check

# Dry-run
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff
```
