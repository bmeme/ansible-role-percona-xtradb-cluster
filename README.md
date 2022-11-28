# Ansible Role: Percona XtraDB Cluster
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://GitHub.com/Naereen/StrapDown.js/graphs/commit-activity)
[![MIT license](https://img.shields.io/badge/License-MIT-blue.svg)](https://lbesson.mit-license.org/)

Installs and configures Percona XtraDB Cluster (only 5.7 at this stage) on EL 7/8. This role takes in consideration a standard Galera architecture: three nodes minimum of which two act as server and one acts as arbitrator.

This role, *at this stage*, **does** not support:

* **node scaling**: you cannot re-run this role on a pre-installed architecture to add one or more nodes.
* **pmm**: You can add [PMM Client](https://docs.percona.com/percona-monitoring-and-management/setting-up/client/index.html) manually to your architecture.
* **ssl**: You can [encrypt your PXC traffic](https://docs.percona.com/percona-xtradb-cluster/5.7/security/encrypt-traffic.html) manually

Some approaches used in this role are definitely inspired by the excellent work of `csuka/ansible_role_percona_xtradb_cluster`, available [here](https://github.com/csuka/ansible_role_percona_xtradb_cluster). But the implementation needs were different, so that we decided to open a new project: other that, the `csuka` project supports only EL8 and Percona 8.0. Hopefully, in a near future, these project could be merged.


## Requirements
No special requirements; note that this role requires root access, so either run it in a playbook with a global `become: yes`, or invoke the role in your playbook like:

```yaml
- hosts: database
  roles:
    - role: bmeme.percona_xtradb_cluster
      become: yes
```

Copy the variables from `default/main.yml` and adjust them for **each node** in it's variable folder.
Take a look at the `molecule/default/group_vars` and `molecule/default/host_vars` folder to have an idea of how to configure every single node of your cluster architecture.

## Installation
This is an Ansible role distributed using Ansible Galaxy. In order to install this role you can use the following command.

`$ ansible-galaxy install bmeme.percona_xtradb_cluster `

## Update
If you want to update the role, you need to pass --force parameter when installing. Please, check the following command:

`$ ansible-galaxy install --force bmeme.percona_xtradb_cluster `

## Clustering
When there are at least 2 nodes in the play, a multi-master cluster is possible. A 2 node galera cluster is a fundamentally broken design, as it cannot maintain uptime without a quorum and the ability of a node to go down to aid recovery. 

## Role Variables 

```yaml
percona_cluster:
  role: master
  bootstrapper: false
  server_id: '1'
  ip_address: "{{ ansible_default_ipv4.address }}"
```
The `percona_cluster` object is what you really need to configure for each node:

* **role**: *string*. Defines what is the role of the specific node. Could be *master* or *arbiter*. If you have three or more nodes dedicated to your cluster, all of them could be *master*. If you want to have only two nodes in the cluster and use an arbiter (installable on any other server in your infrastructure), you will have two nodes as *master* and one as *arbiter*
* **bootstrapper**: *boolean*. Defines the node responsible for bootstrapping the cluster. **Pay attention**: there must only be one bootstrapper and cannot be the arbiter!!! There is no assertion in the role to check any of that (at this stage).
* **server_id**: *int*. Defines the unique ID of a node. **Pay attention**: this is a **unique id**! Do not use the same ID for more than a node. There isn't any assertion in the role to check this (always at this stage).
* **ip_address**: *string*. Defines the IP Address of the node. To automatically calculate IPv4 address of a node, use `{{ ansible_default_ipv4.address }}` ansible variable as used in `default/main.yml` file.

```yaml
percona_cluster_name: "galera-cluster"
```
Defines the name you want to have your cluster. Change it at your need.

```yaml
percona_root_user: "root"
percona_root_password: "S3cr3tS/$"
percona_sst_user: "sstuser"
percona_sst_password: "S3cr3tS/$"
```
Defines `root` user/password and `sst` user/password. `sst` is the user responsible for replication transactions between nodes.

```yaml
percona_databases:
  - mynewbranddb

percona_users:
  - name: mynewbranddb_user
    password: "yie?XuiNaedu"
    privs: 'mynewbranddb.*:ALL,GRANT'
    state: present
    encrypted: false
    host: '%'
```
Create databases and/or users during the role execution. Leave blank to skip. Databases and users are created only on bootstrapper node.

```yaml
# Datafile directory.
mysql_datadir: "/var/lib/mysql"

# Address on which mysql bind to
mysql_bind_address: "{{ percona_cluster.ip_address }}"

# Mysql symbolic links configuration
mysql_symbolic_links: 0

# pxc_strict_mode allowed values: DISABLED,PERMISSIVE,ENFORCING,MASTER
mysql_cluster_pxc_strict_mode: ENFORCING
```
Basic Mysqld configuration. 

```yaml
# All specific configuration items
# Example:
#
# mysql_cluster_configuration: |-
#   max_allowed_packet=64M
#
mysql_cluster_configuration: ""
```
This allows to add custom configuration to `mysqld.conf` file, stored in `/etc/percona-xtradb-cluster.conf.d` directory. Add `mysqld` configuration items exactly as if you were putting them in `mysqld.conf` file.

Here an example of advanced configurations:

```yaml
mysql_cluster_configuration: |-
  max_allowed_packet=64M
  some_other_conf=some_other_value
```

## Supported Linux distro and versions
- EL 7
- EL 8

## Dependencies
N/A

## Example Playbook
    - hosts: db-servers
      become: yes
      roles:
        - { role: bmeme.percona_xtradb_cluster }

*Inside `group_vars/all.yml`*:

	percona_databases:
	  - mynewbranddb
	
	percona_users:
	  - name: mynewbranddb_user
	    password: "yie?XuiNaedu"
	    privs: 'mynewbranddb.*:ALL,GRANT'
	    state: present
	    encrypted: false
	    host: '%'
	    
*Inside `host_vars/master01.yml`*:
	
	percona_cluster:
	  role: master
	  bootstrapper: true
	  server_id: '1'
	  ip_address: "{{ ansible_default_ipv4.address }}"

*Inside `host_vars/master02.yml`*:
	
	percona_cluster:
	  role: master
	  bootstrapper: false
	  server_id: '2'
	  ip_address: "{{ ansible_default_ipv4.address }}"
	  
*Inside `host_vars/arbiter.yml`*:
	
	percona_cluster:
	  role: arbiter
	  bootstrapper: false
	  server_id: '3'
	  ip_address: "{{ ansible_default_ipv4.address }}"
	  
## GitHub Actions and molecule test
A CI actions is not active for this role. This because (I don't know why...) role breaks during execution in GitHub action environment when mysql server is enabled. But molecule test works well.

## License
MIT / BSD

## Author Information
This role was created in 2022 by [Bmeme](https://www.bmeme.com). It is actually maintained by [Daniele Piaggesi](https://github.com/g0blin79), [Roberto Mariani](https://github.com/jean-louis) and [Michele Mondelli](https://github.com/Mithenks)

## Credits
This role was really inspired by [csuka Ansible Role XtraDB Cluster](https://github.com/csuka/ansible_role_percona_xtradb_cluster).