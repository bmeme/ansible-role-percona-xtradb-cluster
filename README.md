# Ansible Role: Percona XtraDB Cluster
![Workflow](https://github.com/bmeme/ansible-role-percona-xtradb-cluster/actions/workflows/ci.yml/badge.svg?branch=main)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://GitHub.com/Naereen/StrapDown.js/graphs/commit-activity)
[![MIT license](https://img.shields.io/badge/License-MIT-blue.svg)](https://lbesson.mit-license.org/)

This Ansible role installs and configures Percona XtraDB Cluster 8.x on Enterprise Linux 9 systems (RHEL, AlmaLinux, Rocky, etc.).
It automates the bootstrap and synchronization of multi-master clusters, optionally including a Galera Arbitrator (garbd) for 2-node topologies.

## Versin matrix 
| Role Series | Percona Server version | Supported OS | Python | Ansible | Status |
|-------------|------------------------|--------------|--------|---------|--------|
|1.x|5.7 |EL 7 / 8 |3.6–3.9|<2.15|_Legacy_ (EOL for Percona 5.7)|
|2.x|8.x |EL 9 |3.10 / 3.11|>2.15|_Current / Maintained_

> Version 2.x drops support for Percona XtraDB Cluster 5.7 and older platforms. EL 9 is the only supported targets. Python 3.10 / 3.11 are required on the controller.

## Currently not supported

* **node scaling**: you cannot re-run this role on a pre-installed architecture to add one or more nodes.
* **pmm**: You can add [PMM Client](https://docs.percona.com/percona-monitoring-and-management/setting-up/client/index.html) manually to your architecture.

## Requirements
* Root privileges (use `become: true`)
* SSH connectivity between nodes
* Static IP addresses (required by Galera)

```yaml
- hosts: database
  become: true
  roles:
    - role: bmeme.percona_xtradb_cluster
```
You must define per-node variables inside `host_vars/`.
See `molecule/default/group_vars` and `molecule/default/host_vars` for working examples.

## Installation
This is an Ansible role distributed using Ansible Galaxy. In order to install this role you can use the following command.

`$ ansible-galaxy install bmeme.percona_xtradb_cluster `

## Update
If you want to update the role, you need to pass --force parameter when installing. Please, check the following command:

`$ ansible-galaxy install --force bmeme.percona_xtradb_cluster `

## Clustering model
When two or more nodes are defined, a multi-master Galera cluster is formed.
A 2-node cluster requires an arbitrator node (`garbd`) to maintain quorum and prevent split-brain conditions.

## Roles variables

### Cluster node configuration

```yaml
percona_cluster:
  role: master
  bootstrapper: false
  server_id: '1'
  ip_address: "{{ ansible_default_ipv4.address }}"
```

| Key | Type | Description |
|-----|------|-------------|
| `role` | string | Node type: `master` or `arbiter` |
| `bootstrapper` | bool | Whether this node bootstraps the cluster (only **one node** should be `true`) |
| `server_id` | int | Unique server ID (must differ across nodes) |
| `ip_address` | string | Node IP address. Use `{{ ansible_default_ipv4.address }}` for automatic detection

### Cluster identity

```yaml
percona_cluster_name: "galera-cluster"
```

### Authentication

```yaml
percona_root_user: "root"
percona_root_password: "S3cr3tS/$"
percona_sst_user: "sstuser"
percona_sst_password: "S3cr3tS/$"
```

where:
* `root`: administrative access
* `sstuser`: used for **SST replication** between nodes

### Databases and users

```yaml
percona_databases:
  - myappdb

percona_users:
  - name: myapp_user
    password: "MyP@ssw0rd!"
    privs: "myappdb.*:ALL,GRANT"
    host: "%"
```

These are created **only on the bootstrap node**.

### Base MySQL settings

```yaml
mysql_datadir: "/var/lib/mysql"
mysql_bind_address: "{{ percona_cluster.ip_address }}"
mysql_symbolic_links: 0
mysql_cluster_pxc_strict_mode: ENFORCING
```

### Extended configuration (inline block)

```yaml
mysql_cluster_configuration: |-
  max_allowed_packet=64M
  innodb_buffer_pool_size=512M
```

The block is appended into `/etc/my.cnf.d/20-mysqld-core.cnf`.

### SSL/TLS for replication

If `percona_ssl_enabled: true`, the role:
* copies SSL certificates (`ca.pem`, `server-cert.pem`, `server-key.pem`) from the bootstrapper node,
* distributes them across all nodes and configures Galera’s `socket.ssl_*` options,
* configures `garbd` to use the same certificates.

Example:

```yaml
percona_ssl_enabled: true
```

## Supported platforms
* EL 9 (RHEL, Almalinux, Rocky)

## Example inventory structure

```css
inventory/
├── group_vars/
│   └── all.yml
├── host_vars/
│   ├── master01.yml
│   ├── master02.yml
│   └── arbiter.yml
└── playbook.yml
```

### `groups_vars/all.yml`

```yaml
percona_databases:
  - mynewbranddb
percona_users:
  - name: mynewbranddb_user
    password: "yie?XuiNaedu"
    privs: "mynewbranddb.*:ALL,GRANT"
    host: "%"
```

### `hosts_var/master01.yml`

```yaml
percona_cluster:
  role: master
  bootstrapper: true
  server_id: 1
  ip_address: "{{ ansible_default_ipv4.address }}"
```

### `host_vars/arbiter.yml`

```yaml
percona_cluster:
  role: arbiter
  bootstrapper: false
  server_id: 3
  ip_address: "{{ ansible_default_ipv4.address }}"
```

## CI / Molecule testing

Due to systemd and container limitations, GitHub Actions currently cannot fully start MySQL under Molecule.
All tests succeed locally via molecule test.
If you have ideas or patches to make CI integration work, PRs are welcome!

## Author & Maintainers

Created by [Bmeme](https://www.bmeme.com)
Maintained by:
* [Daniele Piaggesi](mailto:daniele.piaggesi@bmeme.com)
* [Roberto Mariani](mailto:roberto.mariani@bmeme.com)

## License
MIT/BSD dual license

## Credits
Role inspired by [csuka/ansible_role_percona_xtradb_cluster](https://github.com/csuka/ansible_role_percona_xtradb_cluster)
