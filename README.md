# 🗳️ K8s GitOps Voting App

![App Overview](k8s-kind-voting-app.png)

![Kubernetes](https://img.shields.io/badge/Kubernetes-Kind%20v1.30-326CE5?logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-EF7B4D?logo=argo&logoColor=white)
![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Dashboards-Grafana-F46800?logo=grafana&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/Host-AWS%20EC2-FF9900?logo=amazonaws&logoColor=white)

A **cloud-native, microservices-based voting application** deployed on a local Kubernetes cluster using **Kind (Kubernetes IN Docker)**. This project demonstrates real-world Kubernetes patterns including multi-service orchestration, GitOps with Argo CD, and full-stack observability with Prometheus & Grafana — all running on an AWS EC2 instance.

---

## Why This Project

Deploying a multi-service application by hand (running `kubectl apply` from a laptop and hoping nothing drifts) does not scale and leaves no audit trail. This project solves that by treating Git as the single source of truth.

| Problem | Approach |
|---------|----------|
| Manual, error-prone deployments | Declarative Kubernetes manifests, synced automatically by Argo CD |
| Cluster state drifting from what's in the repo | GitOps: Argo CD compares live state against Git and reconciles differences |
| No visibility into cluster and workload health | Prometheus + Grafana via the kube-prometheus-stack Helm chart |
| Services starting in the wrong order (worker up before Redis/Postgres) | Retry logic in the worker for both Redis and PostgreSQL connections |
| Needing a cluster without paying for managed Kubernetes | Kind (Kubernetes IN Docker), 3 nodes on a single EC2 instance |

## Outcomes

- A 5-service, 3-language application (Python, Node.js, .NET) runs on a 3-node cluster from 9 declarative manifests
- Cluster state is defined in Git and reconciled by Argo CD
- Cluster and workload metrics are visible in Prometheus and Grafana (see screenshots below)
- The same app runs locally with Docker Compose and on Kubernetes without code changes

---

## 📌 Table of Contents

- [About the Project](#-about-the-project)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [How It Works](#-how-it-works)
- [Getting Started](#-getting-started)
- [Running with Docker Compose](#-running-with-docker-compose-local-dev)
- [GitOps with Argo CD](#-gitops-with-argo-cd)
- [Monitoring — Prometheus & Grafana](#-monitoring--prometheus--grafana)
- [Kubernetes Manifests Breakdown](#-kubernetes-manifests-breakdown)
- [Screenshots](#-screenshots)

---

## 📖 About the Project

This project is a hands-on implementation of deploying a **polyglot microservices application** on Kubernetes. The goal was to:

- Understand how Kubernetes manages multi-container workloads
- Set up a local K8s cluster using **Kind** (without needing a managed cloud K8s service)
- Implement **GitOps** principles using **Argo CD** for continuous deployment
- Monitor the cluster and workloads using the **kube-prometheus-stack** (Prometheus + Grafana)

The application is a simple voting app where users vote between two options (e.g. **Cats vs Dogs**), and results are displayed in real-time on a separate results page.

---

## 🏗️ Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              Kind K8s Cluster                │
                        │                                              │
                        │   ┌──────────┐        ┌──────────────────┐  │
User ──── Port 31002 ──►│   │  Vote    │──────► │      Redis       │  │
                        │   │ (Python) │        │  (Message Queue) │  │
                        │   └──────────┘        └────────┬─────────┘  │
                        │                                │             │
                        │                       ┌────────▼─────────┐  │
                        │                       │     Worker       │  │
                        │                       │    (.NET/C#)     │  │
                        │                       └────────┬─────────┘  │
                        │                                │             │
                        │   ┌──────────┐        ┌────────▼─────────┐  │
User ──── Port 31001 ──►│   │  Result  │◄────── │   PostgreSQL     │  │
                        │   │ (Node.js)│        │     (DB)         │  │
                        │   └──────────┘        └──────────────────┘  │
                        │                                              │
                        └─────────────────────────────────────────────┘
```

![Architecture Diagram](architecture.excalidraw.png)

### Data Flow

1. **User votes** via the Vote app (Python/Flask) → vote is pushed to **Redis** as a JSON message
2. **Worker** (.NET/C#) continuously pops votes from **Redis** → upserts them into **PostgreSQL**
3. **Result app** (Node.js) polls **PostgreSQL** every second → pushes live scores to the browser via **Socket.IO**

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Vote App** | Python 3, Flask, Gunicorn |
| **Result App** | Node.js, Express, Socket.IO |
| **Worker** | .NET / C# |
| **Message Queue** | Redis (Alpine) |
| **Database** | PostgreSQL 15 (Alpine) |
| **Containerization** | Docker |
| **Orchestration** | Kubernetes (Kind v1.30.0) |
| **GitOps / CD** | Argo CD |
| **Monitoring** | Prometheus + Grafana (kube-prometheus-stack via Helm) |
| **Local Cluster** | Kind (Kubernetes IN Docker) |
| **Cloud Host** | AWS EC2 |

---

## 📁 Project Structure

```
k8s-kind-voting-app/
│
├── vote/                        # Python Flask voting frontend
│   ├── app.py                   # Flask app — handles votes → pushes to Redis
│   ├── Dockerfile
│   └── requirements.txt
│
├── result/                      # Node.js result frontend
│   ├── server.js                # Express + Socket.IO — polls PostgreSQL, streams results
│   ├── Dockerfile
│   └── views/                   # HTML templates
│
├── worker/                      # .NET/C# background worker
│   ├── Program.cs               # Pops from Redis → upserts into PostgreSQL
│   ├── Dockerfile
│   └── Worker.csproj
│
├── k8s-specifications/          # Kubernetes manifests
│   ├── vote-deployment.yaml
│   ├── vote-service.yaml
│   ├── result-deployment.yaml
│   ├── result-service.yaml
│   ├── worker-deployment.yaml
│   ├── redis-deployment.yaml
│   ├── redis-service.yaml
│   ├── db-deployment.yaml
│   └── db-service.yaml
│
├── kind-cluster/                # Kind cluster setup
│   ├── config.yml               # 3-node cluster (1 control-plane + 2 workers)
│   ├── commands.md              # Full command reference
│   ├── install_kind.sh
│   └── install_kubectl.sh
│
├── healthchecks/                # Health check scripts for Redis & PostgreSQL
├── seed-data/                   # Optional: seed votes into the app
├── docker-compose.yml           # Local development (Docker Compose)
├── architecture.excalidraw.png
└── README.md
```

---

## ⚙️ How It Works

### 🐍 Vote App (Python / Flask)
- Runs on port **5000** (NodePort **31002** in K8s)
- Accepts `GET` and `POST` requests
- Assigns a unique `voter_id` cookie to each visitor (prevents duplicate votes per browser)
- On vote submission, serialises `{voter_id, vote}` as JSON and pushes it to the **Redis** `votes` list via `RPUSH`

### ⚙️ Worker (.NET / C#)
- Runs as a **background service** with no exposed port
- Connects to both Redis and PostgreSQL on startup with **auto-retry logic** (handles startup race conditions)
- Continuously polls Redis every **100ms** using `ListLeftPopAsync`
- Performs an **upsert** on the `votes` table — if the voter has voted before, updates their choice
- Handles reconnection for both Redis and PostgreSQL if connections drop

### 🟢 Result App (Node.js / Express)
- Runs on port **4000** (NodePort **31001** in K8s)
- Uses **Socket.IO** for real-time browser updates (no page refresh needed)
- Queries PostgreSQL every **1 second**: `SELECT vote, COUNT(id) FROM votes GROUP BY vote`
- Broadcasts live scores to all connected browser clients via WebSockets

### 🔴 Redis
- Acts as the **message queue** between Vote and Worker
- Uses a simple list pattern: `RPUSH` (Vote) / `LPOP` (Worker)

### 🐘 PostgreSQL
- Stores canonical vote results in a `votes` table
- Schema is **auto-created** by the Worker on first connection:
  ```sql
  CREATE TABLE IF NOT EXISTS votes (
      id   VARCHAR(255) NOT NULL UNIQUE,
      vote VARCHAR(255) NOT NULL
  )
  ```

---

## 🚀 [Getting Started](https://github.com/SHIVA-SINGHx/k8s-gitops/blob/main/GETTING_STARTED.md)



## Monitoring with Prometheus and Grafana

| Prometheus | Grafana |
|-----------|---------|
| ![Prometheus](prometheus.png) | ![Grafana](grafana.png) |

---

## 📜 Kubernetes Manifests Breakdown

All manifests live in [`k8s-specifications/`](./k8s-specifications/).

| Manifest | Kind | Description |
|---|---|---|
| `vote-deployment.yaml` | Deployment | Flask voting frontend (1 replica) |
| `vote-service.yaml` | Service (NodePort 31002) | Exposes Vote on port 5000 |
| `result-deployment.yaml` | Deployment | Node.js result frontend (1 replica) |
| `result-service.yaml` | Service (NodePort 31001) | Exposes Result on port 5001 |
| `worker-deployment.yaml` | Deployment | .NET worker (no external port) |
| `redis-deployment.yaml` | Deployment | Redis message queue |
| `redis-service.yaml` | Service (ClusterIP) | Internal Redis access |
| `db-deployment.yaml` | Deployment | PostgreSQL database |
| `db-service.yaml` | Service (ClusterIP) | Internal DB access |

### Kind Cluster Config

```yaml
# kind-cluster/config.yml
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

## 🧹 Cleanup

```bash
# Delete the Kind cluster (removes all workloads and containers)
kind delete cluster --name=kind
```

## What I Learned

- Writing Kubernetes Deployments and Services for a multi-tier app, and choosing between NodePort and ClusterIP
- How GitOps replaces manual `kubectl apply` with Git-driven, auditable deployments
- Handling startup dependencies between services with connection retry logic
- Setting up cluster observability with Helm, Prometheus, and Grafana

## 📸 Screenshots

<!-- Voting App UI -->
<!-- ![Vote App](./screenshots/vote-app.png) --><img width="871" height="834" alt="Screenshot 2026-09-26 184844" src="https://github.com/user-attachments/assets/46063b8c-b4c0-43ba-b039-543dbf75fc20" />


<!-- Result App UI -->
<!-- ![Result App](./screenshots/result-app.png) --><img width="1470" height="743" alt="Screenshot 2026-09-26 184830" src="https://github.com/user-attachments/assets/53b94ad7-f6a0-4214-a6d8-03b753c7149a" />


<!-- Argo CD Dashboard -->
<!-- ![Argo CD](./screenshots/argocd.png) --><img width="1919" height="904" alt="Screenshot 2026-09-26 183114" src="https://github.com/user-attachments/assets/c2d80cc2-05ae-4b6b-9f9d-84fcec0e3dfe" />


<!-- Kubernetes Dashboard -->
<!-- ![K8s Dashboard](./screenshots/k8s-dashboard.png) --><img width="1919" height="908" alt="Screenshot 2026-09-26 190317" src="https://github.com/user-attachments/assets/8b7e7227-72a7-4b09-b4dd-aa484dc433b3" />


---

## 📄 License

This project is licensed under the terms of the [LICENSE](./LICENSE) file.

---

## Author

**Shiva Singh** · [GitHub](https://github.com/SHIVA-SINGHx)



