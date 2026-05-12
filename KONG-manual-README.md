# 📘 Kong API Gateway — Manual Setup (Bare Metal / Ubuntu)

## 1. Overview

This project demonstrates a **real-world API Gateway setup using Kong (Community Edition)** installed manually on Ubuntu without:

* Docker
* Kubernetes
* Helm

Kong acts as a **central API Gateway** for multiple microservices:

* User Service
* Order Service
* Payment Service

---

## 2. Architecture

```
Client
   |
   v
Kong API Gateway (Port 8000)
   |
   +-------------------+
   |                   |
User Service     Order Service     Payment Service
(5678)             (5679)              (5680)
```

---

## 3. Kong Ports

| Port | Purpose                    |
| ---- | -------------------------- |
| 8000 | Public API Gateway (Proxy) |
| 8001 | Admin API (Management)     |
| 8002 | Admin GUI (if enabled)     |
| 8007 | Status endpoint            |


##  4. Important Kong Files

## `/etc/kong/kong.conf`

### Purpose:

Main configuration file for Kong runtime.

### Example:

```ini
database = off
declarative_config = /etc/kong/kong.yml

proxy_listen = 0.0.0.0:8000
admin_listen = 127.0.0.1:8001
```

### 🔍 Explanation:

| Key                  | Meaning                 |
| -------------------- | ----------------------- |
| `database = off`     | Enables DB-less mode    |
| `declarative_config` | Path to API definitions |
| `proxy_listen`       | Public API entry point  |
| `admin_listen`       | Internal admin API      |


## `/etc/kong/kong.yml`

### Purpose:

Defines all APIs, routes, services, plugins.

---

## 🧩 Example (Multi-Service Setup)

```yaml
_format_version: "3.0"

services:

  - name: user-service
    url: http://127.0.0.1:5678
    routes:
      - name: user-route
        paths:
          - /users
        strip_path: true

  - name: order-service
    url: http://127.0.0.1:5679
    routes:
      - name: order-route
        paths:
          - /orders
        strip_path: true

  - name: payment-service
    url: http://127.0.0.1:5680
    routes:
      - name: payment-route
        paths:
          - /payments
        strip_path: true
```

---

## 🔍 5. Key Concepts Explained

---

## 🧠 Service

A **Service = backend microservice**

Example:

```
user-service → http://127.0.0.1:5678
```

---

## 🧭 Route

A **Route = URL mapping**

Example:

```
/users → user-service
```

---

## ✂️ strip_path

If:

```
/users/profile
```

Kong sends:

```
/profile
```

to backend.

---

## Plugins (Optional Enhancement)

You can add:

* JWT Authentication
* Rate Limiting
* Logging

Example:

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 5
```

---

## 🚀 6. Systemd Service Setup (Production Mode)

### 📄 `/etc/systemd/system/kong.service`

```ini
[Unit]
Description=Kong API Gateway
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/kong start -c /etc/kong/kong.conf
ExecReload=/usr/local/bin/kong reload -c /etc/kong/kong.conf
ExecStop=/usr/local/bin/kong stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## ▶️ Commands

```bash
sudo systemctl start kong
sudo systemctl enable kong
sudo systemctl status kong
```

---

## 🔁 7. Request Flow (How Kong Works)

```
Client Request → Kong → Route Matching → Plugins → Backend → Response → Client
```

---

## ⚙️ Internal Steps

### 1. Request enters Kong (Port 8000)

### 2. Kong extracts:

* Path
* Method
* Headers

### 3. Route matching:

```
/users → user-service
```

### 4. Plugin execution:

* JWT check
* Rate limiting
* Logging

### 5. Forward request to backend

### 6. Backend response returned

### 7. Response plugins executed

### 8. Final response sent to client

---

## ❗ 8. Common Issue (Your Case)

### Problem:

```
404 File not found
```

### Reason:

Backend service NOT running on:

```
http://127.0.0.1:5678
```

---

## ✅ Fix

Run a test backend:

```bash
python3 -m http.server 5678
```

OR:

```bash
docker run -p 5678:5678 hashicorp/http-echo -text="User Service"
```

Then test:

```bash
curl http://localhost:8000/users
```

---

## 🧪 9. Testing APIs

```bash
curl http://localhost:8000/users
curl http://localhost:8000/orders
curl http://localhost:8000/payments
```

---

## 📊 10. Kong Internal Algorithm (Simplified)

```
1. Receive request
2. Parse URI
3. Match route
4. Run request plugins
5. Select service
6. Rewrite path (if needed)
7. Forward request
8. Receive response
9. Run response plugins
10. Return response to client
```

---

## 🧠 11. Summary

Kong =

* Reverse proxy
* API gateway
* Plugin engine
* Traffic controller

It sits between:

```
Client ↔ Kong ↔ Microservices
```

# ⚙️ Kong Request Processing Pipeline (Deep Explanation)

When a request comes in:

```text
Client → Kong (8000) → Backend Service → Kong → Client
```

Example request:

```text
GET /users/profile?id=10
```

---

# 1️⃣ Receive Request

### What happens:

Kong’s **NGINX layer** receives the HTTP request on:

```text
0.0.0.0:8000
```

### Internally:

* Connection accepted by NGINX worker process
* Request is converted into an internal Lua context
* Kong creates a **request object**

### Think of it like:

📦 “A parcel arrives at the gate of a warehouse”

---

# 2️⃣ Parse URI

### What happens:

Kong extracts request metadata:

* Path → `/users/profile`
* Method → GET / POST / PUT
* Host → domain/IP
* Headers → Authorization, Content-Type
* Query params → `?id=10`

### Internally:

Kong builds a structured object:

```json
{
  "path": "/users/profile",
  "method": "GET",
  "query": {"id": "10"}
}
```

### Why?

Because routing decisions depend on this data.

---

# 3️⃣ Match Route (MOST IMPORTANT STEP)

### What happens:

Kong checks:

👉 “Which route matches this request?”

Example routes:

```yaml
/users
/orders
/payments
```

Request:

```text
/users/profile
```

### Matching logic:

Kong uses **prefix matching algorithm**:

```text
IF request_path STARTS WITH route_path → match
```

So:

```text
/users/profile → matches /users
```

### Internally:

* Routes are stored in a high-performance lookup table
* NGINX + Kong Router engine does O(log n) matching

### Think of it like:

🧭 “Finding correct department in a company based on label”

---

# 4️⃣ Run Request Plugins (PRE-PROXY PHASE)

### What happens:

Before sending request to backend, Kong runs plugins.

Example plugins:

* JWT authentication
* API Key check
* Rate limiting
* IP restriction

### Example flow:

```text
JWT plugin → validate token
Rate limit → check usage
ACL → check permissions
```

### If failure:

```text
→ STOP request here (401 / 403 / 429)
```

### Think of it like:

🛑 “Security check at airport before boarding”

---

# 5️⃣ Select Service

### What happens:

Once route matches, Kong finds:

```text
Route → Service → Backend URL
```

Example:

```yaml
service:
  url: http://127.0.0.1:5678
```

So Kong decides:

```text
Forward request to: User Service
```

### Internally:

* Service object contains upstream details
* Load balancing logic may also apply here (if multiple upstreams exist)

---

# 6️⃣ Rewrite Path (if needed)

### What happens:

Kong modifies request path before sending to backend.

Controlled by:

```yaml
strip_path: true
```

---

### Example:

Incoming request:

```text
/users/profile
```

Route:

```yaml
/users
```

Backend receives:

```text
/profile
```

---

### Why this exists:

Because backend services don’t need to know gateway prefix.

---

### Think of it like:

✂️ “Removing label before sending package inside factory”

---

# 7️⃣ Forward Request (Proxy to Backend)

### What happens:

Kong sends request to upstream service:

```text
http://127.0.0.1:5678/profile
```

### Internally:

* NGINX upstream connection pool is used
* TCP connection reused if possible
* Request forwarded with headers intact

### Think of it like:

🚚 “Delivery truck takes parcel to correct warehouse”

---

# 8️⃣ Receive Response

### What happens:

Backend sends response:

```json
{
  "user": "john",
  "id": 10
}
```

### Kong receives:

* Status code (200/404/500)
* Headers
* Body

### Internally:

Response is buffered into Kong context.

---

# 9️⃣ Run Response Plugins (POST-PROXY PHASE)

### What happens:

Kong processes response before sending to client.

Plugins:

* Response transformation
* Logging
* Caching
* Metrics (Prometheus)

---

### Example:

Add headers:

```text
X-Kong-Proxy-Latency: 5ms
X-Kong-Upstream-Latency: 20ms
```

---

### Think of it like:

📦 “Quality check + labeling before delivery to customer”

---

# 🔟 Return Response to Client

### Final step:

Kong sends response back:

```text
Client ← Kong ← Backend
```

Example:

```json
{
  "user": "john",
  "id": 10
}
```

### Plus headers:

```text
X-Kong-Request-Id: abc123
```

---

# 🧠 FULL FLOW SUMMARY (ONE LINE)

```text
Receive → Parse → Route Match → Request Plugins → Select Service → Rewrite → Forward → Response → Response Plugins → Return
```

---

# 🔥 REAL INTERNAL VIEW (HOW KONG THINKS)

Kong internally behaves like:

```pseudo
function handle_request(req):

    ctx = parse_request(req)

    route = find_route(ctx.path)

    if route == null:
        return 404

    run_plugins(route.request_plugins, ctx)

    if ctx.blocked:
        return ctx.response

    service = route.service

    upstream_req = rewrite_request(ctx, route)

    response = forward(service.url, upstream_req)

    run_plugins(route.response_plugins, response)

    return response
```

---

# 🧠 SIMPLE ANALOGY (VERY IMPORTANT)

Think of Kong like an **airport system**:

| Step | Kong              | Airport                  |
| ---- | ----------------- | ------------------------ |
| 1    | Receive request   | Passenger enters airport |
| 2    | Parse URI         | Check ticket             |
| 3    | Route match       | Find gate                |
| 4    | Plugins (request) | Security check           |
| 5    | Service selection | Assign flight            |
| 6    | Rewrite path      | Boarding pass update     |
| 7    | Forward           | Boarding flight          |
| 8    | Receive response  | Flight lands             |
| 9    | Response plugins  | Immigration check        |
| 10   | Return            | Exit airport             |

---

# 🚀 WHY THIS MATTERS

Once you understand this pipeline, you can:

* Debug 404/401/502 issues easily
* Design microservices correctly
* Add plugins in correct order
* Build production-grade API gateways

Good — now we go one level deeper into how Kong *actually works internally*, not just the surface pipeline.

I’ll break this into **real engine internals** so you understand Kong like a systems engineer.

---

# 🧠 ADVANCED KONG INTERNAL ARCHITECTURE

Kong is not just an API gateway — it is:

> **NGINX + Lua runtime + Router engine + Plugin execution VM + caching layer**

---

# 🧩 1. KONG CORE ARCHITECTURE

Internally Kong has 4 major layers:

```text id="k2q3ua"
        ┌──────────────────────┐
        │   Kong Proxy Layer   │  (NGINX)
        └─────────┬────────────┘
                  │
        ┌─────────▼────────────┐
        │  Lua Plugin Engine   │
        └─────────┬────────────┘
                  │
        ┌─────────▼────────────┐
        │  Router / Cache      │
        └─────────┬────────────┘
                  │
        ┌─────────▼────────────┐
        │  Data Store Layer    │ (DB or DB-less memory)
        └──────────────────────┘
```

---

# ⚙️ 2. HOW REQUEST REALLY ENTERS KONG

When request comes:

```text id="x9j1pd"
Client → TCP → NGINX worker → Lua context → Kong pipeline
```

### Step-by-step:

### 🔹 Step A: NGINX Worker Process

* Each request is handled by a **worker process**
* No thread-per-request model (event-driven)

This is why Kong scales well

### 🔹 Step B: Lua Context Creation

NGINX passes request into Lua VM:

```text id="l8b4hf"
ngx.ctx = {
  request = {...},
  response = nil
}
```

---

### 🔹 Step C: Phases Activated

Kong uses **NGINX request phases**:

```text id="r3w8kc"
init_worker
rewrite
access   ← plugins run here
balancer
header_filter
body_filter
log
```

---

# 🧭 3. ROUTER ENGINE (VERY IMPORTANT)

Kong does NOT scan routes linearly.

It uses a **high-performance radix tree / trie-like structure**

---

## 🔍 How route matching works:

When you define:

```yaml id="r8tq3v"
paths:
  - /users
  - /orders
```

Kong builds:

```text id="9h2xqp"
/
 ├── users
 └── orders
```

---

## ⚡ Matching algorithm:

Instead of:

```text id="n5m8qz"
O(n) scanning routes ❌
```

Kong uses:

```text id="p2k9aa"
O(log n) or O(1)-like prefix tree lookup ✅
```

---

## 🧠 Advanced behavior:

Kong also matches:

* Host header
* Methods (GET/POST)
* Headers-based routing
* Regex routes

Example:

```yaml id="x8p4rt"
paths: ["/users/:id"]
```

→ dynamic parameter extraction

---

# 🔌 4. PLUGIN ENGINE (HEART OF KONG)

Plugins are executed like a **pipeline stack**

---

## 📦 Plugin phases:

```text id="k9w2lh"
1. access (request)
2. header_filter
3. body_filter
4. log
```

---

## 🔥 Execution model:

Plugins are executed in **priority order**

Example:

```text id="m7z1qa"
JWT (priority 1450)
Rate-limit (priority 910)
Logging (priority 10)
```

---

##  Internally:

Kong builds:

```lua id="v4p2zn"
plugin_chain = sort_by_priority(plugins)
```

Then executes:

```text id="j1q9xp"
for plugin in plugin_chain:
    plugin:access()
```

---

## 🚨 Important:

If any plugin fails:

```text id="b7m2qd"
return response immediately → stop pipeline
```

Example:

* JWT invalid → STOP
* Rate limit exceeded → STOP

---

# 🚚 5. UPSTREAM SELECTION (LOAD BALANCING ENGINE)

When service is selected:

```text id="u8k2dp"
service → upstream → node
```

---

## Algorithms supported:

### 1. Round Robin (default)

```text id="r3k9zq"
A → B → C → A
```

### 2. Least Connections

```text id="c2m8xp"
send to least busy node
```

### 3. Consistent Hashing

```text id="h7p1aa"
same user → same backend
```

---

## Internally Kong uses:

```text id="w8q2lm"
balancer.lua
```

---

# 🔁 6. REQUEST REWRITE ENGINE

Before forwarding:

Kong modifies request:

### Example:

```text id="a9m2xq"
/users/profile → /profile
```

---

## Internals:

```lua id="t4k8pz"
if strip_path then
   remove_prefix(route.path)
end
```

---

# 🚀 7. FORWARDING ENGINE (NGINX UPSTREAM)

Kong uses:

> **OpenResty (NGINX + LuaJIT)**

---

## Internals:

```text id="y2k9qm"
client → nginx worker → upstream connection pool → backend
```

---

## Optimization:

* Persistent TCP connections
* Connection pooling
* Keep-alive reuse

---

# 📥 8. RESPONSE HANDLING PIPELINE

Response flows back:

```text id="f8p3qa"
backend → kong → client
```

---

## Steps:

### 1. header_filter phase

* modify headers
* add latency headers

### 2. body_filter phase

* stream response chunks
* optionally transform body

---

## Example injected headers:

```text id="l2m9xz"
X-Kong-Proxy-Latency: 5ms
X-Kong-Upstream-Latency: 20ms
```

---

# 🧾 9. LOGGING SYSTEM (ASYNC PIPELINE)

Kong logs asynchronously:

```text id="z4k8pm"
log phase → queue → file / syslog / http endpoint
```

---

## Why async?

So request is NOT delayed.

---

# 🧠 10. DB vs DB-less INTERNAL DIFFERENCE

## DB Mode:

```text id="q8m1aa"
Admin API → Postgres → cache → runtime
```

## DB-less:

```text id="t9p2kk"
kong.yml → LMDB → in-memory cache → runtime
```

---

## DB-less reload:

```text id="n2k8qp"
kong reload → rebuild in-memory graph
```

---

# ⚡ 11. FINAL FULL INTERNAL FLOW

This is the REAL Kong engine flow:

```text id="v7p1lm"
1. NGINX receives request
2. Lua context created
3. Route lookup (radix tree)
4. Plugin chain execution (request phase)
5. Service selection
6. Load balancer picks upstream
7. Request rewrite
8. Forward via upstream connection pool
9. Receive response
10. Response plugin execution
11. Logging async
12. Return response
```


# 🧠 SIMPLE MENTAL MODEL

Think of Kong as:

> 🧠 “A programmable traffic brain inside NGINX”

Where:

* Router = GPS system
* Plugins = security + rules engine
* Balancer = traffic controller
* NGINX = highway system
* Lua = decision engine
