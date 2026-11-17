# Logging and observability – specification (Elastic Stack)

Load §0 when configuring logging in the services. Load the rest of the document (**optional module**) only when the user has explicitly asked for ELK. Context: [project-spec.md](./project-spec.md). Local environment: [local-dev-spec.md](./local-dev-spec.md). AWS: [infra-spec.md](./infra-spec.md).

## 0. Module status and core minimum

**The Elastic Stack is an optional module. Do not implement it initially** – perform steps `O.1` and `O.2` in [implementation-plan.md](./implementation-plan.md#optional-modules-only-on-explicit-request) only on the user's explicit request. Until the module is enabled, only the following core rules apply:

- Logs to stdout (no files), a text pattern containing `requestId` from the MDC.
- `X-Request-Id` propagation via Feign (see [backend-spec.md §4.4](./backend-spec.md#44-feign-client-catalogclient)).
- Logging levels and prohibitions (tokens, passwords, secrets, personal data) from §2 below already apply in the core.
- Logging configuration in the services kept in one place, so that a later switch to ECS JSON is a configuration change (profile/variable), not a code change.

After enabling the module: ECS JSON format, the `logging` compose profile and ECK on EKS as described in §1–7.

## 1. Goal and stack choice (optional module)

Central search of the logs of both services (and Keycloak) in Kibana, with the ability to trace a single request through `orders` → `catalog` by `requestId`.

| Component | Role | Decision |
|---|---|---|
| Elasticsearch | Log storage and index | yes |
| Kibana | Search, dashboards | yes |
| Filebeat | Collecting container logs and shipping them to Elasticsearch | yes (DaemonSet on EKS, container locally) |
| Logstash | Parsing/transformations | **omitted by default** – logs are already structured (ECS JSON), so Logstash would add a component and RAM usage without benefit. Added only on explicit request (educational purpose) |

Rejected alternative: Amazon OpenSearch Service (it is not Elasticsearch/Kibana, higher cost of the minimal instance). This extends decision D8 from [project-spec.md](./project-spec.md#9-design-decisions).

## 2. Log format (Spring Boot services)

- *(The JSON format concerns the optional module; the levels, `requestId` fields and prohibitions below always apply.)*
- After enabling the module: logs **to stdout** in **ECS JSON** format – Spring Boot's built-in structured logging (`logging.structured.format.console=ecs`; requires a Boot version with this feature, verify in step 0.1). One line = one JSON object.
- Mandatory fields: `@timestamp`, `log.level`, `message`, `service.name` (`catalog-service` / `orders-service`), `service.environment` (`dev`), `requestId` (from the MDC, see [backend-spec.md §2](./backend-spec.md#2-common-conventions)), `error.stack_trace` for exceptions. Verify the exact MDC field name on the chosen version and stabilize it in one configuration place.
- In the IDE a text format is acceptable (overridden by an environment variable); in containers always JSON.
- **Forbidden in logs:** tokens, passwords, secrets, full personal data, request bodies. The user identifier (`sub`) is allowed only at `DEBUG` level.
- Levels: `ERROR` – the operation failed and requires attention (including a failed compensation, KO-1/KO-2); `WARN` – an expected business error or degradation (e.g. open circuit breaker); `INFO` – business events (order created/cancelled); `DEBUG` – diagnostics (disabled by default).
- `requestId` is propagated via Feign (the `X-Request-Id` header), so the same ID appears in the logs of both services.

## 3. Indexing and retention

- Data in the data stream `logs-minishop-<environment>` (or the index `minishop-logs-*`, depending on the version and Filebeat configuration – choose one and document it in the configuration file).
- Retention: **7 days** (ILM/lifecycle: deletion after 7 days) to limit disk usage.
- Mappings: ECS fields; `requestId` as `keyword`.
- Filebeat adds container/pod metadata (name, namespace, image); it collects only containers labeled `logging=true` (compose) or from the application namespace (Kubernetes) – it does not collect the Elastic stack's own logs.

## 4. Kibana – versioned objects

Saved objects exported to `infra/local/elk/kibana-objects.ndjson` and imported with the `scripts/kibana-import.sh` script (the same files for compose and EKS):

- **Data view** `minishop-logs` (time field `@timestamp`).
- **Saved searches:** "Errors" (`log.level: "ERROR"`), "Request tracing" (`requestId : *`, with columns `service.name`, `log.level`, `message`).
- **Dashboard "Mini shop – logs":** log count over time by `service.name`; error count by service; most frequent error messages (top 10); distribution of log levels.

## 5. Local environment (docker-compose, `logging` profile)

The default `docker compose up` does **not** start the Elastic stack (RAM). To enable: `docker compose --profile logging up`.

| Service | Notes |
|---|---|
| `elasticsearch` | single node (`discovery.type=single-node`), limited heap (e.g. 512 MB), container memory limit, data volume; **security disabled – dev only** (explicitly described in the file); port 9200 |
| `kibana` | connection to `elasticsearch`; port 5601; import of the objects from §4 |
| `filebeat` | Docker container autodiscover (read-only access to `/var/lib/docker/containers` and the Docker socket), JSON decoding, `add_docker_metadata`, output to `elasticsearch`; file `infra/local/elk/filebeat.yml` |

- Application services and Keycloak get the label `logging=true`.
- Requirements: at least a few GB of free memory for Docker; on Linux hosts set `vm.max_map_count` according to the Elasticsearch documentation.
- Kibana: `http://localhost:5601`.
- General environment details: [local-dev-spec.md §10](./local-dev-spec.md#10-logs-elk-optional-logging-profile).

## 6. AWS (EKS)

- **ECK (Elastic Cloud on Kubernetes)** installed as an operator (Helm). Resources: `Elasticsearch` (dev: 1 node), `Kibana` (1 replica), `Beat` (Filebeat as a DaemonSet). One Elastic Stack version for all components (version table in project-spec).
- Storage: EBS gp3 volume (StorageClass `gp3`, requires the EBS CSI Driver add-on – added in [infra-spec.md](./infra-spec.md#3-resources)), 20 GB for dev.
- Approximate resources: Elasticsearch – memory request/limit approx. 2 Gi (1 GB heap), Kibana approx. 1 Gi, Filebeat approx. 100–200 Mi per node. **With 2 × `t3.medium` memory will run out** – nodes changed to `t3.large`.
- Security: ECK enables TLS and authentication by default; the `elastic` user's password in a Kubernetes secret (not in the repo). Filebeat uses a separate user with minimal permissions (write to the log index).
- Access to Kibana: `kubectl port-forward` by default. Optionally an Ingress `kibana.<domain>` with HTTPS, restricted to the owner's IP range (ALB annotation). Kibana login via Keycloak (SSO) is **out of scope** (usually requires a paid subscription – verify).
- Collected: logs from the application namespace (catalog, orders, frontend, keycloak).
- After `terraform destroy` the PVC volumes must be deleted (included in `scripts/teardown.sh`).

## 7. Acceptance criteria

- After placing an order in the UI, filtering by `requestId` in Kibana shows entries from `orders-service` **and** `catalog-service` for the same request.
- A deliberately triggered error (e.g. stopped `catalog`) appears as `WARN`/`ERROR` in the "Errors" view.
- There are no tokens or passwords in the logs (sample review + test from [test-spec.md](./test-spec.md#2-backend--layers)).
- 7-day retention configured and verified in Kibana (Stack Management).
