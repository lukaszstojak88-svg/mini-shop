# Backend – specification (catalog-service, orders-service)

Load when working on `services/**`. General rules: [project-spec.md](./project-spec.md#7-rules). Security: [auth-spec.md](./auth-spec.md). Tests: [test-spec.md](./test-spec.md).

## 1. Project layout

Maven multi-module: `services/pom.xml` (parent: `spring-boot-starter-parent`, modules `catalog-service`, `orders-service` – directory name = Maven module = `artifactId`). Each module is a separate Spring Boot application built into a separate image. Java packages stay `pl.minishop.catalog` / `pl.minishop.orders`.

Packages (example for `catalog`; `orders` analogously):

```
pl.minishop.catalog
├── CatalogApplication
├── api            # @RestController, DTO (record), DTO<->domain mapping (MapStruct), ControllerAdvice
├── service        # business logic, @Transactional
├── domain         # JPA entities, domain exceptions
├── repository     # Spring Data JPA
└── config         # SecurityConfig, @ConfigurationProperties
```

`orders` additionally: the `client` package (Feign: `CatalogClient`, configuration, error decoder, fallback).

## 2. Common conventions

- **Error format:** Problem Details (RFC 9457), `Content-Type: application/problem+json`. Fields: `type`, `title`, `status`, `detail`, `instance` + custom `code` (string constant) and optionally `errors` (list of validation errors `{field, message}`).
- **Pagination:** parameters `page` (from 0), `size` (default 12, max 50), `sort`. Response:
  ```json
  { "content": [], "page": 0, "size": 12, "totalElements": 0, "totalPages": 0 }
  ```
- **Money:** `BigDecimal`, scale 2, as a number in JSON. Currency: PLN (default, not modeled).
- **Time:** `Instant` (UTC), ISO-8601 in JSON.
- **Identifiers:** `BIGINT GENERATED ALWAYS AS IDENTITY`.
- **Trace:** a filter assigns/propagates `X-Request-Id`, puts it into the MDC (`requestId`), returns it in the response; `requestId` goes into every log entry.
- **Logs:** stdout, text pattern with `requestId`; levels and prohibitions: [observability-spec.md §0](./observability-spec.md#0-module-status-and-core-minimum). ECS JSON format only after enabling the optional ELK module.
- **Actuator:** only `health` (with `liveness`/`readiness` probes) and `info` are exposed. The `prometheus` endpoint only after enabling the optional module ([monitoring-spec.md](./monitoring-spec.md)). Paths: `/actuator/health/liveness`, `/actuator/health/readiness`.
- **Configuration:** `application.yml` with local default values; overridden by environment variables. The `dev` profile (data seed) only in docker-compose.
- **OpenAPI:** springdoc (`/v3/api-docs`, Swagger UI only in the `dev` profile).

## 3. catalog-service

### 3.1 Data model (`catalog` database)

Table `product`:

| Column | Type | Constraints |
|---|---|---|
| `id` | BIGINT identity | PK |
| `name` | VARCHAR(120) | NOT NULL |
| `description` | VARCHAR(2000) | NULL |
| `price` | NUMERIC(12,2) | NOT NULL, CHECK (price > 0) |
| `stock` | INTEGER | NOT NULL, CHECK (stock >= 0) |
| `version` | BIGINT | NOT NULL DEFAULT 0 (optimistic locking, `@Version`) |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() |

Index: `ix_product_name_lower` on `lower(name)`.

Liquibase: `src/main/resources/db/changelog/db.changelog-master.yaml` includes `changes/001-create-product-table.yaml`, `changes/002-...`. Development seed (8 sample products, including one with `stock = 0` and one with `stock <= 5`) in a changeset with `context: dev`; `spring.liquibase.contexts` set to `dev` only in docker-compose.

### 3.2 Public API

| Method and path | Access | Description |
|---|---|---|
| `GET /api/catalog/products?q=&page=&size=&sort=` | public | List; `q` = name contains (case-insensitive); default sort `name,asc` |
| `GET /api/catalog/products/{id}` | public | 200 / 404 `PRODUCT_NOT_FOUND` |
| `POST /api/catalog/products` | `ADMIN` | 201 + `Location`, returns `ProductResponse` |
| `PUT /api/catalog/products/{id}` | `ADMIN` | Full replacement; requires `version`; mismatch → 409 `PRODUCT_VERSION_CONFLICT` |
| `DELETE /api/catalog/products/{id}` | `ADMIN` | 204 / 404 |

`ProductRequest` (POST/PUT):

```json
{ "name": "Słuchawki BT", "description": "…", "price": 249.00, "stock": 12, "version": 0 }
```
One `ProductRequest` record for POST and PUT; `version` is nullable in the record and required only in PUT (missing → 400 `VALIDATION_ERROR`). Validation: `name` 1–120 characters (after trim, non-empty); `description` ≤ 2000; `price` > 0, ≤ 99,999.99, scale ≤ 2 (the cap keeps the maximum order total – 50 items × 100 × price – within `NUMERIC(12,2)`); `stock` ≥ 0.

`sort` accepts only `name`, `price`, `createdAt` (with `asc`/`desc`); any other property → 400 `VALIDATION_ERROR`.

Note: every stock reservation/release increments `version`, so an admin's PUT may get 409 `PRODUCT_VERSION_CONFLICT` after an order was placed in the meantime – expected behavior (the frontend asks to reload the product).

`ProductResponse`:

```json
{ "id": 1, "name": "…", "description": "…", "price": 249.00, "stock": 12,
  "version": 0, "createdAt": "2026-11-16T10:00:00Z", "updatedAt": "2026-11-16T10:00:00Z" }
```

### 3.3 Internal API (service-to-service only, `SERVICE` role)

Paths under `/internal/catalog/**`; not routed through the Ingress.

`POST /internal/catalog/stock/reservations`

```json
// request
{ "items": [ { "productId": 1, "quantity": 2 }, { "productId": 5, "quantity": 1 } ] }
// 200
{ "items": [ { "productId": 1, "productName": "Słuchawki BT", "unitPrice": 249.00, "quantity": 2 }, … ] }
```

- The operation is **atomic and "all or nothing"** in a single transaction.
- Stock is decreased with a conditional `UPDATE product SET stock = stock - :q, version = version + 1, updated_at = now() WHERE id = :id AND stock >= :q` (0 rows changed = out of stock). Items are processed in ascending `productId` order (deadlock avoidance).
- Errors: 404 `PRODUCT_NOT_FOUND`; 409 `INSUFFICIENT_STOCK` with a list `unavailable: [{productId, requested, available}]`; 400 for duplicate `productId` or `quantity` outside 1–100.

`POST /internal/catalog/stock/releases` – body as above (no response body), increases stock by the given quantities in a single transaction; 204; 404 when the product does not exist.

## 4. orders-service

### 4.1 Data model (`orders` database)

Table `orders`:

| Column | Type | Constraints |
|---|---|---|
| `id` | BIGINT identity | PK |
| `user_id` | VARCHAR(64) | NOT NULL (`sub` claim) |
| `customer_name` | VARCHAR(120) | NOT NULL (`preferred_username` claim) |
| `status` | VARCHAR(20) | NOT NULL, CHECK IN (`NEW`,`CONFIRMED`,`CANCELLED`) |
| `total` | NUMERIC(12,2) | NOT NULL, CHECK (total > 0) |
| `version` | BIGINT | NOT NULL DEFAULT 0 |
| `created_at`, `updated_at` | TIMESTAMPTZ | NOT NULL |

Table `order_item`:

| Column | Type | Constraints |
|---|---|---|
| `id` | BIGINT identity | PK |
| `order_id` | BIGINT | NOT NULL, FK → `orders(id)` ON DELETE CASCADE |
| `product_id` | BIGINT | NOT NULL (no FK – another service) |
| `product_name` | VARCHAR(120) | NOT NULL (snapshot) |
| `unit_price` | NUMERIC(12,2) | NOT NULL, CHECK (unit_price > 0) |
| `quantity` | INTEGER | NOT NULL, CHECK (quantity BETWEEN 1 AND 100) |

Indexes: `orders(user_id, created_at DESC)`, `order_item(order_id)`.

### 4.2 API

| Method and path | Access | Description |
|---|---|---|
| `POST /api/orders` | authenticated | Place an order → 201 + `Location` + `OrderResponse` |
| `GET /api/orders?scope=own\|all&page=&size=` | authenticated; `scope=all` only `ADMIN` | Default `own`; sort `createdAt,desc` |
| `GET /api/orders/{id}` | owner or `ADMIN` | Someone else's order for a regular user → 404 (without revealing existence) |
| `POST /api/orders/{id}/cancel` | owner or `ADMIN` | Only `NEW`; otherwise 409 `INVALID_ORDER_STATE` |
| `POST /api/orders/{id}/confirm` | `ADMIN` | Only `NEW`; otherwise 409 `INVALID_ORDER_STATE` |

`CreateOrderRequest`:
```json
{ "items": [ { "productId": 1, "quantity": 2 } ] }
```
Validation: 1–50 items; `quantity` 1–100; duplicate `productId` → 400 `VALIDATION_ERROR`.

`OrderResponse`:
```json
{ "id": 1042, "status": "NEW", "total": 578.00, "customerName": "user",
  "createdAt": "…",
  "items": [ { "productId": 1, "productName": "Słuchawki BT", "unitPrice": 249.00, "quantity": 1, "lineTotal": 249.00 } ] }
```
`OrderSummary` (list): `id`, `createdAt`, `itemCount` (sum of `quantity` over all items), `total`, `status` (+ `customerName` for `scope=all`).

Status transitions: `NEW → CONFIRMED` (admin), `NEW → CANCELLED` (owner/admin). All others are forbidden.

### 4.3 Order creation flow

1. Validate the request; determine `user_id`/`customer_name` from the JWT.
2. Feign call `reserve(items)` (outside the database transaction) → item snapshot (name, price).
3. Compute `total` on the server side (prices from the catalog response; client-supplied prices do not exist).
4. Save the order and items (`TransactionTemplate`), status `NEW`.
5. If step 4 fails: call `release(items)` (compensation) and raise an error; a compensation failure is logged at `ERROR` level with `requestId` (KO-1).

Cancellation: in a single transaction, load the order with optimistic locking, check the state, set `CANCELLED`, call `release`; a `release` error → rollback and 503 (KO-2).

### 4.4 Feign client (`CatalogClient`)

- `@FeignClient(name = "catalog", url = "${minishop.catalog.base-url}")`; methods `reserve` and `release`.
- Timeouts: connect 2 s, read 3 s (configurable).
- Resilience4j circuit breaker (`catalog`): window of 10 calls, failure threshold 50%, open for 10 s, 3 calls in half-open state. **No retry** (KO-3).
- `ErrorDecoder` maps: 404 → `ProductNotFoundException`, 409 → `InsufficientStockException` (with the list of unavailable items), 5xx/timeout/open circuit → `CatalogUnavailableException` (HTTP 503 `CATALOG_UNAVAILABLE`).
- A Feign interceptor propagates the `X-Request-Id` header from the current request (consistent `requestId` in the logs of both services).
- Call authorization: an interceptor attaches a `client_credentials` token (see [auth-spec.md](./auth-spec.md#5-identity-propagation-between-services)).

## 5. Error codes (`code`)

| Code | HTTP | Where |
|---|---|---|
| `VALIDATION_ERROR` | 400 | both |
| `PRODUCT_NOT_FOUND` | 404 in catalog; **422** in orders (a product from the order does not exist) | catalog, orders |
| `PRODUCT_VERSION_CONFLICT` | 409 | catalog |
| `INSUFFICIENT_STOCK` | 409 | catalog; propagated by orders |
| `ORDER_NOT_FOUND` | 404 | orders |
| `INVALID_ORDER_STATE` | 409 | orders |
| `CATALOG_UNAVAILABLE` | 503 | orders |
| `UNAUTHORIZED` / `FORBIDDEN` | 401 / 403 | both (Problem Details from security handlers) |
| `INTERNAL_ERROR` | 500 | both (no details) |

## 6. Configuration (environment variables)

| Variable | Service | Example (local) |
|---|---|---|
| `SPRING_DATASOURCE_URL` | both | `jdbc:postgresql://postgres:5432/catalog` |
| `SPRING_DATASOURCE_USERNAME` / `_PASSWORD` | both | `catalog_app` / dev-only |
| `SPRING_PROFILES_ACTIVE` | both | `dev` |
| `SPRING_LIQUIBASE_CONTEXTS` | both | `dev` |
| `MINISHOP_SECURITY_ISSUER` | both | `http://localhost:8081/realms/minishop` |
| `MINISHOP_SECURITY_JWK_SET_URI` | both | `http://keycloak:8080/realms/minishop/protocol/openid-connect/certs` |
| `MINISHOP_CATALOG_BASE_URL` | orders | `http://catalog-service:8080` |
| `MINISHOP_CATALOG_TOKEN_URI` | orders | `http://keycloak:8080/realms/minishop/protocol/openid-connect/token` |
| `MINISHOP_CATALOG_CLIENT_SECRET` | orders | dev-only |

Application properties are mapped via `@ConfigurationProperties` (prefix `minishop`). Spring Security and OAuth2 Client details: [auth-spec.md](./auth-spec.md).
