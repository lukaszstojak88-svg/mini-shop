# Mini shop – project specification

Main document. Always loaded first. Details live in the specifications listed in the [Specification map](#specification-map) section.

## 1. Goal

Educational/portfolio project: a small online shop running locally (docker-compose) and in the AWS cloud (EKS). The goal is to gain hands-on experience with: Spring Boot microservices, OAuth2/OIDC (Keycloak), PostgreSQL + Liquibase, Angular, Docker, Terraform, Kubernetes/Helm and CI/CD.

## 2. Scope

**In scope**
- Product catalog: browsing, searching, pagination; CRUD for the administrator.
- Cart (browser-side) and placing orders.
- Orders: list of own orders, details, cancellation (status `NEW`), confirmation by the administrator.
- Login and registration via Keycloak (OIDC, JWT), roles `USER` and `ADMIN`.
- Local startup with a single command (`docker compose up`).
- Running on local Kubernetes (Docker Desktop) with Helm charts, before deploying to AWS.
- Deployment to AWS EKS with Terraform, Helm and GitHub Actions.
**Out of scope (deliberately)**
- Payments, shipping, e-mails, notifications.
- Product images (placeholders with icons in the UI).
- Distributed transactions, Saga, Kafka/queues (see [Known limitations](#8-known-limitations)).
- Internationalization (UI in Polish only).
- In-browser e2e tests (optionally later).
- Elastic Stack (ELK) – **optional module**, described in [observability-spec.md](./observability-spec.md); implemented only on explicit request (steps `O.*` in the plan).
- Prometheus + Grafana – **optional module**, described in [monitoring-spec.md](./monitoring-spec.md); implemented only on explicit request (steps `O.3`, `O.4`).

## 3. Stack

| Area | Choice |
|---|---|
| Backend | Java 25 LTS, Spring Boot (latest stable line), Maven + wrapper |
| Spring modules | Web, Validation, Data JPA, Security (resource server), OAuth2 Client (client credentials), Actuator, Cloud OpenFeign, Resilience4j (circuit breaker) |
| Mapping | MapStruct (no Lombok – DTOs as `record`) |
| Database | PostgreSQL (single instance, separate databases: `catalog`, `orders`, `keycloak`) |
| Migrations | Liquibase (YAML changelogs) |
| Identity | Keycloak (OIDC, Authorization Code + PKCE, JWT) |
| Frontend | Angular (latest stable), standalone components, signals, Angular Material, `angular-oauth2-oidc` |
| Containers | Docker, nginx (frontend), docker-compose |
| Local Kubernetes | Docker Desktop (Kubernetes), ingress-nginx, the same Helm charts as on AWS ([k8s-local-spec.md](./k8s-local-spec.md)) |
| Cloud | AWS: EKS, RDS PostgreSQL, ECR, ALB, Route 53, ACM, Secrets Manager, CloudWatch |
| IaC / deploy | Terraform, Helm, GitHub Actions (OIDC to AWS) |
| Metrics (optional module) | Micrometer → Prometheus → Grafana (locally a compose profile, in Kubernetes kube-prometheus-stack). **Do not implement initially** – see [monitoring-spec.md](./monitoring-spec.md) |
| Logs (optional module) | ECS JSON logs → Filebeat → Elasticsearch → Kibana (locally a compose profile, on EKS via ECK). **Do not implement initially** – see [observability-spec.md](./observability-spec.md) |
| Tests | JUnit 5, Mockito, AssertJ, Testcontainers, WireMock, Jasmine/Karma or Vitest (default runner of the chosen Angular version) |

**Versions** – pin in step 0.1 and record here (single source of truth):

| Component | Version |
|---|---|
| Java | 25 (LTS) |
| Spring Boot | 4.1.2 |
| Spring Cloud | 2025.1.4 "Oakwood" (Spring Cloud OpenFeign 5.0.4) |
| Keycloak | 26.8.2 (image `quay.io/keycloak/keycloak:26.8.2`) |
| PostgreSQL | 18.7 (image `postgres:18.7`) |
| Angular / Angular Material | 22.2.x (`@angular/core` 22.2.6, `@angular/material` 22.2.7) |
| angular-oauth2-oidc | 22.0.2 |
| Node.js | 26 (Active LTS) |
| Elastic Stack (optional module) | _fill in only when the module is enabled_ |
| Prometheus / Grafana / kube-prometheus-stack (optional module) | _fill in only when the module is enabled_ |

> If Spring Cloud OpenFeign does not yet support the latest Spring Boot line, use the latest supported line and note it in the table.
>
> **Compatibility (verified 2026-11-17):** Spring Cloud 2025.1.4 is based on Spring Boot 4.0.9 and officially supports Spring Boot 4.1.2 (the 2025.1.x line covers Boot 4.0.x and 4.1.x since 2025.1.2). Spring Boot 4.2 and Spring Cloud 2026.0 "Paddington" are still release candidates (GA planned for 2026-11-19) – do not use them; upgrade later as a separate change. PostgreSQL 19 GA is also scheduled for 2026-11-19, so the project stays on 18. Angular 22 requires Node `^22.22.3 || ^24.15.0 || >=26.0.0`.

## 4. Architecture

```
Browser
   │  https://shop.<domain>            https://auth.<domain>
   ▼                                         ▼
 Ingress/ALB  ──────────────────────────► Keycloak ──► PostgreSQL (db: keycloak)
   │  /                → frontend (nginx)
   │  /api/catalog/**  → catalog-service ──► PostgreSQL (db: catalog)
   │  /api/orders/**   → orders-service  ──► PostgreSQL (db: orders)
   │                         │
   │                         └── Feign (/internal/catalog/**, client_credentials token) ──► catalog-service
   ▼
 (/internal/** is NOT exposed through the Ingress)
```

- Backend services are **resource servers**: they validate JWTs issued by Keycloak (keys from JWKS).
- A single domain for the application (path-based routing) → no CORS. Locally: nginx in the frontend container reproduces this routing; with `ng serve` it is done by `proxy.conf.json`.
- Service-to-service communication: HTTP using service names (Docker/Kubernetes DNS), no service discovery.

## 5. Repository structure

```
.
├── CLAUDE.md
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitattributes            # LF for *.sh and mvnw (Windows: core.autocrlf)
├── docs/                     # specifications (this directory); docs/pl/ – Polish copies (local only, git-ignored)
├── services/
│   ├── pom.xml               # parent + modules
│   ├── mvnw, .mvn/
│   ├── catalog-service/      # Spring Boot
│   └── orders-service/       # Spring Boot
├── frontend/                 # Angular
├── infra/
│   ├── local/                # Postgres init SQL, Keycloak realm (dev), elk/ (Filebeat, Kibana objects)
│   ├── terraform/            # bootstrap/, envs/dev/, modules/
│   └── helm/                 # charts: catalog-service, orders-service, frontend, keycloak, db-init, postgres-local (local only)
├── scripts/                  # helper scripts (smoke, k8s-local-up/down)
└── .github/workflows/
```

Conventions: `groupId` `pl.minishop`, packages `pl.minishop.catalog` and `pl.minishop.orders`. Service/image names: `catalog-service`, `orders-service`, `frontend`.

## 6. Ports (local)

| Service | Host port | Container port |
|---|---|---|
| frontend (nginx, docker-compose) | 8080 | 80 |
| Angular dev server (`ng serve`) | 4200 | – |
| Keycloak | 8081 | 8080 |
| catalog-service | 8082 | 8080 |
| orders-service | 8083 | 8080 |
| PostgreSQL | 5432 | 5432 |
| Local Kubernetes Ingress (`shop.localtest.me`, `auth.localtest.me`) | 80 | – |
| Elasticsearch (optional module, `logging` profile) | 9200 | 9200 |
| Kibana (optional module, `logging` profile) | 5601 | 5601 |
| Prometheus (optional module, `monitoring` profile) | 9090 | 9090 |
| Grafana (optional module, `monitoring` profile) | 3000 | 3000 |

## 7. Rules

They apply in every step. Violating a rule = step not finished.

### 7.1 Clean code (all languages)
- Readability over cleverness. Names reveal intent; no abbreviations and no names like `data`, `manager`, `util`.
- Small units: a method does one thing (roughly ≤ 20 lines), a class has a single responsibility.
- No dead or commented-out code, no `TODO` without a reference to an item in [Known limitations](#8-known-limitations).
- **YAGNI**: do not add features, abstractions, libraries, configuration or files beyond the specification. Need something more – ask.
- DRY in moderation (rule of three); duplication is better than the wrong abstraction.
- Comments explain "why", not "what". Code, comments, names, commits and documentation in English (Polish copies of the specifications in `docs/pl/`); UI texts in Polish.
- No "magic" numbers and strings – constants or configuration.
- Explicit error handling; never swallow exceptions.

### 7.2 Java / Spring
- Constructor injection (`final` fields), no `@Autowired` on fields.
- Layers: `api` (controllers, DTOs) → `service` → `repository`. Controller without business logic. `@Transactional` on service methods, not on controllers.
- DTOs as `record`. JPA entities do not leave the service layer; mapping via MapStruct.
- No Lombok. `BigDecimal` for money (never `double`). `Optional` only as a return type. No `null` in the public service API.
- Errors: domain exceptions + `@RestControllerAdvice` returning Problem Details (RFC 9457). Never a stack trace in a response.
- Configuration via `@ConfigurationProperties` (typed, validated). No scattered `@Value`. No secrets in the repository.
- SLF4J logging with parameters (`log.info("Order {} created", id)`); do not log tokens, passwords or personal data. Core logging minimum: [observability-spec.md §0](./observability-spec.md#0-module-status-and-core-minimum).
- Formatting: Spotless (google-java-format); `spotless:check` in the build.
- Input validation at the boundary (Bean Validation in DTOs); principle of least privilege in Spring Security.

### 7.3 SQL / Liquibase
- A changeset is immutable once merged – fixes go into a new changeset.
- One changeset = one change; `id` with a number and description (`001-create-product-table`); `rollback` where possible.
- `spring.jpa.hibernate.ddl-auto=validate`. The schema is changed exclusively by Liquibase.
- Constraints (NOT NULL, CHECK, FK, UNIQUE) in the database, not only in code.

### 7.4 Angular / TypeScript
- `strict: true`, no `any`. ESLint + Prettier without warnings.
- Standalone components, `ChangeDetectionStrategy.OnPush`, state via signals.
- Logic in services; thin components. Typed reactive forms.
- Subscriptions managed via the `async` pipe or `takeUntilDestroyed`.
- Accessibility: field labels, focus, contrast, semantic HTML.

### 7.5 Tests
- Every behavior change has a test; a build without green tests = step not finished. Details: [test-spec.md](./test-spec.md).
- Tests are deterministic and independent. Prefer Testcontainers over mocking the database.

### 7.6 Git and process
- Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `refactor:`, `ci:`, `infra:`).
- One plan step = one coherent change (one or several commits, no big "mixed" ones).
- Do not commit or push without the user's request.
- Contract change (API, schema, configuration) = update of the relevant specification in the same step.
- After a step: mark it in [implementation-plan.md](./implementation-plan.md).

### 7.7 Security
- Zero secrets in the repo. Development secrets only in `.env.example` and the `dev` realm, explicitly marked as dev-only.
- `/internal/**` endpoints are never exposed through the Ingress.
- Dependencies without known critical vulnerabilities; do not add libraries without justification.

## Specification map

Load a document **only** when it concerns the current step.

| Document | Contents | Load when implementing |
|---|---|---|
| [backend-spec.md](./backend-spec.md) | Catalog and orders services: API, data model, Liquibase, Feign, resilience, errors | `services/**` code |
| [auth-spec.md](./auth-spec.md) | Keycloak, realm, roles, clients, Spring Security and Angular configuration, identity propagation | login, roles, endpoint security, realm |
| [frontend-spec.md](./frontend-spec.md) | Screens, routing, state, GUI, runtime configuration, nginx | `frontend/**` code |
| [test-spec.md](./test-spec.md) | Test strategy and conventions, tools, coverage thresholds, CI | any production code (section relevant to the layer) |
| [local-dev-spec.md](./local-dev-spec.md) | docker-compose, Postgres init, local Keycloak, environment variables, Dockerfiles | `docker-compose.yml`, `infra/local/**`, Dockerfiles |
| [k8s-local-spec.md](./k8s-local-spec.md) | Local Kubernetes (Docker Desktop): ingress-nginx, `postgres-local`, chart parameterization, `k8s-local-*` scripts | `infra/helm/**`, `values-local.yaml`, `scripts/k8s-local-*.sh` |
| [infra-spec.md](./infra-spec.md) | AWS, Terraform, Helm, EKS, RDS, secrets, CI/CD, observability, costs | `infra/terraform/**`, `infra/helm/**`, `.github/**` |
| [observability-spec.md](./observability-spec.md) | Core logging minimum (§0); **optional module**: ECS JSON, Filebeat/Elasticsearch/Kibana, `logging` profile, ECK on EKS | logging minimum – always when configuring logs; the rest **only on explicit request** (steps `O.*`) |
| [monitoring-spec.md](./monitoring-spec.md) | **Optional module:** Micrometer metrics, Prometheus, Grafana, kube-prometheus-stack, alerts | **only on explicit request** (steps `O.3`, `O.4`) |
| [implementation-plan.md](./implementation-plan.md) | Steps, prompts, acceptance criteria, status | always (indicates what to load in a given step) |

## 8. Known limitations

Deliberate simplifications – do not "fix" them without a request.

- **KO-1** No distributed transaction between `orders` and `catalog`. Stock reservation is atomic in the catalog; if saving the order fails, the reservation is released (compensation). A failure between steps may leave an inconsistency – target solution: outbox/Saga.
- **KO-2** Order cancellation: if the stock release succeeds and the local commit fails, the stock level will be overstated. Rare; see KO-1.
- **KO-3** No idempotency for reservation/release (no retry on these calls).
- **KO-4** The cart exists only in the browser (`localStorage`) and is not synchronized between devices.

## 9. Design decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | Single PostgreSQL instance, separate databases and users per component | Isolation without the cost of multiple RDS instances |
| D2 | Stock reservation in the catalog returns a snapshot (name, price) | One Feign call, consistent price in the order |
| D3 | Reservation/release: `client_credentials` token of the `orders-service` with the `SERVICE` role; endpoints under `/internal/**` | The end user cannot change stock directly |
| D4 | Path-based routing on a single domain | No CORS, simpler frontend |
| D5 | Own, minimal Helm chart for Keycloak (official image) | Fewer dependencies on external charts |
| D6 | Runtime frontend configuration (`/config.json`) | One image for all environments |
| D7 | Maven (not Gradle), no Lombok | Simplicity, ubiquity |
| D8 | **Optional module:** Elastic Stack without Logstash (Filebeat → Elasticsearch), logs in ECS JSON; ECK on EKS | Logs already structured, fewer components and less RAM; ECK is the standard way to run Elastic on Kubernetes |
| D9 | One set of Helm charts for local Kubernetes (Docker Desktop) and EKS; differences only in `values-*.yaml` | Charts tested locally for free, less work in the AWS phase |
| D10 | **Optional module:** Prometheus + Grafana (Micrometer, kube-prometheus-stack, no Alertmanager), independent of ELK | Metrics and alerts without affecting the core; no notifications limits scope and resources |

## 10. Default assumptions (to be confirmed by the project owner)

- Catalog reads (`GET /api/catalog/products/**`) are **public** (no login); orders require login. **Confirmed.**
- User self-registration in Keycloak is **enabled**. **Confirmed.**
- `groupId` and package root `pl.minishop`. **Confirmed.**
- Repository on GitHub, CI/CD = GitHub Actions.
- AWS region: `eu-central-1`.
- A Route 53 domain is available for the AWS deployment (without a domain: see [infra-spec.md](./infra-spec.md), section "Domain and TLS").
- In-browser e2e tests out of scope.

## 11. Glossary

| Term | Meaning |
|---|---|
| Reservation | Atomic decrease of the stock level in the catalog for the duration of the order |
| Snapshot | Copy of the product name and price stored in the order item |
| Resource server | Backend service validating JWTs |
| Realm | Space of users and clients in Keycloak (`minishop`) |
