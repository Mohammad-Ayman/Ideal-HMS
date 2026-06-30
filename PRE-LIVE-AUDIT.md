# Pioneer HMS / Ideal — Pre-Live Audit: Security, Reliability & Performance

> Read-only multi-agent audit of the app (Next.js 16 frontend `ideal/` + Express/PostgreSQL backend `hotelsPro--server/`).
> 94 findings raised → **73 confirmed** against real code with quoted evidence; the highest-impact items were re-verified by hand.
> Tracked in Jira under the **Ideal Hotels** project, epic **"Pre Live"**.

## 1. Executive Summary

The application is functionally rich but carries **critical, actively-exploitable security gaps** that warrant remediation before go-live. The single largest risk is a publicly **unauthenticated `POST /migrate`** that drops every table in the production database for all tenants. Close behind, **multi-tenant isolation is not reliably enforced**: several entire modules (`users`, `hotelFinanceInvoices`, and ~7 `config/*` lookups) lack `authorize()` and tenant scoping, allowing any authenticated user to dump all hotels' user records (including bcrypt password hashes), and to read/modify/delete other tenants' financial and config data via sequential-id IDOR. The intended ownership middleware (`ensureCorrectHotel/Branch/User`) is dead code, so the only tenant boundary is each query remembering to filter by `hotel_id` — which these modules do not. Reliability is undermined by **non-atomic multi-table writes on the core booking and payment paths** (orphaned reservations, ledger drift, double-booking races) and a complete absence of tests, timeouts, graceful shutdown, and observability. Performance is dominated by a **schema with exactly one secondary index** (every tenant list is a sequential scan) and **unbounded, un-paginated list endpoints**. The good news: the codebase already contains correct patterns for nearly every fix (the `reservations`/`finances`/`employees`/`departments` modules for RBAC+validation+tenant scoping, `pool.connect()` transactions, `catchAsync`, `ApiError`, `next/dynamic`, `Promise.all`), so most remediation is replicating existing conventions rather than inventing them.

---

## 2. Critical (Fix Immediately)

### C1. Unauthenticated destructive `POST /migrate` wipes the entire production DB
- **Where:** `hotelsPro--server/src/app.ts:67-103` (DROP loop at `:84`)
- **Impact:** A single anonymous POST drops every public table CASCADE (including the `schema_migrations` ledger) for all tenants — total, irreversible, multi-tenant data loss. The "transaction" uses shared-pool `BEGIN`/`COMMIT` rather than a dedicated client, so it isn't even atomic.
- **Fix:** Remove `/migrate` from the runtime app; schema changes go exclusively through `src/config/runMigrations.ts` (`npm run db:migrate`). If retained, gate behind `ensureAuthenticated` + a platform-admin `authorize()` + `NODE_ENV` guard with a dedicated `pool.connect()` client. **Effort: S**

### C2. `/users` module: no authorize, no tenant scoping → full cross-tenant user dump incl. bcrypt hashes
- **Where:** `src/modules/clients/index.ts:24`, `users/routes.ts`, `users/services/UserService.ts:60`
- **Impact:** Mounted with `ensureAuthenticated` only. `getAll()` = `SELECT * FROM hotel_users WHERE deleted_at IS NULL` — every hotel's users incl. the `password` bcrypt hash. `getById/update/softDelete` take `+req.params.id` with no tenant filter → any authenticated user reads/edits/deletes any other hotel's users (softDelete CASCADE-deletes another hotel if the target is an OWNER).
- **Fix:** Add `authorize('hotel_users.*')` per route (mirror `employee.routes.ts`); source `hotel_id`/`branch_id` from `req.user`; add `AND hotel_id=$ AND branch_id=$` to every query (mirror `IndividualClientService`); replace `SELECT *` with a projection excluding `password`. **Effort: M**

### C3. `UserService.update` mass-assignment + no tenant filter → privilege escalation / account takeover
- **Where:** `users/services/UserService.ts:90-105`, `routes.ts:20`, `controllers/UserController.ts:44`
- **Impact:** `PUT /users/:id` has no `validateReq`/`authorize`. The UPDATE is built from arbitrary `req.body` keys with WHERE `id = $` only. Any authenticated user can set `role`, `password`, or `hotel_id`/`branch_id` on any account → full takeover. Keys are interpolated unparameterized (latent SQL surface).
- **Fix:** `validateReq` with a `.strip()` zod schema whitelisting only safe self-service fields; `authorize('hotel_users.update')`; add tenant `WHERE` (mirror `BaseClientModel.update` + `Client.validation.ts`). **Effort: M**

### C4. `hotelFinanceInvoices` get/update/delete scoped by id only → financial-record IDOR
- **Where:** `hotelFinanceInvoices/{routes,controllers,services,models}`
- **Impact:** Zero `authorize`/`validateReq` on item routes; `findById/update/delete` filter `WHERE id = $1` only. Any authenticated user can read, tamper (incl. `total_amount`, reassign `hotel_id`), or soft-delete any other hotel's invoices via the serial id. The sibling `finances` module is correctly hardened.
- **Fix:** Thread `req.user.hotel_id` into the model WHERE clauses (mirror the module's own `list()`); add `authorize()` + a `validateReq` whitelist. **Effort: M**

---

## 3. Security Improvements

### 3.1 Auth / RBAC / multi-tenant isolation
- **Config lookup modules IDOR (HIGH)** — `config/paymentMethod`, `reservationStatus`, `reservationSource`, `reservationType`, `idType`, `serviceStatus`, `invoiceType` trust body `hotel_id` on create and scope get/update/delete by id only, no `authorize`. `reservationStatus/serviceStatus/invoiceType` `getAll` contain `WHERE (hotel_id=$1 OR 1=1)` tautologies leaking all tenants. `departments` is correct — use as template.
- **Role assignment not hotel-checked (HIGH)** — `HotelUserRoleModel.create/createMany` insert `(user_id, role_id)` with no check that `role_id` belongs to the caller's hotel → a user with `hotel_employees.create` can assign another hotel's admin role.
- **Dead/broken ownership middleware (MEDIUM)** — `ensureCorrectHotel/Branch/User` (`Auth.ts:42,60,78`) are never mounted, read tenant ids from caller-controlled `req.params/body`, and would throw if chained. Root cause of the IDOR findings.
- **Default employee password `'12345678'` + no reset flow (HIGH)** — `Employee.service.ts:19`; no forced first-login change, no forgot/reset endpoint, `logout` is a no-op stub so tokens can't be revoked.
- **JWT algorithm not pinned (LOW)** — no `algorithms` option (`JwtStrategy.ts:9`, `TokenUtils.ts`, `app.ts:132`). Cheap defense-in-depth: `algorithms:['HS256']`.
- **No env/secret boot validation; dead `/auth/verify` route (LOW)** — secrets read with `!`; `/auth/verify` reads an `accessToken` cookie the backend never sets.

### 3.2 Attack surface
- **Unauthenticated `/seed/:id` and `/platform_seed` (MEDIUM)** — `app.ts:106,112`; idempotent but a resource-abuse/data-pollution vector.
- **Public `/auth/register` with no validation (MEDIUM)** — `auth/routes.ts:13`, no `validateReq`; allows unlimited tenant self-provisioning.
- **No rate limiting (HIGH for login/register)** — no `express-rate-limit`; `POST /auth/login` does bare `bcrypt.compare` with no lockout and `password.min(1)`.

### 3.3 Error handling / information disclosure
- **`globalErrorHandler` leaks raw pg messages, masks ApiError (MEDIUM)** — `globalErrorHandler.ts:39-57` returns raw `DatabaseError.message`; ApiError message hardcoded to `"An Custom error occurred."`; nothing logged server-side. Auth controllers bypass it via local try/catch.
- **CORS/helmet/morgan hardening (LOW)** — localhost origins in a credentialed CORS allowlist not gated on `NODE_ENV`; `morgan('dev')` unconditional in prod.

### 3.4 Frontend security
- **Access-token JWT in JS-readable cookie (HIGH)** — `Signin.tsx:65` writes the JWT via `document.cookie` (non-httpOnly) and `console.log`s it; `proxy.ts:106-107` leaves `httpOnly`/`secure` commented out. XSS-exfiltratable for the full token lifetime.
- **Server actions never re-check authorization (MEDIUM)** — no mutating action calls a server-side permission guard; defense-in-depth gap.
- **Proxy/Signin log JWT payload, refreshed tokens, full responses (MEDIUM/LOW)** — `proxy.ts:70,101`, `permissions.ts:6-7`, `Signin.tsx:59,64`, `finances/index.ts:61`.
- **Operational secrets in server `.env` (MEDIUM)** — plaintext production SSH root password + ops runbook in `.env` (gitignored, not committed). Rotate; move runbook out; add committed `.env.example` + boot-time validation.
- **Dead Redux `baseApi.ts` reading token from localStorage (LOW)** — `actions/features/api/baseApi.ts`, unreferenced, imports an undeclared `@reduxjs/toolkit` dependency. Delete.
- **Open-redirect `next` param (LOW)** — guard is correct today; centralize into a shared util.

---

## 4. Reliability Improvements

### 4.1 Transactions / atomicity
- **Booking POST writes 3 tables with no transaction (HIGH)** — `HotelReservation.service.ts:29-83`: `create` + `incrementReservationCount` + `updateStatus` on the autocommit pool; a mid-sequence failure orphans the reservation. Wrap in `pool.connect()` BEGIN/COMMIT (same file's `checkIn` is the template).
- **`changeReservationApartment` refund issued outside its transaction (HIGH)** — `:686-706`: `PaymentFinanceModel.create` called without the `client` arg → refund commits even when the apartment change rolls back.
- **Payment/receipt `isRent` flow non-atomic + unlocked read-modify-write (HIGH)** — `PaymentFinance.service.ts:13-57`, `ReceiptFinance.service.ts:15-53`: concurrent payments lose updates; use `pool.connect()`+BEGIN + `SELECT ... FOR UPDATE` on the balance.
- **`resetPermissions` N inserts without a transaction** — `HotelRole.service.ts:118-130`.

### 4.2 Concurrency / races
- **Apartment double-booking race (HIGH)** — `HotelReservation.model.ts:134-173`: `checkAvailability` is a plain `SELECT 1 ... LIMIT 1` with no `FOR UPDATE`/constraint; two concurrent overlapping bookings both pass. Use a transaction + `SELECT ... FOR UPDATE`, or a Postgres `EXCLUDE` constraint (`btree_gist`, `daterange &&`).

### 4.3 Error handling
- **Auth endpoints swallow errors, lose status codes (MEDIUM)** — `AuthController.ts:12-61`: register→500 w/ leaked message, login→401 for all errors, none call `next(err)`. Wrap in `catchAsync`; throw `ApiError`. Fix missing `return` at `:52` (double-send).
- **Dead trailing error handler / inverted 404 order (LOW)** — `app.ts:144-155`.
- **Frontend: no error.tsx/loading.tsx boundaries (MEDIUM)** — zero boundaries in `src/app`; unguarded `removeAttributes(...)` in `reservations/layout.tsx:76-92` can blank the dashboard segment.
- **Finances/services actions discard the error (MEDIUM)** — `finances/index.ts`, `services/index.ts` `catch { return null }`; migrate to the `ActionResult<T>` + `extractErrorMessage` pattern.
- **Frontend dead/loose handling (LOW)** — dead `TokenExpiredError` branch in `auth.ts` (jose never throws it); unguarded `JSON.parse(localStorage)` in `NavigationContext.tsx:26`; pervasive `any` casts.

### 4.4 Observability
- **No structured logging / error tracking / APM (MEDIUM)** — only `morgan('dev')` + 88 ad-hoc `console.*`; `globalErrorHandler` logs nothing. Add `pino` + Sentry; log inside `globalErrorHandler`.

### 4.5 Infra / deploy
- **No graceful shutdown / pool drain / crash handlers (LOW)** — `server.ts` registers no SIGTERM/SIGINT/`pool.end()`/`pool.on('error')`.
- **No health endpoint / single-instance PM2 (MEDIUM)** — `GET /` returns 200 without a DB check; add `GET /health` (`SELECT 1`); move to `ecosystem.config.js` cluster mode.
- **No request/server/DB timeouts + unsafe SSL (MEDIUM)** — `dbConfig.ts:5-12`: no `max`/`idleTimeoutMillis`/`connectionTimeoutMillis`/`statement_timeout`; `ssl:{rejectUnauthorized:false}`. `server.ts` sets no `requestTimeout`/`headersTimeout`.
- **No frontend fetch timeout/retry (MEDIUM)** — no `AbortSignal`/timeout/retry on 117 `await fetch` calls; add `AbortSignal.timeout(8000)` to the shared helpers + proxy refresh.
- **Destructive schema path + no backups (HIGH)** — `/migrate` re-applying `Hotel_Pro.sql` wipes `schema_migrations`; no `pg_dump`/PITR. Make `npm run db:migrate` the only schema path; add scheduled backups + tested restore.
- **No CI/CD, unpinned Node (MEDIUM)** — no `.github/workflows`, manual rsync deploy, no `engines.node`, `npm install` not `npm ci`.

### 4.6 Tests
- **No automated tests (MEDIUM)** — `npm test` is a failing stub; the financially sensitive atomicity/concurrency flows have zero coverage. Add `vitest` + integration tests against a disposable Postgres.

---

## 5. Performance Improvements

### 5.1 Database — indexes & N+1
- **Only one secondary index in the whole schema (HIGH)** — `Hotel_Pro.sql:239` is the sole index; every tenant query/join/CASCADE is a seq scan. Add (as a numbered additive migration) composite/partial B-tree indexes: `hotel_reservations (hotel_id, branch_id, apartment_id) WHERE deleted_at IS NULL`, availability `(hotel_id, branch_id, check_in_date, check_out_date)`, `hotel_finances (hotel_id, branch_id, finance_type, date)`, partial `(hotel_id, branch_id)` for clients/apartments, FK columns, RBAC join columns.
- **Role permission assignment = N serial INSERTs (MEDIUM)** — `HotelRole.service.ts:30,66,125`; use multi-row INSERT / `unnest($::int[])`.
- **Dashboard recent-activity triple-scans reservations (LOW)** — `Dashboard.model.ts:95`.

### 5.2 Backend
- **`authorize()` runs a 5-table join on every protected request (MEDIUM)** — `authorize.ts:19` → `AuthorizationService.ts:10`; 85 routes, no cache. Read `permissions` from the verified JWT onto `req.user` instead.
- **Per-request `SELECT *` user re-fetch (MEDIUM)** — `JwtStrategy.ts:16` → `UserService.ts:66` on 100% of authenticated requests, pulling the bcrypt hash. Hydrate `req.user` from JWT claims.
- **Unbounded list endpoints — no pagination (HIGH)** — `HotelReservation.model.ts:301`, `ClientService.ts:132`, `BaseFinance.model.ts:84`, `Frontdesk.model.ts:4`. Add keyset pagination + LIMIT.
- **`SELECT *` over-fetch on lists (LOW)** — project only needed columns.
- **Dashboard fan-out: 4 endpoints each re-running auth (LOW)** — single `/dashboard/summary` via `Promise.all`.

### 5.3 Frontend — bundle & data-fetching
- **Reservations layout data-fetching waterfall (HIGH)** — `reservations/layout.tsx:31-58`: 6 sequential awaits before a 9-promise `Promise.all` = 7 sequential hops. Collapse into one `Promise.all`. *Quick win.*
- **Recharts ships eagerly in the dashboard bundle (MEDIUM)** — `DashboardOverview.tsx:7-8`. Use `next/dynamic`; named imports in `ui/chart.tsx`; add `experimental.optimizePackageImports: ['recharts','lucide-react','date-fns']`.
- **FrontdeskCalendar statically imports 6 feature forms (MEDIUM)** — `FrontdeskCalendar.tsx:33-38`; lazy-load via `next/dynamic`.
- **Suspense/loading + provider memoization (LOW)** — add route-level `loading.tsx`; memoize provider value objects.

---

## 6. Prioritized Roadmap

### P0 — Pre-Live emergency security
- **[S]** Remove/guard `POST /migrate`; route schema changes through `runMigrations.ts` only (C1). *Quick win.*
- **[S]** Remove/guard `/seed/:id` and `/platform_seed` (3.2).
- **[M]** Lock down `/users`: `authorize` + tenant scoping + drop `SELECT *` of password (C2).
- **[M]** Fix `UserService.update`: `.strip()` zod whitelist + `authorize` + tenant WHERE (C3).
- **[M]** Scope `hotelFinanceInvoices` get/update/delete + `authorize` (C4).
- **[L]** Tenant-scope + `authorize` all `config/*` lookups; fix `OR 1=1` tautologies (3.1).
- **[M]** Add `express-rate-limit` to `/auth/login`/`/auth/register`/refresh + lockout (3.2).
- **[S]** Rotate plaintext SSH root password in `.env`; add `.env.example` + boot env validation (3.4). *(Reported handled — verify.)*

### P1 — Hardening (security + reliability)
- **[M]** Validate `role_id` belongs to caller's hotel on employee/role assignment (3.1).
- **[M]** Delete/rewrite `ensureCorrect*` middleware; standardize per-query tenant filtering (3.1).
- **[L]** Random temp employee passwords + email + forced first-login change + forgot-password + token revocation (3.1).
- **[M]** Move access token to backend httpOnly+Secure cookie; stop `document.cookie` write & token logging (3.4).
- **[S]** Convert auth controllers to `catchAsync`/`ApiError`; fix `globalErrorHandler` leak (3.3/4.3). *Quick win.*
- **[S]** Wrap booking `create()` + `changeReservationApartment` refund in transactions (4.1). *Quick win.*
- **[M]** Make payment/receipt `isRent` flow atomic + `SELECT ... FOR UPDATE` (4.1).
- **[M]** Close apartment double-booking race (`EXCLUDE`/`FOR UPDATE`) (4.2).
- **[S]** Pool `max`/timeouts/`statement_timeout` + `server.requestTimeout`/`headersTimeout` + prod `rejectUnauthorized:true` (4.5). *Quick win.*
- **[S]** Graceful shutdown + `pool.on('error')` + crash handlers (4.5).
- **[M]** `GET /health` + `error.tsx`/`loading.tsx`/`global-error.tsx` + frontend fetch timeouts (4.3/4.5).
- **[M]** Scheduled `pg_dump`/PITR + tested restore (4.5).
- **[M]** Migrate finances/services actions to `ActionResult`; surface real errors in UI (4.3).

### P2 — Performance
- **[M]** Add composite/partial/FK/RBAC indexes as a numbered migration (5.1). *Highest-leverage perf fix.*
- **[M]** Keyset pagination on reservations/clients/finances/frontdesk lists (5.2).
- **[S]** Read permissions from JWT on `req.user` (skip per-request join); embed full claims in refresh (5.2).
- **[S]** Batch role-permission INSERTs; transaction on `resetPermissions` (5.1).
- **[S]** Collapse reservations-layout fetch waterfall into one `Promise.all` (5.3). *Quick win.*
- **[M]** Lazy-load recharts + FrontdeskCalendar forms; add `optimizePackageImports` (5.3).
- **[S]** Project explicit columns instead of `SELECT *` on lists (5.2).
- **[M]** Single `/dashboard/summary` endpoint via `Promise.all` (5.2).

### P3 — Polish & resilience
- **[L]** Test harness (`vitest`) + integration tests for reservation/check-in/payment atomicity & double-booking (4.6).
- **[L]** CI (`npm ci` + typecheck/build/lint per repo); pin `engines.node`; checked-in deploy script (4.5).
- **[M]** Structured logging (`pino`) + Sentry + log inside `globalErrorHandler` (4.4).
- **[S]** Pin JWT algorithm; boot-time env validation; delete dead code (`/auth/verify`, `baseApi.ts`, dead error handler, dead `TokenExpiredError` branch) (3.1/3.3/4.3). *Quick win.*
- **[S]** Gate localhost CORS on `NODE_ENV`; conditional morgan; centralize safe-redirect util; guard `JSON.parse` (various LOW).
- **[M]** Memoize provider values; loading skeletons; `safeParse` action responses (5.3/4.3).

---

### Patterns to reuse
RBAC+validation+tenant scoping: `employees`, `finances`, `clients` (`IndividualClientService`), `config/departments`. Transactions: `HotelReservation.service.ts` `checkIn`, `AuthService.ts:25`, `runMigrations.ts`. Batch insert: `hotelFinanceInvoice.service.ts:103-111`. Error envelope: `catchAsync`, `ApiError`, `ResponseSender`, `_shared.ts` `ActionResult`/`extractErrorMessage`. Lazy import: `PdfPreviewDialog.tsx`; parallel fetch: `dashboard/page.tsx`.
