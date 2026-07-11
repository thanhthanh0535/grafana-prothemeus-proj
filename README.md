# Grafana + Prometheus Monitoring Lab
 
A hands-on monitoring project that deploys **Grafana** and **Prometheus** three different ways — Docker Compose, Kubernetes, and Terraform (Grafana Cloud + local) — with dashboards managed as code and a GitHub Actions CI/CD pipeline.
 
## Architecture
 
```
┌─────────────────────────────────────────────────────────┐
│                     Ubuntu VM (10.0.0.10)               │
│                                                         │
│  ┌──────────────┐   scrapes   ┌────────────────────┐    │
│  │  Prometheus  │────────────▶│  node_exporter     │    │
│  │  :9090       │             │  :9100             │    │
│  └──────┬───────┘             │  (auth / payment / │    │
│         │                     │   user services)   │    │
│         ▼                     └────────────────────┘    │
│  ┌──────────────┐                                       │
│  │   Grafana    │  ◀── dashboards provisioned           │
│  │   :3000      │      via Terraform                    │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │ terraform apply
              ┌───────────┴───────────┐
              │  GitHub Actions CI/CD │──▶ Grafana Cloud
              └───────────────────────┘
```
 
## Project Structure
 
```
.
├── docker-compose.yml          # Grafana + Prometheus via Docker Compose
├── prometheus.yml              # Prometheus scrape config (node_exporter targets)
│
├── grafana.yaml                # All-in-one K8s manifest (PVC + Deployment + Service)
├── deploy-grafana.yaml         # Grafana Deployment (namespace: my-grafana)
├── svc-grafana.yaml            # LoadBalancer Service (port 3000)
├── pod-grafana.yaml            # Pod spec reference
├── pv-my-grafana.yaml          # PersistentVolume (hostPath, 1Gi)
├── pvc-my-grafana.yaml         # PersistentVolumeClaim (1Gi)
├── custom-resources.yaml       # Calico CNI installation config (kubeadm cluster)
│
└── terraform-proj/
    ├── main.tf                 # Grafana providers (cloud, stack, local)
    ├── variables.tf            # Grafana Cloud URL / token / slug
    ├── resources.tf            # Prometheus data source + dashboards
    ├── dashboards/
    │   └── microservices-dashboard.json   # "Microservices Monitoring" dashboard
    └── .github/workflows/
        └── grafana.yml         # Terraform plan/apply pipeline
```
 
## 1. Docker Compose Deployment
 
The fastest way to get the stack running.
 
**Prerequisites:** Docker and Docker Compose installed.
 
```bash
docker compose up -d
```
 
| Service    | URL                     | Notes                          |
|------------|-------------------------|--------------------------------|
| Grafana    | http://localhost:3000   | Admin credentials set via env vars |
| Prometheus | http://localhost:9090   | Config mounted from `prometheus.yml` |
 
Prometheus scrapes `node_exporter` on the VM (`10.0.0.10:9100`), with targets labeled as `auth-service`, `payment-service`, and `user-service` to simulate a microservices environment.
 
> **Note:** Set the Grafana admin credentials through environment variables or an `.env` file — do not commit real passwords to the repo.
 
## 2. Kubernetes Deployment
 
Deploys Grafana onto a kubeadm cluster with Calico CNI and persistent storage.
 
**Prerequisites:** A running Kubernetes cluster (kubeadm or Minikube), `kubectl` configured.
 
```bash
# (kubeadm only) Install Calico networking
kubectl apply -f custom-resources.yaml
 
# Create the namespace
kubectl create namespace my-grafana
 
# Provision storage
kubectl apply -f pv-my-grafana.yaml
kubectl apply -f pvc-my-grafana.yaml
 
# Deploy Grafana + expose it
kubectl apply -f deploy-grafana.yaml
kubectl apply -f svc-grafana.yaml
 
# Verify
kubectl get pods -n my-grafana
kubectl get svc -n my-grafana
```
 
Access Grafana via the NodePort assigned to the LoadBalancer service:
 
```bash
kubectl get svc grafana -n my-grafana
# http://<node-ip>:<nodePort>
```
 
**Key details:**
- Persistent storage via `hostPath` PV (`/mnt/data/grafana`, 1Gi, storage class `my-grafana-plugin`)
- Liveness (TCP :3000) and readiness (`/robots.txt`) probes configured
- `fsGroup: 472` security context so Grafana can write to the mounted volume
### Troubleshooting
 
**Pod stuck in `Pending` on a single-node cluster** — the control-plane taint blocks scheduling. Remove it:
 
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
 
**PVC stuck in `Pending`** — confirm the PV's `storageClassName` matches the PVC and the PV exists before the PVC binds.
 
## 3. Terraform — Dashboards as Code
 
The `terraform-proj/` directory manages Grafana resources declaratively across **two environments**:
 
- **Grafana Cloud** — creates a service account + token, provisions a Prometheus data source, and deploys the dashboard to the cloud stack
- **Local Grafana** — deploys the same dashboard to the self-hosted instance at `http://10.0.0.10:3000`
```bash
cd terraform-proj
 
# Set your Grafana Cloud credentials
export TF_VAR_grafana_cloud_url="https://<your-stack>.grafana.net/"
export TF_VAR_grafana_cloud_slug="<your-stack-slug>"
export TF_VAR_grafana_cloud_token="<access-policy-token>"
 
terraform init
terraform plan
terraform apply
```
 
The **Microservices Monitoring** dashboard includes panels for CPU utilisation per service, memory consumption, latency trends, response times, and a service health overview.
 
## 4. CI/CD Pipeline
 
`.github/workflows/grafana.yml` runs on every push and pull request:
 
1. **Terraform Init** and **Plan** on all pushes and PRs
2. **Plan output posted as a PR comment** for review before merge
3. **Terraform Apply** runs automatically — only on push to `main`
Required repository secrets:
 
| Secret | Purpose |
|--------|---------|
| `GRAFANA_CLOUD_URL` | Grafana Cloud instance URL |
| `GRAFANA_CLOUD_ACCESS_POLICY_TOKEN` | Access policy token for authentication |
 
## Tech Stack
 
- **Grafana** — visualization and dashboards
- **Prometheus** — metrics collection and storage
- **node_exporter** — host-level metrics
- **Kubernetes (kubeadm)** + **Calico CNI** — container orchestration
- **Docker Compose** — local container deployment
- **Terraform** (grafana/grafana provider) — infrastructure as code
- **GitHub Actions** — CI/CD automation
## What This Project Demonstrates
 
- Deploying the same monitoring stack across Docker, Kubernetes, and cloud
- Kubernetes persistent storage (PV/PVC), probes, and security contexts
- Managing Grafana dashboards and data sources as code with Terraform
- Multi-provider Terraform configuration (cloud + local aliases)
- GitOps-style CI/CD with plan-on-PR and apply-on-merge

