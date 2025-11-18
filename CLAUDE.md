# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible role (`jonaspammer.mediawiki`) for automated MediaWiki installation and configuration. The role:

- Downloads and extracts MediaWiki from wikimedia.org releases
- Installs required system packages (ImageMagick, Inkscape, PHP extensions)
- Manages extensions and skins via Git or Composer
- Optionally generates LocalSettings.php from a Jinja2 template

**Important Limitations:**

- Does NOT install/configure SQL Server
- Does NOT install/configure Web Server
- Does NOT execute MediaWiki maintenance scripts (like `update.php`)
- Role is modular - expects these to be handled by other roles or manual setup

## Development Commands

### Testing

```bash
# Run all tests (all Ansible versions)
tox

# Test specific distribution
MOLECULE_DISTRO=ubuntu2204 tox
MOLECULE_DISTRO=debian12 tox
MOLECULE_DISTRO=debian11 tox
MOLECULE_DISTRO=ubuntu2004 tox

# Debug with persistent container
MOLECULE_DESTROY=never MOLECULE_DISTRO=ubuntu2204 tox

# Show installed package versions (useful for debugging)
CI=true tox

# Run pre-commit hooks manually
pre-commit run --all-files

# Install pre-commit hooks for local development
pre-commit install
```

### Molecule Container Debugging

When a molecule test fails:

```bash
# 1. Find container ID
docker ps

# 2. Enter container
docker exec -it <container_id> /bin/bash

# 3. Debug artifacts available at:
# /var/tmp/vars.yml (host variables)
# /var/tmp/environment.yml (environment variables)

# 4. Cleanup after debugging
docker stop <container_id>
docker container rm <container_id>
# or
docker container prune
```

### Development Setup

```bash
# Create virtual environment (optional but recommended)
python3 -m venv venv
source venv/bin/activate

# Install development dependencies
python3 -m pip install -r requirements-dev.txt

# Install pre-commit hooks
pre-commit install
```

## Architecture & Code Structure

### Task Execution Flow

```
tasks/main.yml (entry point)
├── assert.yml (validate variables)
├── Create destination directory
├── Download MediaWiki archive
├── Extract archive
├── Install system packages
├── Install PHP packages
├── [tag: mediawiki::extensions]
│   ├── install_extension1.yml (loop wrapper)
│   └── install_extension.yml (per-extension logic)
│       ├── Git clone OR Composer require
│       ├── Run composer install in extension dir
│       └── Install system dependencies
├── [tag: mediawiki::skins]
│   └── install_skin.yml (per-skin logic)
│       └── Git clone skin repository
└── Template LocalSettings.php (if mediawiki_create_localsettings is true)
```

### Key Files

**Role Structure:**

- `defaults/main.yml`: 900+ configurable variables (MediaWiki settings, packages, extensions)
- `tasks/main.yml`: Main task orchestration
- `tasks/assert.yml`: Variable validation
- `tasks/install_extension.yml`: Extension installation logic (Git + Composer)
- `tasks/install_skin.yml`: Skin installation logic (Git)
- `handlers/main.yml`: Web server restart handler
- `templates/LocalSettings.php.j2`: 41KB MediaWiki configuration template
- `vars/main.yml`: Platform-specific variables (base paths by distribution)

**Testing:**

- `molecule/default/molecule.yml`: Molecule configuration (Docker driver)
- `molecule/default/converge.yml`: Test playbook
- `molecule/default/verify.yml`: Post-convergence verification
- `molecule/resources/prepare.yml`: Environment setup (installs dependency roles)
- `tox.ini`: Test matrix configuration (4 Ansible versions × 4 distributions)

**Linting & Quality:**

- `.ansible-lint`: Ansible linting rules (skips `name` rule per development guidelines)
- `.yamllint`: YAML linting configuration (140 char line length)
- `.pre-commit-config.yaml`: Git hooks (commitlint, prettier, yamllint, black, flake8)
- `commitlint.config.js`: Conventional commits enforcement

### Variable Architecture

Variables follow Ansible precedence with distribution-specific overrides:

```
defaults/main.yml → vars/main.yml (platform-specific) → host_vars/group_vars
```

Pattern for distribution-specific variables:

```yaml
_variable_name:
  default: [...]
  RedHat: [...]
  Alpine: [...]

variable_name: "{{ _variable_name[ansible_distribution] | default(_variable_name['default']) }}"
```

### Extension Installation Methods

Extensions support two installation methods:

1. **Git (default)**: Clones from Wikimedia gerrit or custom git URL
   - Default URL: `https://github.com/wikimedia/mediawiki-extensions-{name}.git`
   - Default branch: `REL{major}_{minor}` (e.g., `REL1_39`)
   - Can optionally run `composer install` in extension directory

2. **Composer**: Uses `composer require` in MediaWiki root
   - For packages like `mediawiki/semantic-media-wiki`
   - Requires `composer_name` variable

## Development Guidelines

### CookieCutter Template Sync

This project is templated from [`cookiecutter-ansible-role`](https://github.com/JonasPammer/cookiecutter-ansible-role) and kept in sync using `cruft`.

**Before making changes:**

1. Check if the change applies to the template itself
2. If yes, make the change in cookiecutter-ansible-role first
3. Then sync this repo using `cruft update`

**Files managed by template:**

- GitHub Actions workflows (`.github/workflows/`)
- Documentation structure (README.adoc, CONTRIBUTING.adoc, DEVELOPMENT.adoc)
- Development tooling configs (.pre-commit-config.yaml, tox.ini, .editorconfig)
- DevContainer configuration (`.devcontainer/`)

### Versioning & Releases

**CRITICAL**: Version tags must NOT start with `v`. Use `1.0.0`, not `v1.0.0`.

- Uses Semantic Versioning (enforced via Conventional Commits)
- GitHub Actions automatically publishes to Ansible Galaxy on tag push
- Changelog is manually created in GitHub Releases
- Only core contributors need to follow conventional commits strictly (PRs are squash-merged)

### Conventional Commits

Format: `<type>(<scope>): <description>`

Common types: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `ci`

Example: `feat(extensions): add support for composer-based extension installation`

Enforced via commitlint pre-commit hook and CI.

### Testing Requirements

**Supported Platforms:**

- Debian 11 (bullseye), 12 (bookworm)
- Ubuntu 20.04 LTS (focal), 22.04 LTS (jammy)

**Supported Ansible Versions:**

- Ansible 6 (core 2.13)
- Ansible 7 (core 2.14)
- Ansible 8 (core 2.15)
- Ansible 9 (core 2.16)

All changes must pass:

1. yamllint (YAML syntax)
2. ansible-lint (best practices)
3. Molecule tests (integration tests across matrix)

### Making Changes to Role

When modifying role behavior:

1. **Add/modify tasks**: Edit files in `tasks/`
2. **Add variables**: Update `defaults/main.yml` AND document in README.adoc
3. **Test locally**: Run `tox` or target specific distro
4. **Update tests**: Modify `molecule/default/converge.yml` or `verify.yml` if needed
5. **Check CI**: GitHub Actions will run full test matrix

### Working with Tags

Two main task tags allow selective execution:

```bash
# Skip extensions
ansible-playbook playbook.yml --skip-tags mediawiki::extensions

# Only install skins
ansible-playbook playbook.yml --tags mediawiki::skins
```

When adding new features, consider if they should be tagged for selective execution.

## Important Technical Details

### Required Variables

**Must be set by user:**

- `php_version`: PHP version string (e.g., "7.4")
  - CRITICAL: Must satisfy [MediaWiki Compatibility Matrix](https://www.mediawiki.org/wiki/Compatibility)
  - Variable name intentionally matches `geerlingguy.php-versions` role
  - Inconsistent PHP package versions can break system packages

**Typically overridden:**

- `mediawiki_linux_username`: Owner of MediaWiki files
- `mediawiki_linux_group`: Group owner of MediaWiki files
- `mediawiki_destination`: Installation directory (default: distro-specific `/var/www/html/mediawiki`)

### Role Dependencies

**Hard dependencies (must be installed before this role):**

- `community.general` collection (for `composer` module)
- Composer installed on target host
- Git installed on target host
- Unzip recommended for Composer

**Soft dependencies (for complete MediaWiki setup):**

- `geerlingguy.php`
- `geerlingguy.php-mysql`
- `geerlingguy.php-versions`
- `geerlingguy.composer`
- `geerlingguy.mysql`
- `jonaspammer.apache2` (or another web server role)

See `molecule/resources/prepare.yml` for complete setup example.

### Extension Configuration Structure

Extensions are organized as a dictionary of lists:

```yaml
mediawiki_extensions:
  category_name:  # Arbitrary category for organization
    - name: "ExtensionName"
      load: true  # Used in LocalSettings.php template
      gather_type: "git"  # or "composer"

      # Git-specific options
      git_url: "https://github.com/..."
      git_version: "REL1_39"
      git_run_composer_install: true  # or "always"

      # Composer-specific options
      composer_name: "mediawiki/extension-name"
      composer_version: "~3.0"
      composer_install_pre_config_actions:
        - "--no-plugins allow-plugins.composer/installers true"

      # System dependencies
      _system_package_dependencies_:
        - package-name
```

### LocalSettings.php Template

The template (`templates/LocalSettings.php.j2`) is 41KB and covers:

- Database configuration
- File uploads and paths
- Security settings
- Caching configuration
- Extension loading (loops through `mediawiki_extensions`)
- Skin loading (loops through `mediawiki_skins`)

Set `mediawiki_create_localsettings: true` to generate it automatically.

## CI/CD Pipeline

### GitHub Actions Workflows

**ci.yml** (Main CI):

- Triggers: PR, push to master, weekly schedule (Sunday 05:00 UTC)
- Jobs: yamllint + molecule tests across 4 distros × 4 Ansible versions (max 4 parallel)
- Artifacts: Debug info uploaded as CI artifacts

**release-to-galaxy.yml**:

- Triggers: Tag push (without 'v' prefix)
- Publishes role to Ansible Galaxy

**gh-pages.yml**:

- Generates documentation from README.adoc
- Publishes to GitHub Pages

**label-pr-sizes.yml**:

- Automatically labels PRs by size

**issue-label-manager.yml**:

- Manages GitHub issue labels

### Renovate

Automated dependency updates via `.github/renovate.json5`:

- GitHub Actions versions
- Python packages in requirements-dev.txt
- pre-commit hooks

## DevContainer Support

VS Code DevContainer available with:

- Ubuntu 22.04 base
- Docker-in-Docker (allows running Molecule inside container)
- Python 3.12
- Automatic setup via `.devcontainer/postCreateCommand.sh`:
  - Git configuration
  - pip install
  - pre-commit installation

To use: Open folder in VS Code → "Reopen in Container"

## Common Pitfalls

1. **PHP Version Mismatch**: Ensure `php_version` satisfies MediaWiki compatibility and is consistent across all roles
2. **Version Tag Format**: Never prefix versions with 'v' (Galaxy import will fail)
3. **Extension Dependencies**: Extensions may require specific system packages (use `_system_package_dependencies_`)
4. **Composer Plugins**: Some extensions require allowing plugins via `composer_install_pre_config_actions`
5. **File Permissions**: Role manages permissions, but user/group must exist beforehand
6. **Maintenance Scripts**: After installing extensions, manually run `php maintenance/update.php`
7. **Template Changes**: Changes to cookiecutter template should be made upstream, not directly in this repo

## Reference Links

- [Ansible Role Development Guidelines](https://github.com/JonasPammer/cookiecutter-ansible-role/blob/master/ROLE_DEVELOPMENT_GUIDELINES.adoc)
- [MediaWiki Compatibility Matrix](https://www.mediawiki.org/wiki/Compatibility)
- [MediaWiki Releases](https://releases.wikimedia.org/mediawiki/)
- [Molecule Documentation](https://molecule.readthedocs.io/)
- [Conventional Commits Specification](https://www.conventionalcommits.org/)
