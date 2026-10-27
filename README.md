# Ansible Role: Greengage DB

[![Tested with Molecule](https://img.shields.io/badge/tested%20with-Molecule-2ea44f?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/molecule/)
[![Linted with ansible-lint](https://img.shields.io/badge/linted%20with-ansible--lint-1a1918?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/lint/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Ansible Galaxy](https://img.shields.io/badge/halif.greengage-blue?logo=ansible&logoColor=white)](https://galaxy.ansible.com/ui/standalone/roles/halif/greengage/)

Ansible role for installing and configuring Greengage DB on Ubuntu 22.04 LTS.

The role installs Greengage DB, configures the `gpadmin` account and environment, prepares data directories, and optionally generates cluster initialization configuration files.

**Important:** cluster initialization is not performed automatically. The administrator remains responsible for reviewing the generated configuration and deciding when to initialize and start a cluster.

## Requirements

- Ubuntu 22.04 LTS (Jammy).
- Ansible 2.14 or later.
- Network access to the Greengage DB APT repository.
- Privileges required to install packages and manage users, groups, and directories.

Other operating systems and Ubuntu versions are not currently supported by this role.

## Role behavior

The role can perform the following tasks:

- Install required APT prerequisites.
- Configure the Greengage DB APT repository and signing key.
- Install the Greengage DB package.
- Create the `gpadmin` user and group when enabled.
- Configure the Greengage environment for `gpadmin`.
- Create the configured Greengage data directory.
- Validate cluster inventory and supported settings when cluster configuration is enabled.
- Generate `gpinitsystem_config` and the host file used for cluster initialization when explicitly enabled.

### Cluster configuration limitations

The current implementation supports the following cluster configuration:

- One master host.
- At least one segment host in the inventory.
- One primary segment without mirrors.
- The `mirrorless` replication mode.

Unsupported segment counts and replication modes are rejected by validation.

Cluster configuration generation is disabled by default. Generating initialization files does not initialize or start a cluster.

### What is not yet verified

The current CI suite does not perform a complete cluster initialization. In particular, it does not yet verify:

- Successful execution of `gpinitsystem` through completion.
- Successful startup of a fully initialized cluster.
- Cluster health using `gpstate`.
- SQL connectivity using `psql`.

The integration-preparation scenario runs `gpinitsystem` only far enough to verify that its preliminary checks reach the confirmation prompt. It then declines initialization. This checks preparation for initialization, not the successful creation of a working cluster.

## Installation

Install the role from Ansible Galaxy:

```bash
ansible-galaxy role install halif.greengage
```

Alternatively, install it directly from GitHub:

```bash
ansible-galaxy role install git+https://github.com/halif/ansible-role-greengage.git
```

## Basic usage

The following playbook installs and configures Greengage DB without enabling cluster configuration or initializing a cluster.

```yaml
---
- name: Install Greengage DB
  hosts: greengage_servers
  become: true

  roles:
    - role: halif.greengage
```

Replace `greengage_servers` with the appropriate inventory group in your environment.

## Cluster configuration

Cluster configuration generation requires both of the following variables:

- `greengage_cluster_enabled: true`
- `greengage_manage_cluster_initialization: true`

The inventory must contain exactly one host in the configured master group and at least one host in the configured segment group.

### Example inventory

```ini
[greengage_master]
mdw

[greengage_segments]
sdw1
```

The hostnames `mdw` and `sdw1` are examples. They must resolve correctly and refer to reachable hosts in your environment.

Install and configure Greengage DB on the intended hosts first. Then run the following playbook against the master to generate the cluster initialization configuration:

```yaml
---
- name: Generate Greengage cluster configuration
  hosts: greengage_master
  become: true

  vars:
    greengage_cluster_enabled: true
    greengage_manage_cluster_initialization: true

    greengage_master_group: greengage_master
    greengage_segments_group: greengage_segments

    greengage_segment_count: 1
    greengage_replication_mode: mirrorless

    greengage_master_hostname: mdw
    greengage_master_port: 5432

    greengage_master_data_directory: /data/greengage/master
    greengage_segment_data_directory: /data/greengage/primary

  roles:
    - role: halif.greengage
```

Review the generated `gpinitsystem_config` and host file before proceeding.

**This playbook only prepares the initialization configuration. It does not execute `gpinitsystem` or start the cluster.** Before initializing a real cluster, verify that the hosts satisfy Greengage's requirements for networking, hostname resolution, SSH, storage, and operating system configuration.

## Configuration variables

The main variables are defined in `defaults/main.yml`.

| Variable | Default | Description |
|---|---|---|
| `greengage_package_name` | `greengage6` | APT package name |
| `greengage_manage_gpadmin` | `true` | Manage the `gpadmin` user and group |
| `greengage_gpadmin_user` | `gpadmin` | System account name |
| `greengage_gpadmin_group` | `gpadmin` | System group name |
| `greengage_gpadmin_home` | `/home/gpadmin` | Home directory for `gpadmin` |
| `greengage_manage_environment` | `true` | Configure the Greengage environment |
| `greengage_environment_file` | `/home/gpadmin/.bashrc` | Shell configuration file |
| `greengage_path_script` | `/opt/greengagedb/greengage6/greengage_path.sh` | Greengage environment script |
| `greengage_data_directory` | `/data/greengage` | Base data directory |
| `greengage_manage_data_directory` | `true` | Manage the base data directory |
| `greengage_data_directory_mode` | `"0700"` | Permissions for the base data directory |
| `greengage_cluster_enabled` | `false` | Enable cluster inventory validation and configuration |
| `greengage_manage_cluster_initialization` | `false` | Generate cluster initialization configuration |
| `greengage_cluster_name` | `greengage` | Cluster name used by the configuration template |
| `greengage_master_group` | `greengage_master` | Inventory group for the master |
| `greengage_segments_group` | `greengage_segments` | Inventory group for segment hosts |
| `greengage_master_hostname` | `ansible_facts.hostname` | Master hostname |
| `greengage_master_port` | `5432` | Master port in the generated configuration |
| `greengage_master_data_directory` | `/data/greengage/master` | Master data directory |
| `greengage_segment_data_directory` | `/data/greengage/primary` | Primary segment data directory |
| `greengage_mirror_data_directory` | `/data/greengage/mirror` | Mirror data directory |
| `greengage_segment_count` | `1` | Supported primary segment count |
| `greengage_replication_mode` | `mirrorless` | Supported replication mode |

For the complete list of variables, see [`defaults/main.yml`](defaults/main.yml).

## Testing

The repository uses GitHub Actions, `ansible-lint`, `yamllint`, and Molecule.

The current test scenarios cover the following areas.

### Default scenario

The `default` scenario verifies the installation and preparation of Greengage DB, including:

- Package installation.
- Installation directory and executable files.
- The `gpadmin` account and home directory.
- Greengage environment setup.
- Required initialization and status utilities.
- The data directory and its ownership.
- Cluster initialization remaining disabled by default.

### Cluster configuration scenario

The `cluster_config` scenario verifies cluster inventory validation and generation of initialization configuration files with expected values, ownership, and permissions.

### Cluster integration-preparation scenario

The `cluster_integration` scenario prepares two temporary Docker containers representing the master (`mdw`) and a primary segment (`sdw1`).

It verifies:

- Greengage installation and configuration on the test hosts.
- SSH server preparation and passwordless SSH from the master to the segment.
- Preparation of the master and segment data directories.
- Generation and preliminary validation of the Greengage initialization configuration.
- That `gpinitsystem` reaches its confirmation prompt and that initialization is declined.
- Idempotence of the role during the Molecule test sequence.

This scenario does **not** complete cluster initialization. It does not verify a running cluster, `gpstate`, or SQL queries.

### Running tests locally

The tests can be run locally if the required Ansible, Molecule, Docker, and collection dependencies are installed:

```bash
molecule -s default test
molecule -s cluster_config test
molecule -s cluster_integration test
```

The GitHub Actions workflow runs the configured linting and test jobs.

## Supported platform

- Ubuntu 22.04 LTS (Jammy).

## License

MIT. See [LICENSE](LICENSE).
