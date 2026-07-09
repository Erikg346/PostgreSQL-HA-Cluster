# PostgreSQL HA Cluster Automation

## Overview

This project deploys a highly available PostgreSQL cluster using:

- PostgreSQL
- Patroni
- etcd
- Ansible
- Vagrant
- VMware Fusion

The deployment is entirely variable-driven and designed to support multiple PostgreSQL versions without changing the playbook logic.

Current implementation has been validated on:

- Ubuntu 24.04 LTS
- PostgreSQL 17 (PGDG Repository)
- Patroni 4.x
- etcd 3.4.x
- VMware Fusion
- Vagrant

---

# Architecture

```text
                   +----------------+
                   |     etcd       |
                   | 3-node quorum  |
                   +----------------+
                     |           |
                     | Patroni   |
                     v           v

+----------------+   +----------------+   +----------------+
|     pg01       |   |     pg02       |   |     pg03       |
|   Leader       |   |   Replica      |   |   Replica      |
| PostgreSQL 17  |   | PostgreSQL 17  |   | PostgreSQL 17  |
+----------------+   +----------------+   +----------------+
```

Patroni manages:

- Leader election
- Cluster state
- Failover
- Replica provisioning
- PostgreSQL lifecycle management

etcd provides:

- Distributed consensus
- Cluster coordination
- Leader lock management

---

# Design Principles

The project follows a simple separation of concerns:

## Inventory

Answers:

```text
Where should this be deployed?
```

Example:

```ini
[loftware_prod]
pg01
pg02
pg03
```

---

## Variables

Answers:

```text
What should be deployed?
```

Example:

```yaml
postgres_version: 17

cluster_name: postgres-cluster
```

---

## Playbooks

Answers:

```text
How should it be deployed?
```

Example:

```yaml
postgresql.yml
etcd.yml
patroni.yml
cluster.yml
```

---

# Directory Structure

```text
.
├── inventory.ini
├── cluster.yml
│
├── group_vars
│   └── loftware_prod.yml
│
├── playbooks
│   ├── postgresql.yml
│   ├── etcd.yml
│   └── patroni.yml
│
├── templates
│   └── config.yml.j2
│
├── ssh_config
│
└── Vagrantfile
```

---

# Components

## PostgreSQL

Responsible for:

- PGDG repository configuration
- PostgreSQL installation
- Directory creation
- PostgreSQL service management
- Validation

Playbook:

```text
playbooks/postgresql.yml
```

---

## etcd

Responsible for:

- etcd installation
- Cluster member configuration
- Cluster initialization
- Service management

Playbook:

```text
playbooks/etcd.yml
```

---

## Patroni

Responsible for:

- Patroni installation
- PostgreSQL HA management
- Leader election
- Replica creation
- Failover

Playbook:

```text
playbooks/patroni.yml
```

---

## Cluster

Responsible for:

- End-to-end deployment

Playbook:

```text
cluster.yml
```

Example:

```yaml
---
- import_playbook: playbooks/postgresql.yml

- import_playbook: playbooks/etcd.yml

- import_playbook: playbooks/patroni.yml
```

---

# Configuration

## Inventory

```ini
[loftware_prod]
pg01
pg02
pg03
```

---

## Variables

File:

```text
group_vars/loftware_prod.yml
```

### PostgreSQL

```yaml
postgres_version: 17

data_root: /loftware/data

log_root: /loftware/logs

postgres_port: 5432
```

### Cluster

```yaml
cluster_name: postgres-cluster

patroni_rest_port: 8008
```

### etcd

```yaml
etcd_client_port: 2379

etcd_peer_port: 2380

etcd_cluster_token: pgcluster
```

### Nodes

```yaml
etcd_nodes:
  - name: pg01
    ip: 192.168.56.11

  - name: pg02
    ip: 192.168.56.12

  - name: pg03
    ip: 192.168.56.13
```

---

# Build Process

## Start VMs

```bash
vagrant up
```

---

## Generate SSH Configuration

```bash
vagrant ssh-config > ssh_config
```

---

## Verify Connectivity

```bash
ansible loftware_prod -m ping -u vagrant
```

Expected:

```text
pong
```

from all hosts.

---

## Deploy Cluster

```bash
ansible-playbook cluster.yml
```

---

# Validation

## Patroni Cluster Status

```bash
sudo patronictl -c /etc/patroni/config.yml list
```

Expected:

```text
+ Cluster: postgres-cluster

pg01  Leader   running
pg02  Replica  streaming
pg03  Replica  streaming
```

---

## Check PostgreSQL

```bash
sudo -u postgres psql
```

```sql
select version();
```

---

## Check etcd

```bash
etcdctl endpoint health
```

---

## Check Patroni

```bash
systemctl status patroni
```

---

# Failover Testing

## Stop Current Leader

Example:

```bash
sudo systemctl stop patroni
```

on pg01.

---

## Verify New Leader

Run on another node:

```bash
sudo patronictl -c /etc/patroni/config.yml list
```

Expected:

```text
pg02 Leader
```

or

```text
pg03 Leader
```

---

## Restore Original Leader

```bash
sudo systemctl start patroni
```

Expected:

```text
Replica
```

role after rejoining.

---

# Important Implementation Notes

## PostgreSQL Data Directory Permissions

Must be:

```text
0700
```

or

```text
0750
```

Incorrect permissions cause:

```text
FATAL: data directory has invalid permissions
```

---

## Patroni Owns PostgreSQL

When using Patroni:

```bash
systemctl stop postgresql
systemctl disable postgresql
```

Patroni becomes the PostgreSQL process manager.

---

## Required Log Directories

Must exist before bootstrap:

```text
/loftware/logs/postgres_log

/loftware/logs/wal_archive
```

Missing directories cause bootstrap failures.

---

## etcd v2 API

Required for current Patroni configuration.

etcd configuration contains:

```text
ETCD_ENABLE_V2="true"
```

Without this setting Patroni cannot communicate with etcd.

---

## pg_hba Configuration

Patroni bootstrap requires host-based authentication rules.

Missing pg_hba entries can prevent bootstrap completion.

---

# Current Constraints

Validated against:

```text
Ubuntu 24.04
PostgreSQL 17
Patroni 4.x
etcd 3.4.x
Vagrant
VMware Fusion
```

Not yet validated against:

```text
PostgreSQL 18
Physical servers
Cloud deployments
Multiple datacenters
```

---

# Future Enhancements

## Security

- Ansible Vault
- TLS between Patroni nodes
- TLS for PostgreSQL
- TLS for etcd

## Operations

- Automated health checks
- Automated failover validation
- Rolling restarts
- Rolling upgrades

## Backup

- pgBackRest
- WAL archiving validation
- PITR testing

## Monitoring

- PostgreSQL exporter
- Patroni exporter
- Prometheus
- Grafana

---

# Lessons Learned

Key implementation discoveries:

- PostgreSQL data directory must use secure permissions.
- Patroni requires ownership of PostgreSQL processes.
- Patroni needs valid pg_hba entries during bootstrap.
- etcd must expose the API expected by Patroni.
- Log and archive directories must exist before cluster initialization.
- Separating configuration from playbook logic makes the deployment reusable and version-independent.

---

# Version

Current status:

```text
v1.0 Lab / Proof of Concept
```

Validated functionality:

- PostgreSQL Deployment
- etcd Cluster Formation
