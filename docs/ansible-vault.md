# Ansible Vault Guide

Comprehensive guide for managing sensitive credentials and secrets using Ansible Vault in this project.

## Overview

Ansible Vault encrypts sensitive data at rest, allowing you to commit encrypted credentials to version control securely. This project uses Vault to protect:
- Switch and NDFC credentials
- API tokens
- SSH keys
- Certificate passwords
- Any other sensitive configuration

## Quick Start

### Initial Setup

```bash
# Create vault password file (add to .gitignore)
echo "your-secure-vault-password" > .vault_pass
chmod 600 .vault_pass

# Configure ansible.cfg to use password file (already configured)
# vault_password_file = .vault_pass

# Edit encrypted vault file
ansible-vault edit inventory/group_vars/all/vault.yml
```

### Vault File Structure

The project uses these vault files:

```
inventory/
├── group_vars/
│   └── all/
│       ├── common.yml      # Non-sensitive variables
│       └── vault.yml        # ENCRYPTED: Credentials and secrets
└── host_vars/
    └── nexus_dashboard/
        └── vault.yml        # ENCRYPTED: Host-specific secrets (if needed)
```

## Vault Operations

### Creating Encrypted Files

```bash
# Create new encrypted file
ansible-vault create inventory/group_vars/all/vault.yml

# Create with specific password file
ansible-vault create --vault-password-file .vault_pass inventory/group_vars/all/vault.yml
```

### Editing Encrypted Files

```bash
# Edit vault file (decrypts in $EDITOR, re-encrypts on save)
ansible-vault edit inventory/group_vars/all/vault.yml

# Edit with specific password file
ansible-vault edit --vault-password-file .vault_pass inventory/group_vars/all/vault.yml

# Edit with prompted password
ansible-vault edit --ask-vault-pass inventory/group_vars/all/vault.yml
```

### Viewing Encrypted Files

```bash
# View contents without editing
ansible-vault view inventory/group_vars/all/vault.yml

# View with specific password file
ansible-vault view --vault-password-file .vault_pass inventory/group_vars/all/vault.yml
```

### Encrypting Existing Files

```bash
# Encrypt existing plain-text file
ansible-vault encrypt inventory/group_vars/all/vault.yml

# Encrypt multiple files
ansible-vault encrypt inventory/group_vars/*/vault.yml

# Encrypt with specific vault ID
ansible-vault encrypt --vault-id prod@.vault_pass inventory/group_vars/all/vault.yml
```

### Decrypting Files

```bash
# Decrypt to plain text (use with caution!)
ansible-vault decrypt inventory/group_vars/all/vault.yml

# Decrypt to stdout (doesn't modify file)
ansible-vault decrypt --output=- inventory/group_vars/all/vault.yml
```

### Rekeying (Changing Password)

```bash
# Change vault password
ansible-vault rekey inventory/group_vars/all/vault.yml

# Rekey with new password file
ansible-vault rekey --new-vault-password-file .vault_pass_new inventory/group_vars/all/vault.yml

# Rekey multiple files
ansible-vault rekey inventory/group_vars/*/vault.yml
```

## Running Playbooks with Vault

### Using Password File (Recommended)

The `ansible.cfg` in this project is pre-configured:

```ini
[defaults]
vault_password_file = .vault_pass
```

Run playbooks normally:
```bash
ansible-playbook playbooks/01-profile-existing-switches.yml
```

### Using Command-Line Options

```bash
# Prompt for password
ansible-playbook playbooks/01-profile-existing-switches.yml --ask-vault-pass

# Specify password file
ansible-playbook playbooks/01-profile-existing-switches.yml --vault-password-file .vault_pass

# Use vault ID
ansible-playbook playbooks/01-profile-existing-switches.yml --vault-id prod@.vault_pass
```

### Environment Variable

```bash
# Set password via environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
ansible-playbook playbooks/01-profile-existing-switches.yml
```

## Vault File Contents

### inventory/group_vars/all/vault.yml

This file contains credentials used across all hosts:

```yaml
---
# NX-OS Switch Credentials
vault_nxos_username: admin
vault_nxos_password: "SecurePassword123!"

# NDFC/DCNM Credentials
vault_ndfc_username: admin
vault_ndfc_password: "NDFCSecurePass456!"

# Optional: SSH Key Passphrase
vault_ssh_key_passphrase: "SSHKeyPass789!"

# Optional: API Tokens
vault_api_token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Optional: SNMP Community Strings
vault_snmp_ro_community: "public123"
vault_snmp_rw_community: "private456"
```

### Referencing Vault Variables

In `inventory/group_vars/all/common.yml` (unencrypted):

```yaml
---
# Reference vault variables
ansible_user: "{{ vault_nxos_username }}"
ansible_password: "{{ vault_nxos_password }}"
ansible_network_os: nxos
ansible_connection: network_cli

# NDFC credentials
ndfc_login:
  username: "{{ vault_ndfc_username }}"
  password: "{{ vault_ndfc_password }}"
```

## Multiple Vault IDs

For complex environments with different security domains:

### Configuration

Create multiple password files:
```bash
echo "dev-password" > .vault_pass_dev
echo "prod-password" > .vault_pass_prod
chmod 600 .vault_pass_*
```

Update `ansible.cfg`:
```ini
[defaults]
vault_identity_list = dev@.vault_pass_dev, prod@.vault_pass_prod
```

### Usage

```bash
# Encrypt with specific vault ID
ansible-vault encrypt --vault-id dev@.vault_pass_dev inventory/group_vars/dev/vault.yml
ansible-vault encrypt --vault-id prod@.vault_pass_prod inventory/group_vars/prod/vault.yml

# Edit specific vault
ansible-vault edit --vault-id prod@.vault_pass_prod inventory/group_vars/prod/vault.yml

# Run playbook with multiple vaults
ansible-playbook playbooks/01-profile-existing-switches.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod
```

### Vault File with ID Labels

```yaml
---
# Encrypted with vault ID "prod"
vault_nxos_username: !vault |
          $ANSIBLE_VAULT;1.2;AES256;prod
          66386439653966636339613...
```

## Encrypting Individual Variables

For mixed encrypted/plain-text files:

```bash
# Encrypt a string
ansible-vault encrypt_string 'SecurePassword123!' --name 'vault_nxos_password'
```

Output:
```yaml
vault_nxos_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653966636339613961336164323037373834353866643863353630396363346262353033
          ...
```

Use in variable files:
```yaml
---
# inventory/group_vars/all/vault.yml
vault_nxos_username: admin  # Plain text
vault_nxos_password: !vault |  # Encrypted
          $ANSIBLE_VAULT;1.1;AES256
          66386439653966636339613961336164323037373834353866643863353630396363346262353033
          ...
```

## Security Best Practices

### Password File Security

```bash
# Set restrictive permissions
chmod 600 .vault_pass

# Add to .gitignore (already configured)
echo ".vault_pass*" >> .gitignore

# Never commit password files
git add .vault_pass  # ❌ DON'T DO THIS
```

### Password Management

1. **Use strong passwords**: Minimum 20 characters, mixed case, numbers, symbols
2. **Unique per environment**: Dev, staging, prod should have different passwords
3. **Store securely**: Use password manager (1Password, LastPass, Bitwarden)
4. **Share securely**: Use secure channels (encrypted email, password manager sharing)
5. **Rotate regularly**: Change vault passwords periodically

### Password Generation

```bash
# Generate secure random password
openssl rand -base64 32

# Or using Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# Save to vault password file
openssl rand -base64 32 > .vault_pass
chmod 600 .vault_pass
```

### CI/CD Integration

#### GitHub Actions

```yaml
- name: Run Ansible Playbook
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
  run: |
    echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    chmod 600 .vault_pass
    ansible-playbook playbooks/01-profile-existing-switches.yml
    rm .vault_pass
```

Store `ANSIBLE_VAULT_PASSWORD` in GitHub repository secrets.

#### GitLab CI

```yaml
variables:
  ANSIBLE_VAULT_PASSWORD: $VAULT_PASSWORD

before_script:
  - echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
  - chmod 600 .vault_pass

after_script:
  - rm -f .vault_pass
```

Store `VAULT_PASSWORD` in GitLab CI/CD variables (masked).

### Audit and Compliance

```bash
# Check which files are encrypted
find inventory -name "vault.yml" -exec file {} \;

# Verify encryption
ansible-vault view inventory/group_vars/all/vault.yml > /dev/null && echo "Valid" || echo "Invalid"

# Show vault ID (if used)
grep -A1 "ANSIBLE_VAULT" inventory/group_vars/all/vault.yml | head -1
```

## Troubleshooting

### Common Errors

**Error: "Decryption failed"**
```bash
# Verify password file exists and is correct
cat .vault_pass

# Try manual decryption
ansible-vault view --vault-password-file .vault_pass inventory/group_vars/all/vault.yml
```

**Error: "vault password file not found"**
```bash
# Check ansible.cfg configuration
grep vault_password_file ansible.cfg

# Create password file
echo "your-password" > .vault_pass
chmod 600 .vault_pass
```

**Error: "Attempting to decrypt but no vault secrets found"**
```bash
# File is not encrypted, encrypt it
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

**Error: "conflicting action statements: decrypt, encrypt"**
```bash
# File is already encrypted/decrypted
# Use 'view' to check status
ansible-vault view inventory/group_vars/all/vault.yml
```

### Recovery

**Lost password file**:
1. If you have backup of password: Restore `.vault_pass` file
2. If you have plain-text backup: Re-encrypt with new password
3. If completely lost: Re-create vault.yml with new credentials (requires updating devices)

**Corrupted vault file**:
```bash
# Try viewing to verify corruption
ansible-vault view inventory/group_vars/all/vault.yml

# If corrupted, restore from backup
git checkout HEAD^ -- inventory/group_vars/all/vault.yml

# Re-encrypt if needed
ansible-vault rekey inventory/group_vars/all/vault.yml
```

## Team Collaboration

### Onboarding New Team Members

1. **Securely share vault password**:
   - Use encrypted email
   - Use password manager sharing
   - Use secure messaging (Signal, encrypted Slack)
   - Never share via plain email or unencrypted chat

2. **Setup instructions**:
   ```bash
   # Create password file
   echo "provided-vault-password" > .vault_pass
   chmod 600 .vault_pass
   
   # Verify access
   ansible-vault view inventory/group_vars/all/vault.yml
   ```

3. **Document access**: Maintain list of who has vault access

### Password Rotation Procedure

```bash
# 1. Backup current vault files
cp inventory/group_vars/all/vault.yml inventory/group_vars/all/vault.yml.backup

# 2. Create new password file
openssl rand -base64 32 > .vault_pass_new
chmod 600 .vault_pass_new

# 3. Rekey vault files
ansible-vault rekey --new-vault-password-file .vault_pass_new inventory/group_vars/all/vault.yml

# 4. Update ansible.cfg
sed -i 's/.vault_pass/.vault_pass_new/' ansible.cfg

# 5. Test
ansible-playbook playbooks/01-profile-existing-switches.yml --check

# 6. Replace old password file
mv .vault_pass_new .vault_pass

# 7. Notify team of password change
```

## Additional Resources

- [Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Ansible Security Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html#keep-vaulted-variables-safely-visible)
- [Vault and Version Control](https://docs.ansible.com/ansible/latest/user_guide/vault.html#storing-passwords-in-version-control)

## Project-Specific Usage

This project's vault configuration:
- **Password file**: `.vault_pass` (gitignored)
- **Vault files**: `inventory/group_vars/all/vault.yml`
- **ansible.cfg**: Pre-configured with `vault_password_file = .vault_pass`
- **Required secrets**:
  - `vault_nxos_username` / `vault_nxos_password`: NX-OS switch credentials
  - `vault_ndfc_username` / `vault_ndfc_password`: NDFC/DCNM credentials

For questions about vault usage in this project, consult your team lead or security officer.
