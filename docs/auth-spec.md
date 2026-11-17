# Authentication and authorization – specification (Keycloak, JWT)

Load for: login, roles, endpoint security, the Keycloak realm, HTTP interceptors in Angular, Feign calls. Context: [project-spec.md](./project-spec.md). API: [backend-spec.md](./backend-spec.md). Login UI: [frontend-spec.md](./frontend-spec.md).

## 1. Model

- **Identity provider:** Keycloak, realm `minishop`.
- **Protocol:** OIDC / OAuth2. The Angular application (public client) logs the user in via **Authorization Code + PKCE (S256)**. Backend services are **resource servers** and validate JWTs (signature, `iss`, `exp`).
- **Service-to-service communication (orders → catalog):** `client_credentials`, a separate `orders-service` client with the `SERVICE` role.
- Server-side sessions: none (stateless), CSRF disabled (tokens in the `Authorization` header).

## 2. Realm `minishop`

File: `infra/local/keycloak/realm-minishop.json` (imported at startup in docker-compose). Production version (different URIs, no dev users, secret from outside): `infra/helm/keycloak/` (see [infra-spec.md](./infra-spec.md)).

**Realm roles**

| Role | Meaning |
|---|---|
| `USER` | Regular customer; default role (`default-roles-minishop` contains `USER`) |
| `ADMIN` | Shop administrator; also includes `USER` permissions (composite) |
| `SERVICE` | Technical role for services; assigned only to the `orders-service` service account |

**Clients**

| Client ID | Type | Flow | Notes |
|---|---|---|---|
| `minishop-spa` | public | Authorization Code + PKCE (S256) | Redirect URIs: `http://localhost:4200/*`, `http://localhost:8080/*`, `http://shop.localtest.me/*` (local Kubernetes); Web origins: the same; direct access grants **disabled** |
| `orders-service` | confidential | `client_credentials` (service account) | Secret from `ORDERS_CLIENT_SECRET`; standard flow disabled; the service account has the `SERVICE` role |
| `minishop-smoke` | public | Direct access grant (password) | **Dev realm only**, exclusively for `scripts/smoke.sh`; does not exist in the production realm |

**Realm settings:** self-registration enabled; login with e-mail disabled (login with username); e-mail verification disabled (dev); `accessTokenLifespan` 5 min; SSL required: `none` only for dev (`external` in production).

**Development users (dev-only, marked in the file):**

| Username | Password | Roles |
|---|---|---|
| `admin` | `admin` | `ADMIN` |
| `user` | `user` | `USER` |

## 3. Tokens

- Roles in the `realm_access.roles` claim (Keycloak's default mapper). No additional mappers.
- The user is identified by the `sub` claim (for `orders.user_id`), display name from `preferred_username`.
- Required scopes: `openid profile`.

## 4. Resource server (catalog and orders)

Dependencies: `spring-boot-starter-oauth2-resource-server`.

**Properties (`minishop.security.*`):**

| Property | Meaning |
|---|---|
| `issuer` | Expected value of `iss` (e.g. `http://localhost:8081/realms/minishop`) |
| `jwk-set-uri` | JWKS address reachable from the container (e.g. `http://keycloak:8080/realms/minishop/protocol/openid-connect/certs`) |

**Why two addresses:** `iss` in tokens always points to Keycloak's public address (set by `KC_HOSTNAME`), while a service inside the Docker/Kubernetes network fetches the keys via the internal address. That is why the `JwtDecoder` is built explicitly: `NimbusJwtDecoder.withJwkSetUri(...)` + the validator `JwtValidators.createDefaultWithIssuer(issuer)`. **Do not use `issuer-uri` alone with auto-discovery** – it will fail inside the container.

**Role converter:** a custom `Converter<Jwt, AbstractAuthenticationToken>` mapping `realm_access.roles` → `GrantedAuthority` named `ROLE_<ROLE>`.

**Access rules – catalog (`SecurityFilterChain`):**

| Path | Rule |
|---|---|
| `GET /api/catalog/products/**` | `permitAll` |
| `POST/PUT/DELETE /api/catalog/products/**` | `hasRole('ADMIN')` |
| `/internal/catalog/**` | `hasRole('SERVICE')` |
| `/actuator/health/**`, `/actuator/info` | `permitAll` |
| `/v3/api-docs/**`, `/swagger-ui/**` | `permitAll` only in the `dev` profile |
| everything else | `denyAll` |

**Access rules – orders:**

| Path | Rule |
|---|---|
| `POST /api/orders`, `GET /api/orders`, `GET /api/orders/{id}`, `POST /api/orders/{id}/cancel` | `authenticated` |
| `GET /api/orders?scope=all`, `POST /api/orders/{id}/confirm` | `hasRole('ADMIN')` (checked in the service/`@PreAuthorize`) |
| `/actuator/health/**`, `/actuator/info` | `permitAll` |
| everything else | `denyAll` |

Order ownership (`user_id == sub`, unless `ADMIN`) is enforced in the service layer; a violation → 404 (not 403).

**Error responses:** 401 and 403 as Problem Details (`AuthenticationEntryPoint`, `AccessDeniedHandler`), codes `UNAUTHORIZED` / `FORBIDDEN`.

**Session:** `SessionCreationPolicy.STATELESS`. **CORS:** do not configure (single domain).

**Metrics (optional module):** in both services `/actuator/prometheus` is allowed (`permitAll`) only in the `monitoring` profile and is not routed through the Ingress or nginx; see [monitoring-spec.md](./monitoring-spec.md).

## 5. Identity propagation between services

`orders-service` calls `/internal/catalog/**` with a **service** token, not the user's token.

- Dependency: `spring-boot-starter-oauth2-client`.
- Client registration `catalog-client`: `authorization-grant-type: client_credentials`, `client-id: orders-service`, `client-secret: ${MINISHOP_CATALOG_CLIENT_SECRET}`, `token-uri: ${MINISHOP_CATALOG_TOKEN_URI}`.
- `OAuth2AuthorizedClientManager` based on `AuthorizedClientServiceOAuth2AuthorizedClientManager` (context without an HTTP request); a Feign `RequestInterceptor` obtains the token (cached until expiry) and sets `Authorization: Bearer …`.
- Note: custom `SecurityFilterChain` (stateless, resource server) – do not enable `oauth2Login`.
- Additional defense: `/internal/**` endpoints are not routed through the Ingress/nginx.

## 6. Keycloak locally and in the cluster

| Setting | Local (docker-compose) | AWS (EKS) |
|---|---|---|
| Mode | `start-dev --import-realm` | `start --import-realm` (production mode) |
| Public address (`KC_HOSTNAME`) | `http://localhost:8081` | `https://auth.<domain>` |
| Internal address for services | `http://keycloak:8080` | `http://keycloak.<ns>.svc:8080` |
| Database | Postgres from compose (db `keycloak`) | RDS (db `keycloak`) |
| Secrets | dev values in `.env` | Secrets Manager → External Secrets |

**Local Kubernetes (Docker Desktop):** like docker-compose (`start-dev` mode, dev realm), but `KC_HOSTNAME=http://auth.localtest.me`, internal address `http://keycloak.minishop.svc:8080`, database in `postgres-local`. Details: [k8s-local-spec.md](./k8s-local-spec.md).

Required flags/settings (verify against the chosen Keycloak version): full `KC_HOSTNAME` (URL), `KC_HTTP_ENABLED=true` behind TLS termination, `KC_PROXY_HEADERS=xforwarded` behind the ALB, `KC_HEALTH_ENABLED=true`.

## 7. Frontend (Angular) – summary of requirements

Details in [frontend-spec.md](./frontend-spec.md).
- Library `angular-oauth2-oidc`; configuration (`issuer`, `clientId`, `scope`, `redirectUri`) from `/config.json`.
- Code flow + PKCE, token refresh (silent refresh via refresh token).
- An `HttpInterceptor` adds `Authorization: Bearer` to `/api/**` requests (never to other domains).
- Guards: `authGuard` (logged in), `adminGuard` (`ADMIN` role from `realm_access.roles` in the access token). Guards serve UX only; authorization is enforced by the backend.
- Logout: `logOut()` with a redirect to the home page.

## 8. Security checklist

- [ ] No real secrets in the repo; dev secrets in `.env.example`/the dev realm with an annotation.
- [ ] `/internal/**` not reachable from outside the cluster network.
- [ ] Tokens and passwords do not end up in logs.
- [ ] Validation of `iss`, `exp` (and `aud`, if a mapper is introduced) on the service side.
- [ ] Security tests per [test-spec.md](./test-spec.md#4-security-tests).
