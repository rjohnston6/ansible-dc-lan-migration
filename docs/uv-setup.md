# UV Setup and Usage Guide

This guide provides comprehensive information on using [uv](https://docs.astral.sh/uv/) for Python dependency management in this Ansible project.

## What is UV?

UV is a fast Python package and project manager written in Rust. It provides:
- **Speed**: 10-100x faster than pip
- **Reliability**: Reproducible installations via lockfiles
- **Simplicity**: Drop-in replacement for pip, pip-tools, and virtualenv
- **Compatibility**: Works with existing PyPI packages and requirements files

## Installation

### macOS/Linux

```bash
# Via install script
curl -LsSf https://astral.sh/uv/install.sh | sh

# Via Homebrew
brew install uv

# Via pip (not recommended for production)
pip install uv
```

### Windows

```powershell
# Via install script
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Via pip
pip install uv
```

## Project Setup

### Initial Setup

```bash
# Navigate to project directory
cd ansible-dc-lan-migration

# Sync dependencies from pyproject.toml and uv.lock
uv sync

# This creates .venv/ and installs all dependencies
```

### Activating the Environment

```bash
# macOS/Linux
source .venv/bin/activate

# Windows
.venv\Scripts\activate

# Or use uv run to execute commands without activation
uv run ansible-playbook playbooks/01-profile-existing-switches.yml
```

## Dependency Management

### Adding Dependencies

```bash
# Add a new package
uv add <package-name>

# Add with version constraint
uv add "ansible>=2.19.4"

# Add development dependency
uv add --dev pytest

# Add from Git
uv add git+https://github.com/user/repo.git
```

### Removing Dependencies

```bash
# Remove a package
uv remove <package-name>
```

### Updating Dependencies

```bash
# Update all packages
uv sync --upgrade

# Update specific package
uv add --upgrade <package-name>
```

### Viewing Dependencies

```bash
# Show installed packages
uv pip list

# Show dependency tree
uv pip tree

# Show outdated packages
uv pip list --outdated
```

## Lockfile Management

The `uv.lock` file ensures reproducible installations across environments.

```bash
# Generate/update lockfile
uv lock

# Sync from lockfile (installs exact versions)
uv sync

# Update lockfile with latest compatible versions
uv lock --upgrade
```

**Best practices**:
- Commit `uv.lock` to version control
- Run `uv lock` after modifying `pyproject.toml`
- Use `uv sync` for consistent environments across team

## Running Commands

### Direct Execution

```bash
# Run command in virtual environment (auto-activates)
uv run ansible-playbook playbooks/01-profile-existing-switches.yml

# Run Python script
uv run python scripts/my_script.py

# Run any command
uv run which ansible
```

### Inline Scripts

```bash
# Run inline Python code
uv run python -c "import ansible; print(ansible.__version__)"
```

## Python Version Management

UV can manage Python versions for you:

```bash
# Install specific Python version
uv python install 3.11

# Pin project to Python version (updates pyproject.toml)
uv python pin 3.11

# List available Python versions
uv python list

# List installed Python versions
uv python list --only-installed
```

## Virtual Environment Management

### Creating Environments

```bash
# Create environment (done automatically by uv sync)
uv venv

# Create with specific Python version
uv venv --python 3.11

# Create in custom location
uv venv /path/to/venv
```

### Environment Information

```bash
# Show environment information
uv venv --help

# Check Python executable
uv run which python
```

## Integration with Ansible

### Installing Ansible Collections

After setting up the environment, install Ansible collections:

```bash
# Activate environment
source .venv/bin/activate

# Install collections from requirements.yml
ansible-galaxy collection install -r requirements.yml

# Or use uv run
uv run ansible-galaxy collection install -r requirements.yml
```

### Running Playbooks

```bash
# With activated environment
ansible-playbook playbooks/01-profile-existing-switches.yml

# Without activation (using uv run)
uv run ansible-playbook playbooks/01-profile-existing-switches.yml --tags profile-switches
```

## CI/CD Integration

### GitHub Actions

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.11'

- name: Install uv
  run: curl -LsSf https://astral.sh/uv/install.sh | sh

- name: Install dependencies
  run: uv sync

- name: Run playbook
  run: uv run ansible-playbook playbooks/01-profile-existing-switches.yml --check
```

### GitLab CI

```yaml
test:
  image: python:3.11
  before_script:
    - curl -LsSf https://astral.sh/uv/install.sh | sh
    - uv sync
  script:
    - uv run ansible-playbook playbooks/01-profile-existing-switches.yml --check
```

## Performance Tips

### Caching

UV automatically caches downloaded packages in:
- **macOS/Linux**: `~/.cache/uv/`
- **Windows**: `%LOCALAPPDATA%\uv\cache\`

```bash
# Clear cache
uv cache clean

# Show cache info
uv cache dir
```

### Parallel Installation

UV installs packages in parallel by default. For slower connections:

```bash
# Limit concurrent downloads
UV_CONCURRENT_DOWNLOADS=4 uv sync
```

## Troubleshooting

### Common Issues

**Issue: Command not found after installation**
```bash
# Ensure UV is in PATH
export PATH="$HOME/.cargo/bin:$PATH"

# Or reload shell
source ~/.zshrc  # or ~/.bashrc
```

**Issue: SSL certificate errors**
```bash
# Use custom CA bundle
UV_CA_BUNDLE=/path/to/ca-bundle.crt uv sync
```

**Issue: Locked dependencies conflict**
```bash
# Regenerate lockfile
rm uv.lock
uv lock
```

**Issue: Different Python version**
```bash
# Install and pin specific version
uv python install 3.11
uv python pin 3.11
uv sync
```

### Debug Mode

```bash
# Enable verbose logging
UV_LOG=debug uv sync

# Show what UV is doing
uv sync --verbose
```

## Migration from pip/virtualenv

### From requirements.txt

```bash
# Create pyproject.toml from requirements.txt
uv add -r requirements.txt

# Or sync directly
uv pip sync requirements.txt
```

### From Pipenv

```bash
# Convert Pipfile to pyproject.toml
uv init
uv add $(cat Pipfile | grep -A 999 '\[packages\]' | grep -B 999 '\[dev-packages\]' | grep -v '\[')
```

### From Poetry

```bash
# UV can read pyproject.toml from Poetry projects
uv sync
```

## Best Practices

1. **Always commit lockfiles**: Ensures consistent environments
2. **Pin Python version**: Use `uv python pin` for reproducibility
3. **Use uv run in scripts**: Avoids forgetting to activate environments
4. **Regular updates**: Run `uv lock --upgrade` periodically
5. **Separate dev dependencies**: Use `uv add --dev` for testing tools
6. **Cache in CI**: Cache `~/.cache/uv/` for faster builds
7. **Version constraints**: Use specific versions in pyproject.toml for production

## Additional Resources

- [Official UV Documentation](https://docs.astral.sh/uv/)
- [UV GitHub Repository](https://github.com/astral-sh/uv)
- [UV vs pip Performance Benchmarks](https://docs.astral.sh/uv/#performance)
- [Python Packaging Guide](https://packaging.python.org/)

## Project-Specific Configuration

This project uses:
- **Python version**: 3.11+ (specified in pyproject.toml)
- **Key dependencies**: ansible, ansible-core
- **Lockfile**: `uv.lock` (committed to repository)
- **Virtual environment**: `.venv/` (excluded from git)

For questions or issues with UV in this project, consult the team or refer to the official documentation.
