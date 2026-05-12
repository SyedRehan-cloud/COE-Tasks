# # Kong API Gateway Deployment Models

## (VM vs Kubernetes – Architecture, Working, and Comparison)


# 1. Introduction

Kong Gateway is an API gateway built on top of NGINX and LuaJIT that manages traffic between clients and backend services. It acts as a **reverse proxy, router, and policy enforcement layer** for APIs.

Kong can be deployed in two major ways:

* On a **Virtual Machine (VM / EC2 / Bare Metal)**
* Inside **Kubernetes (K8s Ingress / Gateway API)**

Both approaches solve API traffic management, but their design goals differ:

* VM deployment focuses on **simplicity and control**
* Kubernetes deployment focuses on **scalability and automation**


# 2. Kong Architecture Overview (Common for Both)

Regardless of environment, Kong follows the same internal flow:

## Request Flow (Core Lifecycle)

1. Client sends request
2. Kong receives request (Proxy port 8000)
3. URI parsing happens
4. Route matching (path/host/header/method/regex)
5. Plugin execution (request phase)
6. Service selection (upstream target resolution)
7. Path rewrite (if configured)
8. Request forwarded to upstream
9. Response received from upstream
10. Response plugins executed
11. Response returned to client

---

## Internal Components

* **Proxy Layer (8000)** → Handles traffic
* **Admin API (8001)** → Configuration management
* **DB-less mode / DB mode** → Configuration storage
* **NGINX Worker processes** → Handle request execution
* **Plugins engine** → Authentication, logging, rate limiting, etc.

---

# 3. Kong on VM (Traditional Deployment Model)

## 3.1 What it means

In VM-based deployment, Kong runs as a **system service (systemd)** directly on an operating system like Ubuntu or EC2 instance.

Example:

```bash
sudo systemctl start kong
sudo systemctl restart kong
systemctl status kong
```

Configuration is stored in:

* `/etc/kong/kong.conf`
* or `kong.yml` (DB-less mode)

---

## 3.2 Architecture on VM

```
Client
  ↓
VM IP (Kong Proxy: 8000)
  ↓
Route Matching Engine
  ↓
Upstream Service (localhost / internal IP / external API)
```

## 3.3 How VM-based Kong works

* Kong runs as a **single installed binary**
* Uses systemd to manage lifecycle
* Uses static or declarative config
* Routes are manually defined

Example flow:

```
GET /users
→ Kong checks route (/users)
→ Matches service
→ Forwards to http://127.0.0.1:5678
```

## 3.4 Advantages

* Very simple setup
* Fast debugging (`journalctl`, logs, CLI)
* Full OS-level control
* Ideal for learning and PoCs
* No dependency on orchestration systems


## 3.5 Limitations

* No automatic scaling
* Manual service discovery
* Hard to manage large systems
* Single-node failure risk unless manually replicated


## 3.6 When to use VM Kong

* Learning API Gateway concepts
* Small applications
* Development environments
* Proof of concepts
* Simple microservice setups


# 4. Kong on Kubernetes (Cloud-Native Model)

## 4.1 What it means

In Kubernetes, Kong runs as:

* Ingress Controller OR
* Gateway API controller

It integrates with Kubernetes API server for dynamic configuration.

---

## 4.2 Architecture on Kubernetes

```
Client
  ↓
Kubernetes Ingress / LoadBalancer
  ↓
Kong Pods (Ingress Controller)
  ↓
Kubernetes Service (ClusterIP)
  ↓
Pods (Backend applications)
```


## 4.3 How Kubernetes Kong works

* Kong is deployed as a **containerized pod**
* Services and routes are defined using **Kubernetes YAML**
* Kubernetes automatically manages:

  * Service discovery
  * Scaling
  * Health checks

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-ingress
spec:
  rules:
  - host: users.local
    http:
      paths:
      - path: /users
        backend:
          service:
            name: user-service
            port:
              number: 80
```

## 4.4 Advantages

* Auto scaling (horizontal pod autoscaling)
* Built-in service discovery
* Self-healing infrastructure
* GitOps-friendly configuration
* Highly scalable microservice architecture

---

## 4.5 Limitations

* Complex setup and debugging
* Requires Kubernetes expertise
* More components = more failure points
* Higher operational overhead


## 4.6 When to use Kubernetes Kong

* Large-scale microservices
* Production-grade systems
* High traffic APIs
* Multi-team environments
* Cloud-native architecture

# 5. VM vs Kubernetes Kong (Deep Comparison)

| Feature                | VM Deployment           | Kubernetes Deployment |
| ---------------------- | ----------------------- | --------------------- |
| Setup Complexity       | Low                     | High                  |
| Learning Curve         | Easy                    | Moderate–Hard         |
| Scaling                | Manual                  | Automatic             |
| Service Discovery      | Manual config           | Built-in              |
| Deployment Type        | System service          | Containerized         |
| Fault Tolerance        | Manual setup            | Auto-healing          |
| Debugging              | Easy                    | Complex               |
| Best Fit               | Small systems, learning | Enterprise systems    |
| Infrastructure Control | High                    | Abstracted            |
| DevOps Maturity        | Low                     | High                  |

# 6. Key Decision Logic

## Use VM-based Kong when:

* You are learning API Gateway concepts
* You want quick setup
* You are working with few services
* You prefer simplicity over automation


## Use Kubernetes-based Kong when:

* You have microservices architecture
* You need auto-scaling and high availability
* You deploy frequently (CI/CD pipelines)
* You manage many services across teams


# 7. Important Insight (Real-World Understanding)

Kong itself does NOT change between VM and Kubernetes.

What changes is:

* How Kong is deployed
* How configuration is managed
* How services are discovered

So:

> Kong = Traffic Engine
> Kubernetes = Infrastructure Orchestrator
> VM = Manual Infrastructure Layer

# 8. Final Summary

* VM-based Kong = **Simple, manual, controlled**
* Kubernetes-based Kong = **Scalable, automated, production-grade**
* Both use the same core Kong routing engine
* Choice depends on **scale + operational maturity**
