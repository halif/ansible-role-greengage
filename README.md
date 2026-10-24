# Ansible Role: Greengage DB

[![Tested with Molecule](https://img.shields.io/badge/tested%20with-Molecule-2ea44f?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/molecule/)
[![Linted with ansible-lint](https://img.shields.io/badge/linted%20with-ansible--lint-1a1918?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/lint/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Ansible Galaxy](https://img.shields.io/badge/halif.greengage-blue?logo=ansible&logoColor=white)](https://galaxy.ansible.com/ui/standalone/roles/halif/greengage/)

Ansible role for installing and configuring Greengage DB on Ubuntu 22.04.

## Requirements

- Ubuntu 22.04 LTS (Jammy)
- Ansible 2.14 or later
- APT repository access to `greengagedb.org`

## Role behavior

The role performs the following tasks:

- Installs the required APT prerequisites.
- Configures the Greengage DB APT repository and signing key.
- Installs the configured Greengage DB package.
- Creates the `gpadmin` group and user when enabled.
- Configures the Greengage environment for `gpadmin`.
- Creates the configured Greengage data directories.
- Optionally generates a `gpinitsystem_config` file for cluster initialization.

**Cluster configuration limitations:**

- The current cluster configuration supports one primary segment without mirrors.
- Unsupported segment counts or replication modes are rejected by validation.
- Cluster initialization is disabled by default.
- Generating `gpinitsystem_config` does not initialize or start a cluster.
- Actual cluster initialization, mirror configuration, and SQL connectivity are outside the current test coverage.

## Installation

Install the role from Ansible Galaxy:

```bash
ansible-galaxy role install halif.greengage
```

## Example playbook

```yaml
---
- name: Install Greengage DB
  hosts: greengage_servers
  become: true

  roles:
    - role: halif.greengage
```

## Configuration

The main variables are defined in `defaults/main.yml`.

| Variable | Default | Description |
|---|---|---|
| `greengage_package_name` | `greengage6` | APT package name |
| `greengage_manage_gpadmin` | `true` | Manage the `gpadmin` user and group |
| `greengage_manage_environment` | `true` | Configure the Greengage environment |
| `greengage_data_directory` | `/data/greengage` | Base data directory |
| `greengage_manage_data_directory` | `true` | Create Greengage data directories |
| `greengage_manage_cluster_initialization` | `false` | Generate the cluster initialization configuration |

To generate the cluster initialization configuration:

```yaml
---
- name: Configure Greengage initialization
  hosts: greengage_servers
  become: true

  vars:
    greengage_manage_cluster_initialization: true

  roles:
    - role: halif.greengage
```

Review the generated configuration and verify the intended topology before initializing a cluster manually.

## Testing

The repository includes Molecule scenarios for:

- Default installation and configuration.
- Cluster initialization configuration generation.

Run the tests locally with Molecule installed:

```bash
molecule -s default test
molecule -s cluster_config test
```

The CI workflow also runs linting and these Molecule scenarios.

## Supported platform

- Ubuntu 22.04 LTS (Jammy)

## License

MIT
