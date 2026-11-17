# Mini shop

Educational/portfolio project: a small online shop built from Spring Boot microservices (`catalog-service`, `orders-service`), an Angular frontend and Keycloak (OIDC). It runs locally with docker-compose, on local Kubernetes (Docker Desktop) and on AWS EKS.

> Status: work in progress – see the [implementation plan](docs/implementation-plan.md).

## Stack

Java 25, Spring Boot 4.1, Spring Cloud OpenFeign, PostgreSQL 18 + Liquibase, Keycloak 26, Angular 22 + Angular Material, Docker, Helm, Terraform, GitHub Actions. Exact pinned versions: [project-spec.md §3](docs/project-spec.md#3-stack).

## Repository layout

| Path | Contents |
|---|---|
| `docs/` | Specifications (start with [project-spec.md](docs/project-spec.md)) |
| `services/` | Backend: Maven multi-module (`catalog-service`, `orders-service`) |
| `frontend/` | Angular application |
| `infra/local/` | Local environment: Postgres init, Keycloak realm |
| `infra/helm/` | Helm charts (local Kubernetes and EKS) |
| `infra/terraform/` | AWS infrastructure |
| `scripts/` | Helper scripts (smoke test, local Kubernetes) |

## Prerequisites

- JDK 25, Node.js 26 (LTS), Docker Desktop (with Docker Compose).
- Later phases: `jq` (smoke test), `kubectl` and `helm` (local Kubernetes), `terraform` and the AWS CLI (AWS).

## Running locally

_To be completed in steps 1.1 (infrastructure), 4.1 (backend) and 5.5 (full stack)._

```bash
cp .env.example .env
docker compose up --build
```
