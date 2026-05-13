# **Ansible Role – Kong API Gateway (Standalone / DB-less / PostgreSQL Mode)**

<p align="center">
<img width="420" height="260" alt="kong-logo" src="https://github.com/Kong/docs.konghq.com/assets/kong-logo.png" />
</p>

---

| **Author** | **Created on** | **Version** | **Last Edited On** | **Pre Reviewer** | **L0 Reviewer** | **L1 Reviewer** | **L2 Reviewer** |
| ---------- | -------------- | ----------- | ------------------ | ---------------- | --------------- | --------------- | --------------- |
| Rehan      | 13-05-2026     | 1.0         | 13-05-2026         | Internal Team    | TBD             | TBD             | TBD             |

---

<details>
<summary><h2><strong>Table of Contents</strong></h2></summary>

* [Introduction](#introduction)
* [Role Objectives](#role-objectives)
* [Kong Architecture Overview](#kong-architecture-overview)
* [Deployment Modes Supported](#deployment-modes-supported)
* [OS Compatibility Design](#os-compatibility-design)
* [Directory Structure](#directory-structure)
* [Role Architecture](#role-architecture)
* [Role Variables](#role-variables)
* [Dynamic Inventory Design (AWS EC2)](#dynamic-inventory-design-aws-ec2)
* [Ansible Configuration (ansible.cfg)](#ansible-configuration-ansiblecfg)
* [Playbook Structure](#playbook-structure)
* [Task Execution Flow](#task-execution-flow)
* [Tasks Breakdown](#tasks-breakdown)
* [Templates (Jinja2)](#templates-jinja2)
* [Handlers](#handlers)
* [Service Management Design](#service-management-design)
* [Execution Workflow](#execution-workflow)
* [Proxy / Bastion Connectivity Design](#proxy--bastion-connectivity-design)
* [Flow Diagram](#flow-diagram)
* [Best Practices](#best-practices)
* [Troubleshooting Guide](#troubleshooting-guide)
* [Conclusion](#conclusion)
* [References](#references)

</details>

---

# **Introduction**

This document defines a complete automation framework for deploying **Kong API Gateway** using **Ansible Roles** in cloud environments (AWS EC2-based dynamic infrastructure).

The role is designed to:

* Install Kong Gateway
* Configure DB-less or PostgreSQL-backed mode
* Deploy declarative configuration (`kong.yml`)
* Configure systemd service for Kong
* Support dynamic AWS EC2 inventory
* Enable HA-ready architecture design

---

# **Role Objectives**

This Ansible role is built to achieve:

* Automated Kong installation from official `.deb` package
* Support for **DB-less mode (Declarative)** and **PostgreSQL mode**
* Dynamic configuration using Jinja2 templates
* Multi-node deployment using AWS EC2 dynamic inventory
* Standardized service lifecycle management
* Bastion-host-based secure SSH access

---

# **Kong Architecture Overview**

Kong Gateway is an API gateway that sits between clients and backend services.

It provides:

* Routing
* Authentication (JWT, OAuth, API keys)
* Rate limiting
* Logging and monitoring
* Load balancing

### Core Components:

* **Kong Proxy (8000)** → Handles client requests
* **Kong Admin API (8001)** → Management interface
* **Database (optional)** → PostgreSQL (for DB mode)

---

# **Deployment Modes Supported**

## 1. DB-less Mode (Recommended for Dev/Test)

* No database required
* Uses declarative config (`kong.yml`)
* Fast startup and lightweight

Configured via:

```ini
database = off
declarative_config = /etc/kong/kong.yml
```

---

## 2. PostgreSQL Mode (Production)

* Uses PostgreSQL backend
* Supports full Kong features (analytics, dynamic config)

Configured via:

```ini
database = postgres
pg_host = {{ kong_pg_host }}
```

---

# **OS Compatibility Design**

This role is primarily designed for:

* Ubuntu 20.04+
* Ubuntu 22.04+
* AWS Ubuntu 26.04 (your current setup)

It can be extended to:

* Debian
* Amazon Linux (with package modification)

---

# **Directory Structure**

```
ansible-kong-project/
│
├── ansible.cfg
├── inventories/
│   ├── dev/
│   ├── prod/
│   └── dr/
│
├── playbooks/
│   └── site.yml
│
└── roles/
    └── kong-standalone/
        ├── defaults/
        ├── vars/
        ├── tasks/
        ├── templates/
        ├── handlers/
        ├── files/
        └── meta/
```

---

# **Role Architecture**

| Component  | Purpose                                |
| ---------- | -------------------------------------- |
| defaults/  | Default variables (overridable safely) |
| vars/      | Hard-defined variables                 |
| tasks/     | Execution logic                        |
| templates/ | Jinja2 configuration files             |
| handlers/  | Service restart/reload logic           |
| files/     | Static files (unused currently)        |

---

# **Role Variables**

## defaults/main.yml

```yaml
kong_version: "3.9.0"
kong_mode: "dbless"
kong_admin_listen: "127.0.0.1:8001"
kong_proxy_listen: "0.0.0.0:8000"

kong_pg_host: "127.0.0.1"
kong_pg_port: 5432
kong_pg_database: "kong"
kong_pg_user: "kong"
kong_pg_password: "kong"

backend_host: "127.0.0.1"
```

### Explanation:

* `kong_version` → Version to install
* `kong_mode` → DB-less or postgres
* `kong_admin_listen` → Admin API binding
* `backend_host` → Upstream services target

---

# **Dynamic Inventory Design (AWS EC2)**

Your role uses AWS EC2 dynamic inventory plugin:

```yaml
plugin: amazon.aws.aws_ec2

regions:
  - us-east-2

filters:
  instance-state-name: running
  tag:Environment: Prod

keyed_groups:
  - key: tags.Role
    prefix: role

hostnames:
  - private-ip-address
```

### Explanation:

* Pulls live AWS instances
* Groups by `Role` tag → role_kong, role_postgres
* Uses private IPs for internal communication

---

# **Ansible Configuration (ansible.cfg)**

```ini
[defaults]
inventory = inventories/prod/aws_ec2.yml
roles_path = ./roles
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/.ssh/kong-key.pem

[inventory]
enable_plugins = amazon.aws.aws_ec2

[ssh_connection]
ssh_args = -o ProxyJump=ubuntu@3.135.65.89
```

### Explanation:

* `remote_user` → SSH user on EC2
* `roles_path` → where Ansible finds roles
* `ProxyJump` → bastion host access
* `host_key_checking` → disables prompt for known_hosts issues

---

# **Playbook Structure**

```yaml
- name: Deploy Kong HA Cluster
  hosts: role_kong
  become: yes
  serial: 1
  strategy: free

  roles:
    - kong-standalone
```

### Meaning:

* `hosts: role_kong` → targets AWS group
* `serial: 1` → rolling deployment (one node at a time)
* `strategy: free` → parallel execution allowed per task

---

# **Task Execution Flow**

```
main.yml
 ├── install.yml
 ├── postgres.yml (conditional)
 ├── config.yml
 └── service.yml
```

---

# **Tasks Breakdown**

## 1. Install Kong (install.yml)

```yaml
- name: Download Kong package
- name: Install Kong package
- name: Hold version
```

### Purpose:

* Downloads `.deb` package from Kong repo
* Prevents accidental upgrades

---

## 2. PostgreSQL Setup (postgres.yml)

Runs only if:

```yaml
when: kong_mode == "postgres"
```

Tasks:

* Install PostgreSQL
* Create DB user
* Create Kong database

---

## 3. Configuration (config.yml)

Tasks:

* Create `/etc/kong`
* Deploy `kong.conf`
* Deploy `kong.yml` (DB-less only)
* Deploy systemd service
* Reload systemd daemon

---

## 4. Service Management (service.yml)

* Run migrations (Postgres mode)
* Start Kong service
* Enable on boot
* Verify Admin API

---

# **Templates (Jinja2)**

## kong.conf.j2

```jinja2
database = {{ 'off' if kong_mode == 'dbless' else 'postgres' }}
```

### Logic:

* If DB-less → disables DB
* If postgres → enables DB connection

---

## kong.yml.j2 (Declarative config)

Defines:

* Services
* Routes
* Plugins (JWT, rate limiting, logging)

Example:

```yaml
services:
  - name: user-service
    url: http://{{ backend_host }}:5678
```

### Purpose:

This replaces database configuration in DB-less mode.

---

## kong.service.j2

Systemd unit file:

* Defines service startup
* Restart policies
* ExecStart / ExecReload commands

---

# **Handlers**

```yaml
- name: restart kong
  command: kong restart -c /etc/kong/kong.conf
```

### Purpose:

Triggered when:

* config file changes
* systemd file changes

---

# **Service Management Design**

Kong runs as a systemd service:

```bash
systemctl start kong
systemctl enable kong
```

Ensures:

* Auto restart on reboot
* Process supervision
* Logging via journalctl

---

# **Execution Workflow**

1. AWS inventory fetches EC2 instances
2. Hosts grouped by role_kong
3. Ansible connects via bastion host
4. Kong role executes:

   * Install
   * Configure
   * Start service
5. Verification via Admin API (8001)

---

# **Proxy / Bastion Connectivity Design**

Your environment uses:

```
Local Machine → Bastion (3.135.65.89) → Private EC2 (172.31.x.x)
```

### SSH config:

```bash
Host bastion
    HostName 3.135.65.89
    User ubuntu
    IdentityFile ~/.ssh/kong-key.pem

Host 172.31.*
    ProxyJump bastion
```

### Purpose:

* Secure private subnet access
* No public IP exposure for backend nodes

---

# **Flow Diagram**

```
User (Ansible CLI)
        |
        v
AWS EC2 Dynamic Inventory
        |
        v
Bastion Host (Public EC2)
        |
        v
Private EC2 Nodes (Kong)
        |
        v
Install → Configure → Start Kong
```

---

# **Best Practices**

| Practice                 | Benefit                |
| ------------------------ | ---------------------- |
| Use roles                | Reusability            |
| Use DB-less mode for dev | Faster deployments     |
| Use ProxyJump            | Secure access          |
| Use templates            | Dynamic config         |
| Use serial: 1            | Zero-downtime upgrades |
| Use handlers             | Controlled restarts    |

---

# **Troubleshooting Guide**

## 1. SSH Timeout

Cause:

* No bastion route OR security group blocked

Fix:

* Ensure `ProxyJump` configured
* Allow port 22 from bastion

---

## 2. Role not found error

Cause:

* Wrong roles_path

Fix:

```ini
roles_path = ./roles
```

---

## 3. Kong not starting

Cause:

* config error in kong.conf

Fix:

```bash
kong check
```

---

# **Conclusion**

This Ansible role provides a fully automated, production-ready framework for deploying Kong API Gateway with:

* Multi-mode support (DB-less + PostgreSQL)
* AWS dynamic infrastructure compatibility
* Secure bastion-based access
* Scalable HA-ready architecture
* Fully idempotent deployment model

---

# **References**

| Topic             | Link                                                                                                                                                                           |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Kong Gateway Docs | [https://docs.konghq.com/gateway/latest/](https://docs.konghq.com/gateway/latest/)                                                                                             |
| Ansible Roles     | [https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)         |
| AWS EC2 Inventory | [https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html) |
| Jinja2 Templates  | [https://jinja.palletsprojects.com/en/stable/](https://jinja.palletsprojects.com/en/stable/)                                                                                   |
