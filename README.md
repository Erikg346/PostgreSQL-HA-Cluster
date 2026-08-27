# PostgreSQL HA Cluster Automation

## Overview

This project automates the deployment of a three-node highly available PostgreSQL cluster using:

- PostgreSQL
- Patroni
- etcd
- Ansible
- Vagrant
- VMware Fusion

The deployment is entirely variable-driven and designed to support multiple PostgreSQL versions and repository sources without requiring changes to the core playbook logic.

Current implementation has been validated using:

- Ubuntu 24.04 LTS
- PostgreSQL 17
- PGDG Repository
- Patroni 4.x
- etcd 3.4.x
- Vagrant
- VMware Fusion

---

# Why This Project Exists

Building a highly available PostgreSQL cluster manually is often time-consuming and error-prone.

This project automates:

- PostgreSQL installation
- etcd cluster deployment
- Patroni cluster deployment
- Streaming replication
- Leader election
- Failover configuration
- Cluster validation

The goal is to provide a repeatable, reusable PostgreSQL HA platform that can be deployed through Infrastructure as Code instead of manual installation procedures.

---

# Understanding the Components

## PostgreSQL

PostgreSQL is the database engine.

It stores:

- Application data
- System catalogs
- WAL records
- Replication information

Without PostgreSQL there is no database cluster.

---

## Patroni

Patroni is a PostgreSQL high-availability framework.

Patroni is responsible for:

- Leader election
- Automatic failover
- Replica creation
- Cluster management
- PostgreSQL lifecycle management

Without Patroni, PostgreSQL replication can exist, but failover is a manual operation.

---

## etcd

etcd is a distributed key-value store.

Patroni uses etcd as a Distributed Configuration Store (DCS).

etcd stores:

- Cluster state
- Lock ownership
- Leader information
- Patroni cluster metadata

Without etcd, Patroni members cannot coordinate cluster operations.

---

## How They Work Together

```text
                PostgreSQL
                     ▲
                     │
                 Patroni
                     ▲
                     │
                   etcd
```

### Layer Responsibilities

```text
PostgreSQL
    └── Stores data

Patroni
    └── Manages PostgreSQL

etcd
    └── Coordinates Patroni
```

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
- Replica creation
- Cluster state
- Failover operations
- PostgreSQL startup and shutdown

etcd provides:

- Distributed consensus
- Cluster coordination
- Lock management
- Cluster metadata storage

---

# Design Principles

The project follows a strict separation of concerns.

---

## Inventory

Defines:

```text
Where should this be deployed?
```

Example:

```ini
[postgres_cluster]
pg01
pg02
pg03
```

---

## Variables

Define:

```text
What should be deployed?
```

Example:

```yaml
postgres_version: 17

cluster_name: postgres-cluster

postgres_repo: pgdg
```

---

## Playbooks

Define:

```text
How should it be deployed?
```

Example:

```text
postgresql.yml
etcd.yml
patroni.yml
cluster.yml
```

---

# Deployment Flow

Deployment occurs in the following sequence:

```text
cluster.yml
    |
    +--> postgresql.yml
    |       |
    |       +--> Configure repository
    |       +--> Install PostgreSQL
    |       +--> Create directories
    |
    +--> etcd.yml
    |       |
    |       +--> Install etcd
    |       +--> Configure cluster
    |       +--> Validate quorum
    |
    +--> patroni.yml
            |
            +--> Install Patroni
            +--> Configure Patroni
            +--> Bootstrap leader
            +--> Build replicas
```

A complete deployment can be executed using:

```bash
ansible-playbook cluster.yml
```

---

# Directory Structure

```text
.
├── README.md
├── cluster.yml
├── inventory.ini
├── ssh_config
│
├── group_vars
│   └── postgresql_cluster.yml
│
├── playbooks
│   ├── postgresql.yml
│   ├── etcd.yml
│   └── patroni.yml
│
├── templates
│   ├── config.yml.j2
│   └── etcd.conf.yaml.j2
│
└── Vagrantfile
```

---

# Components

## PostgreSQL Layer

Responsible for:

- Repository configuration
- PostgreSQL installation
- Directory creation
- Logging configuration
- Service management

Playbook:

```text
playbooks/postgresql.yml
```

---

## etcd Layer

Responsible for:

- etcd installation
- Cluster configuration
- Quorum formation
- Service management

Playbook:

```text
playbooks/etcd.yml
```

Template:

```text
templates/etcd.conf.yaml.j2
```

---

## Patroni Layer

Responsible for:

- Patroni installation
- PostgreSQL HA management
- Leader election
- Replica provisioning
- Failover

Playbook:

```text
playbooks/patroni.yml
```

Template:

```text
templates/config.yml.j2
```

---

## Cluster Layer

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

# PostgreSQL Repository Support

The framework supports multiple PostgreSQL repository providers.

---

## PGDG (Default)

```yaml
postgres_repo: pgdg
```

Advantages:

- Official PostgreSQL packages
- Closely aligned with PostgreSQL documentation
- Community-supported
- Simplified package management

---

## Percona

```yaml
postgres_repo: percona
```

Advantages:

- Enterprise-oriented distribution
- Percona-supported packages
- Commonly used in enterprise environments

The repository source can be changed through variables without modifying playbook logic.

---

# Configuration

File:

```text
group_vars/postgresql_cluster.yml
```

---

## PostgreSQL

```yaml
postgres_repo: pgdg

postgres_version: 17

data_root: /loftware/data

log_root: /loftware/logs

postgres_port: 5432
```

---

## Patroni

```yaml
cluster_name: postgres-cluster

patroni_rest_port: 8008
```

---

## etcd

```yaml
etcd_client_port: 2379

etcd_peer_port: 2380

etcd_cluster_token: pgcluster
```

---

## Cluster Nodes

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

## Start Virtual Machines

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
ansible postgres_cluster -m ping -u vagrant
```

Expected:

```text
pong
```

from all cluster nodes.

---

## Deploy the Cluster

```bash
ansible-playbook ./playbooks/cluster.yml
```

---

# Validation

## Check Patroni Cluster

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

## Verify PostgreSQL

```bash
sudo -u postgres psql
```

```sql
select version();
```

---

## Verify Replication

```sql
select * from pg_stat_replication;
```

---

## Verify etcd

```bash
curl http://127.0.0.1:2379/health
```

---

## Verify Patroni

```bash
systemctl status patroni
```

---

# Useful Operational Commands

## Show Cluster Status

```bash
patronictl -c /etc/patroni/config.yml list
```

---

## View Patroni Logs

```bash
journalctl -u patroni -f
```

---

## View etcd Logs

```bash
journalctl -u etcd -f
```

---

## Check PostgreSQL Processes

```bash
ps -ef | grep postgres
```

---

## Check Replication Status

```sql
select * from pg_stat_replication;
```

---

## Check etcd Health

```bash
curl http://127.0.0.1:2379/health
```

---

# Failover Testing

## Stop Current Leader

For example:

```bash
sudo systemctl stop patroni
```

on pg01.

---

## Verify New Leader Election

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

after rejoining.

---

# Important Implementation Notes

## Patroni Owns PostgreSQL

When Patroni is deployed:

```bash
systemctl stop postgresql
systemctl disable postgresql
```

Patroni becomes the PostgreSQL process manager.

---

## PostgreSQL Data Directory Permissions

PostgreSQL data directories must be:

```text
0700
```

or

```text
0750
```

Invalid permissions will prevent PostgreSQL from starting.

---

## Required Logging Directories

The following directories must exist prior to bootstrap:

```text
/loftware/logs/postgres_log

/loftware/logs/wal_archive
```

Missing directories can prevent PostgreSQL startup.

---

## etcd Configuration Source

The Ubuntu 24.04 etcd package reads:

```text
/etc/etcd/etcd.conf.yaml
```

The automation templates and manages this file directly.

---

## etcd v2 Compatibility

Current Patroni implementation requires:

```yaml
enable-v2: true
```

Without v2 support Patroni cannot communicate with etcd.

---

## pg_hba Rules

Patroni bootstrap requires valid host-based authentication rules.

Missing pg_hba entries may cause:

```text
Failed to bootstrap cluster
```

errors.

---

# Current Constraints

Validated against:

```text
Ubuntu 24.04
PostgreSQL 17
PGDG Repository
Percona Repository
Patroni 4.x
etcd 3.4.x
Vagrant
VMware Fusion
```

Not yet validated against:

```text
PostgreSQL 18
Cloud platforms
Physical servers
Multi-datacenter deployments
Kubernetes
```

---

# Future Enhancements

## Security

- Ansible Vault
- TLS for PostgreSQL
- TLS for Patroni
- TLS for etcd

---

## Operations

- Automatic health checks
- Automated failover testing
- Rolling restarts
- Rolling cluster upgrades

---

## Backup

- pgBackRest integration
- WAL validation
- Point-in-time recovery testing

---

## Monitoring

- Prometheus
- Grafana
- PostgreSQL Exporter
- Patroni Exporter

---

## Storage

- Tablespace automation
- Backup filesystem configuration
- Archive filesystem validation

---

# Lessons Learned

Key findings during implementation:

- PostgreSQL data directories require strict permissions.
- Patroni must manage PostgreSQL services.
- Valid pg_hba rules are required during bootstrap.
- etcd v2 compatibility is required for Patroni.
- PostgreSQL logging directories must exist before startup.
- Ubuntu's etcd implementation consumes `/etc/etcd/etcd.conf.yaml`.
- Variable-driven configuration greatly improves reusability.
- Separating configuration from playbook logic simplifies maintenance.

---

# Project Status

Current maturity:

```text
v1.0 Lab / Proof of Concept
```

Successfully validated:

- PostgreSQL Deployment
- PostgreSQL Repository Abstraction (PGDG / Percona)
- etcd Cluster Formation
- Patroni Cluster Formation
- Streaming Replication
- Automatic Leader Election
- Automated Failover
- Cluster Rebuilds Through Ansible
- Dynamic Configuration Through Variables
- Template-Based Configuration Management

Validated topology:

```text
3 Node PostgreSQL Cluster

1 Leader
2 Streaming Replicas

3 etcd Members
3 Patroni Members
```
