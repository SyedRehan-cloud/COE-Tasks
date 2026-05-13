_format_version: "3.0"

services:

  # =========================================================
  # 1. PATH-BASED ROUTING (/users)
  # =========================================================
  - name: user-service
    url: http://127.0.0.1:5678
    routes:
      - name: user-path-route
        paths:
          - /users
        strip_path: true

        annotations:
          explanation: >
            When request comes to /users:
            1. Kong matches path "/users"
            2. Route is selected
            3. strip_path=true removes /users before forwarding
            4. Upstream receives "/"
            Example:
              /users → http://127.0.0.1:5678/


  # =========================================================
  # 2. HOST-BASED ROUTING (users.local)
  # =========================================================
  - name: host-user-service
    url: http://127.0.0.1:5678
    routes:
      - name: user-host-route
        hosts:
          - users.local
        paths:
          - /
        strip_path: false

        annotations:
          explanation: >
            When request comes:
              Host: users.local
            Kong checks HOST first.
            If host matches → route selected.
            Path "/" is secondary here.
            No strip_path → full path forwarded.
            Example:
              users.local:8000 → upstream service


  # =========================================================
  # 3. METHOD-BASED ROUTING (GET only)
  # =========================================================
  - name: method-service
    url: http://127.0.0.1:5678
    routes:
      - name: method-route
        paths:
          - /method
        methods:
          - GET
        strip_path: true

        annotations:
          explanation: >
            Kong matches BOTH:
              1. Path = /method
              2. HTTP method = GET

            Behavior:
              GET /method  → ALLOWED
              POST /method → NOT MATCHED

            Then strip_path removes /method before forwarding.


  # =========================================================
  # 4. HEADER-BASED ROUTING (version=v1)
  # =========================================================
  - name: header-service
    url: http://127.0.0.1:5678
    routes:
      - name: header-route
        paths:
          - /versioned
        headers:
          version:
            - v1
        strip_path: true

        annotations:
          explanation: >
            Kong checks:
              1. Path = /versioned
              2. Header version = v1

            Only requests like:
              /versioned + version:v1

            will match this route.

            Without header → route is skipped.


  # =========================================================
  # 5. QUERY PARAM ROUTING (?type=admin)
  # =========================================================
  - name: query-service
    url: http://127.0.0.1:5678
    routes:
      - name: query-route
        paths:
          - /query
        strip_path: true

        annotations:
          explanation: >
            Kong route matching is ONLY path-based here.
            Query params are NOT used in routing decision.

            Example:
              /query?type=admin

            Kong forwards entire query string to backend:
              /query?type=admin → upstream


  # =========================================================
  # 6. REGEX PATH ROUTING (/users/{id})
  # =========================================================
  - name: regex-service
    url: http://127.0.0.1:5678
    routes:
      - name: regex-route
        paths:
          - ~/users/\d+
        strip_path: false

        annotations:
          explanation: >
            Regex routing enables dynamic path matching.

            Matches:
              /users/123 ✔
              /users/999 ✔
              /users/abc ✘

            Because:
              \d+ = only numbers

            Full path is forwarded as-is.


  # =========================================================
  # 7. MULTI PATH ROUTING (/orders, /purchases)
  # =========================================================
  - name: multi-path-service
    url: http://127.0.0.1:5678
    routes:
      - name: multi-path-route
        paths:
          - /orders
          - /purchases
        strip_path: true

        annotations:
          explanation: >
            One service can handle multiple entry points.

            Matches:
              /orders → service
              /purchases → service

            Both routes forward to same backend.

            strip_path=true removes prefix before forwarding.


  # =========================================================
  # 8. DEFAULT FALLBACK ROUTE (/)
  # =========================================================
  - name: fallback-service
    url: http://127.0.0.1:5678
    routes:
      - name: fallback-route
        paths:
          - /
        strip_path: false

        annotations:
          explanation: >
            This acts as catch-all route.

            If no other route matches:
              → this route is used

            Example:
              /random-api → fallback-service

            IMPORTANT:
              Should always be last logical safety route.

# ⚙️ FIRST: KONG ROUTING MENTAL MODEL

Every request goes through this pipeline:

```
1. Receive request (8000)
2. Extract: Host + Path + Method + Headers + Query
3. Match routes (top-down evaluation, but actually score-based)
4. Pick BEST matching route
5. Apply plugins (if any)
6. Rewrite path (strip_path logic)
7. Forward to upstream service
8. Return response
```

---

# 🚀 NOW YOUR CONFIG — EXPLAINED FLOW BY FLOW

---

# 1. PATH-BASED ROUTING (/users)

```yaml
paths:
  - /users
strip_path: true
```

## 🧪 Request

```bash
curl http://localhost:8000/users
```

## 🔍 Kong does:

* Path = `/users`
* Matches route `/users` ✔

## 🔁 What happens next:

### strip_path: true means:

```
Incoming:  /users
Forwarded: /
```

## 📦 Final upstream call:

```
http://127.0.0.1:5678/
```

## 🧠 Summary

| Step     | Result                  |
| -------- | ----------------------- |
| Match    | YES (/users)            |
| Forward  | root path `/`           |
| Use case | clean microservice APIs |

---

# 2. HOST-BASED ROUTING (users.local)

```yaml
hosts:
  - users.local
paths:
  - /
```

## 🧪 Request

```bash
curl http://users.local:8000/
```

(or with header Host: users.local)

## 🔍 Kong checks:

* Host header = `users.local` ✔
* Path = `/` ✔

## 📦 Forwarding:

```
http://127.0.0.1:5678/
```

## 🧠 Important behavior:

* Host match happens BEFORE path match priority
* This is how SaaS systems separate tenants

## 🧠 Summary

| Step         | Result                 |
| ------------ | ---------------------- |
| Host match   | YES                    |
| Path match   | YES                    |
| Routing type | domain-based isolation |

---

# 3. METHOD-BASED ROUTING (GET only)

```yaml
paths: /method
methods: [GET]
```

## Requests

### Allowed:

```bash
curl http://localhost:8000/method
```

### NOT allowed:

```bash
curl -X POST http://localhost:8000/method
```

## Kong logic:

1. Path matches `/method` ✔
2. Method check:

| Method | Allowed |
| ------ | ------- |
| GET    | ✔       |
| POST   | ❌       |
| PUT    | ❌       |

## Behavior:

* POST request will NOT hit this route
* It will fallback to another route or 404

## 🧠 Summary

| Feature           | Behavior           |
| ----------------- | ------------------ |
| Routing condition | Path + HTTP method |
| Use case          | REST control       |

---

# 4. HEADER-BASED ROUTING (version=v1)

```yaml
headers:
  version:
    - v1
```

## Request

```bash
curl http://localhost:8000/versioned -H "version:v1"
```

## Kong checks:

1. Path = `/versioned` ✔
2. Header exists?

```
version: v1 ✔
```

## If missing header:

```bash
curl http://localhost:8000/versioned
```

→ NO MATCH

## Behavior:

Only requests with correct header go to this service.

## Summary

| Condition         | Required |
| ----------------- | -------- |
| Path              | YES      |
| Header version=v1 | YES      |

---

# 5. QUERY PARAM ROUTING (?type=admin)

```yaml
paths:
  - /query
```

## Request

```bash
curl "http://localhost:8000/query?type=admin"
```

## Important truth:

👉 Kong itself does NOT match query params in YAML routes by default.

So here:

* Route matches only `/query`
* Query params are passed downstream

## 📦 Forwarded request:

```
http://127.0.0.1:5678/query?type=admin
```

## 🧠 So what is query routing used for?

Usually combined with:

* request-transformer plugin
* custom logic
* or app-level filtering

## Summary

| Feature     | Behavior             |
| ----------- | -------------------- |
| Route match | path only            |
| Query used  | inside backend logic |

---

# 6. REGEX ROUTING (/users/\d+)

```yaml
paths:
  - ~/users/\d+
```

## Requests

```bash
curl http://localhost:8000/users/123
curl http://localhost:8000/users/9999
```

## Kong logic:

Regex match happens:

```
/users/123  ✔ match
/users/abc  ❌ no match
```

## Forwarding:

```
http://127.0.0.1:5678/users/123
```

## Key idea:

* No stripping unless configured
* Very powerful for dynamic APIs

## Summary

| Feature  | Behavior    |
| -------- | ----------- |
| Matching | regex-based |
| Use case | dynamic IDs |

---

# 7. MULTI PATH ROUTING (/orders, /purchases)

```yaml
paths:
  - /orders
  - /purchases
```

## 🧪 Requests

```bash
curl http://localhost:8000/orders
curl http://localhost:8000/purchases
```

## Kong logic:

Same service, multiple entry points.

| Request    | Match |
| ---------- | ----- |
| /orders    | ✔     |
| /purchases | ✔     |

## Forwarding:

```
http://127.0.0.1:5678/
```

(because strip_path: true)

## Summary

| Feature        | Benefit      |
| -------------- | ------------ |
| Multiple paths | one service  |
| Use case       | API grouping |

---

# 8. FALLBACK ROUTE (/)

```yaml
paths:
  - /
```

## Request

```bash
curl http://localhost:8000/anything-unknown
```

## 🔍 What happens:

Kong tries all routes:

| Route   | Match      |
| ------- | ---------- |
| /users  | ❌          |
| /method | ❌          |
| /orders | ❌          |
| /       | ✔ fallback |

## Final behavior:

All unmatched requests go here.

## VERY IMPORTANT:

This should be LAST route logically.

---

# FINAL EXECUTION ORDER (REAL KONG BEHAVIOR)

When request comes:

```
1. Check Host match (if defined)
2. Check exact path match
3. Check regex match
4. Check method match
5. Check headers match
6. Score best match
7. Pick highest priority route
```

# ONE REAL-WORLD EXAMPLE FLOW

## Request:

```bash
curl http://users.local:8000/users/123 -H "version:v1"
```

## Step-by-step:

| Step         | Result                 |
| ------------ | ---------------------- |
| Host match   | YES (users.local)      |
| Path match   | YES (/users/123 regex) |
| Header match | YES (version=v1)       |
| Method match | default GET            |

BUT ONLY ONE ROUTE WINS (best score)


# FINAL INTUITION

Think of Kong like this:

```
Incoming Request
      ↓
Match ALL possible routes
      ↓
Score each match
      ↓
Pick BEST match
      ↓
Apply plugins
      ↓
Forward to service
```
# Upstream in Kong

In Kong, an **Upstream** is a logical group of backend servers/services that Kong can send traffic to.

Instead of routing traffic directly to a single backend URL, Kong can route traffic to an **Upstream**, and that Upstream can contain multiple backend targets.

This is how Kong performs:

* Load balancing
* High availability
* Failover
* Health checks
* Traffic distribution

# Simple Definition

## Without Upstream

```yaml
services:
  - name: user-service
    url: http://127.0.0.1:5678
```

Here:

* Kong directly forwards requests to ONE backend server.

Problem:

* If backend dies → service fails.
* No load balancing.
  
# With Upstream

```yaml
upstreams:
  - name: user-upstream

targets:
  - upstream: user-upstream
    target: 127.0.0.1:5678

  - upstream: user-upstream
    target: 127.0.0.1:5679
```

Now:

* Kong sends traffic to MULTIPLE servers.
* Kong distributes requests automatically.

This is the real production architecture.

---

# Real-World Analogy

Imagine:

## Restaurant Example

### Kong = Receptionist

### Upstream = Kitchen Team

### Targets = Individual chefs

Customer request:

```
"I want food"
```

Receptionist decides:

* Chef 1?
* Chef 2?
* Chef 3?

depending on:

* who is free
* who is healthy
* load balancing algorithm

That kitchen group = Upstream.


# Kong Traffic Flow with Upstream

```text
Client Request
      |
      v
+----------------+
| Kong Gateway   |
+----------------+
      |
      v
+----------------+
|   Upstream     |
| user-upstream  |
+----------------+
   |     |     |
   v     v     v
Server1 Server2 Server3
```

---

# Why Upstream is Important

Without Upstream:

* Single backend only
* No scaling
* No failover

With Upstream:

* Multiple backend instances
* Traffic balancing
* Automatic failover
* Health checking
* Better performance


# Core Components

# 1. Service

Defines:

* What backend application exists

Example:

```yaml
services:
  - name: user-service
    host: user-upstream
    port: 80
    protocol: http
```

Notice:

* host = upstream name
* NOT actual IP

---

# 2. Upstream

Logical backend group.

```yaml
upstreams:
  - name: user-upstream
```

---

# 3. Targets

Actual backend servers.

```yaml
targets:
  - upstream: user-upstream
    target: 127.0.0.1:5678

  - upstream: user-upstream
    target: 127.0.0.1:5679
```

# Complete Flow

```text
1. Client sends request
2. Kong matches route
3. Kong selects service
4. Service points to upstream
5. Upstream selects target
6. Request forwarded
7. Response returned
```

# Full Example

```yaml
_format_version: "3.0"

services:
  - name: user-service
    host: user-upstream
    port: 80
    protocol: http

    routes:
      - name: user-route
        paths:
          - /users

upstreams:
  - name: user-upstream

targets:
  - upstream: user-upstream
    target: 127.0.0.1:5678

  - upstream: user-upstream
    target: 127.0.0.1:5679

  - upstream: user-upstream
    target: 127.0.0.1:5680
```

---

# What Happens Internally

Suppose request comes:

```bash
curl http://localhost:8000/users
```

Kong internally:

---

## Step 1 — Route Matching

Kong checks:

```yaml
paths:
  - /users
```

Matched.

---

## Step 2 — Service Selection

Kong selects:

```yaml
service: user-service
```

---

## Step 3 — Upstream Resolution

Service host:

```yaml
host: user-upstream
```

Kong now looks for upstream named:

* user-upstream

---

## Step 4 — Load Balancing

Kong chooses one target:

* 5678
* 5679
* 5680

using balancing algorithm.

---

## Step 5 — Request Forwarding

Request forwarded to selected backend.

---

# Load Balancing Algorithms Used with Upstreams

Kong Upstreams support multiple balancing algorithms.

---

# 1. Round Robin 

```text
Req1 → Server1
Req2 → Server2
Req3 → Server3
Req4 → Server1
```

Balanced equally.

---

# 2. Least Connections

Kong sends traffic to server with:

* fewest active connections

Good for:

* uneven workloads

---

# 3. Consistent Hashing

Same user always goes to same backend.

Used for:

* sessions
* caching
* sticky behavior

---

# 4. Latency-Based

Traffic sent to fastest backend.

---

# Health Checks in Upstreams

Kong continuously checks backend health.

If server dies:

* Kong removes it automatically.

---

# Passive Health Checks

Based on failed requests.

Example:

* 5xx errors
* timeouts

---

# Active Health Checks

Kong periodically pings targets.

Example:

```text
GET /health
```

---

# Example with Health Checks

```yaml
upstreams:
  - name: user-upstream
    healthchecks:
      active:
        http_path: /health
```

---

# Difference: Service vs Upstream

| Component | Purpose                     |
| --------- | --------------------------- |
| Service   | Defines backend application |
| Upstream  | Group of backend servers    |
| Target    | Actual backend server       |

---

# VM vs Kubernetes Upstream Usage

# VM Setup

You manually define targets:

```yaml
targets:
  - 127.0.0.1:5678
```

Manual scaling.

---

# Kubernetes Setup

Kong automatically discovers pods via Kubernetes Service.

Dynamic targets.

---

# Production Importance

Upstreams are critical for:

* scalability
* resilience
* high availability
* zero downtime deployments

Without upstreams:

* Kong is just a router.

With upstreams:

* Kong becomes a full API traffic manager.

---

# Final Understanding

## Route

Decides:

> WHICH request matches

---

## Service

Decides:

> WHICH application handles request

---

## Upstream

Decides:

> WHICH backend instance receives request

---

## Target

Actual server instance

---

# Complete Mental Model

```text
ROUTE
  ↓
SERVICE
  ↓
UPSTREAM
  ↓
TARGET
  ↓
BACKEND SERVER
```

This is the core internal architecture of Kong Gateway.
