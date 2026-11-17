# Tests – specification

Load when implementing any production code (section relevant to the layer). General rules: [project-spec.md](./project-spec.md#75-tests).

## 1. Principles

- Every behavior change has a test; a bug = first a reproducing test, then the fix.
- Pyramid: many unit tests, fewer integration tests, a minimum of end-to-end tests (smoke).
- Tests are deterministic, independent of order, without `Thread.sleep` (use Awaitility), without dependence on system time (injected `Clock`).
- Naming: `shouldDoSomething_whenCondition` (Java) / `should do something when condition` (TS). Given–When–Then structure. AssertJ assertions (Java).
- Test data via builders/factories (`ProductTestData`), not copied literals.
- Do not test the framework (getters, Spring configuration) – test behavior.
- Mock boundaries (HTTP, time), not your own domain classes; test the database with Testcontainers, not H2.

## 2. Backend – layers

| Layer | Test type | Tools |
|---|---|---|
| `service` | unit | JUnit 5, Mockito, AssertJ |
| mappers | unit | JUnit 5 |
| `repository` + Liquibase | integration | `@DataJpaTest` + Testcontainers PostgreSQL (Liquibase migrations run on a real database, `ddl-auto=validate`) |
| `api` (controllers) | slice | `@WebMvcTest` + `spring-security-test` (`jwt()`), verification of status codes, Problem Details, validation |
| whole service | integration | `@SpringBootTest` + Testcontainers; in `orders` additionally WireMock as the catalog |

### Required scenarios (minimum)

**catalog**
- Product CRUD, search (`q`, case), pagination and sorting.
- Validation: empty/too long `name`, `price ≤ 0`, negative `stock`.
- Optimistic locking: PUT with a stale `version` → 409.
- Reservation: success; out of stock (nothing changed – "all or nothing"); non-existent product; duplicates; **concurrency** (e.g. 20 parallel reservations with `stock = 10` → exactly 10 successes, stock does not drop below 0).
- Release: stock increases by the given quantities.
- Migrations: apply from scratch on an empty database; schema consistent with the entities.

**orders**
- Creation: correct `total` computed from the snapshot; `user_id` from `sub`; status `NEW`.
- Catalog errors: 409 → `INSUFFICIENT_STOCK` (mapping), 404 → 422 `PRODUCT_NOT_FOUND`, 5xx/timeout → 503 `CATALOG_UNAVAILABLE`.
- Compensation: a save failure after a successful reservation → `release` called with the same items.
- Circuit breaker opens after a series of errors (WireMock with delays/5xx).
- Status transitions: allowed and forbidden (`INVALID_ORDER_STATE`); cancellation calls `release`; a `release` error → rollback + 503.
- Ownership: user A cannot see user B's order (404); `scope=all` only for `ADMIN`.
- Logging: a request with `X-Request-Id` → the same identifier in the `orders` logs and in the header of the catalog call (WireMock); logs contain neither the token nor the password (e.g. `OutputCaptureExtension`).

## 3. Technical conventions (Java)

- Maven: `*Test` (unit, Surefire), `*IT` (integration, Failsafe) – `mvn verify` runs both.
- Testcontainers: shared container (`@ServiceConnection` or `static` + reuse within a module); the same PostgreSQL image as in docker-compose.
- Coverage: JaCoCo, threshold **80% of lines** for the `service` and `api` packages (the build fails below the threshold); other packages without a threshold.
- Formatting: `spotless:check` in `verify`.

## 4. Security tests

Mandatory in `@WebMvcTest`/integration tests for every endpoint (see the tables in [auth-spec.md](./auth-spec.md#4-resource-server-catalog-and-orders)):
- no token → 401; wrong `iss`/expired token → 401;
- insufficient role → 403 (e.g. `USER` on `POST /api/catalog/products`; `USER` on `/internal/**`);
- `SERVICE` has access to `/internal/**`, `USER`/`ADMIN` do not;
- role converter: `realm_access.roles` → `ROLE_*` (unit test, including a missing claim);
- 401/403 responses in Problem Details format.

Keycloak **is not started** in the services' unit/integration tests – tokens are built via `jwt()` / a test `JwtDecoder`.

## 5. Frontend

| Area | Type | Tools |
|---|---|---|
| services (API, `CartStore`, `AuthService`) | unit | the Angular version's default runner, `HttpTestingController` |
| components | component tests (TestBed, interactions by roles/labels) | Angular Testing Library (optional) |
| guards, interceptors | unit | as above |

Minimum scenarios:
- `CartStore`: adding, `stock` limit, removing, total, resilience to corrupted `localStorage`.
- Product list: loading/empty/error states; button disabled when `stock = 0`.
- Placing an order: success; mapping of 409/422/503 errors to messages.
- `authInterceptor`: Bearer only for `/api/**`.
- `authGuard`/`adminGuard`: redirects.
- Product form (admin): validation consistent with the backend.

Lint and build (`ng lint`, `ng build`) without warnings are part of a "green" step.

## 6. Environment smoke test (after `docker compose up`)

The `scripts/smoke.sh` script (bash + curl + jq) accepts `BASE_URL` and `AUTH_URL` (docker-compose defaults: `http://localhost:8080` and `http://localhost:8081`), so it also works on local Kubernetes (`http://shop.localtest.me`, `http://auth.localtest.me`):
1. Obtains an `admin` token (Authorization Code is not suitable for a script – in the **dev** environment the direct access grant is enabled for the test client `minishop-smoke`; the client exists only in the dev realm).
2. Creates a product, fetches the list.
3. As `user`, places an order and checks that the stock decreased.
4. Cancels the order and checks that the stock was restored.
5. Checks that `/internal/**` is not reachable from outside (404/401 via nginx).

## 7. CI

Required green: backend `mvn verify` (tests + JaCoCo + Spotless), frontend `lint` + `test` + `build`, Docker image builds. Pipeline details: [infra-spec.md](./infra-spec.md#7-cicd).

## 8. Optional modules

Tests of optional modules (logs: [observability-spec.md §7](./observability-spec.md#7-acceptance-criteria); metrics: [monitoring-spec.md §8](./monitoring-spec.md#8-tests-and-acceptance)) apply only once they are enabled.
