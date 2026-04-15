# 🐳 ITI K8S Lab4 - Persistent Volumes, Downward API & ConfigMaps

## 📌 Overview
This lab covers Kubernetes storage and configuration management including Persistent Volumes with hostPath, the Downward API for pod metadata injection, and ConfigMaps mounted as environment variables and volume files.

---

## 🏗️ Lab Objectives
- Create and bind PersistentVolumes and PersistentVolumeClaims
- Mount PVCs inside deployments with node pinning
- Use the Downward API to expose pod metadata into files
- Create ConfigMaps from files and literals, mount them as env vars and volumes

---

## ⚙️ Prerequisites
- Running K3s cluster (server + worker)
- `kubectl` configured on master node
- Root access to the host node for `hostPath` directories

---

## 💾 Task 1 — Persistent Volumes (5 pts)

### a & b. Create PV and PVC

Prepare the hostPath directory and `index.html`:

```bash
sudo mkdir -p /data/nginx-pv
echo "Omar Hesham" >> /data/nginx-pv/index.html
```

```yaml
# nginx-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/nginx-data
```

```yaml
# nginx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f nginx-pv.yaml
kubectl apply -f nginx-pvc.yaml
```

📸 PV and PVC YAML files:

![Creating PV and PVC YAML files](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/1-%20creating%20pv%20and%20pvc%20yaml%20files%20.png)

---

### c & d. Create Deployment (3 replicas, pinned to server1)

```yaml
# nginx-pv-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pv-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pv
  template:
    metadata:
      labels:
        app: nginx-pv
    spec:
      nodeName: server1
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-storage
        persistentVolumeClaim:
          claimName: nginx-pvc
```

```bash
kubectl apply -f nginx-pv-deployment.yaml
```

> 📝 `nodeName: server1` pins all 3 replicas to the same node, ensuring they share the same hostPath data.

📸 Deployment YAML:

![Creating the Deployment File](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/2-%20creating%20the%20deployment%20file.png)

**Verify PV, PVC, and Pods:**
```bash
kubectl get pv nginx-pv
kubectl get pvc nginx-pvc
kubectl get pods -o wide
```

**Verify `index.html` content from inside a pod:**
```bash
kubectl exec -it <pod-name> -- /bin/bash
cat /usr/share/nginx/html/index.html
```

📸 Applied and Verified:

![Applying Files and Verifying](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/3-%20Applying%20files%20and%20verifying%20its%20running%20and%20pods.png)

---

## 📡 Task 2 — Downward API (5 pts)

### a & b. Create Downward PV and PVC

```yaml
# downward-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: downward-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /data/downward-data
```

```yaml
# downward-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: downward-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f downward-pv.yaml
kubectl apply -f downward-pvc.yaml
```

📸 Downward PV and PVC:

![Creating Downward PV and PVC](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/4-%20Creating%20downward%20pv%20and%20pvc.png)

---

### c. Create Deployment with Downward API + initContainer

```yaml
# downward-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: downward-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: downward
  template:
    metadata:
      labels:
        app: downward
    spec:
      initContainers:
      - name: write-pod-info
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Pod Name: $POD_NAME" > /data/index.html
          echo "Pod IP: $POD_IP" >> /data/index.html
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: downward-storage
          mountPath: /data
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: downward-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: downward-storage
        persistentVolumeClaim:
          claimName: downward-pvc
```

```bash
kubectl apply -f downward-deployment.yaml
kubectl get pods -o wide
```

📸 Deployment File:

![Creating Deployment File](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/5-%20creating%20deployment%20file.png)

📸 Applied and Running:

![Applying and Verifying Pods Running](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/6-%20applying%20and%20verifying%20the%20pods%20is%20running.png)

**Verify `index.html` contains Pod Name and IP:**
```bash
kubectl exec -it <pod-name> -- /bin/bash
cat /usr/share/nginx/html/index.html
```

📸 Verification from Inside Pod:

![Verifying from the Pod](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/7-%20verifying%20from%20the%20pod.png)

---

## 🗂️ Task 3 — ConfigMaps (10 pts)

### a. Create ConfigMap `birke` from file

```bash
vim /opt/cm.yaml
```

```yaml
# /opt/cm.yaml
apiVersion: v1
data:
  tree: birke
  level: "3"
  department: park
kind: ConfigMap
metadata:
  name: birke
```

```bash
kubectl apply -f /opt/cm.yaml
kubectl get configmap birke
```

---

### b. Create ConfigMap `trauerweide` from literal

```bash
kubectl create configmap trauerweide --from-literal=tree=trauerweide
kubectl get configmap trauerweide
```

📸 ConfigMaps Created:

![Creating CM File and Applied](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/8-%20Creating%20cm%20file%20and%20applied.png)

---

### c. Create Pod `pod1` with env var + volume mount

```yaml
# pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    env:
    - name: TREE1
      valueFrom:
        configMapKeyRef:
          name: trauerweide
          key: tree
    volumeMounts:
    - name: birke-volume
      mountPath: /etc/birke
  volumes:
  - name: birke-volume
    configMap:
      name: birke
```

```bash
kubectl apply -f pod1.yaml
kubectl get pod pod1
```

**Verify env var and mounted files:**
```bash
# Check TREE1 env variable
kubectl exec -it pod1 -- env | grep TREE1

# List mounted ConfigMap keys as files
kubectl exec -it pod1 -- ls /etc/birke

# Read each file
kubectl exec -it pod1 -- cat /etc/birke/tree
kubectl exec -it pod1 -- cat /etc/birke/level
kubectl exec -it pod1 -- cat /etc/birke/department
```

📸 Everything Running and Verified:

![Verifying Everything is Running](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%204/screenshots/9-%20Verifying%20everything%20is%20running.png)

---

## 🎯 Results Summary

| Task | Description | Status |
|------|-------------|--------|
| PV `nginx-pv` | hostPath 1Gi, Retain policy, ReadWriteMany | ✅ |
| PVC `nginx-pvc` | Bound to `nginx-pv` | ✅ |
| nginx-pv-deployment | 3 replicas pinned to server1, PVC mounted | ✅ |
| Downward PV/PVC | `downward-pv` and `downward-pvc` created | ✅ |
| downward-deployment | initContainer writes Pod Name + IP to index.html | ✅ |
| ConfigMap `birke` | Created from `/opt/cm.yaml` (3 keys) | ✅ |
| ConfigMap `trauerweide` | Created from literal `tree=trauerweide` | ✅ |
| Pod `pod1` | `TREE1` env var set, `birke` keys mounted at `/etc/birke/*` | ✅ |
