# 🐳 ITI K8S Lab3 - DNS, Ingress & Services

## 📌 Overview
This lab covers Kubernetes networking fundamentals including DNS resolution, NodePort services, Ingress routing, and ClusterIP services for multi-deployment setups.

---

## 🏗️ Lab Objectives
- Deploy a web application and expose it via NodePort
- Verify DNS/SRV record resolution inside the cluster
- Access services using internal DNS domain names
- Set up Ingress routing for multiple deployments using path-based routing

---

## ⚙️ Prerequisites
- Running K3s cluster (server + worker)
- `kubectl` configured on master node
- Ingress controller installed (K3s includes Traefik by default)

---

## 🌐 Task 1 — DNS

### a. Create `web` Deployment in `iti5` Namespace

```yaml
# deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-deployment
  name: web-deployment
  namespace: iti5
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-deployment
  strategy: {}
  template:
    metadata:
      labels:
        app: web-deployment
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}
```

```bash
kubectl create ns iti5
kubectl apply -f deployment-web.yaml
```

📸 Deployment YAML:

![Task 1b - Web Deployment](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/Task%201%20b.png)

---

### b. Create NodePort Service (port 5000 → target 80)

Generate the service YAML using dry-run:

```bash
kubectl expose deployment web-deployment \
  --name=web-service \
  --type=NodePort \
  --port=5000 \
  --target-port=80 \
  --namespace=iti5 \
  --dry-run=client -o yaml > web-service.yaml

kubectl apply -f web-service.yaml
```

📸 NodePort Service YAML:

![Exposing NodePort](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/Exposing%20NodePort.png)

---

### c. List SRV Record & Verify DNS Resolution

Run a temporary pod with `nslookup` to check the SRV record:

```bash
kubectl run dns-test --image=busybox:1.28 -i --tty --rm -- sh

# Inside the pod:
nslookup -type=SRV _http._tcp.web-service.iti5.svc.cluster.local
```

The SRV record format resolves to:
```
_port._protocol.<service>.<namespace>.svc.cluster.local
```

---

### d. Reach Service from Test Pod Using Domain Name

Spin up a temporary curl pod in the **default** namespace:

```bash
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- /bin/sh

# Inside the pod — reach service using its FQDN:
curl web-service.iti5.svc.cluster.local:5000
```

📸 Curl from Pod to Service via DNS:

![Curl from Pod to Another](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/curl%20from%20pod%20to%20another.png)

---

## 🔀 Task 2 — Ingress & Services

### a. Create Namespaces

```bash
kubectl create ns new-iti-46
kubectl create ns world
```

📸 Namespace & Deployment Creation:

![Task 2a and 2b](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/task%202%20a%20b.png)

---

### b. Create `africa` and `europe` Deployments in `world` Namespace

```bash
# Generate africa deployment YAML
kubectl create deployment africa-deployment \
  --image=husseingalal/africa:latest \
  --replicas=2 \
  -n world \
  --dry-run=client -o yaml > deployment-africa.yaml

# Generate europe deployment YAML
kubectl create deployment europe-deployment \
  --image=husseingalal/europe:latest \
  --replicas=2 \
  -n world \
  --dry-run=client -o yaml > deployment-europe.yaml

kubectl apply -f deployment-africa.yaml
kubectl apply -f deployment-europe.yaml
```

---

### c. Create ClusterIP Services for Both Deployments

```bash
kubectl expose deployment africa-deployment \
  --name=africa \
  --port=8888 \
  --target-port=80 \
  --type=ClusterIP \
  -n world

kubectl expose deployment europe-deployment \
  --name=europe \
  --port=8888 \
  --target-port=80 \
  --type=ClusterIP \
  -n world
```

📸 Services Created:

![Task 2c - Making Services](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/Task%202%20C%20making%20services.png)

---

### d. Create Ingress Resource `world`

First, add the node IP to `/etc/hosts` on your machine:
```bash
echo "<NODE-IP>  world.universe.mine" >> /etc/hosts
```

Create the Ingress:

```yaml
# ingress-world.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: world
  namespace: world
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: world.universe.mine
    http:
      paths:
      - path: /europe
        pathType: Prefix
        backend:
          service:
            name: europe
            port:
              number: 8888
      - path: /africa
        pathType: Prefix
        backend:
          service:
            name: africa
            port:
              number: 8888
```

```bash
kubectl apply -f ingress-world.yaml
```

**Verify Ingress routing:**
```bash
curl http://world.universe.mine/europe
curl http://world.universe.mine/africa
```

📸 Ingress Routing Result:

![Accessing Africa and Europe from Service](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%203/screenshots/Accessing%20to%20africa%20and%20europe%20from%20service.png)

---

## 🎯 Results Summary

| Task | Description | Status |
|------|-------------|--------|
| Namespace `iti5` | Created with `web-deployment` (2 replicas) | ✅ |
| NodePort Service | `web-service` on port 5000 → target 80 | ✅ |
| DNS Resolution | Service reachable via FQDN from test pod | ✅ |
| Namespace `world` | `africa` and `europe` deployments (2 replicas each) | ✅ |
| ClusterIP Services | `africa` and `europe` on port 8888 | ✅ |
| Ingress `world` | Path-based routing `/africa` and `/europe` working | ✅ |
