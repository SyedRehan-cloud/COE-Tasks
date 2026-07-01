# 📘 Consul Cluster POC — Manual Setup (Ubuntu EC2)

---

# 1. Overview

This project demonstrates a **manual 3-node HashiCorp Consul cluster setup** on Ubuntu EC2 instances without:

* Docker
* Kubernetes
* Terraform
* Auto-discovery tools

Consul is used as a:

* Service discovery system
* Distributed key-value store
* Health monitoring system
* Cluster coordination system (Raft-based)

---

# 2. Architecture

```
                ┌──────────────────────┐
                │     Consul UI       │
                │   (8500 HTTP API)   │
                └─────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
  consul-1           consul-2           consul-3
  172.31.38.229      172.31.39.147      172.31.47.192
        │                 │                 │
        └─────────── Cluster Gossip + Raft ───────────┘
```

---

# 3. Node Details

| Node | Name     | Private IP    | Role   |
| ---- | -------- | ------------- | ------ |
| 1    | consul-1 | 172.31.38.229 | Server |
| 2    | consul-2 | 172.31.39.147 | Server |
| 3    | consul-3 | 172.31.47.192 | Server |

---

# 4. Key Concepts Explained

---

## 🧠 4.1 What is Consul?

Consul is a **distributed system coordination tool** that provides:

* Service discovery
* Health checking
* Key-value store
* Leader election (via Raft)

---

## 🧠 4.2 What is a Datacenter (dc1)?

A **datacenter (DC)** in Consul is a logical grouping of nodes.

Example:

```hcl
datacenter = "dc1"
```

### Meaning:

* All 3 nodes belong to same logical cluster
* Enables intra-cluster communication
* Used for multi-region setups later

---

## 🧠 4.3 What is Gossip Protocol?

### Gossip = “Nodes talking to each other continuously”

Consul uses **Serf gossip protocol**.

### Purpose:

* Discover new nodes
* Detect failures
* Share cluster membership

### Ports used:

* 8301 TCP/UDP (LAN gossip)

### How it works:

```
Node A → tells Node B
Node B → tells Node C
Node C → tells Node A
```

Eventually:
👉 all nodes know about all nodes

---

## 🧠 4.4 What is Raft?

### Raft = consensus algorithm used for leader election

Consul uses Raft for:

* Choosing leader
* Replicating cluster state
* Ensuring consistency

### Key idea:

Only **ONE leader exists at a time**

```
Leader → handles writes
Followers → replicate data
```

---

### Example in your cluster:

```
consul-2 → LEADER
consul-1 → follower
consul-3 → follower
```

---

### Raft process:

1. Nodes start as followers
2. No leader → election starts
3. Majority votes → leader chosen
4. Leader replicates logs to followers

---

## 🧠 4.5 What is bootstrap_expect?

```hcl
bootstrap_expect = 3
```

### Meaning:

* Wait for 3 servers before forming cluster
* Prevents split-brain issues

### Your setup:

```
3 servers required → cluster forms only when all 3 join
```

---

## 🧠 4.6 What is bind_addr vs advertise_addr?

### bind_addr

```
bind_addr = "172.31.x.x"
```

👉 Which IP Consul listens on

---

### advertise_addr

```
advertise_addr = "172.31.x.x"
```

👉 Which IP other nodes use to connect

---

## 🧠 4.7 What is retry_join?

```hcl
retry_join = [
  "172.31.39.147",
  "172.31.47.192"
]
```

### Meaning:

* Node automatically tries to join cluster
* No manual join command needed

---

## 🧠 4.8 What is Client vs Server mode?

### Server mode:

```
server = true
```

* Maintains Raft state
* Participates in leader election
* Stores cluster metadata

### Client mode:

* Only forwards requests
* No Raft participation

---

# 5. Directory Structure (IMPORTANT)

## On each node:

```
/etc/consul.d/
    └── consul.hcl          # Main configuration file

/etc/systemd/system/
    └── consul.service      # System service definition

/var/lib/consul/
    └── data files (Raft logs, state)

/usr/local/bin/
    └── consul              # Consul binary
```

---

# 6. Important Files Explained

---

## 📄 6.1 /etc/consul.d/consul.hcl

### Purpose:

Main configuration file for Consul agent

---

### Example:

```hcl
datacenter = "dc1"
node_name = "consul-1"
server = true
bootstrap_expect = 3

data_dir = "/var/lib/consul"

bind_addr = "172.31.x.x"
advertise_addr = "172.31.x.x"

client_addr = "0.0.0.0"

retry_join = [
  "172.31.39.147",
  "172.31.47.192"
]

ui_config {
  enabled = true
}

ports {
  http = 8500
}

log_level = "INFO"
```

---

### Meaning of keys:

| Key              | Meaning                    |
| ---------------- | -------------------------- |
| datacenter       | Logical cluster grouping   |
| node_name        | Unique node identifier     |
| server           | Enables server mode        |
| bootstrap_expect | Expected number of servers |
| data_dir         | Stores Raft + state data   |
| bind_addr        | Internal network binding   |
| advertise_addr   | Cluster communication IP   |
| retry_join       | Auto cluster joining       |
| ui_config        | Enables web UI             |
| ports.http       | API/UI port                |

---

## 📄 6.2 /etc/systemd/system/consul.service

### Purpose:

Runs Consul as a background service

---

### Content:

```ini
[Unit]
Description=Consul Agent
After=network-online.target

[Service]
User=consul
Group=consul
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

### Meaning:

| Section   | Purpose           |
| --------- | ----------------- |
| Unit      | Service metadata  |
| Service   | How to run Consul |
| ExecStart | Start command     |
| Restart   | Auto-restart      |
| Install   | Enable at boot    |

---

# 7. Ports Used

| Port | Type    | Purpose             |
| ---- | ------- | ------------------- |
| 8300 | TCP     | RPC between servers |
| 8301 | TCP/UDP | Gossip LAN          |
| 8302 | TCP/UDP | WAN gossip          |
| 8500 | HTTP    | API + UI            |
| 8600 | TCP/UDP | DNS interface       |

---

# 8. Cluster Formation Flow

```
1. Install Consul binary
2. Create consul user
3. Configure consul.hcl
4. Start service
5. Gossip starts (8301)
6. Nodes discover each other
7. Raft election happens
8. Leader selected
9. Cluster becomes ACTIVE
```

---

# 9. How Your Cluster Actually Started

From your logs:

### Step 1: Nodes start

```
consul-1, consul-2, consul-3 start as followers
```

---

### Step 2: Gossip discovery

```
Nodes find each other via retry_join
```

---

### Step 3: Bootstrap trigger

```
Found expected number of peers: 3
```

---

### Step 4: Raft election

```
New leader elected: consul-2
```

---

# 10. How to Test Cluster

```bash
consul members
```

```bash
consul operator raft list-peers
```

```bash
curl http://<node-ip>:8500/v1/status/leader
```

---

# 11. Common Issues

### ❌ No members shown

→ firewall / SG issue (8301 blocked)

---

### ❌ No leader elected

→ bootstrap_expect mismatch or network issue

---

### ❌ Service not starting

→ systemd or config error

---

# 12. Summary

Consul =

* Distributed cluster coordination system
* Uses Gossip for discovery
* Uses Raft for consistency
* Provides service registry + health system

---

# 🧠 Final Mental Model

Think of Consul like:

```
        🧠 Brain (Raft Leader)
             │
   ┌─────────┼─────────┐
   ▼         ▼         ▼
 Node     Node      Node
 (follower cluster members)

Gossip = communication between them
```

---

Got it — I’ll convert everything you did into a **clean PoC + troubleshooting document**, and also explain Consul concepts (Raft, Gossip, DC) in the same style as your Kong doc.

---

# 📘 Consul Manual Setup PoC (Ubuntu – 3 Node Cluster)

## 1. Overview

This PoC demonstrates a **HashiCorp Consul 3-node server cluster** installed manually on Ubuntu instances (no Kubernetes / Docker).

Consul is used for:

* Service discovery
* Cluster coordination
* Key-value storage
* Health monitoring
* Distributed consensus (Raft)

---

## 2. Architecture

```
                 ┌─────────────────────────┐
                 │     Consul Cluster      │
                 │        (dc1)            │
                 └──────────┬──────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   Node-1              Node-2              Node-3
172.31.38.229      172.31.39.147      172.31.47.192
(consul-1)         (consul-2)         (consul-3)
```

All nodes run:

* Consul Server Agent
* Same datacenter: `dc1`
* Port-based communication over AWS VPC

---

## 3. Key Consul Concepts (VERY IMPORTANT)

---

# 🧠 3.1 Datacenter (DC)

A **Datacenter (dc1)** in Consul is:

> A logical grouping of nodes that trust each other and communicate as one cluster.

### In your setup:

```hcl
datacenter = "dc1"
```

### Meaning:

* All 3 nodes belong to same logical cluster
* They share:

  * Raft leader election
  * Gossip membership
  * Service registry

---

# 🧠 3.2 Gossip Protocol (Serf Layer)

Consul uses **Gossip (Serf protocol)** for:

* Node discovery
* Failure detection
* Membership updates

### Ports used:

| Port | Protocol | Purpose    |
| ---- | -------- | ---------- |
| 8301 | TCP/UDP  | LAN gossip |

---

### How it works:

```
Node A ↔ Node B ↔ Node C
   ↘__________↙
     Gossip layer
```

Each node continuously:

* broadcasts its state
* listens to others
* detects failures automatically

### Why it matters:

* No central server needed for membership
* Fast failure detection

---

# 🧠 3.3 Raft Consensus (MOST IMPORTANT)

Consul uses **Raft algorithm** for:

> Leader election + consistent cluster state

---

### What Raft does:

* Elects 1 leader
* Others become followers
* Ensures data consistency

---

### Your cluster result:

```
New leader elected: consul-2 (172.31.39.147)
```

So:

| Node     | Role     |
| -------- | -------- |
| consul-2 | Leader   |
| consul-1 | Follower |
| consul-3 | Follower |

---

### Ports used:

| Port | Purpose            |
| ---- | ------------------ |
| 8300 | Raft communication |

---

### Raft simple flow:

```
1. All nodes start as followers
2. No leader detected
3. Election triggered
4. Node with majority becomes leader
5. Logs replicated to followers
```

---

# 🧠 3.4 Consul Server vs Agent

| Type   | Role                                         |
| ------ | -------------------------------------------- |
| Server | Maintains cluster state + Raft               |
| Agent  | Runs locally (server + client mode possible) |

Your setup:

```
server = true
```

So all 3 are server nodes.

---

# 🧠 3.5 Ports Summary

| Port | Purpose               |
| ---- | --------------------- |
| 8500 | HTTP API / UI         |
| 8300 | Raft (server RPC)     |
| 8301 | Gossip LAN            |
| 8302 | WAN gossip            |
| 8600 | DNS service discovery |

---

## 4. File Structure (Your Setup)

### `/etc/consul.d/consul.hcl`

This is the **main configuration file**

---

## 4.1 Configuration Breakdown

### Common config (all nodes)

```hcl
datacenter = "dc1"
server = true
bootstrap_expect = 3
data_dir = "/var/lib/consul"
client_addr = "0.0.0.0"
```

---

## 4.2 Node Identity

Each node:

| Node | IP            | Name     |
| ---- | ------------- | -------- |
| 1    | 172.31.38.229 | consul-1 |
| 2    | 172.31.39.147 | consul-2 |
| 3    | 172.31.47.192 | consul-3 |

---

## 4.3 bind vs advertise

### bind_addr

```hcl
bind_addr = "172.31.x.x"
```

👉 Internal listening IP

---

### advertise_addr

```hcl
advertise_addr = "172.31.x.x"
```

👉 IP advertised to other nodes

---

## 4.4 retry_join

```hcl
retry_join = [
  "172.31.39.147",
  "172.31.47.192"
]
```

👉 Auto-discovery mechanism

Meaning:

* Node tries to join cluster automatically
* No manual join command needed

---

## 4.5 UI

```hcl
ui_config {
  enabled = true
}
```

Enables Consul Web UI on:

```
http://<node>:8500
```

---

## 5. systemd Service

### `/etc/systemd/system/consul.service`

```ini
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d
```

### Meaning:

* Runs Consul as a background service
* Loads all config from `/etc/consul.d`

---

### systemd responsibilities:

* Start Consul at boot
* Restart on failure
* Manage lifecycle

---

## 6. Security Group Rules (AWS)

You configured:

| Port | Purpose |
| ---- | ------- |
| 8500 | UI/API  |
| 8300 | Raft    |
| 8301 | Gossip  |
| 8600 | DNS     |

⚠️ Important:

For production:

* DO NOT use `0.0.0.0/0`
* Restrict to VPC CIDR only

---

# 7. Commands You Used (COMPLETE FLOW)

---

## 7.1 Validation

```bash
consul validate /etc/consul.d
```

Checks:

* syntax correctness
* required fields

---

## 7.2 Service control

```bash
sudo systemctl status consul
sudo systemctl restart consul
sudo systemctl daemon-reload
```

---

## 7.3 Manual start (debug mode)

```bash
sudo -u consul consul agent -config-dir=/etc/consul.d
```

Used for:

* debugging
* seeing live logs
* checking join behavior

---

## 7.4 Network check

```bash
ss -lntp | grep consul
```

Used to verify:

* ports are open
* consul is listening

---

## 7.5 logs

```bash
journalctl -u consul -n 200
```

(Initially empty → service wasn’t running via systemd correctly)

---

## 7.6 systemd unit check

```bash
cat /etc/systemd/system/consul.service
```

---

## 7.7 cluster observation

From logs:

```
New leader elected: consul-2
Found expected number of peers: 3
```

---

## 8. Troubleshooting Timeline (YOUR ACTUAL ISSUE)

---

# ❌ Problem 1: Consul not reachable

```
connection refused: 127.0.0.1:8500
```

### Root cause:

* Consul service NOT running via systemd

---

# ❌ Problem 2: systemd service inactive

```
Active: inactive (dead)
No journal logs
```

### Root cause:

* service never started properly OR wrong unit file

---

# ❌ Problem 3: no cluster formed initially

Because:

* service was not running
* no leader election possible

---

# ❌ Problem 4: missing permissions while writing service file

```
Permission denied
```

### Cause:

Used:

```bash
sudo cat > file
```

This fails because `>` is handled by shell (not sudo)

---

### Fix:

Correct way:

```bash
sudo tee /etc/systemd/system/consul.service > /dev/null
```

---

# ✅ Final Fix Steps You Did

### 1. Correct service file created

### 2. daemon reload

```bash
sudo systemctl daemon-reload
```

### 3. restart consul

```bash
sudo systemctl restart consul
```

### 4. verified status

```bash
systemctl status consul
```

---

# 🎉 Final Result

Cluster successfully formed:

```
Nodes joined: 3
Leader elected: consul-2
Raft enabled
Gossip active
UI enabled (8500)
```

---

# 🧠 FINAL SUMMARY

## What Consul is doing here:

* Gossip = node discovery
* Raft = leader election + consistency
* Datacenter = logical cluster grouping
* systemd = service lifecycle manager

---

# If you want next step (recommended)

I can extend this PoC into:

### 🚀 Production-grade version:

* TLS encryption between nodes
* ACL tokens setup
* Consul UI access security
* Service registration example (like microservices)
* Health checks
* Failover simulation
* Adding Consul client agents

Just tell me 👍

