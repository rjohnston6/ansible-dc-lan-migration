# Ansible DC LAN Migration

Automated migration tooling for Cisco NX-OS switches from CLI-based management to Nexus Dashboard Fabric Controller (NDFC). This project provides a complete workflow to profile existing switch configurations and seamlessly migrate them to centralized NDFC management.

## Overview

This project implements a **6-stage sequential workflow**:
1. **Profile** - Extract configurations from existing switches
2. **Configure** - Create fabric definitions in Nexus Dashboard
3. **Add Switches** - Register switches with NDFC
4. **Deploy VLANs** - Push VLAN policies to the fabric
5. **Configure VPC** - Establish VPC domains and peer-links
6. **Deploy Interfaces** - Configure L2/L3/VPC interfaces

## Prerequisites

### Software Requirements
- **Python 3.13+** (managed via UV)
- **Ansible Core 2.15.0+** (installed via UV)
- **UV Package Manager** - Fast Python dependency management

### Ansible Collections
- `cisco.nd` >= 1.3.0 - Nexus Dashboard REST API
- `cisco.dcnm` >= 2.4.0 - NDFC/DCNM modules  
- `cisco.nxos` >= 4.6.0 - NX-OS device management
- `community.general` >= 6.4.0 - Common utilities

### Network Access
- SSH connectivity to existing NX-OS switches (port 22)
- HTTPS access to Nexus Dashboard API (port 443)
- Valid credentials configured in Ansible Vault

## Getting Started

### 1. Environment Setup with UV

This project uses [UV](https://docs.astral.sh/uv/) for fast, reproducible Python dependency management.

#### Install UV
```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or via Homebrew
brew install uv
```

#### Initialize Project
```bash
# Clone repository
git clone <repository-url>
cd ansible-dc-lan-migration

# Install all dependencies (Python packages + Ansible collections)
uv sync

# Activate virtual environment
source .venv/bin/activate

# Install Ansible collections
ansible-galaxy collection install -r requirements.yml
```

> **Note:** Always use `uv` commands, not `pip`, to maintain reproducible environments via `uv.lock`.

For advanced UV usage, see [docs/uv-setup.md](docs/uv-setup.md).

### 2. Configure Ansible Vault

Store sensitive credentials securely using Ansible Vault.

```bash
# Create vault password file (gitignored automatically)
echo "your-secure-password" > .vault_pass

# Edit encrypted vault file
ansible-vault edit inventory/group_vars/all/vault.yml
```

**Required vault variables:**
```yaml
---
vault_switch_password: "password-for-nxos-switches"
vault_nd_password: "password-for-nexus-dashboard"
```

For comprehensive vault management, see [docs/ansible-vault.md](docs/ansible-vault.md).

### 3. Configure Inventory

Edit inventory files to match your environment:

**[inventory/hosts.yml](inventory/hosts.yml)** - Define switches with required attributes:
```yaml
switches:
  children:
    aggregation:
      hosts:
        agg01:
          ansible_host: 198.18.24.66
          fabric: mgmt-fabric        # Fabric assignment
          role: aggregation           # Switch role
          migrate: true               # Migration flag
```

**[inventory/host_vars/nexus_dashboard/fabric_definitions.yml](inventory/host_vars/nexus_dashboard/fabric_definitions.yml)** - Define fabric parameters:
```yaml
nd_fabrics:
  - FABRIC_NAME: mgmt-fabric
    FABRIC_TYPE: classicLan
    IS_READ_ONLY: true
    INBAND_MGMT: true
    MGMT_GW: 198.18.24.65
    # ... additional fabric settings
```

## Project Structure

```
ansible-dc-lan-migration/
├── .github/
│   └── copilot-instructions.md   # AI agent guidance
├── fabrics/                        # Generated artifacts (created by playbook 01)
│   └── <fabric-name>/              # Per-fabric profiled data
│       ├── switch_inventory.csv    # Hardware inventory
│       ├── vlan_database.yml       # VLAN configurations
│       ├── l3_interfaces.yml       # L3 interface configs
│       ├── l2_interfaces.yml       # L2 interface configs
│       └── l2_vpc_interfaces.yml   # VPC interface configs
├── inventory/
│   ├── hosts.yml                   # Switch and ND host definitions
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── common.yml          # Non-sensitive variables
│   │   │   └── vault.yml           # Encrypted credentials
│   │   └── nd/
│   │       └── connection.yml      # ND API connection settings
│   └── host_vars/
│       └── nexus_dashboard/
│           └── fabric_definitions.yml  # Fabric definitions (source of truth)
├── playbooks/                      # Sequential numbered playbooks
│   ├── 01-profile-existing-switches.yml
│   ├── 02-configure-nd-fabric.yml
│   ├── 03-add-switches-to-nd-fabric.yml
│   ├── 04-configure-vlan-policies.yml
│   ├── 05-deploy-vpc-domain.yml
│   └── 06-configure-interfaces.yml
├── templates/                      # Jinja2 templates for API payloads
│   ├── create_fabric.json.j2       # ND fabric creation
│   ├── add_switches_to_fabric.json.j2
│   ├── vpc_pair.json.j2
│   └── ...
├── ansible.cfg                     # Ansible configuration
├── pyproject.toml                  # Python dependencies (UV)
└── requirements.yml                # Ansible collections
```

**Key Directories:**
- **`fabrics/`** - Auto-generated by playbook 01, contains profiled switch data
- **`inventory/host_vars/nexus_dashboard/`** - Source of truth for fabric definitions
- **`templates/`** - Transforms Ansible variables → NDFC API JSON payloads

## Playbooks Overview

Playbooks must be executed **sequentially** - each depends on outputs from the previous stage.

### 01 - Profile Existing Switches

Discovers and extracts configurations from existing NX-OS switches.

**What it profiles:**
- Switch hardware info (model, serial, version)
- VLAN configurations and assignments
- Layer 3 interfaces (SVIs, loopbacks, routed ports) with IP, HSRP, OSPF
- Layer 2 interfaces (access, trunk, port-channels)
- VPC configurations

**Available tags:**
- `profile-switches` - Hardware information only
- `profile-vlans` - VLAN configurations
- `profile-l3-interfaces` - Layer 3 interfaces with routing protocols
- `profile-l2-interfaces` - Layer 2 and VPC interfaces

**Generated files:** `fabrics/<fabric>/` directory with all inventory files

**Usage:**
```bash
# Profile everything
ansible-playbook playbooks/01-profile-existing-switches.yml

# Profile specific components
ansible-playbook playbooks/01-profile-existing-switches.yml --tags profile-vlans,profile-l2-interfaces
```

### 02 - Configure ND Fabric

Creates fabric definitions in Nexus Dashboard via REST API.

**What it does:**
- Queries existing fabrics to prevent duplicates
- Creates new fabrics from `fabric_definitions.yml`
- Supports Classic LAN and VXLAN fabric types

**Usage:**
```bash
ansible-playbook playbooks/02-configure-nd-fabric.yml
```

### 03 - Add Switches to ND Fabric

Registers switches with NDFC fabric management.

**Usage:**
```bash
ansible-playbook playbooks/03-add-switches-to-nd-fabric.yml
```

### 04 - Configure VLAN Policies

Deploys VLAN policies from profiled VLAN database to the fabric.

**Usage:**
```bash
ansible-playbook playbooks/04-configure-vlan-policies.yml
```

### 05 - Deploy VPC Domain

Configures VPC domains and peer-link connectivity between switch pairs.

**What it configures:**
- VPC domain IDs
- Keepalive links and VRF
- Peer-link port-channels
- Allowed VLANs on peer-links

**Usage:**
```bash
ansible-playbook playbooks/05-deploy-vpc-domain.yml
```

### 06 - Configure Interfaces

Deploys Layer 2, Layer 3, and VPC interface configurations to NDFC.

**Available tags:**
- `deploy-l2-interfaces` - L2 access/trunk ports
- `deploy-l3-interfaces` - SVIs, routed interfaces, loopbacks
- `deploy-vpc-interfaces` - VPC port-channels

**Usage:**
```bash
# Deploy all interface types
ansible-playbook playbooks/06-configure-interfaces.yml

# Deploy specific interface types
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-vpc-interfaces
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-l2-interfaces,deploy-l3-interfaces
```

## Common Workflows

### Full Migration Sequence

Execute all playbooks in order for complete CLI → NDFC migration:

```bash
# Step 1: Profile existing switches
ansible-playbook playbooks/01-profile-existing-switches.yml

# Step 2: Create fabric in Nexus Dashboard
ansible-playbook playbooks/02-configure-nd-fabric.yml

# Step 3: Register switches with fabric
ansible-playbook playbooks/03-add-switches-to-nd-fabric.yml

# Step 4: Deploy VLAN policies
ansible-playbook playbooks/04-configure-vlan-policies.yml

# Step 5: Configure VPC domains
ansible-playbook playbooks/05-deploy-vpc-domain.yml

# Step 6: Deploy interface configurations
ansible-playbook playbooks/06-configure-interfaces.yml
```

### Incremental Updates

Re-profile and update specific configurations after changes:

```bash
# Re-profile VPC interfaces after configuration changes
ansible-playbook playbooks/01-profile-existing-switches.yml --tags profile-l2-interfaces

# Deploy updated VPC configurations to NDFC
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-vpc-interfaces
```

### Selective Deployment

Deploy individual interface types for targeted updates:

```bash
# Deploy only SVIs and routed interfaces
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-l3-interfaces

# Deploy only L2 access/trunk ports
ansible-playbook playbooks/06-configure-interfaces.yml --tags deploy-l2-interfaces
```

### Testing and Validation

```bash
# Dry-run mode (check without changes)
ansible-playbook playbooks/01-profile-existing-switches.yml --check

# List all tasks without execution
ansible-playbook playbooks/06-configure-interfaces.yml --list-tasks

# Verbose output for troubleshooting
ansible-playbook playbooks/02-configure-nd-fabric.yml -vvv

# Lint playbooks for best practices
ansible-lint playbooks/*.yml
```

## Key Concepts

### Connection Patterns

This project uses two distinct connection types:

**1. NX-OS Switches (`network_cli`)**
- SSH connection to switches
- Uses `cisco.nxos` collection modules
- Credentials: `vault_switch_password`
- Config: `inventory/hosts.yml` under `switches` group

**2. Nexus Dashboard (`httpapi`)**
- HTTPS REST API connection
- Uses `cisco.nd` collection modules  
- Credentials: `vault_nd_password`
- Config: `inventory/group_vars/nd/connection.yml`

### Interface Naming Transformations

NDFC uses different naming conventions than CLI:

| CLI Name | NDFC Name | Interface Type |
|----------|-----------|----------------|
| `port-channel10` | `vpc10` | VPC interfaces |
| `Vlan100` | mode: `vlan` | SVI interfaces |
| `loopback0` | mode: `lo` | Loopback interfaces |
| `Ethernet1/1` | mode: `routed` | Physical interfaces |

### Interface Exclusions

Playbook 01 automatically excludes from profiling:
- `mgmt0` - Management interface
- `Vlan1` - Default VLAN
- VPC peer-link interfaces - Managed by VPC domain configuration
- Port-channel member interfaces - Only parent port-channel is profiled

### Fabric Organization

- **Multi-fabric support** - Each fabric defined in `fabric_definitions.yml`
- **Isolated artifacts** - Each fabric gets `fabrics/<fabric-name>/` directory
- **VPC domain definition** - Defined per-fabric with peer switch mappings
- **Idempotent operations** - Playbooks check existing state before creating

## Important Notes

### Execution Order
⚠️ **Playbooks must run sequentially** - Later playbooks depend on artifacts from earlier ones:
- Can't deploy VLANs before fabric exists (02 → 04)
- Can't configure interfaces before switches are added (03 → 06)
- Re-profiling (01) is safe and updates existing artifact files

### API Timeouts
NDFC operations can be slow. Connection timeout is set to `1000` seconds in `inventory/group_vars/nd/connection.yml`. Don't reduce this value.

### Python Version
Requires **Python 3.13+** per `pyproject.toml`. Use `pyenv` or `uv` to manage Python versions:
```bash
# Check Python version in venv
source .venv/bin/activate
python --version  # Should be 3.13+
```

### Generated Files
Files in `fabrics/<fabric>/` are **generated artifacts**, not source files:
- Don't manually edit these files
- Re-run playbook 01 to regenerate from switch configs
- Safe to delete and regenerate at any time

## Troubleshooting

### Common Issues

**"dict object has no attribute ipv6" error**
- Fixed in latest version - ensure you have the IPv6 safety checks in playbook 01
- Update your repo: `git pull origin main`

**Playbook fails with "fabric not found"**
- Ensure playbooks run in sequence (02 before 03, 03 before 04, etc.)
- Check `fabrics/` directory exists and contains your fabric subdirectory

**NDFC API timeout errors**
- NDFC operations are inherently slow - this is normal
- Don't reduce `ansible_command_timeout` in connection.yml
- Consider using `--limit` to process fewer switches at once

**Vault password errors**
- Ensure `.vault_pass` file exists with correct password
- Check file permissions: `chmod 600 .vault_pass`
- Verify vault file can be decrypted: `ansible-vault view inventory/group_vars/all/vault.yml`

**Collection not found errors**
```bash
# Reinstall Ansible collections
ansible-galaxy collection install -r requirements.yml --force
```

### Validation Steps

1. **Check generated artifacts** - Review files in `fabrics/<fabric>/` after profiling:
   ```bash
   ls -la fabrics/mgmt-fabric/
   cat fabrics/mgmt-fabric/vlan_database.yml
   ```

2. **Verify inventory structure**:
   ```bash
   ansible-inventory --graph
   ansible-inventory --host agg01
   ```

3. **Test vault access**:
   ```bash
   ansible-vault view inventory/group_vars/all/vault.yml
   ```

4. **Validate playbook syntax**:
   ```bash
   ansible-playbook playbooks/01-profile-existing-switches.yml --syntax-check
   ansible-lint playbooks/*.yml
   ```

5. **Check connectivity**:
   ```bash
   # Test switch connectivity
   ansible switches -m ping
   
   # Test Nexus Dashboard API
   ansible nexus_dashboard -m cisco.nd.nd_rest -a "path=/api/v1/manage/fabrics method=GET"
   ```

### Logs and Debugging

- **Ansible logs**: `./ansible.log` (configured in `ansible.cfg`)
- **Retry files**: `./retry/<playbook>.retry` - Contains failed hosts
- **Verbose mode**: Add `-v`, `-vv`, or `-vvv` for increasing verbosity
- **Step-through mode**: Use `--step` to execute tasks interactively

### Getting Help

1. Review playbook comments - Each playbook has detailed header documentation
2. Check template headers in `templates/*.j2` for API payload structure
3. Consult Cisco collection docs:
   - [cisco.nd documentation](https://galaxy.ansible.com/ui/repo/published/cisco/nd/docs/)
   - [cisco.nxos documentation](https://galaxy.ansible.com/ui/repo/published/cisco/nxos/docs/)
4. See [.github/copilot-instructions.md](.github/copilot-instructions.md) for AI agent guidance

## Contributing

When modifying this project:

- **Adding fabric types**: Update conditional logic in `templates/create_fabric.json.j2`
- **New interface types**: Extend filtering in playbook 01 (search for `when:` conditions)
- **API changes**: Check `cisco.nd.nd_rest` module calls - update `path` parameters
- **Inventory changes**: Switches require `fabric` and `role` attributes in `hosts.yml`

Run linting before committing:
```bash
ansible-lint playbooks/*.yml
```

## License

MIT License - See LICENSE file for details

## Authors

- Russell Johnston (rujohns2@cisco.com)
