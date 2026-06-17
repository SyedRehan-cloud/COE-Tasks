Below is a **structured working document (v1)** of your current repository state. I’ve written it like an internal engineering handbook so you can repeatedly use it while improving PMM, Backup, DR, and HA.

---

# 📘 Percona Server HA Role — Working Documentation (Current State v1)

## 🧭 1. Project Overview

This repository contains an **Ansible Role for Percona Server 8.0 High Availability setup**.

It is designed to deploy and manage a full MySQL HA stack including:

* Percona Server 8.0 (MySQL-compatible)
* Master–Slave replication (GTID-based)
* Orchestrator (failover automation)
* ProxySQL (query routing)
* XtraBackup (backup engine)
* MinIO (object storage backup target)
* mysqld_exporter (Prometheus metrics)
* PMM (Percona Monitoring Management – partial implementation)

---

## 🏗 2. High-Level Architecture

```
                   ┌──────────────────────┐
                   │   Orchestrator       │
                   │  Failover Manager    │
                   └─────────┬────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                                 │
   ┌────────▼────────┐             ┌──────────▼──────────┐
   │   ProxySQL       │             │   PMM Server        │
   │ Query Router     │             │ Monitoring System   │
   └────────┬────────┘             └──────────┬──────────┘
            │                                 │
   ┌────────▼────────┐             ┌──────────▼──────────┐
   │ MySQL Master     │◄──────────►│ MySQL Replica 1     │
   └────────┬────────┘             └──────────┬──────────┘
            │                                 │
            │                                 │
   ┌────────▼────────┐             ┌──────────▼──────────┐
   │ Replica 2        │             │ Backup Node         │
   │ (Backup Server)  │             │ XtraBackup + MinIO  │
   └──────────────────┘             └─────────────────────┘
```

---

## 📁 3. Repository Directory Structure (Current State)

```
percona_server/
├── README.md
├── CHANGELOG.md
├── BackupNRestore.md
│
├── defaults/
│   └── main.yml
│
├── handlers/
│   └── main.yml
│
├── meta/
│   └── main.yml
│
├── tasks/
│   ├── main.yml
│   ├── Ubuntu.yml
│   ├── configure.yml
│   ├── replication.yml
│   ├── database.yml
│   ├── users.yml
│   ├── mysql_metrics.yml
│   ├── orchestrator.yml
│   ├── proxysql.yml
│   ├── proxysql_configure.yml
│   ├── minio.yml
│   ├── backup_scripts.yml
│   ├── post_failover_script.yml
│   ├── pmm.yml
│
├── templates/
│   ├── mysqld.cnf.j2
│   ├── exporter.cnf.j2
│   ├── mysqld_exporter.service.j2
│   ├── orchestrator.conf.json.j2
│   ├── orchestrator.service.j2
│   ├── backup.cnf.j2
│   ├── minio.service.j2
│   ├── mysql-full-backup-minio.sh.j2
│   ├── mysql-full-backup-local.sh.j2
│   ├── mysql-incremental-backup-minio.sh.j2
│   ├── mysql-incremental-backup-local.sh.j2
│   ├── post-failover-action.sh.j2
│
└── vars/
    └── main.yml
```

---

## ⚙️ 4. Execution Flow (main.yml)

File:

```
tasks/main.yml
```

### Flow Order:

1. Install Percona Server
2. Configure MySQL
3. Setup Replication (if enabled)
4. Create Databases
5. Create Users
6. Install Metrics Exporter
7. Setup Orchestrator
8. Install ProxySQL
9. Configure ProxySQL
10. Setup MinIO client
11. Deploy Post Failover Script
12. Deploy Backup Scripts
13. Setup PMM (optional)

---

## 🧠 5. Feature Flags (Core Controls)

From `defaults/main.yml`:

### Installation Flags

```yaml
percona_install_packages: true
percona_replication: false
percona_database_creation: true
percona_users_creation: true
```

### HA Components

```yaml
percona_orchestrator_install: false
percona_orchestrator_configure: false
percona_proxysql_install: false
percona_proxysql_configure: false
```

### Observability

```yaml
percona_metrics_enabled: true
percona_pmm_enabled: false
```

### Backup

```yaml
percona_deploy_backup_scripts: false
percona_setup_backup_cron: false
percona_backup_server: false
percona_ha_backup_storage: "minio"
```

---

## 📊 6. PMM Integration (CURRENT IMPLEMENTATION)

File:

```
tasks/pmm.yml
```

### What it does:

#### 1. PMM Client Install

* Installs `pmm2-client`
* Enables Percona repo
* Checks `/usr/bin/pmm-agent`

---

#### 2. PMM Agent Setup

```bash
pmm-agent setup
--server-address=...
--server-username=admin
--server-password=***
```

---

#### 3. Node Registration (REST API)

Uses:

```
/v1/inventory/Nodes/Add
/v1/inventory/Services/Add
/v1/inventory/Agents/Add
```

Registers:

* MySQL Node
* MySQL Service
* mysqld_exporter agent

---

#### 4. Verification

```bash
pmm-admin status
```

---

### ⚠️ Known Issues

* No QAN setup
* No validation of registration
* Uses REST API instead of `pmm-admin add mysql`
* No retry logic
* No dashboard setup
* PMM server inconsistency (IP vs domain)

---

## 💾 7. Backup System

### File:

```
tasks/backup_scripts.yml
```

### What it does:

#### 1. Installs:

* percona-xtrabackup-80
* qpress

---

#### 2. Storage Modes

### MinIO mode:

```yaml
percona_ha_backup_storage: "minio"
```

### Local mode:

```yaml
percona_ha_backup_storage: "local"
```

---

#### 3. Backup Scripts

Generated via templates:

* Full backup script
* Incremental backup script

Stored in:

```
/usr/local/bin/
```

---

#### 4. Cron Jobs

* Weekly full backup
* Daily incremental backup

---

### ⚠️ Gaps

* No restore validation
* No backup integrity checks
* No backup alerting
* No DR replication of backups

---

## ☁️ 8. MinIO Integration

### File:

```
tasks/minio.yml
```

### Function:

* Downloads `mc` (MinIO client)
* Configures alias
* Verifies connection
* Creates bucket:

```
percona-backup
```

* Creates folder:

```
failover-resync/
```

---

## 🔁 9. Replication System

### File:

```
tasks/replication.yml
```

### Features:

* GTID replication
* Binlog fallback mode
* Auto master discovery
* Replica configuration
* Replica start/stop logic
* Validation checks

---

### Key Logic:

#### Master:

* Creates replication user
* Fetches binlog position

#### Slave:

* Configures replication source
* Starts replica
* Validates status

---

## 🧠 10. Orchestrator (Failover Engine)

### File:

```
tasks/orchestrator.yml
```

### Responsibilities:

* Installs orchestrator binary
* Configures systemd service
* Creates backend DB
* Starts API server
* Discovers MySQL topology
* Handles failover detection

---

### Key API:

```
/api/discover/<host>/<port>
```

---

## 🔁 11. Post Failover Script

### File:

```
templates/post-failover-action.sh.j2
```

### Purpose:

Executed after failover event.

### Responsibilities:

* ProxySQL updates
* Replica recovery logic
* MySQL health checks
* GTID rejoin
* Notifications (Slack/Email optional)
* Orchestrator rediscovery

---

## 📡 12. Observability (mysqld_exporter)

### File:

```
tasks/mysql_metrics.yml
```

### What it installs:

* mysqld_exporter
* systemd service
* MySQL user for exporter
* Prometheus metrics endpoint

---

## 🔐 13. Security Model

* MySQL credentials via variables
* Optional Vault integration
* No hardcoded passwords (intended)
* Uses `no_log: true` for secrets

---

## ⚠️ 14. Current Gaps (IMPORTANT)

### Observability

* PMM incomplete
* No QAN
* No dashboards

---

### Backup/DR

* No restore validation
* No DR replication
* No backup integrity checks

---

### Automation

* No automated patching
* No upgrade strategy

---

### Architecture

* PMM endpoint inconsistency
* No standard observability layer

---

## 🧭 15. Recommended Improvement Roadmap

### Phase 1 — PMM Fix

* Fix registration method
* Add QAN
* Add validation layer

---

### Phase 2 — Backup Hardening

* Restore validation system
* Backup verification checks
* DR backup replication

---

### Phase 3 — DR Automation

* Cross-site MinIO replication
* Failover recovery automation

---

### Phase 4 — Observability Framework

* Standard dashboards
* Metrics normalization
* Grafana integration

---

# 📌 How You Should Use This Document

Whenever you modify the repo:

### Ask:

* Does this change break PMM flow?
* Does backup still work in both modes?
* Is replication impacted?
* Is orchestrator still stable?
* Does this affect failover script?
Excellent. Now you have enough code locally to start a proper enhancement project instead of blindly modifying things.

After reviewing the role, the Semaphore logs, your PMM assessment, and the role structure, here's what I would do if I were taking ownership of this repository.

# First Understand What Already Exists

Current role already provides:

| Component              | Status             |
| ---------------------- | ------------------ |
| Percona Install        | ✅ Implemented      |
| MySQL Configuration    | ✅ Implemented      |
| Replication            | ✅ Implemented      |
| Orchestrator           | ✅ Implemented      |
| ProxySQL               | ✅ Implemented      |
| mysqld_exporter        | ✅ Implemented      |
| MinIO Backup           | ✅ Implemented      |
| Backup Cron            | ✅ Implemented      |
| Restore Playbook       | ✅ Separate Project |
| PMM Integration        | ⚠️ Partial/Broken  |
| Backup Validation      | ❌ Missing          |
| Grafana Dashboards     | ❌ Missing          |
| DR Automation          | ❌ Missing          |
| Cross-site Replication | ❌ Missing          |
| Automated Patching     | ❌ Missing          |

---

# What I Found In PMM Role

Current PMM role:

`tasks/pmm.yml`

It tries to:

1. Install pmm-agent
2. Configure pmm-agent
3. Register node through PMM API
4. Register MySQL service
5. Register mysqld_exporter
6. Run

```bash
pmm-admin status
```

Problem:

This is NOT how PMM is normally managed.

Normally:

```bash
pmm-admin config
pmm-admin add mysql
```

is used.

Current role is using direct REST APIs:

```yaml
/v1/inventory/Nodes/Add
/v1/inventory/Services/Add
/v1/inventory/Agents/Add
```

This is fragile.

---

# First Enhancement I Would Build

Create:

```text
tasks/
 ├── pmm_install.yml
 ├── pmm_register.yml
 ├── pmm_verify.yml
```

instead of one huge:

```text
tasks/pmm.yml
```

Then main.yml becomes:

```yaml
- include_tasks: pmm_install.yml
- include_tasks: pmm_register.yml
- include_tasks: pmm_verify.yml
```

Much easier to maintain.

---

# PMM Assessment Mapping

You reported:

> PMM agents installed but not registered

That means:

```bash
pmm-agent status
```

exists

but

```bash
pmm-admin inventory list services
```

probably empty.

So Enhancement #1:

Create verification tasks.

Example:

```yaml
- name: Verify PMM registration
  command: pmm-admin status
```

Fail if:

```yaml
failed_when:
  - "'Connected' not in pmm_status.stdout"
```

Currently role never checks this properly.

---

# Enhancement #2 — QAN

Currently missing.

Add:

```yaml
percona_pmm_qan_enabled: true
```

Then:

```bash
pmm-admin add mysql \
 --query-source=perfschema
```

This gives:

* Slow Queries
* Query Analytics
* Top Queries

inside PMM.

This is probably your biggest observability gap.

---

# Enhancement #3 — Standardize PMM Endpoint

You found:

```text
https://pmm.opstree.dev
https://192.168.8.204
```

Two endpoints.

Create variable:

```yaml
percona_ha_pmm_server_url
```

Only one source of truth.

Example:

```yaml
percona_ha_pmm_server_url: "https://pmm.opstree.dev"
```

Never use IPs in inventory.

---

# Enhancement #4 — Backup Validation

This is the biggest missing feature.

Current flow:

```text
Backup
 ↓
MinIO
 ↓
Done
```

No validation.

Need:

```text
Backup
 ↓
Restore
 ↓
Integrity Check
 ↓
Report
```

---

Create new file:

```text
tasks/backup_validation.yml
```

Flow:

```yaml
1. Download latest backup
2. Restore into sandbox mysql
3. Run validation
4. Generate report
```

---

Validation examples:

```sql
SHOW DATABASES;
```

```sql
SELECT COUNT(*) FROM important_table;
```

```sql
CHECKSUM TABLE important_table;
```

```sql
SHOW REPLICA STATUS;
```

---

# Enhancement #5 — Automated Restore Test

Your PMM assessment mentions:

> Weekly restore in isolated environment

This should become a new Semaphore template.

Example:

```text
Percona-Restore-Validation
```

Workflow:

```text
Create temporary VM
↓
Restore latest backup
↓
Run validation
↓
Send report
↓
Destroy VM
```

This is enterprise-grade DR validation.

---

# Enhancement #6 — Cross Site Replication

Current MinIO:

```yaml
percona-backup
```

single bucket.

Need:

```text
Primary MinIO
 ↓
Replication
 ↓
DR MinIO
```

Variables:

```yaml
percona_ha_dr_minio_endpoint
percona_ha_dr_bucket
```

Then backup scripts sync:

```bash
mc mirror
```

after backup completion.

---

# Enhancement #7 — Automated Patching

Create:

```text
tasks/patching.yml
```

Install:

```bash
unattended-upgrades
```

or controlled patching playbook.

Targets:

```text
Percona
ProxySQL
Orchestrator
```

---

# Enhancement #8 — Observability Framework

This matches your review document.

Create:

```text
tasks/pmm_dashboards.yml
```

Metrics:

### MySQL

```text
Connections
QPS
TPS
Threads Running
```

### Replication

```text
Replica Lag
Replica Broken
GTID Gap
```

### InnoDB

```text
Buffer Pool Hit Ratio
Dirty Pages
Redo Generation
```

### Backup

```text
Last Backup Time
Backup Size
Backup Success
```

### Orchestrator

```text
Failovers
Recovery Events
```

### ProxySQL

```text
Backend Health
Connection Pool
Routing Stats
```

---

# What About Restore Role?

You asked:

> should restore stay separate?

YES.

100%.

Keep restore separate.

Architecture should be:

```text
percona_server
│
├── Install
├── Configure
├── Replication
├── Orchestrator
├── ProxySQL
├── PMM
├── Backup
│
└── Restore (NO)
```

Separate repository/playbook:

```text
percona_restore
```

because restore is:

* destructive
* emergency operation
* different lifecycle

Never mix with installation role.

---

# Branch Strategy For You

Create your own enhancement branch.

From master:

```bash
git checkout master

git pull origin master

git checkout -b pmm-observability-enhancement
```

Then work in phases.

### Phase 1

```text
PMM Fixes
QAN
Registration Verification
```

### Phase 2

```text
Backup Validation
Restore Validation
```

### Phase 3

```text
Cross-site DR
Automated Patching
```

### Phase 4

```text
Grafana Dashboards
Observability Framework
```

I would start with **PMM + Backup Validation first**, because those directly address the gaps you identified and can be implemented without redesigning the HA architecture.
