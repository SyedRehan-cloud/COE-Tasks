# Ansible Role – Kong API Gateway (Production-Ready End-to-End Guide)

<p align="center">
<img width="420" height="260" alt="kong-logo" src="https://github.com/Kong/docs.konghq.com/assets/kong-logo.png" />
</p>


| Field           | Value                  |
| --------------- | ---------------------- |
| Author          | Rehan                  |
| Technology      | Ansible + Kong Gateway |
| Deployment Type | AWS EC2                |
| Supported Modes | DBLess + PostgreSQL    |
| Kong Version    | 3.9.0                  |
| Last Updated    | 19-05-2026             |


# 1. WHAT IS THIS PROJECT?

This project automates the deployment and configuration of:

#  Kong API Gateway (Community Edition)

using:

* Ansible Automation
* AWS EC2 Infrastructure
* CI/CD Pipeline (Semaphore)
* systemd Service Management

---

# 2. WHAT THIS AUTOMATION SOLVES

Without automation:

| Problem                           | Impact                  |
| --------------------------------- | ----------------------- |
| Manual Kong installation          | Configuration mismatch  |
| PostgreSQL startup race condition | Kong boot failure       |
| Manual migrations                 | Human errors            |
| Service configuration differences | Environment instability |
| Repeated setup work               | Slow deployments        |

---

With this automation:

One-command deployment
Fully repeatable infrastructure
Production-safe service management
DBLess + PostgreSQL mode support
CI/CD friendly
Environment-consistent setup

# 3. SUPPORTED DEPLOYMENT MODES

This role supports TWO architectures.

# MODE 1 → DBLESS MODE (CURRENT ACTIVE MODE)

```yaml
kong_mode: "dbless"
```

## What happens internally?

Kong does NOT use PostgreSQL.

Instead:

* Kong reads all routes/services/plugins from:

```bash
/etc/kong/kong.yml
```

## Architecture

```text
Client
   ↓
Kong Gateway
   ↓
Backend Services
```

---

## Advantages

Simpler setup
Faster deployments
No migrations required
Lower infrastructure cost
Easier CI/CD

---

## Limitations

No Kong Manager
No runtime API persistence
Changes require config redeploy

---

# MODE 2 → POSTGRES MODE

```yaml
kong_mode: "postgres"
```

---

## What happens internally?

Kong stores all configuration in PostgreSQL.

Kong will:

* Install PostgreSQL
* Create DB + user
* Run migrations
* Persist routes/plugins/services in DB

---

## Architecture

```text
Client
   ↓
Kong Gateway
   ↓
PostgreSQL
   ↓
Backend Services
```

---

## Advantages
HA ready
Dynamic Admin API changes
Persistent configuration
Enterprise-ready architecture

---

## Limitations

 More complex
 DB maintenance required
 Migrations required

---

# 4. HOW TO SWITCH BETWEEN DBLESS AND POSTGRES

File:

```bash
roles/kong-api-gateway/defaults/main.yml
```

---

## Enable DBLess

```yaml
kong_mode: "dbless"
```

---

## Enable PostgreSQL

```yaml
kong_mode: "postgres"
```

---

# IMPORTANT INTERNAL BEHAVIOR

| Feature               | DBLess     | PostgreSQL |
| --------------------- | ---------- | ---------- |
| PostgreSQL install    |  skipped  |  executed |
| Migrations            |  skipped  |  executed |
| kong.yml used         |  yes      |  no       |
| kong.conf DB settings |  disabled |  enabled  |

---

# 5. FULL PROJECT STRUCTURE

```text
kong-setup/
│
├── ansible.cfg
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
│   ├── install.yml
│   ├── postgres.yml
│   ├── config.yml
│   ├── service.yml
│   └── verify.yml
│
├── templates/
│   ├── kong.conf.j2
│   ├── kong.service.j2
│   └── kong.yml.j2
│
└── vars/
    └── main.yml
```

---

# 6. FILE-BY-FILE EXPLANATION

---

# ansible.cfg

Path:

```bash
kong-setup/ansible.cfg
```

---

## Purpose

Controls global Ansible behavior.

---

## Backend Functionality

Defines:

* inventory source
* SSH user
* SSH key
* role lookup path
* inventory plugins

---

## FINAL VERSION

```ini
[defaults]
inventory = inventories/prod/aws_ec2.yml
roles_path = ./roles
host_key_checking = False
remote_user = ubuntu
private_key_file = /home/ubuntu/.ssh/kong-key.pem

stdout_callback = yaml
bin_ansible_callbacks = True

[inventory]
enable_plugins = amazon.aws.aws_ec2
```

---

# defaults/main.yml

Path:

```bash
defaults/main.yml
```

---

# Purpose

Stores all configurable default variables.

---

# Backend Functionality

Controls:

* Kong version
* deployment mode
* DB configuration
* listening ports
* backend routing

---

# FINAL VERSION

```yaml
---
kong_version: "3.9.0"

# deployment mode
# values:
# - dbless
# - postgres
kong_mode: "dbless"

kong_admin_listen: "127.0.0.1:8001"
kong_proxy_listen: "0.0.0.0:8000"

# postgres variables
kong_pg_host: "127.0.0.1"
kong_pg_port: 5432
kong_pg_database: "kong"
kong_pg_user: "kong"
kong_pg_password: "kong"

# backend service
backend_host: "127.0.0.1"
```

---

# vars/main.yml

Path:

```bash
vars/main.yml
```

---

# Purpose

Stores fixed role-level variables.

---

# FINAL VERSION

```yaml
---
kong_version: "3.9.0"
```

---

# meta/main.yml

Path:

```bash
meta/main.yml
```


# Purpose

Makes role compatible with:

* Ansible Galaxy
* Semaphore role installation
* CI/CD role validation


# Why this was required

Without this file:

```text
role does not appear to have a meta/main.yml file
```

Semaphore failed role installation.

---

# FINAL VERSION

```yaml
---
galaxy_info:
  role_name: kong_api_gateway
  author: Rehan
  description: Kong API Gateway installation role
  company: Opstree

  license: MIT

  min_ansible_version: "2.14"

  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
        - noble

dependencies: []
```


# tasks/main.yml

Path:

```bash
tasks/main.yml
```

---

# Purpose

Main orchestrator.

Controls execution order.

---

# Backend Execution Flow

```text
install.yml
   ↓
postgres.yml (ONLY if postgres mode)
   ↓
config.yml
   ↓
service.yml
   ↓
verify.yml
```

---

# FINAL VERSION

```yaml
---
- import_tasks: install.yml

- import_tasks: postgres.yml
  when: kong_mode == "postgres"

- import_tasks: config.yml

- import_tasks: service.yml

- import_tasks: verify.yml
```

---

# tasks/install.yml

Path:

```bash
tasks/install.yml
```

---

# Purpose

Downloads and installs Kong package.

---

# Backend Functionality

* Downloads Kong `.deb`
* Installs package
* Prevents auto-upgrade

---

# FINAL VERSION

```yaml
---
- name: Download Kong package
  get_url:
    url: "https://packages.konghq.com/public/gateway-39/deb/ubuntu/pool/noble/main/k/ko/kong_{{ kong_version }}/kong_{{ kong_version }}_amd64.deb"
    dest: /tmp/kong.deb
    mode: '0644'
    force: yes

- name: Install Kong package
  apt:
    deb: /tmp/kong.deb
    state: present

- name: Hold Kong package version
  dpkg_selections:
    name: kong
    selection: hold
```

---

# tasks/postgres.yml

Path:

```bash
tasks/postgres.yml
```


# Purpose

Handles PostgreSQL installation and DB provisioning.


# IMPORTANT

This file executes ONLY when:

```yaml
kong_mode: "postgres"
```


# Backend Functionality

* installs PostgreSQL
* waits for DB readiness
* creates DB user
* creates Kong database
* ensures stable DB connectivity

---

# FINAL VERSION

```yaml
---
- name: Install PostgreSQL
  apt:
    name:
      - postgresql
      - postgresql-contrib
    state: present
    update_cache: yes

- name: Find PostgreSQL service name
  shell: systemctl list-units --type=service | grep postgres | awk '{print $1}' | head -n1
  register: postgres_service
  changed_when: false

- name: Start PostgreSQL service
  systemd:
    name: "{{ postgres_service.stdout }}"
    state: started
    enabled: yes
  when: postgres_service.stdout != ""

- name: Wait for PostgreSQL TCP port
  wait_for:
    host: 127.0.0.1
    port: 5432
    timeout: 120

- name: Wait for PostgreSQL fully ready
  shell: pg_isready -h 127.0.0.1 -p 5432
  register: pg_ready
  retries: 30
  delay: 3
  until: pg_ready.rc == 0
  changed_when: false

- name: Find PostgreSQL config
  find:
    paths: /etc/postgresql
    patterns: postgresql.conf
    recurse: yes
  register: pg_conf

- name: Ensure PostgreSQL listens on IPv4 only
  lineinfile:
    path: "{{ pg_conf.files[0].path }}"
    regexp: '^#?listen_addresses'
    line: "listen_addresses = '127.0.0.1'"
  notify: restart postgresql
  when: pg_conf.matched > 0

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

# tasks/config.yml

Path:

```bash
tasks/config.yml
```

---

# Purpose

Deploys:

* Kong configs
* DBLess config
* systemd service

---

# Backend Functionality

Creates:

```text
/etc/kong/kong.conf
/etc/kong/kong.yml
/etc/systemd/system/kong.service
```

---

# FINAL VERSION

```yaml
---
- name: Create Kong config directory
  file:
    path: /etc/kong
    state: directory
    mode: '0755'

- name: Deploy kong.conf
  template:
    src: kong.conf.j2
    dest: /etc/kong/kong.conf
    mode: '0644'
  notify: restart kong

- name: Deploy kong.yml (dbless mode only)
  template:
    src: kong.yml.j2
    dest: /etc/kong/kong.yml
    mode: '0644'
  when: kong_mode == "dbless"
  notify: reload kong

- name: Deploy systemd service
  template:
    src: kong.service.j2
    dest: /etc/systemd/system/kong.service
    mode: '0644'
  notify:
    - reload systemd
    - restart kong

- name: Reload systemd
  systemd:
    daemon_reload: yes
```

---

# tasks/service.yml

Path:

```bash
tasks/service.yml
```

---

# Purpose

Handles:

* config validation
* migrations
* Kong startup

---

# Backend Functionality

## DBLess mode

* validates declarative config
* starts Kong directly

---

## PostgreSQL mode

* validates DB connectivity
* runs migrations
* starts Kong

---

# FINAL VERSION

```yaml
---
- name: Validate Kong config
  command: kong check /etc/kong/kong.conf
  changed_when: false
  when: kong_mode == "postgres"

- name: Validate Kong config (dbless mode)
  command: kong check /etc/kong/kong.conf
  changed_when: false
  when: kong_mode == "dbless"

- name: Stop Kong safely
  systemd:
    name: kong
    state: stopped
  ignore_errors: yes

- name: Wait for PostgreSQL stable connection
  shell: |
    PGPASSWORD={{ kong_pg_password }} psql \
    -h 127.0.0.1 \
    -U {{ kong_pg_user }} \
    -d postgres \
    -c "SELECT 1;"
  register: db_ping
  retries: 20
  delay: 3
  until: db_ping.rc == 0
  changed_when: false
  when: kong_mode == "postgres"

- name: Check Kong schema existence
  shell: sudo -u postgres psql -d {{ kong_pg_database }} -c "\dt"
  register: schema_check
  failed_when: false
  changed_when: false
  when: kong_mode == "postgres"

- name: Run migrations bootstrap (first time only)
  command: kong migrations bootstrap -c /etc/kong/kong.conf
  when:
    - kong_mode == "postgres"
    - "'kong' not in schema_check.stdout"
  register: bootstrap
  retries: 10
  delay: 5
  until: bootstrap.rc == 0

- name: Run migrations up
  command: kong migrations up -c /etc/kong/kong.conf
  register: mig_up
  retries: 10
  delay: 5
  until: mig_up.rc == 0
  when: kong_mode == "postgres"

- name: Run migrations finish
  command: kong migrations finish -c /etc/kong/kong.conf
  when: kong_mode == "postgres"

- name: Start Kong
  systemd:
    name: kong
    state: started
    enabled: yes
```

---

# tasks/verify.yml

Path:

```bash
tasks/verify.yml
```

---

# Purpose

Verifies Kong health after deployment.

---

# Backend Functionality

Checks:

```text
http://127.0.0.1:8001
```

If healthy:

```text
HTTP 200
```

---

# FINAL VERSION

```yaml
---
- name: Wait for Kong Admin API to be ready
  uri:
    url: http://127.0.0.1:8001
    method: GET
    status_code: 200
    timeout: 5
  register: kong_api
  retries: 30
  delay: 5
  until: kong_api.status == 200

- name: Confirm Kong Admin API is responding
  debug:
    msg: "Kong Admin API is up and running"
```

---

# handlers/main.yml

Path:

```bash
handlers/main.yml
```

---

# Purpose

Handles service reload/restart events.

---

# Backend Functionality

Uses:

```text
systemd
```

instead of raw Kong commands.

---

# FINAL VERSION

```yaml
---
- name: restart kong
  systemd:
    name: kong
    state: restarted
    daemon_reload: yes
  listen: restart kong

- name: reload kong
  systemd:
    name: kong
    state: reloaded
  listen: reload kong

- name: reload systemd
  systemd:
    daemon_reload: yes
```

---

# templates/kong.conf.j2

Path:

```bash
templates/kong.conf.j2
```

---

# Purpose

Main Kong configuration file.

---

# Backend Functionality

Controls:

* database mode
* ports
* listeners
* DB connectivity

---

# FINAL VERSION

```jinja
{% if kong_mode == "postgres" %}
database = postgres

pg_host = {{ kong_pg_host }}
pg_port = {{ kong_pg_port }}
pg_user = {{ kong_pg_user }}
pg_password = {{ kong_pg_password }}
pg_database = {{ kong_pg_database }}

{% else %}
database = off
declarative_config = /etc/kong/kong.yml
{% endif %}

admin_listen = {{ kong_admin_listen }}
proxy_listen = {{ kong_proxy_listen }}
```

---

# templates/kong.service.j2

Path:

```bash
templates/kong.service.j2
```

---

# Purpose

Creates systemd-managed Kong service.

---

# IMPORTANT FIX

This template was fixed to support:

✅ DBLess mode
✅ PostgreSQL mode

without hardcoding PostgreSQL dependency.

---

# FINAL VERSION

```ini
[Unit]
Description=Kong API Gateway
After=network.target

{% if kong_mode == "postgres" %}
After=postgresql.service
{% endif %}

[Service]
Type=forking
ExecStart=/usr/local/bin/kong start -c /etc/kong/kong.conf
ExecStop=/usr/local/bin/kong stop -c /etc/kong/kong.conf
ExecReload=/usr/local/bin/kong reload -c /etc/kong/kong.conf
Restart=on-failure
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```

---

# templates/kong.yml.j2

Path:

```bash
templates/kong.yml.j2
```

---

# Purpose

DBLess declarative configuration.

---

# Backend Functionality

Defines:

* services
* routes
* plugins

WITHOUT PostgreSQL.

---

# FINAL VERSION

```yaml
_format_version: "3.0"
_transform: true

services:
  - name: user-service
    url: http://{{ backend_host }}:5678

    plugins:
      - name: rate-limiting
        config:
          minute: 5
          policy: local

    routes:
      - name: user-route
        paths:
          - /users
        strip_path: true
```

---

# 7. REAL EXECUTION FLOW

```text
1. Semaphore triggers playbook
2. Role gets installed
3. install.yml executes
4. postgres.yml skipped (dbless mode)
5. config.yml deploys configs
6. systemd service created
7. service.yml validates config
8. Kong service starts
9. verify.yml checks Admin API
10. handlers restart/reload services
```

---

# 8. FINAL RESULT

Your deployment now provides:

✅ Production-ready Kong deployment
✅ DBLess + PostgreSQL support
✅ CI/CD-safe architecture
✅ Semaphore-compatible role structure
✅ Dynamic service management
✅ Stable startup sequence
✅ Health verification
✅ Reusable Ansible automation
