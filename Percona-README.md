PERCONA MYSQL HA CLUSTER – OPERATIONAL AUDIT & ONBOARDING GUIDE
1. 🔐 ACCESSING THE INFRASTRUCTURE
1.1 SSH Access Pattern

All nodes are accessed via IP:

ssh user@192.168.8.61
ssh user@192.168.8.77
ssh user@192.168.8.12
Typical roles:
MySQL Master
MySQL Replica
Orchestrator node
1.2 Verify you are on correct server
hostname
ip a
whoami
Meaning:
hostname → identifies server identity
ip a → confirms correct network interface/IP
whoami → confirms SSH user permissions
2. 🧠 IDENTIFY IF SYSTEM IS VM OR CONTAINER
2.1 Primary check
systemd-detect-virt
Output meaning:
Output	Meaning
kvm/vmware/xen	VM
docker/container	container
none	physical/VM without detection
2.2 cgroup inspection
cat /proc/1/cgroup
Interpretation:
/docker/ → container
/kubepods/ → Kubernetes pod
/system.slice/ → VM/bare metal
2.3 Kernel verification
uname -a
Meaning:

Shows OS kernel used by MySQL process (VM shares host kernel).

3. 🧾 MYSQL INSTALLATION VERIFICATION
3.1 Check MySQL service
systemctl status mysql
Meaning:
Confirms MySQL is installed as system service
Shows active/running state
3.2 Check Percona version
mysql --version

Expected:

Percona Server 8.0.x
3.3 Check MySQL process
ps -ef | grep mysqld
Meaning:

Confirms actual database engine running

4. 📂 UNDERSTANDING MYSQL DATA DIRECTORY
4.1 Main data location
/var/lib/mysql

This contains:

Core files:
File	Meaning
ibdata1	InnoDB system tablespace
ib_logfile / redo logs	crash recovery logs
undo_001/002	rollback logs
mysql-bin.*	binary logs (replication source)
relay-bin.*	replica logs
*.ibd	per-table data files
4.2 What your output shows
Example:
mysql-bin.000001
relay-bin.000006
ibdata1
undo_001
Meaning:
mysql-bin.* → Master replication logs
relay-bin.* → Replica receiving logs
ibdata1 → shared InnoDB metadata
undo_* → transaction rollback system
5. 🔁 REPLICATION ARCHITECTURE UNDERSTANDING

From your cluster:

Master:
192.168.8.61
Replica:
192.168.8.77
5.1 Check replication status (IMPORTANT)

Run:

mysql -e "SHOW SLAVE STATUS\G"

OR (MySQL 8+):

mysql -e "SHOW REPLICA STATUS\G"
Key fields:
Field	Meaning
Replica_IO_Running	receives logs from master
Replica_SQL_Running	applies logs
Seconds_Behind_Master	replication lag
Master_Host	upstream server
5.2 Check master status
mysql -e "SHOW MASTER STATUS;"
Meaning:

Shows:

current binlog file
position
GTID set

Used for replication sync

6. ⚙️ MYSQL CONFIGURATION AUDIT
6.1 Main config file
/etc/mysql/mysql.conf.d/mysqld.cnf
6.2 Important parameters
Replication:
server-id = X
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
Meaning:
Config	Meaning
server-id	unique ID per node
log_bin	enables replication logs
ROW	safest replication mode
GTID	global transaction tracking
Replica settings:
read_only = ON
super_read_only = ON
relay-log = relay-bin
Meaning:

Prevents writes on replica

Performance:
innodb_buffer_pool_size
max_connections
tmp_table_size

Controls memory + query performance

7. 📊 CHECK DISK DATA (WHAT DATA EXISTS)
7.1 List databases
mysql -e "SHOW DATABASES;"

Expected output:

mysql
performance_schema
sys
application DBs
7.2 Check tables inside DB
mysql -e "USE demo; SHOW TABLES;"
7.3 Inspect table size
mysql -e "
SELECT table_schema, table_name, data_length
FROM information_schema.tables;
"
7.4 Physical file mapping
ls -lh /var/lib/mysql/<db_name>/
Meaning:

Each .ibd file = one table

8. 📡 ORCHESTRATOR UNDERSTANDING

From your logs:

orchestrator table exists
GitHub orchestrator UI
failover automation
8.1 What Orchestrator does
Detects master failure
Promotes replica
Updates topology automatically
8.2 Check topology
orchestrator-client -c topology
9. ⚙️ ANSIBLE / SEMAPHORE AUDIT

You are using:

Semaphore UI
GitHub repo: OT-COE/LDCManager
Playbooks:
percona-server-role
percona-restore
9.1 What your jobs are doing
Example job:
Configure MySQL Master
Install Percona
Add repository
Install mysql role
Apply config
Enable replication
9.2 Key roles used
1. MySQL role
roles/mysql/tasks/Ubuntu.yml
Does:
installs Percona server
configures repo
installs dependencies
2. Restore playbook
restore-playbook.yml
Does:
uses xtrabackup
pulls backup from MinIO
restores old master
3. GTID validation playbook
Implement GTID checks and resets
Does:
validates replication consistency
resets GTID if mismatch
10. ☁️ BACKUP SYSTEM (MINIO + XTRABACKUP)
10.1 Backup tool
xtrabackup
Meaning:

Hot backup without stopping DB

10.2 Storage
MinIO (S3 compatible storage)
10.3 Flow:
backup created
pushed to MinIO
restore playbook pulls it
applies on target node
11. 🚨 LOGGING & TROUBLESHOOTING
11.1 Error logs
tail -f /var/log/mysql/error.log
11.2 Slow queries
tail -f /var/log/mysql/mysql-slow.log
Meaning:

Queries taking > 2 sec

11.3 Binlog check
ls -lh /var/lib/mysql/mysql-bin.*
12. 🧪 HEALTH CHECK CHECKLIST (DAILY)

Run:

mysql -e "SHOW SLAVE STATUS\G"
mysql -e "SHOW MASTER STATUS;"
systemctl status mysql
df -h
free -m
13. 🧭 HOW TO MAP PLAYBOOK → REAL INFRA CHANGE
When you see:
"Configure MySQL Master"

→ installs Percona + sets master role

"Restore Playbook"

→ restores old master using backup

"GTID checks"

→ ensures replication consistency

"requirements.yml updates"

→ installs Ansible roles (mysql role version changes)

14. 🧠 KEY UNDERSTANDING OF YOUR CURRENT CLUSTER
You currently have:
3–N MySQL nodes
GTID-based replication
Percona Server 8.0
Ansible-managed configuration
Orchestrator for failover
MinIO backups
xtrabackup restore system
15. 📌 FINAL SUMMARY (FOR NEW ENGINEER)

If someone joins fresh, they should do:

Step 1:

Check server type

systemd-detect-virt
Step 2:

Check MySQL

systemctl status mysql
Step 3:

Check role

SHOW SLAVE STATUS\G
SHOW MASTER STATUS;
Step 4:

Check data

SHOW DATABASES;
Step 5:

Check replication lag

Step 6:

Check orchestrator UI

Step 7:

Check Ansible repo + playbook mapping

16. 🏗️ ARCHITECTURE DIAGRAM (MASTER / REPLICA / ORCHESTRATOR FLOW)
16.1 Logical Topology
                    ┌─────────────────────────────┐
                    │        ORCHESTRATOR         │
                    │   (Failover Controller)     │
                    └────────────┬────────────────┘
                                 │
                                 │ monitors
                                 ▼
        ┌──────────────────────────────────────────────┐
        │              MYSQL MASTER                    │
        │           192.168.8.61                      │
        │                                              │
        │  - Accepts WRITE traffic                     │
        │  - Generates binlogs                        │
        │  - GTID source                              │
        └──────────────┬───────────────────────────────┘
                       │
                       │ replication (binlog stream)
                       ▼
        ┌──────────────────────────────────────────────┐
        │              MYSQL REPLICA                   │
        │           192.168.8.77                      │
        │                                              │
        │  - READ ONLY                                │
        │  - Applies relay logs                       │
        │  - Follows GTID stream                     │
        └──────────────────────────────────────────────┘
16.2 Data Flow
Master writes → binlog.00000x
Replica pulls → relay-bin.00000x
Replica applies → InnoDB tables
Orchestrator monitors lag + health
16.3 Failure Detection Flow
MySQL Master Failure
        ↓
Orchestrator detects (health check fail)
        ↓
Select best replica (lowest lag)
        ↓
Promote replica → becomes new master
        ↓
Repoint remaining replicas
        ↓
Update topology automatically
17. 🔁 FAILOVER STEP-BY-STEP RUNBOOK
17.1 Preconditions
GTID enabled
Orchestrator running
At least 1 healthy replica
17.2 Step 1: Detect failure
systemctl status mysql

OR

orchestrator-client -c topology
17.3 Step 2: Confirm master down
mysql -h 192.168.8.61 -e "SELECT 1;"

If no response → failure confirmed

17.4 Step 3: Check replicas
mysql -e "SHOW REPLICA STATUS\G"

Check:

IO Running = Yes
SQL Running = Yes
17.5 Step 4: Orchestrator auto-promotion

Orchestrator executes:

Stops replication
Promotes replica
17.6 Step 5: New master verification
mysql -e "SHOW MASTER STATUS;"
17.7 Step 6: Repoint apps

Update application connection:

OLD: 192.168.8.61
NEW: 192.168.8.77
17.8 Step 7: Validate cluster
orchestrator-client -c topology
18. ♻️ RESTORE PROCEDURE (MINIO → PRODUCTION USING XTRABACKUP)
18.1 Overview

Restore uses:

Percona XtraBackup
MinIO object storage
Ansible restore-playbook
18.2 Step 1: Identify backup
mc ls minio/backups/
18.3 Step 2: Download backup
mc cp minio/backups/resync_xxx /backup/
18.4 Step 3: Prepare restore node
systemctl stop mysql
rm -rf /var/lib/mysql/*
18.5 Step 4: Apply backup
xtrabackup --prepare --target-dir=/backup/resync_xxx
xtrabackup --copy-back --target-dir=/backup/resync_xxx
18.6 Step 5: Fix permissions
chown -R mysql:mysql /var/lib/mysql
18.7 Step 6: Start MySQL
systemctl start mysql
18.8 Step 7: Rejoin replication
CHANGE REPLICATION SOURCE TO ...;
START REPLICA;
19. WHAT HAPPENS DURING OUTAGE (REAL PRODUCTION VIEW)
19.1 Types of outages
1. Master crash
Writes stop
Orchestrator triggers failover
2. Replica lag spike
Read traffic still works
Risk of stale reads
3. Disk full
MySQL stops writing
Binlog stops
19.2 System reaction flow
Failure occurs
   ↓
MySQL health check fails
   ↓
Orchestrator evaluates cluster
   ↓
Auto failover OR alert triggered
   ↓
Replica promoted
   ↓
Traffic redirected
19.3 Key risk points
GTID mismatch
Incomplete replication
Slow backup restore
Binlog corruption
20. 📊 FULL PRODUCTION SRE CHECKLIST
20.1 Daily checks
mysql -e "SHOW REPLICA STATUS\G"
mysql -e "SHOW MASTER STATUS;"
systemctl status mysql
df -h
free -m
20.2 Replication health
IO thread running
SQL thread running
Lag = 0
20.3 Disk health
df -h
du -sh /var/lib/mysql
20.4 Slow query monitoring
tail -f /var/log/mysql/mysql-slow.log
20.5 Error monitoring
tail -f /var/log/mysql/error.log
20.6 Backup validation
Last backup exists in MinIO
Restore test successful
20.7 Orchestrator health
orchestrator-client -c status
20.8 Security checks
read_only=ON on replicas
No anonymous users
No remote root access
20.9 Performance checks
Buffer pool usage
Query latency
Connection saturation
20.10 Weekly checks
Binlog rotation
Backup restore test
Failover simulation
Disk cleanup
20.11 Emergency checklist
Is master alive?
Is replication broken?
Is lag increasing?
Is disk full?
Is orchestrator responding?
📌 FINAL NOTE

This system is:

✔ GTID-based MySQL replication
✔ Percona Server 8.0
✔ Ansible-managed infrastructure
✔ Orchestrator-controlled failover
✔ MinIO-based backup system
✔ XtraBackup restore pipeline
Good question — this is exactly where a **Percona HA setup moves from “working system” → “production-grade platform”**.


# 1. CURRENT STATE (WHAT YOU HAVE NOW)

From your logs + configs:

### You already have:

* Percona Server 8.0
* GTID-based replication
* Master / Replica architecture
* Orchestrator for failover
* Ansible-based provisioning
* Semaphore CI execution
* MinIO + XtraBackup backup system
* Slow query + error logs enabled

---

### So current maturity level:

> ✅ “Semi-automated HA database system”

NOT fully automated yet.

# 2. ⚠️ CURRENT GAPS / RISKS

These are the real production gaps I see in your setup:

## ❌ 2.1 Manual dependency still exists

Even with Ansible:

* Failover is not fully user-free
* App cutover is likely manual
* Restore execution still triggered manually
* Playbooks depend on humans selecting jobs

👉 Risk: slow recovery during outage

## 2.2 Orchestrator not fully integrated with app routing

You have:

* Orchestrator detects failover

But missing:

* automatic load balancer update
* automatic DNS switch
* automatic service discovery update

---

## 2.3 No unified control plane

Right now systems are separate:

| Component    | Tool               |
| ------------ | ------------------ |
| Provisioning | Ansible            |
| Failover     | Orchestrator       |
| Backup       | XtraBackup + MinIO |
| Execution    | Semaphore          |

Problem: No single brain controlling everything

---

## 2.4 No automatic recovery workflows

Missing automation:

* Auto rebuild failed replica
* Auto re-seed from master
* Auto rejoin cluster after failure

---

## ❌ 2.5 No continuous validation

No evidence of:

* automated replication testing
* automated failover testing (chaos testing)
* restore verification jobs

---

## ❌ 2.6 Observability gaps

You have logs, but missing:

* Prometheus metrics integration
* Grafana dashboards
* Alert correlation (lag vs disk vs CPU)

---

## ❌ 2.7 Security gaps (common in such setups)

Likely missing:

* TLS enforced replication (you have TLS disabled in config)
* encrypted backups
* secrets management (Vault not seen)

---

# 3. 🚀 IMPROVEMENTS ROADMAP

I’ll split this into 3 phases:

---

# PHASE 1 — Stabilization (quick wins)

## 3.1 Auto failover integration with app layer

### Add:

* VIP (keepalived) OR
* HAProxy OR
* ProxySQL

Goal:
Apps never connect to raw IPs

---

## 3.2 Enable TLS replication

In MySQL:

```ini id="tls1"
ssl-ca
ssl-cert
ssl-key
```

👉 Prevents data sniffing between nodes

---

## 3.3 Central monitoring

Add:

* Prometheus exporter for MySQL
* Grafana dashboards
* Alertmanager

---

## 3.4 Backup validation automation

Right now backups exist.

Add:

👉 “restore test job every week”

---

## 3.5 Standardize Ansible execution

Move from:

* multiple ad-hoc Semaphore templates

To:

* single pipeline:

```
deploy → configure → verify → register
```

---

# 🟠 PHASE 2 — Automation (major upgrade)

## 3.6 Auto-rebuild of failed replicas

When replica dies:

```text
Detect failure
→ provision new VM
→ install Percona
→ re-seed from master backup
→ attach to replication
```

👉 Fully hands-off recovery

---

## 3.7 Self-healing replication

If replication breaks:

* auto stop replica
* auto re-clone from master
* auto rejoin GTID

---

## 3.8 Orchestrator + ProxySQL integration

Instead of manual routing:

```text
Orchestrator promotes master
↓
ProxySQL updates query routing automatically
↓
App sees no downtime
```

---

## 3.9 Unified deployment pipeline

Replace multiple jobs:

👉 One GitOps-style flow:

```text
Git commit → Ansible → validation → rollout
```

---

## 3.10 Automated failover testing (CHAOS)

Weekly job:

* kill master
* observe failover
* validate recovery time

---

# 🔴 PHASE 3 — FULL AUTONOMOUS PLATFORM (Target State)

This is “Google-grade DB automation”

---

## 3.11 Control Plane (VERY IMPORTANT)

Introduce:

### DB Control Service (custom or tooling layer)

It handles:

* orchestrator API
* ansible execution
* backup restore triggers
* monitoring alerts

👉 One brain for everything

---

## 3.12 Event-driven automation

Instead of manual runs:

```text
MySQL failure event
→ webhook triggered
→ orchestrator + ansible + proxy update
→ auto recovery
```

---

## 3.13 Auto scaling replicas

When load increases:

* spin new replicas automatically
* attach to GTID stream

---

## 3.14 Self-healing full stack

System automatically handles:

| Failure        | Auto action         |
| -------------- | ------------------- |
| Master crash   | failover            |
| Replica crash  | rebuild             |
| Disk full      | alert + rotate logs |
| Lag increase   | throttle writes     |
| Backup failure | retry + alert       |

---

## 3.15 Zero-touch restore system

Instead of manual restore:

```text
Select timestamp
→ click restore
→ system rebuilds full cluster
```

---

# 4. 🤖 HOW TO MAKE IT FULLY AUTOMATED (FINAL ARCHITECTURE)

## 4.1 Target architecture

```text
                ┌────────────────────────┐
                │   CONTROL PLANE        │
                │ (NEW: Automation Brain)│
                └──────────┬─────────────┘
                           │
     ┌─────────────────────┼─────────────────────┐
     │                     │                     │
     ▼                     ▼                     ▼
Orchestrator        Ansible Engine        Monitoring System
(failover)          (provisioning)        (Prometheus/Grafana)
     │                     │                     │
     └─────────────┬───────┴─────────────┬─────┘
                   ▼                     ▼
           ProxySQL / HAProxy     MinIO Backup System
                   │
                   ▼
            MySQL Cluster
```

---

## 4.2 Automation layers required

### Layer 1 — Infrastructure automation

* Terraform (VM provisioning)
* Ansible (configuration)

---

### Layer 2 — DB automation

* Orchestrator (failover)
* ProxySQL (routing)

---

### Layer 3 — Data automation

* XtraBackup scheduled
* restore automation

---

### Layer 4 — Observability automation

* Prometheus
* Alertmanager
* Grafana

---

### Layer 5 — Event automation

* Webhooks
* Event triggers
* Runbooks-as-code

---

# 5. 🧭 FINAL SUMMARY (WHERE YOU ARE VS WHERE YOU SHOULD BE)

## CURRENT STATE:

✔ Semi-automated MySQL HA
✔ Manual dependency still exists
✔ Separate systems working independently

---

## TARGET STATE:

🚀 Fully autonomous DB platform

* self-healing
* auto failover
* auto recovery
* auto restore
* zero manual intervention

---

# 💡 MOST IMPORTANT IMPROVEMENT (IF YOU DO ONLY ONE THING)

👉 Add this first:

> **ProxySQL or HAProxy in front of MySQL**

Because it immediately enables:

* zero-downtime failover
* no app changes during master switch
* clean abstraction layer

---
To achieve **full automation (self-healing Percona HA platform)** from your current setup, you don’t “add one tool” — you build **automation layers on top of what you already have**.

I’ll show you **exactly how to evolve your current system step-by-step into a fully automated one**.

---

# 🚀 TARGET: FULLY AUTOMATED PERCONA HA PLATFORM

You currently have:

* Percona MySQL cluster
* GTID replication
* Orchestrator (failover engine)
* Ansible (provisioning engine)
* MinIO + XtraBackup (backup system)

👉 What’s missing is **integration + automation glue**

---

# 🧭 HOW TO ACHIEVE FULL AUTOMATION (STEP-BY-STEP ROADMAP)

---

# 🟡 PHASE 1 — MAKE FAILOVER “APP INVISIBLE”

## 🎯 Goal:

When master fails → apps automatically switch with ZERO manual action

---

## 1. Add Proxy Layer (MOST IMPORTANT STEP)

### Install ONE of:

* ProxySQL (recommended)
* OR HAProxy

---

## 🔧 Architecture change:

Before:

```
App → MySQL IP (manual change needed)
```

After:

```
App → ProxySQL → MySQL Cluster
```

---

## 🔥 Why this is critical:

Without this:

* failover happens
* but apps still point to old master ❌

With this:

* failover happens
* proxy reroutes automatically ✅

---

## 2. Connect ProxySQL with Orchestrator

When Orchestrator promotes new master:

👉 it must trigger ProxySQL update

---

### Example flow:

```text
Orchestrator detects failure
        ↓
Promotes replica → new master
        ↓
Webhook triggers ProxySQL script
        ↓
ProxySQL updates writer hostgroup
        ↓
App continues without downtime
```

---

## 3. How to implement it

### A. Enable Orchestrator hooks:

```bash
PostFailoverProcesses
```

### B. Add script:

```bash
update_proxysql_writer.sh
```

### C. Script does:

* removes old master
* adds new master
* updates writer group

---

# 🟠 PHASE 2 — AUTO-RECOVERY (SELF HEALING)

## 🎯 Goal:

If a server dies → system rebuilds itself

---

## 4. Auto-rebuild replica (CRITICAL UPGRADE)

### Problem today:

If replica dies → manual rebuild

---

### Solution:

Add automation:

```text
Replica failure detected
        ↓
Ansible triggered automatically
        ↓
Provision new VM (or reuse old)
        ↓
Install Percona
        ↓
Clone from master using XtraBackup
        ↓
Attach to replication
```

---

## 🔧 Tools needed:

* Ansible dynamic inventory
* Orchestrator hooks OR Prometheus alerts
* XtraBackup restore playbook

---

## 5. Implement event trigger

Use one:

* Prometheus Alertmanager webhook
* Orchestrator hooks
* Semaphore API trigger

---

Example:

```text
Replica down alert
→ trigger Ansible job
→ rebuild node automatically
```

---

# 🔴 PHASE 3 — AUTO BACKUP + AUTO RESTORE PIPELINE

## 🎯 Goal:

One-click or fully automatic restore

---

## 6. Automate backup validation

Right now:
✔ backups exist
❌ not validated automatically

---

### Add job:

```text
Weekly:
  - restore backup in sandbox
  - run health checks
  - destroy sandbox
```

---

## 7. Auto restore system

Instead of manual restore:

### Add workflow:

```text
Backup selected (MinIO)
        ↓
Restore playbook triggered
        ↓
Cluster rebuilt automatically
        ↓
Replication reattached
```

---

## 8. Add “restore API trigger”

Example:

```bash
POST /restore
{
  "timestamp": "latest"
}
```

This triggers Ansible playbook.

---

# 🟣 PHASE 4 — EVENT-DRIVEN AUTOMATION (THE REAL UPGRADE)

## 🎯 Goal:

No humans involved in DB operations

---

## 9. Introduce event pipeline

You need:

### Event sources:

* MySQL failure
* replication lag
* disk full
* node down

---

### Event processor:

* Alertmanager OR custom webhook service

---

### Event actions:

| Event        | Action           |
| ------------ | ---------------- |
| Master down  | failover         |
| Replica down | rebuild          |
| Lag high     | throttle / alert |
| Disk full    | cleanup          |

---

## 10. Example flow:

```text
MySQL crash detected
        ↓
Alertmanager webhook
        ↓
Orchestrator failover
        ↓
ProxySQL update
        ↓
Ansible rebuild replica
        ↓
Monitoring validates cluster
```

---

# 🟢 PHASE 5 — CONTROL PLANE (FINAL LEVEL)

## 🎯 Goal:

ONE SYSTEM controls everything

---

## 11. Build DB Control Plane

This is a lightweight service (can be Python/Go):

It manages:

* Orchestrator API
* Ansible execution
* Backup restore
* ProxySQL updates

---

## 12. Control plane responsibilities:

### It becomes the “brain”

```text
Input:
  failure / backup / scaling event

Output:
  execute automation workflows
```

---

## 13. Example actions:

* promote replica
* rebuild node
* restore backup
* update routing
* trigger alerts

---

# 🧠 FINAL TARGET ARCHITECTURE

```
                    ┌──────────────────────┐
                    │   CONTROL PLANE      │
                    │ (Automation Brain)   │
                    └─────────┬────────────┘
                              │
     ┌────────────────────────┼────────────────────────┐
     ▼                        ▼                        ▼
Orchestrator           Ansible Engine          Monitoring System
(failover)             (provisioning)          (Prometheus)
     │                        │                        │
     └──────────────┬─────────┴──────────┬────────────┘
                    ▼                    ▼
              ProxySQL Layer       MinIO Backup
                    │
                    ▼
            MySQL Cluster (Self-healing)
```

---

# ⚡ WHAT YOU WILL ACHIEVE AFTER THIS

## ✔ Fully automated failover

No manual intervention

## ✔ Self-healing replicas

Dead node → auto rebuild

## ✔ Zero downtime app switching

Proxy handles routing

## ✔ One-click restore

Backup → full cluster restore

## ✔ Event-driven DB ops

Everything triggered by signals

---

# 💡 MOST IMPORTANT INSIGHT

If you do only ONE thing first:

> 🔥 Implement ProxySQL + Orchestrator integration

Because that unlocks:

* real zero downtime
* automatic routing
* foundation for full automation

---
