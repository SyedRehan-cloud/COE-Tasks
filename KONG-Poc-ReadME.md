# Kong Kubernetes POC

# 1. Objective

* Deploy Kong on Kubernetes
* Deploy microservice
* Expose API via Kong
* Test routing

---

# 2. Architecture

```mermaid id="pocarch"
flowchart LR
    Client["Client"] --> Kong["Kong Gateway"]
    Kong --> Service["Kubernetes Service"]
    Service --> Pod["Application Pod"]
```

---

# 3. Folder Structure

```txt id="pocstruct"
kong-poc/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── ingress.yaml
```

---

# 4. Install Kong

```bash id="pocinstall"
helm repo add kong https://charts.konghq.com
helm repo update

helm install kong kong/kong \
  --namespace kong \
  --create-namespace
```

---

# 5. Namespace

```yaml id="pocns"
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

---

# 6. Deployment

```yaml id="pocdep"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Kong POC"
        ports:
        - containerPort: 5678
```

---

# 7. Service

```yaml id="pocsvc"
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: demo
spec:
  selector:
    app: user
  ports:
  - port: 80
    targetPort: 5678
```

---

# 8. Ingress

```yaml id="pocing"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kong-ingress
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

# 9. Deploy

```bash id="pocdeploy"
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

---

# 10. Flow

```mermaid id="pocflow"
flowchart LR
    Client["Client"] --> Kong["Kong"]
    Kong --> Service["Service"]
    Service --> Pod["Pod"]
```

---

# 11. Test API

```bash id="poctest"
curl http://<KONG-IP>/users
```

Expected:

```txt id="pocres"
Hello from Kong POC
```

---

# 12. Cleanup

```bash id="pocclean"
kubectl delete namespace demo
helm uninstall kong -n kong
```
