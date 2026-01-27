# Copilot Instructions: Ansible DC LAN Migration

## Architecture Overview

Ansible automation migrating Cisco NX-OS switches from CLI to **Nexus Dashboard Fabric Controller (NDFC)**, plus POAP pre-provisioning for new switches.

### Data Flow (Critical Path)
```
inventory/hosts.yml           → Defines switches (fabric, role, add_to_fabric)
fabric_definitions.yml        → Source of truth: nd_fabrics[], vpc_domains[]
discovery/1.0-profile*        → SSH to switches → generates fabrics/<fabric>/*.yml
templates/*.j2                → Transform YAML → NDFC API payloads
provision-switch/1.x-*        → Deploy via NDFC httpapi
```

### Workflow Sequence (`0.0-full-provision-switch.yml` runs all)
| Playbook | Purpose | Tags |
|----------|---------|------|
| 1.0-provision-switches | POAP or discovery | `preprovision-switches`, `discover-switches` |
| 1.1-create-discovery-user | SSH user for NDFC | — |
| 1.2-provision-features | NX-OS features | — |
| 1.3-deploy-vpc-domain | VPC peer-link (aggregation only) | — |
| 1.4-provision-interfaces | L2/VPC/SVI | `deploy-l2-interfaces`, `deploy-vpc-interfaces`, `deploy-svi-interfaces` |
| 1.5-provision-vlan-policies | VLAN database | — |
| 1.6-provision-default-route | Static routes | — |
| 1.7-check-poap-status | Verify POAP completion | — |
| 1.8-bootstrap-switches | POAP Day-0 config | — |

### Discovery Playbook Tags (`1.0-profile-existing-switches.yml`)
| Tag | Output File |
|-----|-------------|
| `profile-switches` | `switch_details.yml` |
| `profile-features` | `switch_features.yml` |
| `profile-vlans` | `vlan_database.yml` |
| `profile-l3-interfaces` | `l3_interfaces.yml` |
| `profile-l2-interfaces` | `l2_interfaces.yml`, `l2_vpc_interfaces.yml` |
| `profile-static-routes` | `static_routes.yml` |

## Connection Patterns (NEVER Mix!)

| Target | `hosts:` | Connection | Collection |
|--------|----------|------------|------------|
| NX-OS Switches | `switches` | `ansible.netcommon.network_cli` | `cisco.nxos` |
| Nexus Dashboard | `nexus_dashboard` | `ansible.netcommon.httpapi` | `cisco.nd`, `cisco.dcnm` |

**Credentials**: `vault_switch_password` (switches), `vault_nd_password` (ND) → `inventory/group_vars/all/vault.yml`

## Development Setup

```bash
# Python 3.13+ required (UV only, never pip)
uv sync && source .venv/bin/activate
ansible-galaxy collection install -r requirements.yml

# Vault setup
echo "password" > .vault_pass
ansible-vault edit inventory/group_vars/all/vault.yml

# Run examples
ansible-playbook playbooks/provision-switch/0.0-full-provision-switch.yml -v
ansible-playbook playbooks/provision-switch/1.4-provision-interfaces.yml --tags deploy-vpc-interfaces
ansible-playbook playbooks/discovery/1.0-profile-existing-switches.yml --tags profile-vlans
```

## Key Patterns

### Switch Filtering (used in every playbook)
```yaml
- set_fact:
    switches_to_provision: >-
      {{ groups['switches'] | map('extract', hostvars)
         | selectattr('add_to_fabric', 'defined')
         | selectattr('add_to_fabric', 'equalto', true) | list }}
```

### Interface Naming Transforms (templates)
```jinja2
port-channel113 → vpc113           {# VPC dcnm_interface format #}
Ethernet1/4 → e1/4                 {# regex_replace('^Ethernet', 'e') #}
```

### VPC ID 1 Reserved
Templates skip `vpc_id: 1` — it's the VPC peer-link managed by `vpc_domains[]` in `fabric_definitions.yml`.

### hosts.yml Switch Structure
```yaml
switch_name:
  ansible_host: 198.18.24.81
  fabric: mgmt-fabric              # Must match nd_fabrics[].FABRIC_NAME
  role: access                     # access | aggregation
  add_to_fabric: true              # false = excluded from provisioning
  mgmt_int: Vlan199
  # POAP only (new switches):
  destination_switch_sn: ABC123
  destination_switch_model: N9K-C9300v
  destination_switch_version: "10.6(1)"
```

## Modules Reference

| Module | Use Case | State |
|--------|----------|-------|
| `cisco.nd.nd_rest` | Raw NDFC REST API (POAP, policies) | — |
| `cisco.dcnm.dcnm_interface` | Interface config | `replaced` |
| `cisco.dcnm.dcnm_vpc_pair` | VPC domain | `merged` |
| `cisco.nxos.nxos_facts` | SSH discovery | — |

## Critical Notes

- **NDFC Timeouts**: `ansible_command_timeout: 1000` in `inventory/group_vars/nd/connection.yml`
- **Feature Mapping**: `templates/feature_lookup.yml` maps NX-OS features → NDFC template names
- **Generated Files**: `fabrics/<fabric>/` created by discovery playbook, never edit manually
- **Gateway Format**: POAP requires prefix length (e.g., `198.18.24.65/26`)
- **Python 3.13+** required (managed via `uv`, never use pip directly)
- **VPC ID 1**: Skipped in templates — reserved for peer-link managed by VPC domain
- **Inventory Variants**: Use `hosts.rbc.yml` for production (`-i inventory/hosts.rbc.yml`)

## Template Organization

Templates follow playbook numbering (`1.4-provision-interfaces-*.j2` for playbook `1.4`):

| Template | Module | Output Format |
|----------|--------|---------------|
| `1.4-provision-interfaces-eth.j2` | `dcnm_interface` | JSON config list |
| `1.4-provision-interfaces-vpc.j2` | `dcnm_interface` | JSON config list |
| `1.4-provision-interfaces-svi.j2` | `dcnm_interface` | JSON config list |
| `1.5-provision-vlan-policies.j2` | `nd_rest` | NDFC REST payload |
| `1.8-bootstrap-switches.j2` | `nd_rest` | POAP bootstrap payload |

Templates read from `fabrics/<fabric>/*.yml` and output NDFC-compatible payloads via `| from_json` filter.

## Playbook Conventions

### Multi-Play Structure
Playbooks targeting NDFC use multiple plays when handling different switch types:
- **Aggregation switches**: `deploy: false` (config staged, no immediate push)
- **Access/POAP switches**: `deploy: false` (config staged for POAP bootstrap)

### Role-Based Deployment Strategy
```yaml
# Separate configs by role before deploying
eth_interface_config_agg: "{{ config | selectattr('switch', 'in', agg_switches) }}"
eth_interface_config_access: "{{ config | rejectattr('switch', 'in', agg_switches) }}"
```

### Loading Fabric Data Pattern
```yaml
- find:
    paths: "{{ playbook_dir }}/../../fabrics"
    patterns: "l2_interfaces.yml"
    recurse: true
  register: interface_files

- include_vars:
    file: "{{ item.path }}"
    name: "data_{{ item.path | dirname | basename }}"  # Namespace by fabric
  loop: "{{ interface_files.files }}"
```

## Adding New Switches

1. Add entry to `inventory/hosts.yml` under `switches.children.access` or `switches.children.aggegation`
2. Set `add_to_fabric: true` to include in provisioning
3. For POAP (new hardware): include `destination_switch_sn`, `destination_switch_model`, `destination_switch_version`
4. Run discovery first if migrating existing switch: `--tags profile-switches,profile-vlans,...`

## Debugging Tips

- **Retry files**: `.retry` files in `retry/` folder list failed hosts from previous runs
- **Dry-run NDFC API**: Use `scripts/dry-run-ndfc-api.sh` to test REST calls
- **Template debugging**: Add `| to_nice_json` after template output to inspect generated payloads
- **Connection issues**: Check `ansible_command_timeout: 1000` in `inventory/group_vars/nd/connection.yml`
