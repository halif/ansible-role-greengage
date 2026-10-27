# Ansible Role: Greengage DB

[![Tested with Molecule](https://img.shields.io/badge/tested%20with-Molecule-2ea44f?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/molecule/)
[![Linted with ansible-lint](https://img.shields.io/badge/linted%20with-ansible--lint-1a1918?logo=ansible&logoColor=white)](https://ansible.readthedocs.io/projects/lint/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Ansible role for installing and configuring Greengage DB on Ubuntu 22.04 LTS.

The role prepares the operating system, installs Greengage DB, configures the `gpadmin` account and environment, and can generate cluster initialization configuration files.

**Important:** this role does not automatically initialize or start a database cluster. Cluster initialization is an explicit operational step performed by the administrator.

## Status and scope

This is an initial release focused on installation and cluster preparation.

| Feature | Status |
|---|---|
| Install Greengage DB package | Covered by Molecule tests |
| Configure APT repository and prerequisites | Covered by role tests |
| Create the `gpadmin` user and group | Covered by Molecule tests |
| Configure the Greengage environment | Covered by Molecule tests |
| Create the Greengage data directory | Covered by Molecule tests |
| Generate cluster initialization configuration | Covered by Molecule tests |
| Validate supported cluster settings | Covered by validation tests |
| Prepare two test nodes and configure passwordless SSH | Covered by the CI integration-preparation scenario |
| Initialize a real Greengage cluster | Not yet verified by CI |
| Verify cluster health using `gpstate` | Not yet verified by CI |
| Verify SQL connectivity using `psql` | Not yet verified by CI |

Passing CI confirms the behaviors listed as covered above. It does not yet establish that a fully initialized, operational Greengage cluster can be created successfully.

## Requirements

- Ubuntu 22.04 LTS (Jammy).
- Ansible 2.14 or later.
- Network access to the Greengage DB APT repository.
- Sufficient privileges to install system packages and manage users, groups, files, and directories.

Other Ubuntu versions and operating systems are not currently supported by this role.

## What the role does

By default, the role:

- Installs required APT prerequisites.
- Configures the Greengage DB APT repository and signing key.
- Installs the configured Greengage DB package.
- Creates the `gpadmin` group and user when enabled.
- Configures the Greengage environment for `gpadmin`.
- Creates the configured Greengage data directory.
- Leaves cluster initialization disabled.

When explicitly enabled, the role can also generate `gpinitsystem_config` and the host file used for cluster initialization.

### What the role does not do

The role does not automatically:

- Initialize a Greengage cluster with `gpinitsystem`.
- Start a newly initialized cluster.
- Create application databases or users.
- Configure mirrored segments.
- Guarantee that the resulting cluster is operational or accepts SQL connections.

These steps must be performed separately by the administrator.

## Installation

After the role is available on Ansible Galaxy:

```bash
ansible-galaxy role install halif.greengage
```

Alternatively, install it directly from GitHub:

```bash
ansible-galaxy role install git+https://github.com/halif/ansible-role-greengage.git
```

## Basic usage

The following playbook installs and configures Greengage DB without initializing a cluster:

```yaml
---
- name: Install Greengage DB
  hosts: greengage_servers
  become: true

  roles:
    - role: halif.greengage
```

Use an inventory containing the intended Greengage servers.

## Cluster configuration

Cluster configuration generation is disabled by default.

To enable it, both of the following variables must be enabled:

- `greengage_cluster_enabled: true`
- `greengage_manage_cluster_initialization: true`

The inventory must contain the configured master group with exactly one host and a segment group with at least one host.

The current supported configuration is limited to one primary segment without mirrors. Unsupported segment counts and replication modes are rejected by validation.

### Example inventory

```ini
[greengage_master]
mdw

[greengage_segments]
sdw1
```

The names `mdw` and `sdw1` are examples. They must correspond to hosts that are resolvable and reachable in your environment.

### Example playbook

```yaml
---
- name: Prepare Greengage cluster configuration
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

After running the playbook, review the generated initialization configuration and host file before using them.

**Generating these files does not initialize or start the cluster.** Before manual initialization, verify that the master and segment hosts meet Greengage's system, networking, hostname, SSH, and storage requirements.

## Configuration variables

The defaults are defined in `defaults/main.yml`.

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
| `greengage_cluster_enabled` | `false` | Enable cluster inventory validation and cluster configuration |
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

For the complete list of variables and APT settings, see [`defaults/main.yml`](defaults/main.yml).

## Testing

The repository uses GitHub Actions, `ansible-lint`, `yamllint`, and Molecule.

## Supported platform

- Ubuntu 22.04 LTS (Jammy).

## License

MIT. See [LICENSE](LICENSE).
