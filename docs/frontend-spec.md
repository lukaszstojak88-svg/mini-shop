# Frontend – specification (Angular)

Load when working on `frontend/**`. General rules: [project-spec.md](./project-spec.md#74-angular--typescript). API: [backend-spec.md](./backend-spec.md). Login: [auth-spec.md](./auth-spec.md). Tests: [test-spec.md](./test-spec.md#5-frontend).

> UI texts stay in Polish (see project-spec, §2 and §7.1). Polish strings quoted below are the actual UI copy – keep them verbatim.

## 1. Technology and conventions

- Angular (latest stable), standalone components, signals, `OnPush`, strict TypeScript, no `any`.
- UI: Angular Material (light theme; dark optionally later). Interface language: Polish. Currency/date format: `pl-PL` (PLN).
- API access: `HttpClient` + typed services; relative paths `/api/catalog/...` and `/api/orders/...`.
- Dev: `ng serve` + `proxy.conf.json` (`/api/catalog` → `http://localhost:8082`, `/api/orders` → `http://localhost:8083`).
- Runtime configuration: a `/config.json` file loaded before the application starts (`APP_INITIALIZER`/`provideAppInitializer`):
  ```json
  { "issuer": "http://localhost:8081/realms/minishop", "clientId": "minishop-spa" }
  ```
  In the container it is generated from environment variables (`envsubst`) when nginx starts.
- ESLint (angular-eslint) + Prettier; scripts `lint`, `test`, `build`.

## 2. Directory structure

```
src/app/
├── core/            # auth (service, guards, interceptor), config, error handling, layout (header)
├── shared/          # reusable UI components, pipes, shared models
├── features/
│   ├── products/    # product list, product card
│   ├── cart/        # cart store, cart panel
│   ├── orders/      # order list, details, placing orders
│   └── admin/       # products (CRUD), orders (admin)
└── app.routes.ts
```

## 3. Routing

| Path | Screen | Access |
|---|---|---|
| `/` | redirect → `/products` | – |
| `/products` | Product list + cart panel | public |
| `/orders` | My orders | `authGuard` |
| `/orders/:id` | Order details | `authGuard` |
| `/admin/products` | Product management (table) | `adminGuard` |
| `/admin/products/new`, `/admin/products/:id` | Product form | `adminGuard` |
| `/admin/orders` | All orders (confirm/cancel) | `adminGuard` |
| `**` | 404 page | – |

Lazy loading per `features/*`. An anonymous user entering a protected path → login → return to the original address.

## 4. GUI

### 4.1 Header (all screens)
Left: shop icon + "Mini shop". Navigation: **Produkty** (Products), **Zamówienia** (Orders; logged-in users only), **Administracja** (Administration; `ADMIN` only). Right: a **Zaloguj** (Log in) button or the username + **Wyloguj** (Log out). The active item is underlined. Cart icon with an item counter.

### 4.2 Products (`/products`) – two-column layout
- **Left column (2/3):** search field "Szukaj produktu" (Search product; debounce 300 ms, parameter `q`), grid of product cards (auto-fit, min. 150 px), pagination (Material paginator, 12 per page).
- **Product card:** image placeholder (substitute icon), name, price (`249,00 zł`), availability indicator, **Dodaj do koszyka** (Add to cart) button.
  - Availability: `stock > 5` → "Dostępne: N" (Available: N; success color); `1–5` → "Dostępne: N" (warning); `0` → "Brak w magazynie" (Out of stock; error), button disabled with the label "Niedostępne" (Unavailable).
  - The quantity in the cart does not exceed `stock` (the button locks once the limit is reached).
- **Right column (1/3):** **Koszyk** (Cart) panel: items (`name × quantity`, item price, +/− and remove buttons), total **Razem** (Total), **Złóż zamówienie** (Place order) button. Empty cart: "Koszyk jest pusty" (The cart is empty) + disabled button. On narrow screens the panel moves below the list.
- States: loading (card skeletons), empty result ("Brak produktów pasujących do wyszukiwania" – No products match the search), error (message + **Spróbuj ponownie** – Try again).

### 4.3 Placing an order
Click **Złóż zamówienie**: anonymous user → login while preserving the cart; logged-in user → `POST /api/orders` (button in loading state, double-click protection). Success → cart emptied, redirect to `/orders/:id`, notification "Zamówienie #N zostało złożone" (Order #N has been placed). Errors:
- 409 `INSUFFICIENT_STOCK` → message with the names of unavailable products, list refreshed, cart preserved with corrected quantities.
- 422 `PRODUCT_NOT_FOUND` → item removed from the cart + message.
- 503 `CATALOG_UNAVAILABLE` → "Sklep jest chwilowo niedostępny. Spróbuj ponownie za chwilę." (The shop is temporarily unavailable. Try again in a moment.)

### 4.4 My orders (`/orders`)
Table: **Nr** (No.), **Data** (Date), **Pozycje** (Items), **Suma** (Total), **Status**, actions. Status as a badge: `NEW` "Nowe" (accent), `CONFIRMED` "Potwierdzone" (success), `CANCELLED` "Anulowane" (error). Actions: **Anuluj** (Cancel; only `NEW`, with confirmation in a dialog) and **Szczegóły** (Details). Pagination, empty state: "Nie masz jeszcze zamówień" (You have no orders yet) + link to products.

### 4.5 Order details (`/orders/:id`)
Header (number, status, date, total), table of items (name, unit price, quantity, value), **Anuluj zamówienie** (Cancel order) action for `NEW`. Non-existent/someone else's order → "Nie znaleziono zamówienia" (Order not found) screen.

### 4.6 Administration
- **Products:** table (id, name, price, stock, edit/delete actions), **Dodaj produkt** (Add product) button; form (name, description, price, stock) with validation matching the backend; deletion with confirmation; version conflict (409 `PRODUCT_VERSION_CONFLICT`) → message "Produkt został zmieniony przez kogoś innego. Wczytaj go ponownie." (The product was changed by someone else. Reload it.) + refresh button.
- **Orders:** table of all orders (with a **Klient** (Customer) column), actions **Potwierdź** (Confirm; `NEW`) and **Anuluj** (Cancel; `NEW`).

### 4.7 Texts and accessibility
- UI texts in constants/text files (no scattered literals in templates where they repeat).
- Every form field has a label and an error message; icon buttons have `aria-label`; visible focus; contrast compliant with WCAG AA.
- API error messages from `ProblemDetail.code` mapped to Polish texts in one place (`core/errors`).

## 5. State and data

- **Cart:** `CartStore` (signals): items `{productId, name, unitPrice, quantity, maxStock}`; `total` as `computed`; persisted in `localStorage` (key `minishop.cart.v1`, resilient to corrupted data). The price in the cart is informational only – the server computes its own.
- **Auth:** `AuthService` wraps `angular-oauth2-oidc`: signals `isAuthenticated`, `username`, `roles`; methods `login()`, `logout()`.
- **HTTP:** `authInterceptor` (Bearer for `/api/**`), `errorInterceptor` (`ProblemDetail` mapping, 401 → login, global snackbar for errors not handled locally).
- **API clients:** `ProductsApi`, `OrdersApi` with types matching [backend-spec.md](./backend-spec.md) (`ProductResponse`, `OrderResponse`, `Page<T>`, `ProblemDetail`).

## 6. Build and container

- Multi-stage image: `node` (build) → `nginx` (serving `dist`). The `nginx.conf` file:
  - `try_files $uri /index.html` (SPA fallback),
  - `location /api/catalog/ { proxy_pass http://catalog-service:8080; }`, `location /api/orders/ { proxy_pass http://orders-service:8080; }` (only in the docker-compose variant; in Kubernetes routing is done by the Ingress – upstream names from environment variables),
  - `location /internal/ { return 404; }` (no routing to `/internal/**`, no SPA fallback),
  - security headers (`X-Content-Type-Options`, `Referrer-Policy`, `Content-Security-Policy` adapted to the Keycloak address),
  - generating `/config.json` from a template (`envsubst`) in the entrypoint.
- Caching: hashed files – long cache; `index.html` and `config.json` – `no-cache`.
