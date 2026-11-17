# Local environment – specification (docker-compose)

Load when working on `docker-compose.yml`, `infra/local/**`, Dockerfiles and `.env.example`. Context: [project-spec.md](./project-spec.md). Keycloak: [auth-spec.md](./auth-spec.md). Service configuration: [backend-spec.md](./backend-spec.md#6-configuration-environment-variables).

## 1. Goal

`docker compose up --build` starts everything: PostgreSQL, Keycloak, `catalog-service`, `orders-service`, frontend. For working in the IDE you can start only the infrastructure: `docker compose up postgres keycloak`, and run the services and `ng serve` locally.

## 2. Services

| Service | Image / build | Host port | Dependencies | Healthcheck |
|---|---|---|---|---|
| `postgres` | official `postgres` (pinned version) | 5432 | – | `pg_isready` |
| `keycloak` | `quay.io/keycloak/keycloak:<pinned tag>` | 8081 → 8080 | `postgres` (healthy) | `/health/ready` on the management port (9000); the image has no `curl` – use a TCP/`bash` test |
| `catalog-service` | build `services/catalog-service` | 8082 → 8080 | `postgres`, `keycloak` (healthy) | `/actuator/health/readiness` |
| `orders-service` | build `services/orders-service` | 8083 → 8080 | `postgres`, `keycloak`, `catalog-service` (healthy) | `/actuator/health/readiness` |
| `frontend` | build `frontend` | 8080 → 80 | `catalog-service`, `orders-service` (healthy) | `GET /` |

Shared default compose network; hostnames = service names. Volume `postgres-data` (persistent data). `restart: unless-stopped` only for `postgres` and `keycloak`.

## 3. PostgreSQL – initialization

`infra/local/postgres/init/01-init.sql` (run only on the first start with an empty volume):

- Roles (LOGIN): `catalog_app`, `orders_app`, `keycloak_app` – dev-only passwords from `.env`.
- Databases: `catalog` (owner `catalog_app`), `orders` (owner `orders_app`), `keycloak` (owner `keycloak_app`).
- `REVOKE CONNECT ON DATABASE ... FROM PUBLIC` for each database; each role connects only to its own.

Passwords are not hardcoded in the SQL file: a `.sh` script in the init directory (or `envsubst`) reads them from environment variables.

## 4. Keycloak

- Command: `start-dev --import-realm`; the realm file is mounted at `/opt/keycloak/data/import/realm-minishop.json` (read-only).
- Variables: `KC_DB=postgres`, `KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak`, `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KC_HOSTNAME=http://localhost:8081`, `KC_HTTP_ENABLED=true`, `KC_HEALTH_ENABLED=true`, `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` (Keycloak admin account – dev-only, verify the variable names for the pinned version).
- Addresses: browser → `http://localhost:8081`; containers → `http://keycloak:8080`. Tokens have `iss=http://localhost:8081/realms/minishop` (see [auth-spec.md §4](./auth-spec.md#4-resource-server-catalog-and-orders)).
- Admin console: `http://localhost:8081` (credentials from `.env`).

## 5. Backend services

Environment variables per [backend-spec.md §6](./backend-spec.md#6-configuration-environment-variables); `SPRING_PROFILES_ACTIVE=dev`, `SPRING_LIQUIBASE_CONTEXTS=dev` (product seed).

### Dockerfile (both services, multi-stage)
1. **build:** image with JDK 25 + Maven wrapper; copy `pom.xml` before the sources (dependency cache); `./mvnw -pl <module> -am package -DskipTests`.
2. **runtime:** JRE 25 (Temurin, slim/alpine variant matching the architecture), non-root user, `EXPOSE 8080`, `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`, `ENTRYPOINT ["java","-jar","/app/app.jar"]`.
3. The image contains no secrets; `.dockerignore` excludes `target/`, `.git/`, `node_modules/`.

## 6. Frontend

Image per [frontend-spec.md §6](./frontend-spec.md#6-build-and-container). Variables: `KEYCLOAK_ISSUER` (`http://localhost:8081/realms/minishop`), `OIDC_CLIENT_ID` (`minishop-spa`), `CATALOG_UPSTREAM` (`catalog-service:8080`), `ORDERS_UPSTREAM` (`orders-service:8080`).

## 7. `.env.example`

Contains only dev-only values, with the comment "do not use outside the local environment": Postgres role passwords, Keycloak admin credentials, `ORDERS_CLIENT_SECRET` (must match the realm). The `.env` file is in `.gitignore`.

## 8. Commands

| Purpose | Command |
|---|---|
| Everything | `docker compose up --build` |
| Infrastructure only | `docker compose up postgres keycloak` |
| Reset data | `docker compose down -v` |
| Service logs | `docker compose logs -f orders-service` |
| Smoke test | `./scripts/smoke.sh` ([test-spec.md §6](./test-spec.md#6-environment-smoke-test-after-docker-compose-up)) |

## 9. Common problems (for the README)

- **401 despite a valid token:** `iss` mismatch – check `KC_HOSTNAME` and `MINISHOP_SECURITY_ISSUER` (both must point to `http://localhost:8081/realms/minishop`).
- **A service does not start because Keycloak is not ready yet:** check the healthcheck and `depends_on: condition: service_healthy`.
- **A realm change has no effect:** the import runs only for a non-existent realm – `docker compose down -v` and start again.

## 10. Logs (ELK, optional `logging` profile)

> **Optional module – do not implement initially.** This section applies only after an explicit request from the user (plan step `O.1`).

Details and rationale: [observability-spec.md](./observability-spec.md). The Elastic stack is in a separate compose profile, so a plain `docker compose up` stays lightweight.

| Service | Image | Host port | Notes |
|---|---|---|---|
| `elasticsearch` | official (pinned version) | 9200 | single node, limited heap, `elastic-data` volume, security disabled (dev-only) |
| `kibana` | official (same version) | 5601 | `depends_on` Elasticsearch (healthy) |
| `filebeat` | official (same version) | – | configuration `infra/local/elk/filebeat.yml`; collects containers labeled `logging=true` |

- Startup: `docker compose --profile logging up --build`; reset logs: `docker compose down -v` (also removes volumes).
- Application services (`catalog-service`, `orders-service`) and `keycloak` have the label `logging=true`.
- Dashboard import: `scripts/kibana-import.sh` (objects from `infra/local/elk/kibana-objects.ndjson`).
- Common problems: Elasticsearch exits with code 137 (not enough memory for Docker) or reports a `vm.max_map_count` error (Linux) – see the Elasticsearch documentation for the pinned version.

## 11. Alternative: local Kubernetes

The same system can be run on Kubernetes in Docker Desktop with the same charts as on AWS – see [k8s-local-spec.md](./k8s-local-spec.md) (plan step 6.2). docker-compose remains the primary, simplest local environment.

## 12. Metrics (Prometheus + Grafana, optional `monitoring` profile)

> **Optional module – do not implement initially.** Applies after an explicit request from the user (plan step `O.3`). Details: [monitoring-spec.md](./monitoring-spec.md).

| Service | Host port | Notes |
|---|---|---|
| `prometheus` | 9090 | configuration `infra/local/monitoring/prometheus.yml`, targets `catalog-service:8080` and `orders-service:8080` (`/actuator/prometheus`), 3-day retention |
| `grafana` | 3000 | datasource and dashboard provisioning from `infra/local/monitoring/grafana/`; dev-only admin password from `.env` |

Startup: `docker compose --profile monitoring up --build` (services get the Spring `monitoring` profile). Can run together with the `logging` profile.
