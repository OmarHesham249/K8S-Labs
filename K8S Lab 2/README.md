# 🐳 ITI K8S Lab2 - Kubectl Config, K3S & Deployments

## 📌 Overview
This lab covers essential Kubernetes operational skills including cluster context management, custom kubectl plugins, and deployment configuration with environment variables.

---

## 🏗️ Lab Objectives
- Build a K3s cluster with 1 server and 1 worker
- Create a namespace and configure a custom kubectl context
- Write a custom `kubectl` plugin to list node hostnames
- Deploy NGINX with 3 replicas and custom environment variables

---

## ⚙️ Prerequisites
- Two Linux machines (RHEL / Rocky / AlmaLinux)
- Root or sudo access
- K3s installed on both nodes
- `kubectl` available on master node

---

## 🔧 Task 1 — Kubectl Config (5 pts)

### a. Create K3s Cluster (Server + Agent)

**On Master (server1):**
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_SELINUX_RPM=true sh -
```

Get the node token for the worker to join:
```bash
cat /var/lib/rancher/k3s/server/node-token
```

**On Worker (worker1):**
```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_SKIP_SELINUX_RPM=true \
  K3S_URL=https://<MASTER-IP>:6443 \
  K3S_TOKEN=<NODE-TOKEN> \
  sh -
```

**Verify both nodes are Ready:**
```bash
kubectl get nodes
```

📸 Cluster Status:

![Worker Joined Successfully](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%202/screenshots/Day%202%20The%20worker%20joined%20master%20successfully.png)

---

### b. Create Namespace `iti-46`

```yaml
# deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: iti-46
```

```bash
kubectl apply -f deployment.yaml
```

📸 Namespace Created:

![Adding iti-46 Namespace](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%202/screenshots/Adding%20iti-46%20namespace.png)

---

### c. Add `iti-46-context` to kubectl config

Edit `/etc/rancher/k3s/k3s.yaml` to add the new context:

```yaml
contexts:
- context:
    cluster: default
    user: default
  name: default
- context:
    cluster: default
    namespace: iti-46
    user: default
  name: iti-46-context

current-context: iti-46-context
```

Switch to the new context:
```bash
kubectl config use-context iti-46-context
```

📸 Context Configured:

![Adding iti Context](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%202/screenshots/Adding%20iti%20context.png)

---

## 🔌 Task 2 — Kubectl Plugin (5 pts)

### Create `kubectl-hostnames` plugin

```bash
cat <<'EOF' > /usr/local/bin/kubectl-hostnames
#!/bin/bash
kubectl get nodes -o custom-columns="HOSTNAMES:.metadata.name"
EOF

chmod +x /usr/local/bin/kubectl-hostnames
```

Run the plugin:
```bash
kubectl hostnames
```

📸 Plugin Output:

![Printing Hostnames Plugin](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%202/screenshots/Printing%20hostnames%20plugin.png)

---

## 📦 Task 3 — Creating Deployments (10 pts)

### NGINX Deployment with `FOO=ITI` env variable

```yaml
# deployment-main.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        env:
        - name: FOO
          value: "ITI"
```

```bash
kubectl apply -f deployment-main.yaml
kubectl get pods
kubectl describe pod | grep FOO
```

📸 Pods with FOO=ITI:

![Making 3 Pods with FOO in env](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%202/screenshots/Making%203%20pods%20with%20FOO%20in%20env.png)

---

## 🎯 Results Summary

| Task | Description | Status |
|------|-------------|--------|
| Cluster Setup | K3s server + agent joined and Ready | ✅ |
| Namespace | `iti-46` namespace created | ✅ |
| Context | `iti-46-context` added and active | ✅ |
| Plugin | `kubectl hostnames` prints all node names | ✅ |
| Deployment | 3 NGINX replicas with `FOO=ITI` env | ✅ |
