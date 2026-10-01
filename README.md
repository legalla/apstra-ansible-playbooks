# Juniper Apstra — Ansible Playbooks (public share)

This directory contains a set of Ansible playbooks that automate a **Juniper
Apstra** fabric through the `juniper.apstra` collection: from the design catalog
up to day-2 operations (services, DCI, firewalling, upgrades, ZTP…).

> These playbooks were **anonymised for public sharing**. Every IP address uses
> the documentation ranges reserved by **RFC 5737** (`192.0.2.0/24`,
> `198.51.100.0/24`, `203.0.113.0/24`) and every secret has been replaced by a
> placeholder (`CHANGE_ME`) or externalised into `vars/credentials.yml` (template: [`vars/credentials.example.yml`](vars/credentials.example.yml)).
> No real credential, hostname or public address is present.

---

## Prerequisites

1. **Ansible** and the **Juniper Apstra collection**:
   ```bash
   ansible-galaxy collection install juniper.apstra
   ```
2. **Apstra controller credentials**, exported as environment variables (they are
   consumed by the `juniper.apstra.authenticate` module — never stored in the
   playbooks):
   ```bash
   export APSTRA_URL="https://<apstra-ip>:443"
   export APSTRA_USERNAME="admin"
   export APSTRA_PASSWORD="<your-password>"
   export APSTRA_VERIFY_CERTIFICATES=0
   ```
3. **Sensitive service values** — copy [`vars/credentials.example.yml`](vars/credentials.example.yml)
   to `vars/credentials.yml` and edit it (vCenter, RADIUS, NTP, sFlow). Keep the
   real values in an **Ansible Vault** file rather than in clear text.
4. For ZTP playbooks, export `ZTP_USERNAME` / `ZTP_PASSWORD`.

---

## How to run

Every workflow is triggered from a **numbered orchestrator** playbook:

```bash
ansible-playbook 0_create_base.yml
ansible-playbook 1_create_fabric.yml
# …and so on
```

You can override the target blueprint on the command line:

```bash
ansible-playbook 1_create_fabric.yml -e blueprint_name=DCX
```

---

## Repository layout

| Path | Role |
|------|------|
| `N_*.yml` (numbered) | **Orchestrators** — they *only* chain `import_playbook` calls (see below). |
| `init.yml` / `logout.yml` | Shared **bootstrap / teardown** imported by every orchestrator (open & close the Apstra session, set `blueprint_name`). |
| `create_*`, `assign_*`, `commit_*`, … | **Feature playbooks** — a single functional step, imported by the orchestrators or run standalone. |
| `vars/` | Variable files (`credentials.example.yml`). |

### Orchestrators are import-only

All numbered playbooks are **normalised**: they contain *no* inline task, only a
sequence of `ansible.builtin.import_playbook`. The functional logic lives in the
feature playbooks they import. For example:

```yaml
- name: Connect to Apstra
  ansible.builtin.import_playbook: init.yml
- name: Register vCenter VIM and create ESX generic system
  ansible.builtin.import_playbook: create_servers.yml
- name: Commit
  ansible.builtin.import_playbook: commit_servers.yml
- name: Logout from Apstra
  ansible.builtin.import_playbook: logout.yml
```

### ⚠️ Feature playbooks are *not* self-contained

Most feature playbooks (`create_*`, `assign_*`, `commit_*`, `connect_*`, …) are
designed to be **imported by an orchestrator**. To stay lightweight and avoid
opening a new session at every step, they **do not authenticate** (no
`juniper.apstra.authenticate`) and **do not set the `blueprint_name` fact** (no
`set_fact`). They rely on the Apstra session (`auth`) and the `blueprint_name`
fact previously opened by [`init.yml`](init.yml).

**If an end user wants to run one of these feature playbooks on its own**, they
must first add the missing bootstrap tasks, for example:

```yaml
    - name: Set blueprint name fact
      ansible.builtin.set_fact:
        blueprint_name: "{{ blueprint_name | default('DCX') }}"

    - name: Connect to Apstra
      juniper.apstra.authenticate:
        logout: false
      register: auth
```

The simplest way is to run them through their numbered orchestrator (which
imports `init.yml` / `logout.yml` for you), or to import `init.yml` before the
feature playbook.

---

## The numbered workflow

Run the orchestrators roughly in this order to build a fabric end-to-end:

| Step | Playbook | Purpose |
|------|----------|---------|
| 0 | `0_create_base.yml` | Design catalog: resource pools, rack type, design template. |
| 1 | `1_create_fabric.yml` | Blueprint creation, device acknowledge/assign, resources, commit. |
| 1bis | `1bis_env_iba_housekeeping.yml` | Disable noisy environment IBA probes. |
| 1ter | `1ter_update_cabling_map.yml` | Refresh the cabling map from LLDP. |
| 2 | `2_deploy_configlets.yml` | Configlets + property sets, import & commit. |
| 3 | `3_configure_dci.yml` | Data Center Interconnect (underlay/overlay + routing policy). |
| 4 | `4_create_servers.yml` | Register vCenter VIM, create ESX generic system, commit. |
| 4bis | `4bis_vim_iba_housekeeping.yml` | Disable VIM/hypervisor IBA probes. |
| 5 | `5_create_tenants.yml` | Create tenant 2 (L3) and tenant 5 (L2): VRF + virtual networks. |
| 6 | `6_connect_to_srx.yml` | Connect an SRX firewall (generic system, CTs, IPs). |
| 7 | `7_connect_to_legacy.yml` | LACP bond towards an external legacy network. |
| 8 | `8_create_wan_external.yml` | WAN external connectivity + OSPF. |
| 9 | `9_create_new_service.yml` | Add a new service (VLAN45/46 + servers). |
| 10 | `10_connect_dynamic_hosts.yml` | BGP-to-host dynamic routing + floating IP. |

> Steps `1bis`, `1ter` and `4bis` are optional branches
> of the main path.

---

## Documentation convention

Every playbook starts with a `##` header describing its purpose, and inline
`##` comments highlight the important sections. Feature playbooks rely on the
Apstra session and the `blueprint_name` fact opened by `init.yml`; when used
standalone they must be given their own `authenticate` + `set_fact` tasks (see
[⚠️ Feature playbooks are *not* self-contained](#️-feature-playbooks-are-not-self-contained)).

---

## Security notes

- Never commit real secrets: keep `vars/credentials.yml` values in Ansible Vault.
- Apstra credentials are read from the environment, not from the playbooks.
- All example addressing is RFC 5737 documentation space and is safe to publish.
