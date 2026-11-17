# Metrics and alerts – specification (Prometheus + Grafana) – optional module

Load **only when the user has explicitly asked for monitoring** (steps `O.3` and `O.4` in [implementation-plan.md](./implementation-plan.md#optional-modules-only-on-explicit-request)). Context: [project-spec.md](./project-spec.md). Logs (a separate optional module): [observability-spec.md](./observability-spec.md). Service configuration: [backend-spec.md](./backend-spec.md).

## 0. Module status

**Optional module – do not implement initially.** It is independent of the ELK module (you can enable one, the other, or both). The project core exposes only Actuator's `health` and `info` plus resource metrics from metrics-server (HPA, `kubectl top`). The service code changes required by this module are configuration-only and fit within step `O.3`.

## 1. Goal and stack

Application and infrastructure metrics, dashboards and simple alerts: traffic, errors, latency, JVM, connection pool, circuit breaker, business metrics, and in Kubernetes also pod resources.

| Component | Role |
|---|---|
| Micrometer + Prometheus registry (in the Spring Boot services) | Export of metrics at `/actuator/prometheus` |
| Prometheus | Scraping and storing metrics |
| Grafana | Dashboards |
| kube-prometheus-stack (Kubernetes) | Prometheus operator, Grafana, kube-state-metrics, node-exporter, rules |
| Alertmanager | **Disabled** (out of scope; alert rules are defined and visible in Prometheus/Grafana) |

## 2. Metrics in the applications (`catalog`, `orders`)

- Dependencies: `micrometer-registry-prometheus` (with Actuator). The `prometheus` endpoint is exposed **only in the Spring `monitoring` profile** (profile enabled by an environment variable in compose/Helm).
- Common tags `application` (`catalog-service` / `orders-service`) and `environment`.
- Enabled: HTTP server metrics (`http.server.requests` with a percentile histogram), JVM (memory, GC, threads), HikariCP, Tomcat, process; Resilience4j (state and calls of the `catalog` circuit breaker); HTTP client/Feign metrics (verify which are available in the pinned versions).
- Access to `/actuator/prometheus`: `permitAll` only in the `monitoring` profile ([auth-spec.md](./auth-spec.md#4-resource-server-catalog-and-orders)); the endpoint **is not routed** through the Ingress or the frontend nginx – reachable only inside the network (compose/cluster).
- Forbidden: high-cardinality tags (user, product, order identifiers, paths with parameters – use URI templates).

## 3. Business metrics (minimum)

| Metric | Type | Service | Tags |
|---|---|---|---|
| `minishop.orders.created` | counter | orders | – |
| `minishop.orders.cancelled` | counter | orders | – |
| `minishop.stock.reservation.failures` | counter | catalog | `reason` = `insufficient_stock` / `product_not_found` |

Metrics are registered in the `service` layer via an injected `MeterRegistry`; tested per §8. Do not add more without agreement (YAGNI).

## 4. Prometheus

- Scrape every 15 s, `scrape_timeout` 10 s; targets: `catalog` and `orders` at `/actuator/prometheus` (compose: static targets by service name; Kubernetes: `ServiceMonitor`).
- Retention: 3 days (local) / 7 days (EKS); limited volume size (10 GB).
- Local configuration: `infra/local/monitoring/prometheus.yml`.

## 5. Grafana

- Provisioning from files (Prometheus datasource, dashboards) – no manual configuration; files in `infra/local/monitoring/grafana/` (provisioning + `dashboards/minishop.json`), the same JSON in Kubernetes (ConfigMap with a label for the dashboards sidecar).
- **Dashboard "Mini shop – services"** (filter `application`):
  - requests per second by status,
  - 5xx error rate,
  - p50/p95/p99 latency (`http.server.requests`),
  - JVM: heap, GC, threads,
  - HikariCP: active/pending connections,
  - circuit breaker: state and number of calls (success/failure/rejected),
  - business metrics from §3,
  - (Kubernetes) pod CPU/memory/restarts from kube-state-metrics.
- Access: locally `http://localhost:3000` (dev-only admin password from `.env`); in Kubernetes `kubectl port-forward` (no Ingress).

## 6. Kubernetes (Docker Desktop and EKS)

- Install `kube-prometheus-stack` (Helm, pinned version; check the current chart) in the `monitoring` namespace, with Alertmanager disabled; values: `values-local.yaml` (Docker Desktop) and `values-dev.yaml` (EKS).
- The `catalog` and `orders` charts contain a `ServiceMonitor` template, enabled with `monitoring.serviceMonitor.enabled=true` (default `false`, because it requires the operator's CRDs; see [k8s-local-spec.md §5](./k8s-local-spec.md#5-charts-and-parameterization)). The services get `SPRING_PROFILES_ACTIVE=...,monitoring`.
- Order: first kube-prometheus-stack, then upgrade the application charts with `ServiceMonitor` enabled.
- Storage: Docker Desktop – default StorageClass; EKS – gp3 via the EBS CSI Driver (add-on shared with the ELK module, see [infra-spec.md](./infra-spec.md#3-resources)).
- Approximate resources (dev): Prometheus approx. 1 Gi of memory, Grafana approx. 256 Mi, operator and kube-state-metrics together approx. 300 Mi, node-exporter small per node.

## 7. Alerts (Prometheus rules)

Defined as `PrometheusRule` (Kubernetes) or a rules file (compose); no notifications (no Alertmanager), visible in the Prometheus/Grafana UI:

| Alert | Condition (sketch) | Duration |
|---|---|---|
| `HighErrorRate` | 5xx response rate > 5% for a service | 5 min |
| `HighLatencyP95` | p95 `http.server.requests` > 1 s | 10 min |
| `CircuitBreakerOpen` | `open` state for `catalog` | 1 min |
| `ServiceDown` | target not scraped (`up == 0`) | 2 min |
| `PodRestartsHigh` (K8s) | > 3 restarts in 15 min | – |

Starting thresholds – to be adjusted after observing real traffic.

## 8. Tests and acceptance

**Tests in code** (after enabling the module):
- With the `monitoring` profile: `/actuator/prometheus` returns 200 and contains `http_server_requests_seconds`; without the profile the endpoint is unavailable (404/401).
- Business metrics: creating an order increments `minishop.orders.created`; a failed reservation increments the counter with the appropriate `reason`.
- No high-cardinality tags (review).

**Acceptance:**
- Compose (`docker compose --profile monitoring up`): both targets `UP` in Prometheus; the Grafana dashboard shows traffic after generating orders.
- Kubernetes: the `ServiceMonitor` finds the targets, the dashboard is accessible via `port-forward`; stopping `catalog` triggers `ServiceDown` and opens `CircuitBreakerOpen` after a series of requests to `orders`.
- `/actuator/prometheus` not reachable through the Ingress nor via `http://localhost:8080` / `http://shop.localtest.me`.

## 9. Resources and cost

- Local (compose): Prometheus + Grafana take approx. 0.5–1 GB of additional RAM.
- Docker Desktop Kubernetes: kube-prometheus-stack adds approx. 1.5–2 GB of RAM – increase Docker's memory limit.
- EKS: approx. 2 GB of additional RAM and an EBS volume; with ELK enabled as well, `t3.large` nodes. Estimates to be verified in the AWS calculator ([infra-spec.md §9](./infra-spec.md#9-costs-and-cleanup)).

## 10. Security

- Prometheus, Grafana and `/actuator/prometheus` are not exposed publicly; access via `port-forward` or the local network.
- Grafana admin password: dev-only in `.env`/`values-local.yaml`; on EKS in a secret (not in the repo).
- Metrics contain no personal data or tokens.
