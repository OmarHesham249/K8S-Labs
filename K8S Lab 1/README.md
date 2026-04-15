# 🐳 ITI K8S Lab1 - Kubernetes Cluster Setup using kubeadm
## 📌 Overview
This lab demonstrates how to build a basic Kubernetes cluster using `kubeadm`, consisting of:
- One Master Node
- One Worker Node
- Verified cluster connectivity
- Simple NGINX Deployment with 3 replicas
---
## 🏗️ Lab Objectives
- Install Kubernetes using kubeadm
- Initialize a master node
- Join a worker node to the cluster
- Verify node communication and readiness
- Deploy a simple application (NGINX)
---
## ⚙️ Prerequisites
Before starting, ensure you have:
- Two Linux machines (or VMs) running RHEL / Rocky Linux / AlmaLinux
- Root or sudo access
- Internet connectivity
- Container runtime installed (containerd)
- Swap disabled

---

## 🔒 SELinux Configuration
Run the following on **both Master and Worker nodes**:


### Check current SELinux status
```bash
sestatus
```
### Set SELinux to permissive mode (required for Kubernetes networking)
```bash
sudo setenforce 0
```
### Make the change persistent across reboots
```bash
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```
### Verify the change
```bash
grep ^SELINUX= /etc/selinux/config
```

> ⚠️ **Note:** Setting SELinux to `permissive` means violations are logged but not enforced.
> This is the recommended approach for Kubernetes on RHEL.
> Do **not** set it to `disabled` unless absolutely necessary, as that requires a full reboot.

---

## 📥 Step 1: Install kubeadm, kubelet, kubectl
Run the following commands on **both Master and Worker nodes**:

```bash
# Install required dependencies
sudo dnf install -y curl wget

# Add Kubernetes repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
### Install kubelet, kubeadm, kubectl
```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
### Enable and start kubelet
```bash
sudo systemctl enable --now kubelet
```

## 🚀 Step 2: Initialize Master Node
On the Master node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Then configure kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 🌐 Step 3: Install Pod Network (Flannel)
On Master node:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 🤝 Step 4: Join Worker Node
On Worker node, run the command provided by kubeadm init, example:
```bash
kubeadm join <MASTER-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

## ✅ Step 5: Verify Cluster
On Master:
```bash
kubectl get nodes
```
Expected output:
Master → Ready
Worker → Ready

📸 Cluster Status Screenshot:

![Status](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%201/screenshots/The%20worker%20joined%20and%20ready.png)

## 📦 Step 6: Deploy NGINX Application
Create deployment:
```bash
kubectl create deployment nginx-deployment --image=nginx
```
Scale to 3 replicas:
```bash
kubectl scale deployment nginx-deployment --replicas=3
```
Check pods:
```bash
kubectl get pods
```
📸 Deployment Screenshot (3 NGINX Pods):

![Deploying 3 replicas](https://github.com/OmarHesham249/K8S-Labs/blob/main/K8S%20Lab%201/screenshots/After%20deploying%20the%203%20replicas.png)

🎯 Result
Kubernetes cluster successfully created using kubeadm
Master and Worker nodes are connected and Ready
NGINX deployed with 3 running replicas
