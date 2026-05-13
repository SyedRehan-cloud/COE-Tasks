# # **Ansible Role – Kong API Gateway (Complete End-to-End Implementation Guide)**

<p align="center">
<img width="420" height="260" alt="kong-logo" src="https://github.com/Kong/docs.konghq.com/assets/kong-logo.png" />
</p>

---

| **Author** | **Created on** | **Version** | **Last Updated** |
| ---------- | -------------- | ----------- | ---------------- |
| Rehan      | 13-05-2026     | 2.0         | 13-05-2026       |

---

# 🧭 Table of Contents

* Introduction
* Architecture Overview
* Directory Structure (FULL EXPLAINED)
* ansible.cfg (Line-by-line explanation)
* Inventory (AWS Dynamic Inventory FULL)
* SSH Bastion Setup (Critical)
* Playbook (site.yml FULL FLOW)
* Role: kong-standalone (FULL FILE BREAKDOWN)

  * defaults
  * vars
  * tasks
  * templates
  * handlers
* Execution Flow (Step-by-step runtime)
* Commands to Run Project
* Debugging Guide (your errors fixed)
* Security Design
* Final Architecture Flow

---

# 🧠 1. Introduction

This project automates the deployment of **Kong API Gateway** using Ansible on AWS EC2 infrastructure with:

* Dynamic EC2 inventory
* Bastion host SSH access
* DB-less + PostgreSQL modes
* Systemd-based service management
* Plugin-based API gateway configuration

---

# 🏗️ 2. Architecture Overview

```
Local Machine (Ansible)
        |
        v
Bastion Host (Public EC2 - 3.135.65.89)
        |
        v
Private EC2 Instances (172.31.x.x)
        |
        v
Kong Gateway + Backend Services
```

---

# 📁 3. FULL DIRECTORY STRUCTURE (EXPLAINED)

```
ansible-kong-project/
│
├── ansible.cfg                # Ansible configuration
├── inventories/               # Dynamic AWS inventory
│   ├── dev/
│   ├── dr/
│   └── prod/
│
├── playbooks/
│   ├── site.yml               # MAIN PLAYBOOK
│
└── roles/
    └── kong-standalone/       # MAIN ROLE
        ├── defaults/
        │   └── main.yml       # default variables
        │
        ├── vars/
        │   └── main.yml       # fixed variables
        │
        ├── tasks/
        │   ├── main.yml       # task orchestrator
        │   ├── install.yml    # installation
        │   ├── postgres.yml   # DB setup
        │   ├── config.yml     # config generation
        │   └── service.yml    # service control
        │
        ├── templates/
        │   ├── kong.conf.j2   # main config
        │   ├── kong.yml.j2     # declarative config
        │   └── kong.service.j2 # systemd service
        │
        ├── handlers/
        │   └── main.yml       # restart/reload handlers
        │
        ├── files/
        └── meta/
```

---

# ⚙️ 4. ansible.cfg (DETAILED)

```ini
[defaults]
inventory = inventories/prod/aws_ec2.yml
roles_path = ./roles
host_key_checking = False
remote_user = ubuntu
private_key_file = /home/rehan/.ssh/kong-key.pem

[inventory]
enable_plugins = amazon.aws.aws_ec2

[ssh_connection]
ssh_args = -F /home/rehan/.ssh/config
```

### Meaning:

* Uses AWS dynamic inventory
* Connects via bastion host
* Uses Ubuntu user
* Uses EC2 private key

---

# 🌩️ 5. AWS INVENTORY (FULL EXPLANATION)

```yaml
plugin: amazon.aws.aws_ec2

regions:
  - us-east-2

filters:
  instance-state-name: running
  tag:Environment: Prod

hostnames:
  - private-ip-address

keyed_groups:
  - key: tags.Role
    prefix: role
```

### What happens:

* Pulls all running EC2 instances
* Filters only Production
* Groups machines by Role tag:

  * role_kong
  * role_postgres

---

# 🔐 6. SSH BASTION SETUP (IMPORTANT)

```bash
mkdir -p ~/.ssh
touch ~/.ssh/config
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config

cat <<EOF > ~/.ssh/config

Host bastion
    HostName 3.135.65.89
    User ubuntu
    IdentityFile ~/.ssh/kong-key.pem

Host 172.31.*
    User ubuntu
    IdentityFile ~/.ssh/kong-key.pem
    ProxyJump bastion
EOF
```

### Purpose:

* All private EC2s accessed via bastion
* No public access to backend servers

---

# 🚀 7. PLAYBOOK (site.yml FULL FLOW)

```yaml
- name: Deploy Kong API Gateway
  hosts: role_kong
  become: yes
  serial: 1
  strategy: free

  roles:
    - kong-standalone
```

### Meaning:

* `serial: 1` → one server at a time (safe rollout)
* `role_kong` → dynamic inventory group
* `become: yes` → sudo required

---

# 🧩 8. ROLE BREAKDOWN (FULL DETAIL)

---

## 📌 defaults/main.yml

```yaml
kong_version: "3.9.0"
kong_mode: "dbless"
kong_admin_listen: "127.0.0.1:8001"
kong_proxy_listen: "0.0.0.0:8000"
backend_host: "127.0.0.1"
```

### Purpose:

Default safe configuration values

---

## 📌 vars/main.yml

```yaml
kong_version: "3.9.0"
```

### Purpose:

Hard override variables (not easily changed)

---

# 🛠️ TASKS FLOW

## 📌 tasks/main.yml

```yaml
- import_tasks: install.yml

- import_tasks: postgres.yml
  when: kong_mode == "postgres"

- import_tasks: config.yml

- import_tasks: service.yml
```

### Flow:

1. Install Kong
2. Setup DB if required
3. Configure files
4. Start service

---

## 📌 install.yml

### What it does:

```yaml
- Download Kong .deb package
- Install package using apt
- Hold version to prevent auto-upgrade
```

### Why:

Prevents version mismatch in clusters

---

## 📌 postgres.yml

### Runs only if:

```yaml
kong_mode == "postgres"
```

### Tasks:

* Install PostgreSQL
* Start PostgreSQL service
* Create Kong database user
* Create Kong database

### Current Working Implementation:

```yaml
---
- name: Install PostgreSQL
  apt:
    name:
      - postgresql
      - postgresql-contrib
    state: present
    update_cache: yes

- name: Start PostgreSQL
  service:
    name: postgresql
    state: started
    enabled: yes

- name: Create Kong DB user
  shell: |
    sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='{{ kong_pg_user }}'" | grep -q 1 || \
    sudo -u postgres psql -c "CREATE USER {{ kong_pg_user }} WITH PASSWORD '{{ kong_pg_password }}';"
  args:
    executable: /bin/bash

- name: Create Kong database
  shell: |
    sudo -u postgres psql -lqt | cut -d \| -f 1 | grep -qw {{ kong_pg_database }} || \
    sudo -u postgres psql -c "CREATE DATABASE {{ kong_pg_database }} OWNER {{ kong_pg_user }};"
  args:
    executable: /bin/bash
```

### Why shell commands were used:

Initially, `become_user: postgres` with `postgresql_user` and `postgresql_db` modules caused privilege escalation temp-file permission issues:

```bash
Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user
```

Using `sudo -u postgres` shell commands resolved the issue and kept the setup idempotent.

---

## 📌 config.yml

### Tasks:

* Create `/etc/kong`
* Deploy kong.conf
* Deploy kong.yml (DB-less only)
* Deploy systemd service
* Reload systemd

---

## 📌 service.yml

* Run migrations (Postgres mode)
* Start Kong
* Enable service
* Verify API (8001)

---

# 🧾 9. TEMPLATES (JINJA2)

---

## 📌 kong.conf.j2

```jinja2
database = {{ 'off' if kong_mode == 'dbless' else 'postgres' }}
```

### Meaning:

Switch DB mode dynamically

---

## 📌 kong.yml.j2

Defines:

* Services
* Routes
* Plugins:

  * JWT authentication
  * Rate limiting
  * HTTP logging

### Purpose:

This is **DB-less configuration engine**

---

## 📌 kong.service.j2

Systemd service file:

* Start Kong on boot
* Restart on failure
* Reload support

---

# 🔁 10. HANDLERS

```yaml
- name: restart kong
  command: kong restart -c /etc/kong/kong.conf
```

### Triggered when:

* config changes
* service file changes

---

# ▶️ 11. HOW TO RUN (COMMANDS)

## Step 1: Check inventory

```bash
ansible-inventory --graph
```

## Step 2: Test SSH

```bash
ansible all -m ping
```

## Step 3: Run playbook

```bash
ansible-playbook playbooks/site.yml
```

## Step 4: Verify Kong

```bash
curl http://localhost:8001
systemctl status kong
```

---

# 🐛 12. DEBUGGING (YOUR ISSUES FIXED)

---

## ❌ Issue 1: Role not found

✔ Fix:

```ini
roles_path = ./roles
```

---

## ❌ Issue 2: SSH timeout

✔ Cause:

* Missing ProxyJump
* SG blocking port 22

---

## ❌ Issue 3: Identity file error

✔ Fix:

```bash
chmod 600 ~/.ssh/kong-key.pem
```

---

## ❌ Issue 4: Ansible can’t find SSH config

### Error:

```bash
Can't open user config file ~/.ssh/config
```

✔ Fix:

```bash
mkdir -p ~/.ssh
touch ~/.ssh/config
```

✔ Updated ansible.cfg:

```ini
[ssh_connection]
ssh_args = -F /home/rehan/.ssh/config
```

---

## ❌ Issue 5: PostgreSQL become_user permission issue

### Error:

```bash
Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user
```

### Cause:

Using:

```yaml
become_user: postgres
```

with PostgreSQL Ansible modules caused privilege escalation issues.

✔ Fix:

Replaced PostgreSQL modules with:

```bash
sudo -u postgres psql ...
```

shell-based commands.

---

# 🔐 13. SECURITY DESIGN

* Bastion only public access
* Private EC2 fully isolated
* Kong Admin API not exposed publicly
* SSH key-based authentication only

---

# 📊 14. FINAL FLOW

```
Ansible CLI
   ↓
AWS Dynamic Inventory
   ↓
Bastion Host
   ↓
Private EC2 (Kong)
   ↓
Install → Configure → Run → Validate
```

---

# 🎯 FINAL SUMMARY

This role is a:

✔ Production-ready API gateway automation
✔ AWS dynamic infrastructure deployment system
✔ Bastion-secured architecture
✔ Multi-mode Kong deployment (DB-less + DB)
✔ Fully idempotent Ansible role
