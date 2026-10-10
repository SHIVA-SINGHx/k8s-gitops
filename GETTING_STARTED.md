# 🚀 Getting Started — K8s Kind Voting App

A complete step-by-step guide to set up the K8s Kind Voting App from scratch on an AWS EC2 instance — including Argo CD for GitOps, Kubernetes Dashboard, and Prometheus + Grafana monitoring.

---

## 📋 Table of Contents

1. [Launch EC2 & Install Docker](#1-launch-ec2--install-docker)
2. [Install Kind](#2-install-kind)
3. [Create the Kubernetes Cluster](#3-create-the-kubernetes-cluster)
4. [Install kubectl](#4-install-kubectl)
5. [Set Up Argo CD (GitOps)](#5-set-up-argo-cd-gitops)
6. [Expose the Voting & Result Apps](#6-expose-the-voting--result-apps)
7. [Set Up Kubernetes Dashboard](#7-set-up-kubernetes-dashboard)
8. [Install Prometheus & Grafana (Monitoring)](#8-install-prometheus--grafana-monitoring)

---

## 1. Launch EC2 & Install Docker

### Launch EC2 Instance
- **Instance type:** `t2.medium`
- **Storage:** `30 GiB`
- **OS:** Ubuntu
- Open the following **inbound ports** in your Security Group as you go:
  - `8443` — Argo CD UI
  - `8080` — Kubernetes Dashboard
  - `9090` — Prometheus
  - `3000` — Grafana
  - `5000` — Vote App
  - `5001` — Result App
 


### Install Docker

```bash
sudo apt-get update
sudo apt-get install docker.io -y

# Add your user to the docker group (no sudo needed after this)
sudo usermod -aG docker $USER && newgrp docker
```

> ✅ Verify Docker is working: `docker ps`

---

## 2. Install Kind

Kind (Kubernetes IN Docker) lets you run a full Kubernetes cluster inside Docker containers locally.

```bash
# Create the install script
vim install_kind.sh
```

Paste the following content into the file:

```bash
#!/bin/bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo cp ./kind /usr/local/bin/kind
rm -rf kind
```

```bash
# Make executable and run
chmod +x install_kind.sh
./install_kind.sh
```

> ✅ Verify: `kind --version`

---

## 3. Create the Kubernetes Cluster

Create a **3-node cluster** — 1 control-plane + 2 worker nodes — running Kubernetes v1.30.0.

```bash
vim config.yml
```

Paste the following content:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
```

```bash
# Create the cluster
kind create cluster --config=config.yml --name=shiva-cluster
```

> ✅ Verify: `kubectl get nodes`

<img width="651" height="91" alt="Screenshot 2026-09-22 185339" src="https://github.com/user-attachments/assets/6e594ee1-f20a-48da-a2c5-f7da179a9b4d" />


---

## 4. Install kubectl

kubectl is the CLI tool to interact with your Kubernetes cluster.

```bash
vim install_kubectl.sh
```

Paste the following content:

```bash
#!/bin/bash

VERSION="v1.30.0"
URL="https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubectl"
INSTALL_DIR="/usr/local/bin"

curl -LO "$URL"
chmod +x kubectl
sudo mv kubectl $INSTALL_DIR/
kubectl version --client

rm -f kubectl
echo "kubectl installation complete."
```

```bash
chmod +x install_kubectl.sh
./install_kubectl.sh
```

> ✅ Verify: `kubectl version --client`

---

## 5. Set Up Argo CD (GitOps)

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes. It watches your Git repo and automatically syncs the cluster state.

### Install Argo CD

```bash
# Create the Argo CD namespace
kubectl create namespace argocd

# Apply the official Argo CD manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Watch pods come up (wait until all are Running)
kubectl get pods -n argocd

# Check services
kubectl get svc -n argocd
```

### Expose Argo CD UI

```bash
# Patch the service to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Forward port 8443 to the Argo CD server
kubectl port-forward -n argocd service/argocd-server 8443:443 --address=0.0.0.0 &
```

> 🔓 Make sure **port 8443** is open in your EC2 Security Group inbound rules.

### Access Argo CD UI

Open your browser: `https://<EC2-PUBLIC-IP>:8443`

- **Username:** `admin`
- **Password:** Run the command below to get it:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
<img width="1919" height="917" alt="Screenshot 2026-09-26 181749" src="https://github.com/user-attachments/assets/7c141dcb-54b8-43f2-b9ec-e716ebebd3f4" />


### Configure Argo CD for GitOps

1. Log in to the Argo CD UI
2. Create a new **Application** pointing to this Git repository
3. Set the **path** to `k8s-specifications/`
4. Set **sync policy** to `Automatic`

> 💡 **Live scaling demo:** Change the replica count from `1` to `3` in any deployment YAML, push to Git, then refresh the Argo CD UI — you'll see the pods scale up in real time!

<img width="924" height="531" alt="Screenshot 2026-09-26 183548" src="https://github.com/user-attachments/assets/d33d2853-5e64-44af-9bb8-10edaa877353" />
<img width="919" height="562" alt="Screenshot 2026-09-26 183335" src="https://github.com/user-attachments/assets/63f13d3a-3f29-4a07-9b61-9cce026ca7d4" />
<img width="1919" height="904" alt="Screenshot 2026-09-26 183114" src="https://github.com/user-attachments/assets/8c20eadf-7096-4970-8350-e1121e436986" />


### Deploy the Voting App via kubectl (alternatively)

```bash
kubectl apply -f k8s-specifications/
kubectl get all
```

---

## 6. Expose the Voting & Result Apps

```bash
# Expose Vote App on port 5000
kubectl port-forward svc/vote 5000:5000 --address=0.0.0.0 &

# Expose Result App on port 5001
kubectl port-forward svc/result 5001:5001 --address=0.0.0.0 &
```

| App | URL |
|---|---|
| 🗳️ **Vote App** | `http://<EC2-PUBLIC-IP>:5000` |
| 📊 **Result App** | `http://<EC2-PUBLIC-IP>:5001` |

<img width="871" height="834" alt="Screenshot 2026-09-26 184844" src="https://github.com/user-attachments/assets/02ad5972-182c-4d92-9a0f-0b7b8154095d" />
<img width="1470" height="743" alt="Screenshot 2026-09-26 184830" src="https://github.com/user-attachments/assets/3a01e715-a475-4532-b7d0-f89d33cec0e3" />


> 🔓 Make sure ports **5000** and **5001** are open in your EC2 inbound rules.

---

## 7. Set Up Kubernetes Dashboard

The Kubernetes Dashboard gives you a visual, browser-based UI to manage your cluster.

### Install the Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### Create an Admin Service Account

```bash
vim dashboard.yml
```

Paste the following:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dashboard.yml
kubectl get svc -n kubernetes-dashboard
```

### Access the Dashboard

```bash
# Port-forward the dashboard
kubectl port-forward svc/kubernetes-dashboard -n kubernetes-dashboard 8080:443 --address=0.0.0.0 &

# Generate an access token
kubectl -n kubernetes-dashboard create token admin-user
```

Open: `https://<EC2-PUBLIC-IP>:8080`  
Paste the token from the command above to log in.

<img width="1670" height="620" alt="Screenshot 2026-09-26 190245" src="https://github.com/user-attachments/assets/e3ecebf8-8e4f-46b8-b6d8-980b4940abbf" />
<img width="1919" height="908" alt="Screenshot 2026-09-26 190317" src="https://github.com/user-attachments/assets/b79dac44-a2a7-4129-8d80-8913c62b7a78" />


> 🔓 Make sure **port 8080** is open in your EC2 inbound rules.

---

## 8. Install Prometheus & Grafana (Monitoring)

Monitor your cluster and pods using the **kube-prometheus-stack** via Helm.

### Install Helm

```bash
cd ~

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

> ✅ Verify: `helm version`

### Install kube-prometheus-stack

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install the full Prometheus + Grafana + Alertmanager stack
helm install kind-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.nodePort=30000 \
  --set prometheus.service.type=NodePort \
  --set grafana.service.nodePort=31000 \
  --set grafana.service.type=NodePort \
  --set alertmanager.service.nodePort=32000 \
  --set alertmanager.service.type=NodePort \
  --set prometheus-node-exporter.service.nodePort=32001 \
  --set prometheus-node-exporter.service.type=NodePort

# Verify services are up
kubectl get svc -n monitoring
```

### Expose Prometheus & Grafana

```bash
# Expose Prometheus
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 &

# Expose Grafana
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 3000:80 --address=0.0.0.0 &
```

| Tool | URL |
|---|---|
| 🔥 **Prometheus** | `http://<EC2-PUBLIC-IP>:9090` |
| 📊 **Grafana** | `http://<EC2-PUBLIC-IP>:3000` |

> 🔓 Open ports **9090** and **3000** in your EC2 inbound rules.

### Grafana Login Credentials

- **Username:** `admin`
- **Password:** Run this command to get it:

```bash
kubectl --namespace monitoring get secrets kind-prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

### Useful Prometheus Queries

Once inside Prometheus, try these PromQL queries:

```promql
# CPU usage (%) across all pods in the default namespace
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[1m])) / sum(machine_cpu_cores) * 100

# Memory usage per pod
sum(container_memory_usage_bytes{namespace="default"}) by (pod)

# Network receive bytes per pod (5m rate)
sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)

# Network transmit bytes per pod (5m rate)
sum(rate(container_network_transmit_bytes_total{namespace="default"}[5m])) by (pod)
```

---

## 🧹 Cleanup

```bash
# Delete the Kind cluster (removes everything)
kind delete cluster --name=shiva-cluster
```

---

## 🔓 EC2 Inbound Ports Summary


| Port | Service |
|---|---|
| `5000` | Vote App |
| `5001` | Result App |
| `8443` | Argo CD UI |
| `8080` | Kubernetes Dashboard |
| `9090` | Prometheus |
| `3000` | Grafana |

<img width="1779" height="787" alt="Screenshot 2026-09-26 194402" src="https://github.com/user-attachments/assets/7abecaa9-1576-4382-aaad-4efda787bbff" />
---


## Some Screenshot
<img width="871" height="834" alt="Screenshot 2026-09-26 184844" src="https://github.com/user-attachments/assets/08a5ad47-3487-4ffd-9626-e1afc7003e64" />




<div align="center">
  <strong>Happy Shipping! 🚢 — Built with Kubernetes, Kind, Argo CD, Prometheus & Grafana</strong>
</div>
