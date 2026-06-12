# PERCONA MYSQL HA CLUSTER – OPERATIONAL AUDIT & ONBOARDING GUIDE


## 1. ACCESSING THE INFRASTRUCTURE

### 1.1 SSH Access Pattern

All nodes are accessed via IP:

```bash
ssh user@192.168.8.61
ssh user@192.168.8.77
ssh user@192.168.8.12
```

Typical roles:

* MySQL Master
* MySQL Replica
* Orchestrator node

---

### 1.2 Verify you are on correct server

```bash
hostname
ip a
whoami
```

Meaning:

* hostname → identifies server identity
* ip a → confirms correct network interface/IP
* whoami → confirms SSH user permissions

---

## 2. IDENTIFY IF SYSTEM IS VM OR CONTAINER

### 2.1 Primary check

```bash
systemd-detect-virt
```

Output meaning:

| Output           | Meaning                       |
| ---------------- | ----------------------------- |
| kvm/vmware/xen   | VM                            |
| docker/container | container                     |
| none             | physical/VM without detection |

---

### 2.2 cgroup inspection

```bash
cat /proc/1/cgroup
```

Interpretation:

* /docker/ → container
* /kubepods/ → Kubernetes pod
* /system.slice/ → VM/bare metal

---

### 2.3 Kernel verification

```bash
uname -a
```

Meaning:

Shows OS kernel used by MySQL process (VM shares host kernel).

---

## 3. MYSQL INSTALLATION VERIFICATION

### 3.1 Check MySQL service

```bash
systemctl status mysql
```

Meaning:
Confirms MySQL is installed as system service
Shows active/running state


### 3.2 Check Percona version

```bash
mysql --version
```

Expected:

```
Percona Server 8.0.x
```

---

### 3.3 Check MySQL process

```bash
ps -ef | grep mysqld
```

Meaning:
Confirms actual database engine running

---

## 4. UNDERSTANDING MYSQL DATA DIRECTORY

### 4.1 Main data location

```
/var/lib/mysql
```

This contains:

Core files:

| File                   | Meaning                          |
| ---------------------- | -------------------------------- |
| ibdata1                | InnoDB system tablespace         |
| ib_logfile / redo logs | crash recovery logs              |
| undo_001/002           | rollback logs                    |
| mysql-bin.*            | binary logs (replication source) |
| relay-bin.*            | replica logs                     |
| *.ibd                  | per-table data files             |

---

### 4.2 What your output shows

Example:

```
mysql-bin.000001
relay-bin.000006
ibdata1
undo_001
```

Meaning:

* mysql-bin.* → Master replication logs
* relay-bin.* → Replica receiving logs
* ibdata1 → shared InnoDB metadata
* undo_* → transaction rollback system

---

## 5. REPLICATION ARCHITECTURE UNDERSTANDING

From your cluster:

* Master: 192.168.8.61
* Replica: 192.168.8.77

---

### 5.1 Check replication status (IMPORTANT)

```bash
mysql -e "SHOW SLAVE STATUS\G"
```

OR (MySQL 8+):

```bash
mysql -e "SHOW REPLICA STATUS\G"
```

Key fields:

| Field                 | Meaning                   |
| --------------------- | ------------------------- |
| Replica_IO_Running    | receives logs from master |
| Replica_SQL_Running   | applies logs              |
| Seconds_Behind_Master | replication lag           |
| Master_Host           | upstream server           |

---

### 5.2 Check master status

```bash
mysql -e "SHOW MASTER STATUS;"
```

Meaning:

Shows:

* current binlog file
* position
* GTID set

Used for replication sync

---

## 6. MYSQL CONFIGURATION AUDIT

### 6.1 Main config file

```
/etc/mysql/mysql.conf.d/mysqld.cnf
```

---

### 6.2 Important parameters

Replication:

```
server-id = X
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

Meaning:

| Config    | Meaning                     |
| --------- | --------------------------- |
| server-id | unique ID per node          |
| log_bin   | enables replication logs    |
| ROW       | safest replication mode     |
| GTID      | global transaction tracking |

---

Replica settings:

```
read_only = ON
super_read_only = ON
relay-log = relay-bin
```

Meaning:
Prevents writes on replica

---

Performance:

```
innodb_buffer_pool_size
max_connections
tmp_table_size
```

Controls memory + query performance

---

## 7. CHECK DISK DATA (WHAT DATA EXISTS)

### 7.1 List databases

```bash
mysql -e "SHOW DATABASES;"
```

Expected output:

* mysql
* performance_schema
* sys
* application DBs

---

### 7.2 Check tables inside DB

```bash
mysql -e "USE demo; SHOW TABLES;"
```

---

### 7.3 Inspect table size

```bash
mysql -e "
SELECT table_schema, table_name, data_length
FROM information_schema.tables;
"
```

---

### 7.4 Physical file mapping

```bash
ls -lh /var/lib/mysql/<db_name>/
```

Meaning:
Each .ibd file = one table

---

## 8. ORCHESTRATOR UNDERSTANDING

From your logs:

* orchestrator table exists
* GitHub orchestrator UI
* failover automation

---

### 8.1 What Orchestrator does

* Detects master failure
* Promotes replica
* Updates topology automatically

---

### 8.2 Check topology

```bash
orchestrator-client -c topology
```

---

## 9. ANSIBLE / SEMAPHORE AUDIT

You are using:

* Semaphore UI
* GitHub repo: OT-COE/LDCManager

Playbooks:

* percona-server-role
* percona-restore

---

### 9.1 What your jobs are doing

Example job:

* Configure MySQL Master
* Install Percona
* Add repository
* Install mysql role
* Apply config
* Enable replication

---

### 9.2 Key roles used

#### 1. MySQL role

`roles/mysql/tasks/Ubuntu.yml`

Does:

* installs Percona server
* configures repo
* installs dependencies

---

#### 2. Restore playbook

`restore-playbook.yml`

Does:

* uses xtrabackup
* pulls backup from MinIO
* restores old master

---

#### 3. GTID validation playbook

Does:

* validates replication consistency
* resets GTID if mismatch

---

## 10. ☁️ BACKUP SYSTEM (MINIO + XTRABACKUP)

### 10.1 Backup tool

```
xtrabackup
```

Meaning:
Hot backup without stopping DB

---

### 10.2 Storage

MinIO (S3 compatible storage)

---

### 10.3 Flow:

* backup created
* pushed to MinIO
* restore playbook pulls it
* applies on target node

---

## 11. 🚨 LOGGING & TROUBLESHOOTING

### 11.1 Error logs

```bash
tail -f /var/log/mysql/error.log
```

---

### 11.2 Slow queries

```bash
tail -f /var/log/mysql/mysql-slow.log
```

Meaning:
Queries taking > 2 sec

---

### 11.3 Binlog check

```bash
ls -lh /var/lib/mysql/mysql-bin.*
```

---

## 12. 🧪 HEALTH CHECK CHECKLIST (DAILY)

```bash
mysql -e "SHOW SLAVE STATUS\G"
mysql -e "SHOW MASTER STATUS;"
systemctl status mysql
df -h
free -m
```

---

## 13. HOW TO MAP PLAYBOOK → REAL INFRA CHANGE

* "Configure MySQL Master" → installs Percona + sets master role
* "Restore Playbook" → restores old master using backup
* "GTID checks" → ensures replication consistency
* "requirements.yml updates" → installs Ansible roles

---

## 14. KEY UNDERSTANDING OF YOUR CURRENT CLUSTER

You currently have:

* 3–N MySQL nodes
* GTID-based replication
* Percona Server 8.0
* Ansible-managed configuration
* Orchestrator for failover
* MinIO backups
* xtrabackup restore system

---

## 15. FINAL SUMMARY (FOR NEW ENGINEER)

Step 1:

```bash
systemd-detect-virt
```

Step 2:

```bash
systemctl status mysql
```

Step 3:

```sql
SHOW SLAVE STATUS\G
SHOW MASTER STATUS;
```

Step 4:

```sql
SHOW DATABASES;
```

Step 5:
Check replication lag

Step 6:
Check orchestrator UI

Step 7:
Check Ansible repo + playbook mapping

---

## 16. 🏗️ ARCHITECTURE DIAGRAM

### 16.1 Logical Topology

```
                    ┌─────────────────────────────┐
                    │        ORCHESTRATOR         │
                    │   (Failover Controller)     │
                    └────────────┬────────────────┘
                                 │ monitors
                                 ▼
        ┌──────────────────────────────────────────────┐
        │              MYSQL MASTER                    │
        │           192.168.8.61                      │
        │                                              │
        │  - Accepts WRITE traffic                     │
        │  - Generates binlogs                         │
        │  - GTID source                               │
        └──────────────┬───────────────────────────────┘
                       │ replication
                       ▼
        ┌──────────────────────────────────────────────┐
        │              MYSQL REPLICA                   │
        │           192.168.8.77                      │
        │                                              │
        │  - READ ONLY                                 │
        │  - Applies relay logs                        │
        │  - Follows GTID stream                       │
        └──────────────────────────────────────────────┘
```

---

### 16.2 Data Flow

* Master writes → binlog
* Replica pulls → relay-bin
* Replica applies → InnoDB

---

### 16.3 Failure Detection Flow

* MySQL Master Failure
  → Orchestrator detects
  → Select best replica
  → Promote replica
  → Repoint replicas
  → Update topology

---

## 17. FAILOVER STEP-BY-STEP RUNBOOK

### 17.1 Preconditions

* GTID enabled
* Orchestrator running
* At least 1 healthy replica

---

### 17.2 Step 1: Detect failure

```bash
systemctl status mysql
orchestrator-client -c topology
```

---

### 17.3 Step 2: Confirm master down

```bash
mysql -h 192.168.8.61 -e "SELECT 1;"
```

---

### 17.4 Step 3: Check replicas

```bash
mysql -e "SHOW REPLICA STATUS\G"
```

---

### 17.5 Step 4: Orchestrator auto-promotion

Stops replication → Promotes replica

---

### 17.6 Step 5: New master verification

```bash
mysql -e "SHOW MASTER STATUS;"
```

---

### 17.7 Step 6: Repoint apps

* OLD: 192.168.8.61
* NEW: 192.168.8.77

---

### 17.8 Step 7: Validate cluster

```bash
orchestrator-client -c topology
```

---

## 18. RESTORE PROCEDURE (MINIO + XTRABACKUP)

### 18.1 Overview

* Percona XtraBackup
* MinIO object storage
* Ansible restore-playbook

---

### 18.2 Step 1: Identify backup

```bash
mc ls minio/backups/
```


### 18.3 Step 2: Download backup

```bash
mc cp minio/backups/resync_xxx /backup/
```


### 18.4 Step 3: Prepare restore node

```bash
systemctl stop mysql
rm -rf /var/lib/mysql/*
```

---

### 18.5 Step 4: Apply backup

```bash
xtrabackup --prepare --target-dir=/backup/resync_xxx
xtrabackup --copy-back --target-dir=/backup/resync_xxx
```

---

### 18.6 Step 5: Fix permissions

```bash
chown -R mysql:mysql /var/lib/mysql
```

---

### 18.7 Step 6: Start MySQL

```bash
systemctl start mysql
```

---

### 18.8 Step 7: Rejoin replication

```sql
CHANGE REPLICATION SOURCE TO ...;
START REPLICA;
```

---

## 19. WHAT HAPPENS DURING OUTAGE

### 19.1 Types of outages

* Master crash → writes stop
* Replica lag spike → stale reads
* Disk full → MySQL stops writing

---

### 19.2 System reaction flow

Failure → Health check → Orchestrator → Failover → Replica promoted → Traffic redirected

---

### 19.3 Key risk points

* GTID mismatch
* Incomplete replication
* Slow restore
* Binlog corruption

---

## 20. 📊 FULL PRODUCTION SRE CHECKLIST

### Daily checks

```bash
mysql -e "SHOW REPLICA STATUS\G"
mysql -e "SHOW MASTER STATUS;"
systemctl status mysql
df -h
free -m
```

---

### Replication health

* IO thread running
* SQL thread running
* Lag = 0

---

### Error logs

```bash
tail -f /var/log/mysql/error.log
```

---

### Backup validation

* Backup exists in MinIO
* Restore test successful

---

### Orchestrator

```bash
orchestrator-client -c status
```

### Emergency checklist

* Is master alive?
* Is replication broken?
* Is disk full?
* Is orchestrator responding?
