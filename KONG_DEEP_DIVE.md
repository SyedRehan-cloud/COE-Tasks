# 🧠 Kong Gateway Deep Architecture Guide (Full Detailed Notes)

This document explains in depth how :contentReference[oaicite:0]{index=0} works internally, including:

- Routing engine design
- Request processing algorithm
- Load balancing system
- Routing types (deep explanation + behavior)
- Real-world execution flow

This is written as **engineering-level learning notes**.

---

# 📌 PAGE 1–2: KONG ARCHITECTURE + CORE REQUEST ALGORITHM

---

# 1. What Kong Really Is (Deep Understanding)

:contentReference[oaicite:1]{index=1} is not just a proxy.

It is a **distributed API execution engine** that sits between clients and backend services.

### It combines 4 systems:

## 1. Reverse Proxy
- Receives HTTP requests
- Forwards them to upstream services

## 2. Routing Engine
- Matches request → correct backend

## 3. Plugin Engine
- Adds behavior without changing backend code:
  - auth
  - rate limit
  - logging
  - transformations

## 4. Load Balancer
- Chooses which backend instance to hit

---

# 2. Internal Request Processing Pipeline (VERY IMPORTANT)

When a request enters Kong, it follows a strict pipeline.

---

## STEP 1 — Request Reception

Client sends:

```

GET /users/123 HTTP/1.1
Host: api.example.com

```

Kong receives it on:
- 8000 (proxy HTTP)
- 8443 (proxy HTTPS)

---

## STEP 2 — Request Parsing

Kong extracts:

- Path → `/users/123`
- Method → GET
- Host → api.example.com
- Headers → all metadata
- Query params → if present

This forms a **request context object**.

---

## STEP 3 — Route Matching Engine (CORE LOGIC)

Kong now compares request against all routes.

### Matching priority:

```

1. Host match (strongest)
2. Path match
3. Headers match
4. Methods match

```

### Rule:
👉 The more specific route wins

---

### Example:

| Route Type | Match Priority |
|------------|---------------|
| Host + Path + Header | Highest |
| Host + Path | High |
| Path only | Medium |
| Catch-all | Lowest |

---

## STEP 4 — Plugin Execution (Request Phase)

Before forwarding:

Kong runs request plugins:

- Authentication (JWT, API key)
- Rate limiting
- IP restriction
- Request transformation

👉 Important:
Plugins run BEFORE backend is called.

---

## STEP 5 — Service Resolution

Route maps to Service:

```

Route → Service → Upstream

```

Example:

```

/users → user-service → user-upstream

````

Service defines:
- upstream URL OR upstream group

---

## STEP 6 — Load Balancer Selection

If multiple backend instances exist:

Kong selects one using algorithm.

We will cover this in PAGE 3.

---

## STEP 7 — Path Rewriting (strip_path)

If:

```yaml
strip_path: true
````

Then:

```
/users → /
```

This ensures backend gets clean path.

---

## STEP 8 — Forward Request

Kong forwards request to selected backend.

---

## STEP 9 — Response Handling

Backend responds:

```
200 OK + JSON
```

---

## STEP 10 — Response Plugins

Kong processes response:

* add headers
* logging
* caching
* transformations

---

## STEP 11 — Response Returned

Final response sent to client.

---

# 📌 PAGE 3–4: ROUTING SYSTEM (DEEP EXPLANATION)

---

# 3. What is Routing in Kong?

Routing in Kong Gateway is:

> A rule-based decision engine that maps requests → services

It does NOT use simple IF conditions.

Instead, it uses:

* pattern matching
* scoring system
* specificity ranking

---

# 4. Routing Decision Algorithm

When multiple routes match:

Kong uses:

### Priority scoring:

```
Host match → +50
Path match → +30
Headers match → +20
Method match → +10
```

Highest score wins.

---

# 5. Routing Types (DETAILED)

---

## 🔹 5.1 Path-Based Routing

### Example:

```
/users → service A
```

### Internal behavior:

* checks URI prefix
* supports exact + regex
* matches longest prefix first

### Why it works:

Kong uses a **Trie-like pattern matcher** internally.

---

### Real Flow:

```
/users/123
↓
matches /users
↓
route selected
```

---

## 🔹 5.2 Host-Based Routing

### Example:

```
api.company.com → Service A
admin.company.com → Service B
```

### How it works:

* Kong reads Host header
* matches exact string

### Why important:

Used for:

* multi-tenant APIs
* domain separation

---

## 🔹 5.3 Method-Based Routing

### Example:

```
GET /users → allowed
POST /users → blocked
```

### Internal logic:

Kong treats method as filter:

```
(route.path match) AND (method match)
```

---

## 🔹 5.4 Header-Based Routing

### Example:

```
version: v1 → Service A
version: v2 → Service B
```

### Internal behavior:

* headers stored in hash map
* exact match required

---

## 🔹 5.5 Regex Routing

### Example:

```
~/users/\d+
```

### Meaning:

* `\d+` → only numbers allowed

### Internal engine:

* regex compiled at startup
* evaluated per request

---

## 🔹 5.6 Multi-path Routing

### Example:

```
/orders
/purchases → same service
```

### Why used:

* reduces duplication
* supports API grouping

---

# 📌 PAGE 5–6: LOAD BALANCING SYSTEM (VERY DEEP)

---

# 6. What is Load Balancing in Kong?

Load balancing in Kong Gateway is:

> The process of distributing requests across multiple upstream servers

---

# 7. Architecture

```
Service
  ↓
Upstream
  ↓
Targets (multiple servers)
```

Example:

```
user-service
  ↓
user-upstream
  ↓
127.0.0.1:5678
127.0.0.1:5679
127.0.0.1:5680
```

---

# 8. Load Balancing Algorithms

---

## 🔹 8.1 Round Robin (DEFAULT)

### Flow:

```
Req1 → A
Req2 → B
Req3 → C
Req4 → A
```

### Why used:

* simple fairness
* predictable distribution

---

## 🔹 8.2 Least Connections

### Flow:

Sends request to server with least load.

Example:

```
A → 10 connections
B → 2 connections → selected
```

### Why used:

* performance optimization

---

## 🔹 8.3 Consistent Hashing

### Flow:

```
hash(user_id) → backend
```

### Result:

Same user always hits same server.

### Used for:

* sessions
* caching consistency

---

## 🔹 8.4 Weighted Load Balancing

### Example:

```
Server A → weight 80
Server B → weight 20
```

### Flow:

* A receives 80% traffic
* B receives 20% traffic

---

## 🔹 8.5 Health Checks (Active/Passive)

### Active checks:

* Kong pings backend periodically

### Passive checks:

* detects failed requests

### If server fails:

* removed automatically

---

# 9. Load Balancing Execution Flow

When request arrives:

```
Client → Route → Service → Upstream → LB Algorithm → Target → Response
```

---

# 10. Why Kong replaces traditional LB (partially)

Kong can replace LB because it supports:

✔ routing
✔ load balancing
✔ authentication
✔ rate limiting
✔ observability

BUT:

❌ It does NOT replace:

* global load balancers (AWS ALB)
* CDN edge routing
* multi-region failover systems
