# Implementation plan

The order of steps is intentional (each step builds on the previous ones). Project context: [project-spec.md](./project-spec.md).

## How to use

1. Copy the **prompt** of the chosen step and paste it into Claude Code (in the repository root).
2. Claude loads `CLAUDE.md`, `docs/project-spec.md` and the documents listed in the **Load** field, presents a short plan, implements, runs build/tests, and marks the step in this list.
3. Review the changes; you make the commit (or ask for it).

**Prompt template (for your own changes):**
> Implement: *\<description\>*. First read `CLAUDE.md`, `docs/project-spec.md` and `docs/<spec>.md`. Present a plan (max. 10 points), then implement within the scope of the description, run the tests and the build, update the specification if a contract changed. Do not add anything beyond the specification; if in doubt, ask.

## Definition of Done (every step)

- [ ] Step scope delivered, with no additions beyond the specification.
- [ ] Rules from [project-spec.md §7](./project-spec.md#7-rules) satisfied (clean code, good practices).
- [ ] Tests required by [test-spec.md](./test-spec.md) written and green; build (`mvn verify` / `npm run lint && npm test && npm run build`) green.
- [ ] Specifications updated if a contract changed.
- [ ] Step marked below; README updated if the way of running things changed.

## Open questions

The default assumptions from [project-spec.md §10](./project-spec.md#10-default-assumptions-to-be-confirmed-by-the-project-owner) apply until changed. To be confirmed before the indicated steps:
- Route 53 domain and HTTPS on AWS (before 7.5).
- Public catalog reads and self-registration (before 1.1 and 2.3).
- Group/package name (`pl.minishop`) (before 0.1).
- Elastic Stack (optional module): whether and when to enable it; the EKS variant requires cost approval (`t3.large` nodes, EBS volume) before `O.2`.
- Prometheus + Grafana (optional module): whether and when to enable it; the EKS variant (`O.4`) requires separate cost approval.

---

## Phase 0 – Repository foundation

### [ ] 0.1 Repository skeleton
**Load:** `project-spec.md`
**Scope:** `.gitignore`, `.editorconfig`, `README.md` (description, how to run – draft), `.env.example` (empty template with a comment), empty directories with `.gitkeep` according to the structure from §5, filling in the "Versions" table in `project-spec.md` with current, verified versions (Spring Boot, Spring Cloud compatible with Boot, Keycloak, PostgreSQL, Angular/Node).
**Acceptance:** directory structure matches the specification; version table without "to be filled"; note on Spring Cloud/Boot compatibility.
**Prompt:**
> Implement step 0.1 from `docs/implementation-plan.md`. Verify the current versions and the compatibility of Spring Boot with Spring Cloud OpenFeign in the official documentation, and record the results in the "Versions" table in `docs/project-spec.md`. Do not create application code yet.

---

## Phase 1 – Local environment: PostgreSQL + Keycloak

### [ ] 1.1 Postgres and Keycloak in docker-compose
**Load:** `local-dev-spec.md`, `auth-spec.md`
**Scope:** `docker-compose.yml` (only `postgres` and `keycloak`), `infra/local/postgres/init/*`, `infra/local/keycloak/realm-minishop.json` (roles, clients `minishop-spa`, `orders-service`, `minishop-smoke`, users `admin`/`user`), `.env.example`, README section "Running the infrastructure".
**Acceptance:** `docker compose up` starts both components as healthy; the three databases and roles exist; the `client_credentials` token for `orders-service` contains the `SERVICE` role; the `password` token (client `minishop-smoke`) for `admin` contains `ADMIN`, and `iss` = `http://localhost:8081/realms/minishop`; browser login to the Keycloak console works.
**Prompt:**
> Implement step 1.1 from `docs/implementation-plan.md`. Load `docs/local-dev-spec.md` and `docs/auth-spec.md`. When finished, show the `curl` commands verifying the acceptance criteria and their output (without revealing secrets in the log).

---

## Phase 2 – catalog-service

### [ ] 2.1 Skeleton, Liquibase, entity
**Load:** `backend-spec.md` (§1, §2, §3.1), `test-spec.md` (§1–3)
**Scope:** `services/pom.xml` (parent + `catalog` module), Maven wrapper, Spotless, JaCoCo, `catalog` module: application, configuration, Liquibase (`001-create-product-table`, `dev` seed), `Product` entity, repository, Actuator (probes), `X-Request-Id` filter; migration and repository tests with Testcontainers.
**Acceptance:** `./mvnw verify` green; migrations on an empty database; `ddl-auto=validate` passes; health `/actuator/health/readiness` = UP.
**Prompt:**
> Implement step 2.1 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§1, §2, §3.1) and `docs/test-spec.md`. Create the parent POM and the `catalog` module according to the specification. At this stage no controllers and no security.

### [ ] 2.2 Product API (CRUD, search, errors)
**Load:** `backend-spec.md` (§2, §3.2, §5), `test-spec.md` (§2)
**Scope:** `ProductController`, DTOs (`record`), MapStruct, service, domain exceptions, `@RestControllerAdvice` (Problem Details, `code`), validation, pagination, `q`, `@Version` (409 on conflict).
**Acceptance:** all "catalog" scenarios from test-spec for CRUD/search/validation/versioning; coverage ≥ 80% for `service` and `api`.
**Prompt:**
> Implement step 2.2 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§2, §3.2, §5) and `docs/test-spec.md` (§2). Security will be added in step 2.3 – for now the endpoints are open.

### [ ] 2.3 Catalog security (JWT)
**Load:** `auth-spec.md` (§1–4, §8), `test-spec.md` (§4)
**Scope:** resource server dependency, `JwtDecoder` (jwk-set-uri + issuer validator), role converter, `SecurityFilterChain` per the table, Problem Details for 401/403, `@ConfigurationProperties` `minishop.security`.
**Acceptance:** security tests from test-spec §4 for the catalog green; public reads, writes only for `ADMIN`; `/internal/**` reserved for `SERVICE` (endpoints in 2.4).
**Prompt:**
> Implement step 2.3 from `docs/implementation-plan.md`. Load `docs/auth-spec.md` and `docs/test-spec.md` (§4). Do not use `issuer-uri` alone with auto-discovery (see auth-spec §4).

### [ ] 2.4 Stock reservation and release (internal API)
**Load:** `backend-spec.md` (§3.3, §5), `auth-spec.md` (§4), `test-spec.md` (§2)
**Scope:** `POST /internal/catalog/stock/reservations` and `/releases`, conditional `UPDATE`, "all or nothing" transaction, ordering by `productId`, error handling, `SERVICE` role.
**Acceptance:** reservation scenarios from test-spec, **including the concurrency test** (20 parallel reservations with `stock = 10`); `USER`/`ADMIN` get 403 on `/internal/**`.
**Prompt:**
> Implement step 2.4 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§3.3, §5), `docs/auth-spec.md` (§4) and `docs/test-spec.md` (§2). The concurrency test is mandatory.

---

## Phase 3 – orders-service

### [ ] 3.1 Skeleton, Liquibase, entities
**Load:** `backend-spec.md` (§1, §2, §4.1), `test-spec.md` (§1–3)
**Scope:** `orders` module in the parent POM; configuration, `X-Request-Id` filter, Actuator; Liquibase (`orders`, `order_item`, indexes); entities, repositories; migration/repository tests.
**Acceptance:** `./mvnw verify` green for both modules; schema matches the specification.
**Prompt:**
> Implement step 3.1 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§1, §2, §4.1) and `docs/test-spec.md`. Keep the same conventions as in the `catalog` module; no controllers.

### [ ] 3.2 Feign client, resilience, service token
**Load:** `backend-spec.md` (§4.4, §5, §6), `auth-spec.md` (§5), `test-spec.md` (§2)
**Scope:** `CatalogClient`, timeouts, circuit breaker, `ErrorDecoder`, domain exceptions, OAuth2 `client_credentials` client + interceptor, `@ConfigurationProperties` `minishop.catalog`.
**Acceptance:** WireMock tests: success, 404, 409, 5xx, timeout, circuit opening, presence of the `Authorization` and `X-Request-Id` headers in the request to the catalog; no retry.
**Prompt:**
> Implement step 3.2 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§4.4, §5, §6), `docs/auth-spec.md` (§5) and `docs/test-spec.md` (§2). In tests, replace token verification with WireMock for the token endpoint.

### [ ] 3.3 Order use cases and API
**Load:** `backend-spec.md` (§2, §4.2, §4.3, §5), `test-spec.md` (§2)
**Scope:** order service (creation per the §4.3 flow, list, details, cancellation, confirmation, status transitions), controller, DTOs, MapStruct, error handling, compensation, `TransactionTemplate` outside the Feign call.
**Acceptance:** "orders" scenarios from test-spec (creation, catalog errors, compensation, status transitions); coverage ≥ 80% `service`/`api`. No security (step 3.4) – user identity provided by an abstraction (e.g. `CurrentUser`), substituted in tests.
**Prompt:**
> Implement step 3.3 from `docs/implementation-plan.md`. Load `docs/backend-spec.md` (§2, §4.2, §4.3, §5) and `docs/test-spec.md` (§2). Introduce user identity via a `CurrentUser` interface so that step 3.4 only swaps the implementation.

### [ ] 3.4 Orders security (JWT, ownership)
**Load:** `auth-spec.md` (§3, §4, §8), `test-spec.md` (§4)
**Scope:** resource server configuration as in the catalog (consider shared classes only if the duplication is real – rule of three), `CurrentUser` implementation from the JWT, access rules, order ownership, `scope=all` for `ADMIN`.
**Acceptance:** security tests from test-spec §4 for orders; user A cannot see B's orders (404).
**Prompt:**
> Implement step 3.4 from `docs/implementation-plan.md`. Load `docs/auth-spec.md` and `docs/test-spec.md` (§4). Do not introduce a shared library between the services unless justified – ask if you see such a need.

---

## Phase 4 – Backend in docker-compose

### [ ] 4.1 Dockerfiles and running the whole backend
**Load:** `local-dev-spec.md` (§2, §5, §7–9), `backend-spec.md` (§6), `test-spec.md` (§6)
**Scope:** Dockerfiles for `catalog` and `orders`, `.dockerignore`, extending `docker-compose.yml` (healthchecks, `depends_on`), variables in `.env.example`, `scripts/smoke.sh` (steps 1–4 per test-spec §6), README (running, common problems).
**Acceptance:** `docker compose up --build` → everything healthy; `scripts/smoke.sh` passes (product creation, order, stock decrease and restoration); images run as a non-root user.
**Prompt:**
> Implement step 4.1 from `docs/implementation-plan.md`. Load `docs/local-dev-spec.md`, `docs/backend-spec.md` (§6) and `docs/test-spec.md` (§6). At the end, start everything and show the output of `scripts/smoke.sh`.
---

## Phase 5 – Frontend

### [ ] 5.1 Angular skeleton and login
**Load:** `frontend-spec.md` (§1–3, §5), `auth-spec.md` (§7), `test-spec.md` (§5)
**Scope:** Angular project (strict, ESLint, Prettier), Angular Material, loading `/config.json`, `AuthService` (`angular-oauth2-oidc`, code + PKCE), `authInterceptor`, `errorInterceptor`, guards, header, routing (placeholders), `proxy.conf.json`.
**Acceptance:** login as `user` and `admin` via Keycloak, the header shows the name and the right tabs, logout works; service/guard/interceptor tests green; `lint` and `build` without warnings.
**Prompt:**
> Implement step 5.1 from `docs/implementation-plan.md`. Load `docs/frontend-spec.md` (§1–3, §5), `docs/auth-spec.md` (§7) and `docs/test-spec.md` (§5). Start the backend and Keycloak via docker compose.

### [ ] 5.2 Products and cart
**Load:** `frontend-spec.md` (§4.1, §4.2, §4.7, §5), `backend-spec.md` (§3.2), `test-spec.md` (§5)
**Scope:** `ProductsApi`, list with search and pagination, product card, states (loading/empty/error), `CartStore` with `localStorage`, cart panel, limits based on `stock`.
**Acceptance:** behavior consistent with §4.2; `CartStore` and component tests per test-spec §5.
**Prompt:**
> Implement step 5.2 from `docs/implementation-plan.md`. Load `docs/frontend-spec.md` (§4.1, §4.2, §4.7, §5), `docs/backend-spec.md` (§3.2), `docs/test-spec.md` (§5). The look must match the description in the specification (two columns: products and cart).

### [ ] 5.3 Placing orders and order views
**Load:** `frontend-spec.md` (§4.3–4.5, §4.7), `backend-spec.md` (§4.2, §5), `test-spec.md` (§5)
**Scope:** `OrdersApi`, placing an order with handling of 409/422/503 errors, "My orders" list, details, cancellation with a confirmation dialog.
**Acceptance:** scenarios from test-spec §5 (success and error mapping); manual flow: product → cart → order → cancellation.
**Prompt:**
> Implement step 5.3 from `docs/implementation-plan.md`. Load `docs/frontend-spec.md` (§4.3–4.5, §4.7), `docs/backend-spec.md` (§4.2, §5) and `docs/test-spec.md` (§5).

### [ ] 5.4 Admin panel
**Load:** `frontend-spec.md` (§4.6, §4.7), `backend-spec.md` (§3.2, §4.2), `test-spec.md` (§5)
**Scope:** product table, form (validation matching the backend, version conflict handling), deletion with confirmation, list of all orders with Confirm/Cancel actions, `adminGuard`.
**Acceptance:** `user` does not see the panel and cannot enter it via URL; admin performs CRUD and confirms an order; form and guard tests.
**Prompt:**
> Implement step 5.4 from `docs/implementation-plan.md`. Load `docs/frontend-spec.md` (§4.6, §4.7), `docs/backend-spec.md` (§3.2, §4.2) and `docs/test-spec.md` (§5).

### [ ] 5.5 Frontend image and full docker-compose
**Load:** `frontend-spec.md` (§6), `local-dev-spec.md` (§2, §6), `test-spec.md` (§6)
**Scope:** Dockerfile (node → nginx), `nginx.conf` (SPA fallback, `/api/*` proxy, `/internal/` → 404, headers), generating `/config.json` from env, adding the `frontend` service to compose, updating the README and the smoke test (step 5 of test-spec §6).
**Acceptance:** `docker compose up --build` → the application at `http://localhost:8080` works end-to-end (login, shopping, admin); `curl http://localhost:8080/internal/catalog/stock/reservations` returns 404.
**Prompt:**
> Implement step 5.5 from `docs/implementation-plan.md`. Load `docs/frontend-spec.md` (§6), `docs/local-dev-spec.md` and `docs/test-spec.md` (§6). Verify everything end-to-end on the running compose.

---

## Phase 6 – CI and local Kubernetes

### [ ] 6.1 GitHub Actions – CI
**Load:** `infra-spec.md` (§7), `test-spec.md` (§3, §5, §7)
**Scope:** `.github/workflows/ci.yml`: backend `verify`, frontend lint/test/build, Docker image builds (no push), Maven/npm cache; badge in the README.
**Acceptance:** workflow green on a PR; a formatting error or coverage dropping below the threshold breaks the build.
**Prompt:**
> Implement step 6.1 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§7) and `docs/test-spec.md`. `helm lint` will be added in step 6.2, and Terraform in phase 7.

### [ ] 6.2 Helm charts and local Kubernetes (Docker Desktop)
**Load:** `k8s-local-spec.md` (entire), `infra-spec.md` (§5), `local-dev-spec.md` (§2, §5, §6), `auth-spec.md` (§2, §6), `test-spec.md` (§6)
**Scope:** charts `postgres-local`, `db-init`, `keycloak`, `catalog`, `orders`, `frontend` with `values.yaml` and `values-local.yaml` (switches from k8s-local-spec §5, no environment-specific constants); `scripts/k8s-local-up.sh` and `k8s-local-down.sh` with a `docker-desktop` context check; additional redirect URI in the dev realm; parameterization of `scripts/smoke.sh` (`BASE_URL`, `AUTH_URL`); `helm lint` in `ci.yml`; README ("Running on Kubernetes – Docker Desktop", common pitfalls).
**Acceptance:** criteria from k8s-local-spec §8 (all pods Ready, full browser flow at `http://shop.localtest.me`, smoke test, `/internal/**` → 404, script idempotency, refusal to work in another context).
**Prompt:**
> Implement step 6.2 from `docs/implementation-plan.md`. Load `docs/k8s-local-spec.md`, `docs/infra-spec.md` (§5), `docs/local-dev-spec.md`, `docs/auth-spec.md` (§2, §6) and `docs/test-spec.md` (§6). The charts must be shared with the future AWS deployment – do not hardcode anything environment-specific. Verify the current ingress-nginx versions and the image behavior in Docker Desktop Kubernetes in the documentation. At the end, run everything on the local cluster and show the results of the acceptance criteria.

---

## Phase 7 – AWS (EKS)

> Before starting the phase: confirm the domain/HTTPS (see "Open questions"). Each step ends with a cost summary and a "keep / destroy" decision.

### [ ] 7.1 Terraform bootstrap, budget, OIDC for GitHub
**Load:** `infra-spec.md` (§1, §2, §3, §9)
**Scope:** `bootstrap/` (state bucket), `envs/dev` skeleton (backend, providers), AWS Budgets with alerts, IAM OIDC provider + role for GitHub Actions, ECR (3 repositories).
**Acceptance:** `terraform plan/apply` with remote state; the budget and alerts exist; the OIDC role is restricted to the repo and the `main` branch.
**Prompt:**
> Implement step 7.1 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§1, §2, §3, §9). Do not run `terraform apply` without my approval – show the `plan` and a cost summary.

### [ ] 7.2 VPC and EKS
**Load:** `infra-spec.md` (§3)
**Scope:** VPC (one NAT), EKS with a managed node group, add-ons, access entries; outputs needed later.
**Acceptance:** `kubectl get nodes` shows 2 Ready nodes; `terraform validate`/`fmt` clean.
**Prompt:**
> Implement step 7.2 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§3). Verify the current module and Kubernetes versions in the documentation. Show the `plan`, do not run `apply` without my approval.

### [ ] 7.3 Cluster controllers, RDS and secrets
**Load:** `infra-spec.md` (§3, §4, §5)
**Scope:** AWS Load Balancer Controller, External Secrets Operator, metrics-server (Helm via Terraform or a separate script); IAM roles (Pod Identity/IRSA); RDS PostgreSQL; secrets in Secrets Manager; using the `db-init` chart from step 6.2 (values for RDS and ExternalSecret).
**Acceptance:** controllers Running; RDS reachable only from the cluster; the `db-init` Job is idempotent (two runs without errors); `ExternalSecret` synchronizes the secrets.
**Prompt:**
> Implement step 7.3 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§3, §4, §5). Do not print secrets in logs or put them in the repository.

### [ ] 7.4 Helm charts and service deployment
**Load:** `infra-spec.md` (§5, §6), `local-dev-spec.md` (§5), `backend-spec.md` (§6)
**Scope:** AWS `values-dev.yaml` for the `catalog`, `orders`, `frontend` charts from step 6.2 (ALB Ingress, images from ECR, ExternalSecret, HPA, PDB), possible template additions (without breaking `values-local.yaml`), manual push of images to ECR and `helm upgrade --install`.
**Acceptance:** pods Ready; `helm lint` clean; `/internal/**` absent from the Ingresses; Liquibase migrations executed on RDS.
**Prompt:**
> Implement step 7.4 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§5, §6), `docs/backend-spec.md` (§6). Keycloak will be added in 7.5, so at this stage only check pod health and the database connection.

### [ ] 7.5 Keycloak on EKS, DNS and TLS
**Load:** `infra-spec.md` (§5, §8), `auth-spec.md` (§2, §6)
**Scope:** production values of the `keycloak` chart from step 6.2 (production mode, production realm passed via `--set-file`), Route 53 records, ACM certificate, `auth.<domain>` Ingress, `iss`/JWKS configuration in the services, the frontend's `config.json`.
**Acceptance:** full login at `https://shop.<domain>`; `USER`/`ADMIN` roles work; admin console not exposed publicly; end-to-end order on AWS.
**Prompt:**
> Implement step 7.5 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§5, §8) and `docs/auth-spec.md` (§2, §6). Verify the Keycloak settings for the pinned version (hostname, proxy).

### [ ] 7.6 CD – GitHub Actions
**Load:** `infra-spec.md` (§7)
**Scope:** `deploy.yml` (OIDC, build/push with the SHA tag, `helm upgrade --atomic`, post-deployment smoke test), adding `terraform fmt/validate` to `ci.yml` (`helm lint` is there since step 6.2).
**Acceptance:** a push to `main` deploys the changed components; a failed deployment is rolled back; no AWS keys in the repo secrets.
**Prompt:**
> Implement step 7.6 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§7). Do not add long-lived AWS keys – OIDC only.

### [ ] 7.7 Scaling and basic diagnostics
**Load:** `infra-spec.md` (§5, §10), `observability-spec.md` (§0)
**Scope:** metrics-server, HPA for `catalog` and `orders`, PDB, verification of scaling under test load, `scripts/teardown.sh` (order: `helm uninstall` → `terraform destroy`), README with basic diagnostics (`kubectl logs`, `kubectl top`, searching by `requestId`).
**Acceptance:** HPA scales up under load and back down; teardown leaves no orphaned ALBs or volumes.
**Prompt:**
> Implement step 7.7 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§5, §10) and `docs/observability-spec.md` (§0). Do not add the Elastic Stack – it is an optional module.

### [ ] 7.8 (optional) Argo CD / GitOps
**Load:** `infra-spec.md` (§7)
**Scope:** installing Argo CD, applications pointing to the charts in the repo, removing the `helm upgrade` step from `deploy.yml` in favor of updating the image tag in the repo.
**Acceptance:** a tag change in the repo deploys automatically; cluster state consistent with the repo.
**Prompt:**
> Implement step 7.8 from `docs/implementation-plan.md`. Load `docs/infra-spec.md` (§7). First present the proposed repository layout for GitOps and wait for my approval.

---

## Optional modules (only on explicit request)

> **Do not implement initially.** Skip the `O.*` steps until the user explicitly asks for them. They are performed after the core is finished (phases 0–6; `O.2` and `O.4` on EKS after phase 7).

### [ ] O.1 Elastic Stack locally (ELK)
**Load:** `observability-spec.md` (entire: §0–5, §7), `local-dev-spec.md` (§10), `backend-spec.md` (§2, §4.4), `test-spec.md` (§2)
**Scope:** switching logs to ECS JSON (profile/variable, no changes to business code), compose profile `logging` (Elasticsearch, Kibana, Filebeat), `infra/local/elk/filebeat.yml`, 7-day retention, `kibana-objects.ndjson` (data view, searches, dashboard), `scripts/kibana-import.sh`, updating the README and the version table (Elastic Stack) in `project-spec.md`.
**Acceptance:** `docker compose --profile logging up` → Kibana shows the logs of `catalog`, `orders` and `keycloak`; filtering by `requestId` shows entries from both services for a single request; no tokens in logs; the default `docker compose up` unchanged (no Elastic stack).
**Prompt:**
> Implement step O.1 from `docs/implementation-plan.md` (optional module, I am enabling it deliberately). Load `docs/observability-spec.md`, `docs/local-dev-spec.md` (§10), `docs/backend-spec.md` (§2, §4.4) and `docs/test-spec.md` (§2). Verify property and field names in the documentation of the pinned Spring Boot and Elastic Stack versions; do not add Logstash.

### [ ] O.2 Elastic Stack on EKS (ECK)
**Load:** `observability-spec.md` (§3, §4, §6, §7), `infra-spec.md` (§3, §9, §10)
**Scope:** EBS CSI add-on and StorageClass `gp3`, switching nodes to `t3.large`, ECK operator, resources `Elasticsearch` (1 node), `Kibana`, `Beat` (Filebeat DaemonSet), 7-day retention, Kibana object import, access via `port-forward` (optionally an Ingress with IP restriction), extending `scripts/teardown.sh` with PVC deletion.
**Acceptance:** Kibana on EKS shows the logs of all components; filtering by `requestId` works across services; passwords only in secrets; no orphaned volumes after teardown.
**Prompt:**
> Implement step O.2 from `docs/implementation-plan.md` (optional module, I am enabling it deliberately; `O.1` is already done). Load `docs/observability-spec.md` and `docs/infra-spec.md` (§3, §9, §10). Verify the current ECK and Elastic Stack versions. Show the `plan` and a cost summary before `apply`; passwords and tokens must not end up in the repository or in logs.

### [ ] O.3 Prometheus + Grafana locally (docker-compose)
**Load:** `monitoring-spec.md` (§0–5, §7, §8), `local-dev-spec.md` (§12), `backend-spec.md` (§2), `auth-spec.md` (§4), `test-spec.md` (§2)
**Scope:** Micrometer/Prometheus registry, Spring `monitoring` profile (`/actuator/prometheus` endpoint and access rule), business metrics from §3, compose profile `monitoring` (Prometheus, Grafana, provisioning of the datasource and the "Mini shop – services" dashboard), alert rules (rules file), tests from §8, README, updating the version table in `project-spec.md`.
**Acceptance:** `docker compose --profile monitoring up` → both targets `UP` in Prometheus, the Grafana dashboard shows traffic after generating orders; `/actuator/prometheus` not reachable via `http://localhost:8080`; tests from §8 green; the default `docker compose up` unchanged.
**Prompt:**
> Implement step O.3 from `docs/implementation-plan.md` (optional module, I am enabling it deliberately). Load `docs/monitoring-spec.md` (§0–5, §7, §8), `docs/local-dev-spec.md` (§12), `docs/backend-spec.md` (§2), `docs/auth-spec.md` (§4) and `docs/test-spec.md`. Verify the available Resilience4j/Feign metrics in the documentation of the pinned versions. Do not add Alertmanager or additional business metrics.

### [ ] O.4 Prometheus + Grafana on Kubernetes (Docker Desktop, then EKS)
**Load:** `monitoring-spec.md` (§4–10), `k8s-local-spec.md` (§5, §10), `infra-spec.md` (§3, §9, §10)
**Scope:** installing kube-prometheus-stack (Alertmanager disabled) with `values-local.yaml` and `values-dev.yaml`, `ServiceMonitor` template in the `catalog` and `orders` charts (`monitoring.serviceMonitor.enabled`), `PrometheusRule` from §7, dashboard as a ConfigMap, `monitoring` profile in the chart values, updating `scripts/k8s-local-up.sh`/`down.sh` (`--monitoring` option), on EKS: EBS CSI and StorageClass `gp3` (if not already present), `t3.large` nodes when enabled together with ELK.
**Acceptance:** locally: targets `UP` in Prometheus, dashboard accessible via `port-forward`, stopping `catalog` triggers `ServiceDown`; on EKS the same plus no orphaned volumes after teardown; metrics not reachable through the Ingress.
**Prompt:**
> Implement step O.4 from `docs/implementation-plan.md` (optional module, I am enabling it deliberately; `O.3` is already done). Start with local Kubernetes (Docker Desktop). Do the EKS part only after my separate approval: show the `plan` and a cost summary before `apply`. Load `docs/monitoring-spec.md`, `docs/k8s-local-spec.md` and `docs/infra-spec.md` (§3, §9, §10). Verify the current version of the kube-prometheus-stack chart.

---

## Extras (outside the plan, on request)

E2E tests (Playwright), JSON logging and distributed tracing, outbox/Saga instead of compensation (KO-1), a shared dark theme, resource limits (`LimitRange`/`ResourceQuota`), image/dependency scanning in CI.
