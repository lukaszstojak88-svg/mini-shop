# AWS infrastructure – specification (Terraform, EKS, Helm, CI/CD)

Load when working on `infra/terraform/**`, `infra/helm/**`, `.github/workflows/**`. Context: [project-spec.md](./project-spec.md). Keycloak: [auth-spec.md](./auth-spec.md#6-keycloak-locally-and-in-the-cluster). Tests in CI: [test-spec.md](./test-spec.md#7-ci).

> **Verify module, EKS and chart versions in the documentation at implementation time** and record them in a comment/the `versions.tf` file and in the version table in project-spec.

## 1. Principles

- All infrastructure in Terraform; no manual clicking beyond the initial step (account, domain, budget).
- A single environment: `dev`. The structure allows adding more.
- Least privilege (IAM), no static AWS keys (OIDC for GitHub Actions, Pod Identity/IRSA for pods).
- Secrets exclusively in AWS Secrets Manager; they reach the cluster via the External Secrets Operator.
- Costs: see [§9](#9-costs-and-cleanup). Every work session ends with a conscious decision: keep it or `terraform destroy`.

## 2. Terraform structure

```
infra/terraform/
├── bootstrap/            # S3 bucket for state (with versioning and encryption), state locking
├── modules/              # own modules (e.g. ecr, secrets, github-oidc)
└── envs/dev/
    ├── main.tf, variables.tf, outputs.tf, versions.tf, backend.tf
    └── terraform.tfvars.example
```
Remote state in S3 with locking. For VPC and EKS, proven community modules (`terraform-aws-modules`) are allowed – verify the current major version. `terraform fmt`, `validate` and (optionally) `tflint` in CI.

## 3. Resources

| Area | Resource | Parameters (dev) |
|---|---|---|
| Network | VPC | 2–3 AZs, public subnets (ALB) and private subnets (nodes, RDS), **one** NAT Gateway (savings) |
| Cluster | EKS | currently supported Kubernetes version; managed node group: 2 × `t3.medium` (amd64), autoscaling 2–3; when the optional ELK is enabled, switch to `t3.large` ([observability-spec.md §6](./observability-spec.md#6-aws-eks)); access via EKS access entries |
| EKS add-ons | vpc-cni, coredns, kube-proxy, EKS Pod Identity Agent | managed add-ons; `aws-ebs-csi-driver` + StorageClass `gp3` added only with the optional ELK |
| Controllers (Helm) | AWS Load Balancer Controller, External Secrets Operator, metrics-server | each with an IAM role (Pod Identity/IRSA) with minimal permissions |
| Registry | ECR: `catalog-service`, `orders-service`, `frontend` | image scanning on push, lifecycle policy (e.g. last 20 images), tag = commit SHA |
| Database | RDS PostgreSQL | `db.t4g.micro`, single-AZ, 20 GB gp3, encryption, private subnets, SG only from the nodes' SG, no public access, automated backups (short retention) |
| Secrets | Secrets Manager | `minishop/dev/rds-master`, `minishop/dev/{catalog,orders,keycloak}-db`, `minishop/dev/orders-client-secret`, `minishop/dev/keycloak-admin` |
| DNS/TLS | Route 53 (existing zone), ACM | see [§8](#8-domain-and-tls) |
| CI identity | IAM OIDC provider for GitHub + a role for the pipeline | restricted to the repository and the `main` branch |
| Logs | `kubectl logs` (core); **optionally** Elastic Stack via ECK | optional module, plan step `O.2`; see [observability-spec.md §6](./observability-spec.md#6-aws-eks) |
| Metrics | metrics-server (HPA); **optionally** kube-prometheus-stack | metrics-server is enough for the core; optional module, step `O.4` – see [monitoring-spec.md §6](./monitoring-spec.md#6-kubernetes-docker-desktop-and-eks) |

## 4. Database in RDS

- One instance, three databases (`catalog`, `orders`, `keycloak`) and three application roles (per D1).
- Creating databases and roles: a **Kubernetes Job** (`postgres` image, `psql`) run as a Helm `pre-install`/`pre-upgrade` hook, idempotent (`CREATE ... IF NOT EXISTS` / `DO` blocks), with passwords from External Secrets. Terraform does not connect to the database.
- Table schemas are managed exclusively by Liquibase (application startup).

## 5. Helm

```
infra/helm/
├── catalog-service/    ├── orders-service/    ├── frontend/    ├── keycloak/    └── db-init/ (or a hook in one of the charts)
```
Each chart: `Deployment` (2 replicas for catalog/orders/frontend; 1 for Keycloak), `Service` (ClusterIP), `ServiceAccount`, `ExternalSecret`, `HorizontalPodAutoscaler` (CPU, after adding metrics-server), `PodDisruptionBudget`, `Ingress` (where exposed).

**The charts are shared with local Kubernetes** (Docker Desktop) and are created in step 6.2; on AWS only `values-dev.yaml` is added (ALB Ingress, images from ECR, ExternalSecret, HPA). Parameterization rules: [k8s-local-spec.md §5](./k8s-local-spec.md#5-charts-and-parameterization).

**Pod requirements:**
- `resources.requests/limits` set; `securityContext`: `runAsNonRoot`, `readOnlyRootFilesystem` (with `emptyDir` on `/tmp`), `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`.
- Probes: `livenessProbe` → `/actuator/health/liveness`, `readinessProbe` → `/actuator/health/readiness`, `startupProbe` for the JVM and Keycloak.
- Configuration via `values.yaml` + `values-dev.yaml`; image = ECR repository + tag passed at deploy time.

**Ingress (ALB, one shared group `alb.ingress.kubernetes.io/group.name`):**

| Host | Path | Backend |
|---|---|---|
| `shop.<domain>` | `/api/catalog` | `catalog-service` |
| `shop.<domain>` | `/api/orders` | `orders-service` |
| `shop.<domain>` | `/` | `frontend` |
| `auth.<domain>` | `/` | `keycloak` |

`/internal/**` has no rule in any Ingress. HTTP → HTTPS redirect on the ALB, certificate from ACM.

**Keycloak (own chart, official image):** command `start --import-realm`, variables: `KC_HOSTNAME=https://auth.<domain>`, `KC_HTTP_ENABLED=true`, `KC_PROXY_HEADERS=xforwarded`, RDS database, secrets from External Secrets; production realm from a ConfigMap (redirect URI `https://shop.<domain>/*`, no dev users and no `minishop-smoke` client, `orders-service` secret from outside). Administrative access to Keycloak: do not expose the admin console publicly if possible (restrict the `/admin` and `/realms/master` paths on the ALB or access via `kubectl port-forward`).

## 6. Deployment flow (manual, before CI/CD)

1. `terraform apply` (bootstrap, then `envs/dev`).
2. Build and push images to ECR.
3. `helm upgrade --install` in order: db-init → keycloak → catalog → orders → frontend.
4. Verification: `kubectl get pods`, login at `https://shop.<domain>`, smoke test.

## 7. CI/CD

GitHub repository, GitHub Actions:

- **`ci.yml`** (pull request and push): backend `./mvnw verify`; frontend `npm ci && npm run lint && npm test && npm run build`; image builds (no push); `terraform fmt -check` + `validate`; `helm lint`. Maven/npm dependency cache.
- **`deploy.yml`** (push to `main`, manual `workflow_dispatch`): login to AWS via OIDC (role from §3), build and push images tagged with the SHA, `aws eks update-kubeconfig`, `helm upgrade --install --atomic` for changed components, post-deployment smoke test.
- Zero AWS keys in repository secrets. `permissions: id-token: write` required.
- Optionally (stage 7.8): Argo CD instead of the `helm upgrade` step in the pipeline.

## 8. Domain and TLS

Assumption: a Route 53 zone exists for the user's domain. The `shop.<domain>` and `auth.<domain>` records (alias to the ALB) and the ACM certificate (DNS validation) are created in Terraform; optionally ExternalDNS.

**Without a domain:** Keycloak in production mode requires HTTPS for addresses other than localhost, so the full login flow on AWS will not work without a certificate. To be settled with the project owner before step 7.5 (see the questions in [implementation-plan.md](./implementation-plan.md#open-questions)).

## 9. Costs and cleanup

- Billed even when idle: EKS control plane, EC2 nodes, NAT Gateway, ALB, RDS, public IPv4. Check approximate prices in the AWS calculator before `apply`.
- **AWS Budgets** with an alert (e.g. at 50% and 90% of the monthly limit) created in step 7.1.
- After a work session: `helm uninstall` (removes the ALBs created by the controller **before** the cluster is destroyed), then `terraform destroy`. The `scripts/teardown.sh` script does this in the correct order.
- The optional Elastic Stack raises the cost: larger nodes (`t3.large`) and an EBS volume for Elasticsearch. Delete the PVCs before `destroy`, otherwise the volumes will be orphaned. Similarly the optional Prometheus + Grafana (approx. 2 GB RAM and an EBS volume); with both modules enabled, `t3.large` nodes.
- Check after destroy: no orphaned ALBs, EBS volumes, network interfaces, CloudWatch logs with long retention.

## 10. Observability and scaling

- Core: applications log to stdout with `requestId` (`kubectl logs`), resource metrics from metrics-server (`kubectl top`).
- Optional module (only on request): Elastic Stack (ECK) per [observability-spec.md](./observability-spec.md). Application and infrastructure metrics: the optional Prometheus + Grafana module ([monitoring-spec.md](./monitoring-spec.md)), independent of ELK.
- HPA: CPU target ~70%, min 2 / max 4 replicas for `catalog-service` and `orders-service`.
