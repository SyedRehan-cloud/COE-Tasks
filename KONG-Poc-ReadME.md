# KONG POC — SIMPLE KUBERNETES IMPLEMENTATION

<img width="340" height="230" alt="image" src="https://github.com/user-attachments/assets/90f82e30-a73d-4d75-8eb5-94a8ccc85887" />


# 1. Overview

This POC demonstrates:

* Kong deployment on Kubernetes
* API routing using ingress
* Kubernetes service discovery
* Request forwarding through Kong

# 2. Objective

The objective is to:

* Deploy Kong Gateway
* Deploy sample microservice
* Configure ingress
* Test API routing

---

# 3. Architecture

```txt id="4sht0m"
Client
   ↓
Kong Gateway
   ↓
Kubernetes Service
   ↓
Application Pod
```

# 4. Prerequisites

| Requirement        | Description    |
| ------------------ | -------------- |
| Kubernetes Cluster | Minikube / EKS |
| kubectl            | Installed      |
| Helm               | Installed      |


# 5. Folder Structure

```txt id="cfxb7l"
kong-poc/
├── namespace.yaml
├── app-deployment.yaml
├── app-service.yaml
├── ingress.yaml
├── deploy.sh
└── cleanup.sh
```

---

# 6. Install Kong

## Add Helm Repository

```bash id="hvrv6n"
helm repo add kong https://charts.konghq.com
helm repo update
```

---

## Install Kong

```bash id="9r6vbo"
helm install kong kong/kong \
  --namespace kong \
  --create-namespace
```

---

## Verify Installation

```bash id="wgtmly"
kubectl get pods -n kong
```

Expected:

```txt id="w2l4db"
kong-controller-manager Running
kong-gateway Running
```

---

# 7. Deploy Application

# namespace.yaml

```yaml id="w7nfrx"
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

---

# app-deployment.yaml

```yaml id="lbhjj1"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Kong POC"
        ports:
        - containerPort: 5678
```

---

# app-service.yaml

```yaml id="mvh0qe"
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: demo
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 5678
```

---

# Apply Resources

```bash id="h1m8mx"
kubectl apply -f namespace.yaml
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml
```

---

# 8. Configure Ingress

# ingress.yaml

```yaml id="hkrg90"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kong-demo
  namespace: demo
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

---

# Apply Ingress

```bash id="lg6mv9"
kubectl apply -f ingress.yaml
```

---

# 9. Verify Setup

## Check Pods

```bash id="l0d5yf"
kubectl get pods -A
```

---

## Check Services

```bash id="p2gc6k"
kubectl get svc -A
```

---

## Check Ingress

```bash id="v52oc5"
kubectl get ingress -A
```

---

# Get Kong External IP

```bash id="0pjsl0"
kubectl get svc -n kong
```

Look for:

```txt id="8mrfhn"
kong-kong-proxy
```

---

# 10. Testing

## Test API

```bash id="55zg6i"
curl http://<EXTERNAL-IP>/users
```

Expected Output:

```txt id="i7g74z"
Hello from Kong POC
```

---

# Request Flow

```txt id="jglg7m"
Client
   ↓
Kong Gateway
   ↓
Ingress Rule
   ↓
Kubernetes Service
   ↓
Application Pod
```

# 11. Cleanup

```bash id="l3x5uk"
kubectl delete namespace demo
helm uninstall kong -n kong
```

# 12. Troubleshooting

| Issue               | Cause              | Solution       |
| ------------------- | ------------------ | -------------- |
| 404 from Kong       | Route mismatch     | Verify ingress |
| Service unreachable | Wrong service name | Check service  |
| External IP pending | No cloud LB        | Use NodePort   |
| Pod crash           | Invalid manifest   | Check logs     |
