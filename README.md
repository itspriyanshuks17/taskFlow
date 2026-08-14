# TaskFlow

A self-hosted, Jira-style task and project tracker — built end-to-end with a production-style DevOps pipeline: containerized services, CI/CD, infrastructure as code across two clouds, Kubernetes, and observability.

The goal of this project wasn't just "build a task tracker." It was to build one the way a small engineering team actually would ship and operate one.

---

## What it does

- **Projects → Tasks → Subtasks**, with types (Epic / Story / Task / Bug), priorities, labels, due dates, and story points
- **Kanban board** with drag-and-drop status columns (Backlog → To Do → In Progress → In Review → Done)
- **Real-time updates** — board changes sync live across open sessions via WebSockets
- **Comments and a full activity/audit log** on every task (who changed what, when)
- **Auth** via email/password with JWT sessions

## Architecture

```
                        ┌──────────────────────┐
                        │   Ingress / ALB      │
                        │  (TLS termination)   │
                        └──────────┬───────────┘
                                   │
                 ┌─────────────────┴───────────────────┐
                 │                                     │
        ┌────────▼─────────┐                  ┌────────▼───────────┐
        │  Frontend        │                  │  Backend API       │
        │  React + Vite    │──── REST/WS ──   │  Express + Socket  │
        │  (nginx, static) │                  │  .io               │
        └──────────────────┘                  └───────────┬────────┘
                                                          │
                                               ┌──────────▼───────────┐
                                               │   PostgreSQL         │
                                               │   (RDS / Flexible    │
                                               │    Server)           │
                                               └──────────────────────┘

        Both services run as Kubernetes Deployments with HPA,
        readiness/liveness probes, and PodDisruptionBudgets.
        Metrics are scraped by Prometheus and visualized in Grafana.
```

**Primary region:** AWS (EKS + RDS Postgres + ECR)
**Secondary/DR region:** Azure (AKS + Postgres Flexible Server + ACR)

Both are provisioned from the same Terraform patterns so the infra is genuinely portable, not just theoretically multi-cloud.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind v4 | Fast dev loop, small bundle, type-safe |
| Backend | Node.js, Express, TypeScript, Prisma | Familiar, strongly-typed ORM with migrations |
| Database | PostgreSQL | Relational fits the project/task/comment graph |
| Real-time | Socket.io | Simple room-based broadcast for board updates |
| Containers | Docker (multi-stage builds) | Small, non-root, healthchecked images |
| CI/CD | GitHub Actions | Lint → test → scan → build → push → deploy |
| IaC | Terraform | AWS (primary) + Azure (DR), same module patterns |
| Orchestration | Kubernetes (EKS / AKS) | HPA, rolling deploys, PDBs |
| Observability | Prometheus + Grafana | Request rate, p95 latency, error rate, event-loop lag |

## Repository layout

```
taskflow/
├── backend/              Express API (TypeScript, Prisma, Socket.io)
├── frontend/             React app (Vite, TypeScript, Tailwind)
├── infra/
│   ├── terraform/aws/    EKS, RDS, ECR, VPC — primary region
│   ├── terraform/azure/  AKS, Postgres, ACR — DR region
│   ├── k8s/               Kubernetes manifests (Deployments, HPA, Ingress, PDB)
│   └── monitoring/        Prometheus scrape config + Grafana dashboards
├── .github/workflows/    CI/CD pipeline
└── docker-compose.yml    One-command local stack
```

## Running it locally

Requires Docker and Docker Compose.

```bash
git clone <your-repo-url> taskflow
cd taskflow
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
docker compose up --build
```

This starts Postgres, the API (with migrations + seed data applied automatically), the frontend dev server, plus Prometheus and Grafana.

| Service | URL |
|---|---|
| App | http://localhost:5173 |
| API | http://localhost:4000 |
| API metrics | http://localhost:4000/metrics |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (admin / admin) |

**Demo login:** `demo@taskflow.dev` / `password123`

### Running without Docker

```bash
# Backend
cd backend
npm install
npx prisma migrate dev
npx prisma db seed
npm run dev

# Frontend (separate terminal)
cd frontend
npm install
npm run dev
```

## Deploying to AWS

```bash
cd infra/terraform/aws

# One-time: create the S3 bucket + DynamoDB table used for remote state
aws s3api create-bucket --bucket taskflow-terraform-state --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws dynamodb create-table --table-name taskflow-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

terraform init
terraform plan -var="db_password=<strong-password>"
terraform apply -var="db_password=<strong-password>"

# Point kubectl at the new cluster
aws eks update-kubeconfig --name taskflow-cluster --region ap-south-1

# Create the app secrets (see infra/k8s/01-secrets.example.yaml)
kubectl create secret generic taskflow-secrets -n taskflow \
  --from-literal=DATABASE_URL="$(terraform output -raw rds_endpoint)" \
  --from-literal=JWT_SECRET="$(openssl rand -base64 48)"

kubectl apply -f ../../k8s/
```

From here, pushes to `main` (via the GitHub Actions workflow) build new images, push them to GHCR, and roll out the deployment automatically.

## Deploying to Azure (DR)

```bash
cd infra/terraform/azure
terraform init
terraform apply -var="db_admin_password=<strong-password>"
az aks get-credentials --resource-group taskflow-aks-rg --name taskflow-aks
kubectl apply -f ../../k8s/
```

## Observability

The API exposes Prometheus metrics at `/metrics`: request rate and p95 latency by route, error rate, and a business metric (`taskflow_tasks_created_total`). The bundled Grafana dashboard (`infra/monitoring/grafana/dashboards/taskflow-api.json`) visualizes all of it and is auto-provisioned when running via Docker Compose — open Grafana and it's already there, no manual setup.

## What I'd add next

- Integration tests hitting a real test database in CI (currently only unit tests run in the pipeline)
- OpenTelemetry tracing across the API → Postgres call path
- WIP limits per Kanban column and a burndown chart on the dashboard
- Automated Terraform plan-on-PR via `terraform plan` comments in GitHub Actions

## License

MIT — this is a personal portfolio project.
