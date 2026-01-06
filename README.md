# Ansible DC LAN Migration

Automated migration tooling for Cisco NX-OS switches from CLI-based management to Nexus Dashboard Fabric Controller (NDFC). This project provides complete workflows to:
- **Profile** existing switch configurations
- **Migrate** switches to centralized NDFC management
- **Pre-provision** new access switches before physical deployment

## Overview

This project implements two main workflows:

### Migration Workflow (Existing Switches)
Sequential playbooks to migrate existing CLI-managed switches to NDFC:
1. **Discovery** → Extract configurations from switches
2. **Provision Fabric** → Create fabric definitions in NDFC
3. **Deploy** → Configure VLANs, VPC, and interfaces

### Pre-Provisioning Workflow (New Switches)
Configure new access switches in NDFC before physical deployment:
1. **Pre-provision** → Register switch with serial number
2. **Features** → Enable NX-OS features
3. **Uplinks** → Configure Ethernet uplink interfaces
4. **Port-channels** → Create port-channel bundles

---

## Quick Start

```bash
# 1. Clone and setup environment
git clone <repository-url>
cd ansible-dc-lan-migration
uv sync && source .venv/bin/activate
ansible-galaxy collection install -r requirements.yml

# 2. Configure vault
echo "your-password" > .vault_pass
ansible-vault edit inventory/group_vars/all/vault.yml

# 3. Run playbooks
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml
```

---

## Project Structure

```
ansible-dc-lan-migration/
├── fabrics/                                    # Generated artifacts per fabric
│   └── <fabric-name>/
│       ├── mgmt_interface.yml                  # Management interface configs
│       ├── switch_features.yml                 # Enabled NX-OS features
│       ├── vlan_database.yml                   # VLAN configurations
│       ├── l2_interfaces.yml                   # L2 interface configs
│       └── l3_interfaces.yml                   # L3 interface configs
│
├── inventory/
│   ├── hosts.yml                               # Switch and ND host definitions
│   ├── group_vars/
│   │   ├── all/vault.yml                       # Encrypted credentials
│   │   └── nd/connection.yml                   # ND API settings
│   └── host_vars/nexus_dashboard/
│       └── fabric_definitions.yml              # Fabric source of truth
│
├── playbooks/
│   ├── discovery/                              # Profile existing switches
│   │   └── 01-profile-existing-switches.yml
│   ├── provision-fabric/                       # Fabric configuration
│   │   └── 02-configure-nd-fabric.yml
│   ├── provision-access/                       # NEW: Pre-provision access switches
│   │   ├── 1.0-preprovision-new-switches.yml
│   │   ├── 1.1-preprovision-features.yml
│   │   ├── 1.2-preprovision-uplink-interfaces.yml
│   │   └── 1.3-preprovision-portchannel.yml
│   └── provision-agg/                          # Aggregation switch provisioning
│
├── templates/                                  # Jinja2 templates for API payloads
│   ├── preprovision_switch.json.j2             # POAP pre-provision payload
│   ├── feature_policy.json.j2                  # Feature policy payload
│   ├── feature_lookup.yml                      # Feature to template mapping
│   ├── uplink_ethernet_interface.json.j2       # Ethernet interface payload
│   └── uplink_portchannel_interface.json.j2    # Port-channel payload
│
├── ansible.cfg
├── pyproject.toml
└── requirements.yml
```

---

## Pre-Provisioning Workflow (New Access Switches)

Pre-provision new switches in NDFC **before** physical installation. This allows the switch to automatically configure itself via POAP when it boots.

### Workflow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACCESS SWITCH PRE-PROVISIONING                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1.0 Pre-provision    Register switch with NDFC (serial, IP, role)      │
│         ↓                                                                │
│  1.1 Features         Enable NX-OS features (LACP, LLDP, etc.)          │
│         ↓                                                                │
│  1.2 Uplinks          Configure Ethernet member interfaces              │
│         ↓                                                                │
│  1.3 Port-channel     Create port-channel bundle with VLANs             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Execution

#### Step 1.0: Pre-provision Switch

Register the new switch with NDFC using its serial number:

```bash
ansible-playbook playbooks/provision-access/1.0-preprovision-new-switches.yml
```

**What it does:**
- Registers switch serial number, model, and version
- Assigns management IP and default gateway
- Sets switch hostname and role

**NDFC API:**
```
POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/fabrics/<fabric>/inventory/poap
```

**Example Payload:**
```json
{
  "serialNumber": "9XDN0DYD1TM",
  "model": "N9K-C9300v",
  "version": "10.6(1)",
  "hostname": "mgmt-acc01",
  "ipAddress": "198.18.24.82",
  "role": "access",
  "data": "{\"modulesModel\": [\"N9K-C9300v\"], \"gateway\": \"198.18.24.65/26\"}"
}
```

---

#### Step 1.1: Configure Features

Enable required NX-OS features on the switch:

```bash
ansible-playbook playbooks/provision-access/1.1-preprovision-features.yml
```

**What it does:**
- Reads enabled features from `switch_features.yml`
- Maps features to NDFC template names
- Creates feature policies on the switch

**NDFC API:**
```
POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/policies/bulk-create
```

**Feature Mapping:**
| NX-OS Feature | NDFC Template |
|---------------|---------------|
| `tacacs+` | `feature_tacacs` |
| `interface-vlan` | `feature_interface_vlan_11_1` |
| `lacp` | `feature_lacp` |
| `lldp` | `feature_lldp` |
| `nxapi` | `feature_nxapi` |

**Example Payload:**
```json
{
  "templateName": "feature_lacp",
  "serialNumber": "9XDN0DYD1TM",
  "entityType": "SWITCH",
  "entityName": "SWITCH",
  "priority": 500,
  "nvPairs": {}
}
```

---

#### Step 1.2: Configure Uplink Ethernet Interfaces

Configure the physical Ethernet interfaces that will be port-channel members:

```bash
ansible-playbook playbooks/provision-access/1.2-preprovision-uplink-interfaces.yml
```

**What it does:**
- Reads port-channel members from `l2_interfaces.yml`
- Creates Ethernet interface configurations
- Sets trunk mode with jumbo MTU

**NDFC API:**
```
POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/globalInterface
```

**Example Payload:**
```json
{
  "policy": "int_trunk_host",
  "interfaceType": "INTERFACE_ETHERNET",
  "interfaces": [{
    "serialNumber": "9XDN0DYD1TM",
    "fabricName": "mgmt-fabric",
    "ifName": "Ethernet1/1",
    "nvPairs": {
      "MTU": "jumbo",
      "SPEED": "Auto",
      "ADMIN_STATE": "true",
      "CDP_ENABLE": "true"
    }
  }]
}
```

---

#### Step 1.3: Configure Port-Channel

Create the port-channel interface with member interfaces and VLANs:

```bash
ansible-playbook playbooks/provision-access/1.3-preprovision-portchannel.yml
```

**What it does:**
- Creates port-channel with LACP mode
- Assigns member interfaces
- Configures allowed VLANs and native VLAN

**NDFC API:**
```
POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/globalInterface
```

**Example Payload:**
```json
{
  "policy": "int_port_channel_trunk_host",
  "interfaceType": "INTERFACE_PORT_CHANNEL",
  "interfaces": [{
    "serialNumber": "9XDN0DYD1TM",
    "fabricName": "mgmt-fabric",
    "ifName": "Port-channel113",
    "nvPairs": {
      "MEMBER_INTERFACES": "Ethernet1/1,Ethernet1/2",
      "PC_MODE": "active",
      "ALLOWED_VLANS": "101-199",
      "NATIVE_VLAN": "2",
      "DESC": "Uplink to Agg VPC"
    }
  }]
}
```

---

### Complete Pre-Provisioning Example

Run all playbooks in sequence:

```bash
# Profile source switch first (to generate YAML data files)
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml

# Pre-provision workflow
ansible-playbook playbooks/provision-access/1.0-preprovision-new-switches.yml
ansible-playbook playbooks/provision-access/1.1-preprovision-features.yml
ansible-playbook playbooks/provision-access/1.2-preprovision-uplink-interfaces.yml
ansible-playbook playbooks/provision-access/1.3-preprovision-portchannel.yml
```

### Inventory Requirements

Add switches to provision in `inventory/hosts.yml`:

```yaml
switches:
  children:
    access:
      hosts:
        mgmt-acc01:
          ansible_host: 198.18.24.81          # Source switch IP (for profiling)
          fabric: mgmt-fabric
          role: access
          add_to_fabric: true                  # Flag to include in provisioning
          destination_switch_sn: 9XDN0DYD1TM   # NEW switch serial number
          destination_switch_model: N9K-C9300v
          destination_switch_version: "10.6(1)"
          destination_switch_ip_addr: 198.18.24.82  # NEW switch IP
```

### Idempotency

All pre-provisioning playbooks are **idempotent**:
- Query NDFC for existing resources before creating
- Skip already-configured items
- Safe to re-run multiple times

```
✅ Switches to pre-provision (excluding existing): 0
   Already pre-provisioned: 3
```

---

## Discovery Workflow

Profile existing switches to extract configurations:

```bash
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml
```

### Available Tags

```bash
# Profile specific components only
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml --tags profile-switches
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml --tags profile-vlans
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml --tags profile-l2-interfaces
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml --tags profile-l3-interfaces
```

### Generated Files

| File | Contents |
|------|----------|
| `mgmt_interface.yml` | Management interface, IP, VRF, default gateway |
| `switch_features.yml` | Enabled NX-OS features |
| `vlan_database.yml` | VLAN IDs and names |
| `l2_interfaces.yml` | L2 interfaces, port-channels, members |
| `l3_interfaces.yml` | SVIs, loopbacks, HSRP, OSPF |

---

## Fabric Configuration

Create fabric definitions in NDFC:

```bash
ansible-playbook playbooks/provision-fabric/02-configure-nd-fabric.yml
```

Edit fabric settings in `inventory/host_vars/nexus_dashboard/fabric_definitions.yml`.

---

## Future Playbooks (Planned)

The following playbooks are planned for future implementation:

| Playbook | Purpose | API Endpoint |
|----------|---------|--------------|
| `1.4-preprovision-svi.yml` | Configure management SVI | `/globalInterface` (INTERFACE_VLAN) |
| `1.5-preprovision-static-route.yml` | Configure default route | `/policies/bulk-create` (static_route_v4_v6) |
| `1.6-import-switch-bootstrap.yml` | Trigger POAP bootstrap | `/inventory/poap` |

---

## Connection Patterns

### NX-OS Switches (`network_cli`)
- SSH connection to switches
- Uses `cisco.nxos` collection
- Credentials: `vault_switch_password`

### Nexus Dashboard (`httpapi`)
- HTTPS REST API connection
- Uses `cisco.nd` collection
- Credentials: `vault_nd_password`
- Timeout: 1000 seconds

---

## Troubleshooting

### Common Commands

```bash
# Dry-run mode
ansible-playbook playbooks/discovery/01-profile-existing-switches.yml --check

# Verbose output
ansible-playbook playbooks/provision-access/1.0-preprovision-new-switches.yml -vvv

# List tasks
ansible-playbook playbooks/provision-access/1.3-preprovision-portchannel.yml --list-tasks

# Lint playbooks
ansible-lint playbooks/**/*.yml
```

### Verify Generated Files

```bash
ls -la fabrics/mgmt-fabric/
cat fabrics/mgmt-fabric/l2_interfaces.yml
cat fabrics/mgmt-fabric/switch_features.yml
```

### Check Vault

```bash
ansible-vault view inventory/group_vars/all/vault.yml
```

---

## Prerequisites

### Software
- **Python 3.13+** (via UV)
- **Ansible Core 2.15.0+**
- **UV Package Manager**

### Ansible Collections
- `cisco.nd` >= 1.3.0
- `cisco.dcnm` >= 2.4.0
- `cisco.nxos` >= 4.6.0

### Install

```bash
# Install UV
curl -LsSf https://astral.sh/uv/install.sh | sh

# Setup project
uv sync
source .venv/bin/activate
ansible-galaxy collection install -r requirements.yml
```

---

## License

MIT License

## Authors

- Russell Johnston (rujohns2@cisco.com)
