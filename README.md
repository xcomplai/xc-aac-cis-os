> # ⚠️ RETIRED / DEPRECATED — 2026-05-29
>
> **`xcomplai.cis_os` is retired.** Its framework-agnostic Linux fact-gathering
> was generalized into **[`xcomplai.aac_common`](https://github.com/xcomplai/xc-aac-common)**
> under the AAC "decoupled-gather" end state: ONE Linux gatherer, with frameworks
> (CIS / STIG / NIST / …) living purely as OPA rego. All new work happens there;
> the core EE (`xc-aac-ee` v2.3.0+) ships `aac_common`, not `cis_os`.
>
> This repo is archived (read-only). The **`v1.2.0`** release is kept for history.
> See **[xcomplai/xc-aac](https://github.com/xcomplai/xc-aac)** for the current architecture.

---

# xcomplai.cis_os

Ansible Collection: CIS Operating Systems benchmark fact gathering + OPA assessment wrapper.

First reference implementation of the **fact_contract/v1** framework-collection pattern documented at [xcomplai/xc-aac → docs/architecture/FACT_CONTRACT.md](https://github.com/xcomplai/xc-aac/blob/main/docs/architecture/FACT_CONTRACT.md).

## What this collection provides

| Component | Purpose |
|---|---|
| `roles/gather` | Project CIS RHEL 9 host facts into `framework_facts` matching the contract. Read-only; does not POST or store anything. |
| `playbooks/assess.yml` | Thin wrapper: gather → POST OPA → store PG. Inlines the OPA + PG steps for v1.0.0; will move to `xcomplai.aac_common.evaluate` once that collection lands. |

## Install

```bash
# From Galaxy (when published):
ansible-galaxy collection install xcomplai.cis_os

# From this repo (until first release):
ansible-galaxy collection install git+https://github.com/xcomplai/xc-aac-cis-os.git
```

## Use

### As an AAP job template (production path)

The orchestration layer in [xc-aac](https://github.com/xcomplai/xc-aac) will eventually dispatch into this collection via the generic **AAC – Run Framework Assessment** template (Tier 3.I work). Until that lands, point a template directly at `assess.yml`:

| Setting | Value |
|---|---|
| Project | (one pointing at this repo, or any project where the collection is installed) |
| Playbook | `playbooks/assess.yml` |
| Inventory | hosts you want to assess |
| Survey vars | `opa_server_url`, `compliance_db_password` |
| EE | One that has `community.postgresql` |

### Direct invocation (laptop / standalone)

```bash
ansible-playbook -i hosts \
  xcomplai.cis_os.assess \
  -e opa_server_url=http://aac-host:8181 \
  -e compliance_db_password=<vault-secret>
```

The collection's `meta/runtime.yml` requires `ansible-core >= 2.16`.

## Status — what's covered today

This is **v1.0.0 — proof of the pattern**. The gather role projects:

- `filesystem` (auto_updates_enabled, yum_repos, gpg_keys_present)
- `ssh` (sshd_config_raw + parsed permit_root_login / permit_empty_passwords / protocol)
- `selinux` (status, mode, type, policy_version)
- `services` (service_mgr, running list, enabled list)
- `filesystem_permissions` (passwd/shadow/group/gshadow mode + owner)

Plus the full `ansible_facts` projection per fact_contract/v1.

Subsequent minor versions extend coverage to the remaining CIS sections:

| Status | Section |
|---|---|
| ✅ v1.0.0 | filesystem, ssh, selinux, services, filesystem_permissions |
| 🚧 v1.1+ | auth (passwd/shadow parsing, pam_lines) |
| 🚧 v1.1+ | audit (auditd rules, audit.log fields) |
| 🚧 v1.2+ | pam (PAM policy stack) |
| 🚧 v1.2+ | sudo (sudoers + .d entries) |
| 🚧 v1.3+ | network (firewalld, iptables, listening ports) |
| 🚧 v1.3+ | cron (cron jobs + perms) |
| 🚧 v1.4+ | boot_security (grub.cfg, password, single-user) |
| 🚧 v1.4+ | initial_setup (cramfs / squashfs / udf disabled, fs.suid_dumpable, etc.) |
| 🚧 v1.5+ | logging (rsyslog, journald) |
| 🚧 v1.5+ | user_group (account expiry, login.defs, INACTIVE) |

The rego library at [ynotbhatc/rego_policy_libraries](https://github.com/ynotbhatc/rego_policy_libraries) validates against whatever `framework_facts` keys are present — missing nouns return `undefined` and rego rules degrade gracefully. Adding a section here doesn't require rego changes.

## How this maps to the fact contract

```jsonc
{
  "schema": "fact_contract/v1",
  "input": {
    "ansible_facts": { /* setup-gathered, projected to the contract's standard subset */ },
    "framework_facts": {
      "filesystem":    { /* xcomplai.cis_os.gather sets these */ },
      "ssh":           { /* ... */ },
      "selinux":       { /* ... */ },
      "services":      { /* ... */ },
      "filesystem_permissions": { /* ... */ }
    },
    "metadata": {
      "inventory_hostname": "...",
      "framework": "cis_rhel9",
      "framework_version": "2.0.0",
      "collection": "xcomplai.cis_os",
      "collection_version": "1.0.0",
      "gathered_at": "2026-05-29T..."
    }
  }
}
```

Posted to OPA at the path declared in the rego library's `data.cis_rhel9.metadata.opa_endpoint_path` (today: `/v1/data/cis_rhel9/main/compliance_report`). See the contract doc for the full shape.

## Repo

Source: https://github.com/xcomplai/xc-aac-cis-os — issues, PRs, releases.

Pairs with:
- [xcomplai/xc-aac](https://github.com/xcomplai/xc-aac) — orchestration / chart / install.
- [xcomplai/xc-aac-policies](https://github.com/xcomplai/xc-aac-policies) — OPA bundle releases (rego CIS RHEL 9 lives here).
- [ynotbhatc/rego_policy_libraries](https://github.com/ynotbhatc/rego_policy_libraries) — rego source upstream.

## License

MIT.
