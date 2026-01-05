# Ansible DC LAN Migration

Ansible automation for profiling existing Cisco NX-OS switches and migrating configurations to Nexus Dashboard Fabric Controller (NDFC).

## Prerequisites

- Ansible 2.19.4 or later
- Cisco DCNM collection: `cisco.dcnm`
- Cisco NX-OS collection: `cisco.nxos`
- Network connectivity to existing switches and NDFC
- Valid credentials configured in inventory vault

Install required collections:
```bash
ansible-galaxy collection install -r requirements.yml
```

## Getting Started

### Local Environment Setup with UV

This project uses [uv](https://docs.astral.sh/uv/) for Python dependency management. UV provides fast, reliable package installation and environment management.

**Install UV** (if not already installed):
```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or via Homebrew
brew install uv
```

**Setup project environment**:
```bash
# Sync dependencies from pyproject.toml
uv sync

# Activate the virtual environment
source .venv/bin/activate

# Install Ansible collections
ansible-galaxy collection install -r requirements.yml
```

For detailed UV documentation and advanced usage, see [docs/uv-setup.md](docs/uv-setup.md).

### Ansible Vault Configuration

Sensitive credentials are stored using Ansible Vault. You'll need to configure vault encryption for your inventory variables.

**Quick setup**:
```bash
# Create vault password file (add to .gitignore)
echo "your-vault-password" > .vault_pass

# Edit vault file
ansible-vault edit inventory/group_vars/all/vault.yml
```

For comprehensive vault usage, encryption strategies, and best practices, see [docs/ansible-vault.md](docs/ansible-vault.md).

## Project Structure

```
├── fabrics/              # Generated fabric-specific data (created during profiling)
│   └── mgmt-fabric/      # Per-fabric inventory files
├── inventory/            # Ansible inventory and variables
│   ├── hosts.yml         # Inventory hosts
│   └── group_vars/       # Group variables and vault
├── playbooks/            # Ansible playbooks (numbered execution order)
├── templates/            # Jinja2 templates for config generation
└── source_configs/       # Original switch configurations
```

**Note:** The `fabrics/` directory is automatically created when running the profiling playbook (01-profile-existing-switches.yml). A subdirectory is created for each fabric defined in `inventory/host_vars/nexus_dashboard/fabric_definitions.yml`.

## Playbooks

### 01-profile-existing-switches.yml

Profiles existing NX-OS switches to extract configuration data for migration.

**Tags:**
- `profile-switches` - Gather switch hardware information
- `profile-vlans` - Extract VLAN configurations
- `profile-l3-interfaces` - Profile Layer 3 interfaces (IP, HSRP, OSPF)
- `profile-l2-interfaces` - Profile Layer 2 interfaces and VPC configurations

**Outputs:**
- `fabrics/<fabric>/switch_inventory.csv` - Switch hardware inventory
- `fabrics/<fabric>/vlan_database.yml` - Consolidated VLAN database
- `fabrics/<fabric>/l3_interfaces.yml` - Layer 3 interface configurations
- `fabrics/<fabric>/l2_interfaces.yml` - Layer 2 interface configurations
- `fabrics/<fabric>/l2_vpc_interfaces.yml` - VPC interface configurations

**Usage:**
```bash
# Profile all configurations
ansible-playbook playbooks/01-profile-existing-switches.yml

# Profile only VLANs and L2 interfaces
ansible-playbook playbooks/01-profile-existing-switches.yml --tags profile-vlans,profile-l2-interfaces
```

### 02-configure-nd-fabric.yml

Creates fabric configuration in Nexus Dashboard.

**Usage:**
```bash
ansible-playbook playbooks/02-configure-nd-fabric.yml
```

### 03-add-switches-to-nd-fabric.yml

Adds switches to the NDFC fabric.

**Usage:**
```bash
ansible-playbook playbooks/03-add-switches-to-nd-fabric.yml
```

### 04-configure-vlan-policies.yml

Deploys VLAN policies to the fabric.

**Usage:**
```bash
ansible-playbook playbooks/04-configure-vlan-policies.yml
```

### 05-deploy-vpc-domain.yml

Configures VPC domain and peer-link between switch pairs.

**Usage:**
```bash
ansible-playbook playbooks/05-deploy-vpc-domain.yml
```

### 06-configure-interfaces.yml

Deploys Layer 2, Layer 3, and VPC interface configurations to NDFC.

**Tags:**
- `deploy-l2-interfaces` - Deploy Layer 2 interface configurations
- `deploy-l3-interfaces` - Deploy Layer 3 interface configurations (SVI, routed ports, loopbacks)
- `deploy-vpc-interfaces` - Deploy VPC interface configurations

**Usage:**
```bash
# Deploy all interface configurations
ansible-playbook playbooks/06-configure-interfaces.yml

# Deploy only VPC interfaces
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-vpc-interfaces

# Deploy L2 and L3 interfaces only
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-l2-interfaces,deploy-l3-interfaces
```

## Common Workflows

### Complete Migration

Execute playbooks in sequence:
```bash
ansible-playbook playbooks/01-profile-existing-switches.yml
ansible-playbook playbooks/02-configure-nd-fabric.yml
ansible-playbook playbooks/03-add-switches-to-nd-fabric.yml
ansible-playbook playbooks/04-configure-vlan-policies.yml
ansible-playbook playbooks/05-deploy-vpc-domain.yml
ansible-playbook playbooks/06-configure-interfaces.yml
```

### Re-profile and Update Interfaces

Update existing deployment after configuration changes:
```bash
# Re-profile only L2 and VPC configurations
ansible-playbook playbooks/01-profile-existing-switches.yml --tags profile-l2-interfaces

# Deploy updated VPC configurations
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-vpc-interfaces
```

### Deploy Specific Interface Types

Deploy individual interface types for targeted updates:
```bash
# Deploy only Layer 3 interfaces (SVIs, routed ports)
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-l3-interfaces
```

## Configuration Notes

- **VPC Interfaces**: Port-channel interfaces in VPC configurations are automatically converted to VPC interface names (port-channel10 → vpc10)
- **Interface Exclusions**: mgmt0, Vlan1, VPC peer-links, and port-channel member interfaces are excluded from profiling
- **Interface Modes**: SVIs use mode `vlan`, loopbacks use mode `lo`, physical and port-channel interfaces use mode `routed`
- **Inventory**: Inventory files in `fabrics/<fabric>/` are organized by fabric and switch hostname

## Troubleshooting

- Review generated inventory files in `fabrics/<fabric>/` to verify profiled configurations
- Use `--check` mode to validate playbook execution without making changes
- Retry files in `retry/` contain hosts that failed during playbook execution
- Check vault credentials in `inventory/group_vars/all/vault.yml`

## Support

For issues or questions, review the comprehensive comments in playbooks and template headers for detailed implementation information.
