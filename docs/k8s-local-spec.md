# Local Kubernetes (Docker Desktop) – specification

Load for: plan step 6.2, Helm charts, `values-local.yaml`, the `scripts/k8s-local-*.sh` scripts, ingress-nginx. Context: [project-spec.md](./project-spec.md). Charts on AWS: [infra-spec.md](./infra-spec.md#5-helm). Keycloak: [auth-spec.md](./auth-spec.md).

## 1. Goal

Run the whole system on local Kubernetes in Docker Desktop with **the same Helm charts** that will later go to EKS. This way the charts, probes, Ingress and configuration are verified for free, and the AWS phase only adds Terraform, ALB, RDS, ECR and AWS secrets.

## 2. Requirements

- Docker Desktop with Kubernetes enabled (Settings → Kubernetes → Enable). `kubectl` context: `docker-desktop`.
- Docker resources: min. 8 GB RAM and 4 CPUs (Keycloak and the two Spring applications are the heaviest).
- Installed: `kubectl`, `helm`.
- **Cluster mode:** the default single-node cluster (kubeadm) shares images with the local Docker – built images are immediately visible to Kubernetes (`imagePullPolicy: IfNotPresent`, no registry). If the `kind` mode (multi-node) is selected, images have to be loaded into the nodes – verify the behavior on the Docker Desktop version in use and describe it in the README.

## 3. Differences compared to AWS

| Area | Local (Docker Desktop) | AWS (EKS) |
|---|---|---|
| Ingress | ingress-nginx (`ingressClassName: nginx`) | AWS Load Balancer Controller (ALB) |
| Images | local, tag `local`, no registry | ECR, tag = commit SHA |
| Database | PostgreSQL in the cluster (`postgres-local` chart, StatefulSet + PVC) | RDS |
| Secrets | plain `Secret` from dev values (`values-local.yaml`) | External Secrets ← Secrets Manager |
| Autoscaling | disabled (`autoscaling.enabled=false`) | HPA |
| Addresses | `http://shop.localtest.me`, `http://auth.localtest.me` | `https://shop.<domain>`, `https://auth.<domain>` |
| TLS | none (HTTP) | ACM |
| Namespace | `minishop` | `minishop` |

## 4. Addresses and DNS

| Host | Purpose |
|---|---|
| `shop.localtest.me` | application: `/` frontend, `/api/catalog`, `/api/orders` |
| `auth.localtest.me` | Keycloak |

`*.localtest.me` resolves publicly to `127.0.0.1` (requires internet for DNS – verify availability; fallback: entries in the `hosts` file). ingress-nginx is exposed by Docker Desktop on `localhost:80`. The `iss` value in tokens: `http://auth.localtest.me/realms/minishop`; services fetch JWKS via the internal address (`http://keycloak.minishop.svc:8080/...`) – the same "two addresses" principle as in [auth-spec.md §4](./auth-spec.md#4-resource-server-catalog-and-orders). The development realm contains an additional redirect URI `http://shop.localtest.me/*`.

## 5. Charts and parameterization

One set of charts in `infra/helm/`, differences only via values:

| Chart | Notes |
|---|---|
| `postgres-local` | **local only**; StatefulSet, PVC (Docker Desktop default StorageClass), init script from `infra/local/postgres/init` (the same as in compose) |
| `db-init` | Job (hook) creating databases and roles, idempotent; database host from values (`postgres-local` locally, RDS on AWS) |
| `keycloak` | official image; realm JSON passed via `--set-file realmJson=<path>` (a chart cannot read files outside its directory): dev realm locally, production realm on AWS |
| `catalog`, `orders`, `frontend` | Deployment, Service, probes, securityContext, resources, Ingress (frontend and `/api/*`), optionally HPA/PDB |

Common switches (names to be stabilized in step 6.2 and recorded in the chart README):

| Value | Local | AWS |
|---|---|---|
| `image.repository` / `image.tag` / `image.pullPolicy` | `minishop/<name>` / `local` / `IfNotPresent` | ECR / SHA / `IfNotPresent` |
| `ingress.className` + `ingress.annotations` | `nginx` / – | `alb` / ALB, ACM, group annotations |
| `secrets.create` | `true` (dev values) | `false` |
| `externalSecrets.enabled` | `false` | `true` |
| `autoscaling.enabled` | `false` | `true` |
| `monitoring.serviceMonitor.enabled` | `false` (optional module) | `false` (optional module; `true` after step `O.4`) |
| `database.host` / `database.port` | `postgres-local` / 5432 | RDS address |
| `keycloak.publicUrl` / `keycloak.internalUrl` | `http://auth.localtest.me` / `http://keycloak:8080` | `https://auth.<domain>` / in-cluster service |

Values files in each chart: `values.yaml` (neutral defaults), `values-local.yaml`, and in the AWS phase `values-dev.yaml`.

Rules: templates contain no environment-specific constants; dev secrets never in `values.yaml`, only in `values-local.yaml` with a "dev-only" annotation; `/internal/**` without a rule in the Ingress.

## 6. Installation and scripts

`scripts/k8s-local-up.sh` (idempotent):
1. **Safeguard:** aborts if `kubectl config current-context` ≠ `docker-desktop` (protection against deploying to EKS/another cluster).
2. Builds the `catalog-service`, `orders-service`, `frontend` images with the `local` tag.
3. Installs ingress-nginx (official chart; pinned version) with `--wait`, if not installed.
4. Creates the `minishop` namespace.
5. Installs in order with `--wait`: `postgres-local` → `db-init` → `keycloak` (with `--set-file realmJson`) → `catalog` → `orders` → `frontend`.
6. Prints the addresses (`http://shop.localtest.me`) and test credentials (`admin/admin`, `user/user`).

`scripts/k8s-local-down.sh`: `helm uninstall` of the application components, deletion of the namespace; flags `--purge` (also removes the database PVC) and `--ingress` (removes ingress-nginx). The same context check as above.

Note on ordering: the secrets needed by the `db-init` hook must exist before it runs (a hook with a lower weight or a separate release).

## 7. Application configuration in the cluster

- `catalog`/`orders`: environment variables as in [backend-spec.md §6](./backend-spec.md#6-configuration-environment-variables), with Kubernetes service addresses (`http://catalog-service:8080`, `http://keycloak:8080`) and `MINISHOP_SECURITY_ISSUER=http://auth.localtest.me/realms/minishop`.
- `frontend`: `config.json` with `issuer` = `http://auth.localtest.me/realms/minishop`; in Kubernetes nginx serves only static files – `/api/*` routing is done by the Ingress (see [frontend-spec.md §6](./frontend-spec.md#6-build-and-container)).
- Keycloak: `KC_HOSTNAME=http://auth.localtest.me`, development mode (`start-dev --import-realm`), database in `postgres-local`.

## 8. Acceptance criteria (step 6.2)

- `./scripts/k8s-local-up.sh` on a clean cluster succeeds; all pods `Ready`.
- `http://shop.localtest.me`: login as `user` and `admin`, shopping, ordering, cancellation, admin panel – as in docker-compose.
- `scripts/smoke.sh` also passes with `BASE_URL=http://shop.localtest.me` and `AUTH_URL=http://auth.localtest.me` ([test-spec.md §6](./test-spec.md#6-environment-smoke-test-after-docker-compose-up)); `/internal/**` returns 404.
- `helm lint` clean for all charts (added to `ci.yml`).
- Re-running the `up` script breaks nothing; the `down --purge` script leaves the cluster clean.
- The scripts refuse to work in a context other than `docker-desktop`.

## 9. Common pitfalls (for the README)

- Port 80 taken by another process → ingress-nginx won't get an address.
- Too little memory for Docker → pods in `OOMKilled`/`Pending`.
- `*.localtest.me` unavailable offline → entry in the `hosts` file.
- The Keycloak realm import runs only on the first start with an empty database – after changing the realm: `k8s-local-down.sh --purge`.
- Docker Compose and local Kubernetes can run in parallel (different ports), but together they use a lot of memory.

## 10. Optional modules on local Kubernetes

Not part of step 6.2. On explicit request: Prometheus + Grafana via kube-prometheus-stack in the `monitoring` namespace ([monitoring-spec.md §6](./monitoring-spec.md#6-kubernetes-docker-desktop-and-eks), step `O.4`) – requires approx. 2 GB of additional RAM for Docker.
