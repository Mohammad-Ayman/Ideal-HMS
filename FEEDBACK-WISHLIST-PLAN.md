# Implementation Plan — Feedback / Feature Wishlist

A hotel-owner-submitted feedback / feature-request capability with an admin review queue (`pending → accepted | rejected`). The data is **shared across both realms**: hotel owners write+read their own rows via `/api/clients/feedback`; platform admins read all rows and change status via `/api/platform/feedback`. Both realms hit the **same** `hotel_feedback` Postgres table over the one backend base URL — the realm is distinguished only by path prefix and cookie/secret, exactly as the existing `clients` vs `platform` split.

> **Reality check baked into this plan (verified against the repo, not the task wording):**
> - The backend is **raw parameterized PostgreSQL via `query(sql, params)` — NO Mongoose/ORM.** The "schema" is a `CREATE TABLE` in `src/config/Hotel_Pro.sql`, not a Mongoose model. Status/category enums are **INT columns with a SQL comment**, not Postgres ENUMs.
> - The clients side mirrors **`modules/clients/quotations`** (layered model/controller/service/routes/validation, raw-JSON responses).
> - The platform side mirrors **`modules/platform/hotels`** — a **flatter** layout (`routes.ts`, `controllers/HotelController.ts`, `services/HotelService.ts`, `validation/`, **no model layer** — the service holds the SQL) using the **`ResponseSender.sendResponse({success,statusCode,data})` envelope**, a `PATCH /:id/status` action route, and `PlatformAuthMiddleware.authorize("platform_*.action")`. There is **no** platform `quotations` module to copy; `platform/hotels` is the only correct template.
> - **Permissions are NOT inserted ad-hoc into the DB.** Both realms' permission strings are seeded from **static arrays in `src/modules/platform/seed.ts`** (`hotelPermissions[]` already contains `hotel_quotations.*`; `platformPermissions[]` contains `platform_hotels.*`), provisioned by `GET /platform_seed`. Adding the new permissions to those arrays is the real registration point (see §3.8 / §4.7). `clients/seed.ts` seeds **no** roles/permissions.
> - **The owner Navbar does NOT permission-filter its links** (`getPermissableLinks` is only used in `dashboard/setting/[settingType]/layout.tsx`). A `protectedRoutes` entry gates the **page** (via `proxy.ts`) but does **not** hide the nav link (see §5.10 / §5.11).

---

## 1. Overview & data model

### 1.1 Where the "model" lives

There is **no shared model file** — **both** realms get their own OOP model class over the same table:
- **Clients realm** gets an OOP model class `Feedback` (mirrors `Quotation.model.ts`) at
  `hotelsPro--server/src/modules/clients/feedback/models/Feedback.model.ts` (**tenant-scoped**).
- **Platform realm also gets an OOP model class** `Feedback` at
  `hotelsPro--server/src/modules/platform/feedback/models/Feedback.model.ts` — **cross-tenant** (no `hotel_id` scope; `LEFT JOIN hotel_hotels` for `hotel_name`), holding the raw SQL and the atomic terminal guard. The platform **service** is the policy layer (throws `ApiError`); the **controller** returns **raw JSON** (no `ResponseSender` envelope).

The single shared artifact is the **table DDL** in `src/config/Hotel_Pro.sql` (§1.2) — which does **not yet exist** and must be added; it is the source of truth both realms read.

### 1.2 Table DDL — migration `0003_hotel_feedback.sql` (+ `Hotel_Pro.sql` for fresh installs)

**All DB schema changes ship as additive migration files** under `src/config/migrations/`, applied by **`npm run db:migrate`** (`runMigrations.ts` — transactional, ledger-tracked in `schema_migrations`, re-run is a no-op, and it **never drops a table**, so it is safe on a populated/prod DB). The next number after the existing `0001_hotel_quotations.sql` / `0002_grant_quotation_permissions.sql` is **`0003`**. Do **not** use the destructive `POST /migrate` (it drops every `public` table then re-runs `Hotel_Pro.sql`) against a populated DB.

Create `src/config/migrations/0003_hotel_feedback.sql`, mirroring `0001`'s style — `CREATE TABLE IF NOT EXISTS` with **inline** FKs (so a re-run, with the table already present, skips them — no duplicate-constraint error). **Not** separate `ALTER TABLE … ADD FOREIGN KEY` statements, which are not re-run-safe:

```sql
-- Additive migration: adds hotel_feedback to an EXISTING database without
-- dropping anything. Safe to re-run (CREATE TABLE/INDEX IF NOT EXISTS; inline FKs).
-- Applied by `npm run db:migrate`, NOT the destructive POST /migrate.
-- The same table also lives in Hotel_Pro.sql for fresh installs.

CREATE TABLE IF NOT EXISTS "hotel_feedback" (
  "id" SERIAL PRIMARY KEY,
  "hotel_id" INT,
  "branch_id" INT,
  "title" VARCHAR(255) NOT NULL,
  "body" TEXT NOT NULL,
  "category" INT NOT NULL DEFAULT 1,   -- 1=feature_request, 2=bug, 3=improvement, 4=other
  "priority" INT NOT NULL DEFAULT 2,   -- 1=low, 2=medium, 3=high
  "status" INT NOT NULL DEFAULT 1,     -- 1=pending, 2=accepted, 3=rejected
  "admin_note" TEXT,                   -- admin's reason on accept/reject (visible to owner)
  "reviewed_by" INT,                   -- platform_users.id of the acting admin (no cross-realm FK, matches 0001 restraint)
  "reviewed_at" TIMESTAMP,             -- when status left 'pending'
  "created_at" TIMESTAMP DEFAULT (CURRENT_TIMESTAMP),
  "created_by" INT,                    -- hotel_users.id of the submitting owner
  "deleted_at" TIMESTAMP,
  FOREIGN KEY ("hotel_id")  REFERENCES "hotel_hotels"   ("id") ON DELETE CASCADE,
  FOREIGN KEY ("branch_id") REFERENCES "hotel_branches" ("id") ON DELETE CASCADE
);

-- Speeds up the admin review queue (status filter + newest-first ordering).
CREATE INDEX IF NOT EXISTS "idx_hotel_feedback_review"
  ON "hotel_feedback" ("status", "created_at")
  WHERE "deleted_at" IS NULL;
```

Then **also** add the identical `CREATE TABLE` / `CREATE INDEX` block to `src/config/Hotel_Pro.sql` (next to the `hotel_quotations` block, ~line 209) so a fresh-install `POST /migrate` includes it — the same dual-write `0001` did (`hotel_quotations` lives in both the migration and `Hotel_Pro.sql`).

Conventions honored: `SERIAL` PK, `hotel_id`/`branch_id`, `created_at DEFAULT CURRENT_TIMESTAMP`, `created_by`, `deleted_at` soft-delete, INT-status-with-comment enums (no Postgres ENUM), **inline re-run-safe FKs**.

### 1.3 Enum maps (single source, replicated in TS on both sides)

| Concept | INT | Meaning |
|---|---|---|
| `status` | 1 / 2 / 3 | pending / accepted / rejected |
| `category` | 1 / 2 / 3 / 4 | feature_request / bug / improvement / other |
| `priority` | 1 / 2 / 3 | low / medium / high |

These ints are mapped to badge labels/classes in the UI (same pattern as quotation status `1/2/3 → active/converted/cancelled`). The dictionary holds the human labels; **never hardcode strings**. Keep the enum maps **identical** on both realms — they are the contract that aligns the owner (zod input) and admin (read interface) type files.

---

## 2. Status lifecycle

States: `pending (1)` → `accepted (2)` | `rejected (3)`. **Terminal** once it leaves pending.

| Transition | Actor | Realm | Enforced where |
|---|---|---|---|
| (none) → pending | hotel owner (submitter) | clients | `FeedbackService.create()` forces `status: 1` |
| pending → accepted | platform admin | platform | `PlatformFeedbackService.setStatus(id, 2, adminId, note)` |
| pending → rejected | platform admin | platform | `PlatformFeedbackService.setStatus(id, 3, adminId, note)` |

Rules (live in the **service**, throwing `ApiError`, never the controller):
- Owner may **create** and **read own**. Owner may **not** mutate status or read another hotel's rows — the clients SQL is always scoped `hotel_id = $X AND branch_id = $Y AND deleted_at IS NULL`, and `hotel_id`/`branch_id`/`created_by` are taken from `req.user`, **never** from the request body.
- Admin may **read all**, **accept**, **reject**.
- **Atomic terminal guard (concurrency-safe).** The transition guard lives **inside the model's `Feedback.setStatus()` `UPDATE … WHERE … status = 1 AND deleted_at IS NULL`**, not in a separate read-then-write. If the UPDATE matches 0 rows, `PlatformFeedbackService.setStatus()` distinguishes the two cases with **one** follow-up read (`Feedback.currentStatus()`): a row that doesn't exist (or is soft-deleted) → `ApiError(404, "Feedback not found")`; a row that exists but is no longer pending → `ApiError(409, "Feedback already <accepted|rejected>")`. This closes the two-admins-double-act race the naive read-then-write would allow.
- On a successful transition the platform service stamps `reviewed_by` (admin id from the platform token), `reviewed_at = NOW()`, and the optional `admin_note`. The owner sees the resulting `status` + `admin_note` reflected back via the clients read endpoints (same table) — see §7 for the cache-staleness caveat.

---

## 3. Backend — hotel-owner API (`modules/clients`)

New module folder: `hotelsPro--server/src/modules/clients/feedback/`, mirroring `quotations/` exactly.

```
src/modules/clients/feedback/
  models/Feedback.model.ts
  controllers/Feedback.controller.ts
  services/Feedback.service.ts
  routes/feedback.routes.ts
  validation/Feedback.validation.ts
  index.ts            # barrel: export { feedbackRoutes }
```

### 3.1 `models/Feedback.model.ts` (mirror `Quotation.model.ts`)

OOP class, `static tableName = "hotel_feedback"`, raw parameterized SQL, tenancy + soft-delete filters in every statement, optional `client?` param for transactions.

```ts
import query from "../../../../config";

export interface FeedbackAttributes {
  id?: number;
  hotel_id: number;
  branch_id: number;
  title: string;
  body: string;
  category: number;   // 1=feature_request,2=bug,3=improvement,4=other
  priority: number;   // 1=low,2=medium,3=high
  status: number;     // 1=pending,2=accepted,3=rejected
  admin_note?: string | null;
  reviewed_by?: number | null;
  reviewed_at?: Date | null;
  created_at?: Date;
  created_by?: number;
  deleted_at?: Date | null;
}

export class Feedback {
  static tableName = "hotel_feedback";
  // ...assignable fields + constructor copying row...

  static async create(data: Partial<FeedbackAttributes>, client?: any): Promise<Feedback> {
    const executor = client ? client.query.bind(client) : query;
    const sql = `INSERT INTO ${this.tableName}
      (hotel_id, branch_id, title, body, category, priority, status, created_by)
      VALUES ($1,$2,$3,$4,$5,$6,$7,$8) RETURNING *;`;
    const r = await executor(sql, [
      data.hotel_id, data.branch_id, data.title, data.body,
      data.category ?? 1, data.priority ?? 2, data.status ?? 1, data.created_by,
    ]);
    return new Feedback(r.rows[0]);
  }

  static async findAll(hotel_id: number, branch_id: number, q: Record<string, any> = {}): Promise<Feedback[]> {
    const allowedFields = ["status", "category", "priority"];   // allowlist — never interpolate raw keys
    let sql = `SELECT * FROM ${this.tableName} WHERE hotel_id = $1 AND branch_id = $2 AND deleted_at IS NULL`;
    const values: any[] = [hotel_id, branch_id]; let i = 3;
    for (const [k, v] of Object.entries(q)) {
      if (v === undefined || v === null || v === "") continue;
      if (!allowedFields.includes(k)) continue;
      const n = Number(v);                                       // query values arrive as strings
      if (!Number.isInteger(n)) continue;                        // drop ?status=abc -> NaN silently
      sql += ` AND ${k} = $${i}`; values.push(n); i++;
    }
    sql += ` ORDER BY id DESC;`;
    const r = await query(sql, values);
    return r.rows.map((row: any) => new Feedback(row));
  }

  static async findById(id: number, hotel_id: number, branch_id: number): Promise<Feedback | null> {
    const r = await query(
      `SELECT * FROM ${this.tableName} WHERE id=$1 AND hotel_id=$2 AND branch_id=$3 AND deleted_at IS NULL LIMIT 1;`,
      [id, hotel_id, branch_id]
    );
    return r.rows.length ? new Feedback(r.rows[0]) : null;
  }
}
```

> **Query-param coercion (resolved):** the clients `GET /` list has **no `validateReq`** (so `req.query` values are raw strings). `findAll` now coerces each allowlisted filter to an integer with `Number.isInteger` and **silently drops non-numeric values** (e.g. `?status=abc`), so it never feeds `NaN` or an unsanitized string into the parameterized query. Postgres would coerce `'1' → int` anyway, but the explicit guard keeps the contract clear and matches the int-typed columns.

> The clients model is **read-mostly**: owners create + list-own + get-one. **No owner-side update/delete in v1** (a submitted request is immutable to the owner once filed — keeps the lifecycle clean). To add owner edits while still `pending`, add `update`/`softDelete` exactly like `Quotation.model.ts`, guard `status === 1` in the service, and follow §9.

### 3.2 `services/Feedback.service.ts` (mirror `Quotation.service.ts`)

```ts
import ApiError from "../../../shared/errors/ApiError";
import { Feedback, FeedbackAttributes } from "../models/Feedback.model";

type FeedbackInput = Partial<FeedbackAttributes> & { hotel_id: number; branch_id: number; created_by: number };

export class FeedbackService {
  async create(data: FeedbackInput): Promise<Feedback> {
    // server-authoritative defaults; owner can never set status
    return Feedback.create({ ...data, status: 1 });
  }
  async getAll(hotel_id: number, branch_id: number, q: Record<string, any> = {}) {
    return Feedback.findAll(hotel_id, branch_id, q);
  }
  async getById(id: number, hotel_id: number, branch_id: number) {
    const item = await Feedback.findById(id, hotel_id, branch_id);
    if (!item) throw new ApiError(404, "Feedback not found");   // 404 lives in the service, thrown to globalErrorHandler
    return item;
  }
}
```

### 3.3 `controllers/Feedback.controller.ts` (mirror `Quotation.controller.ts`)

Class with injected service, `public X: RequestHandler = catchAsync(...)`, tenancy pulled off `req.user`, **never** trusting client-supplied `hotel_id`/`branch_id`. Raw-JSON responses (quotations style). `getById` lets the service throw the 404 (so the central error envelope is used uniformly).

```ts
import { RequestHandler } from "express";
import catchAsync from "../../../../utils/catchAsync";
import { FeedbackService } from "../services/Feedback.service";

export class FeedbackController {
  private readonly service: FeedbackService;
  constructor(service?: FeedbackService) { this.service = service || new FeedbackService(); }

  public create: RequestHandler = catchAsync(async (req, res) => {
    const { id, hotel_id, branch_id } = req.user as { id: number; hotel_id: number; branch_id: number };
    const feedback = await this.service.create({ ...req.body, hotel_id, branch_id, created_by: id });
    res.status(201).json(feedback);
  });

  public getAll: RequestHandler = catchAsync(async (req, res) => {
    const { hotel_id, branch_id } = req.user as { hotel_id: number; branch_id: number };
    res.json(await this.service.getAll(hotel_id, branch_id, req.query || {}));
  });

  public getById: RequestHandler = catchAsync(async (req, res) => {
    const { hotel_id, branch_id } = req.user as { hotel_id: number; branch_id: number };
    const item = await this.service.getById(parseInt(req.params.id, 10), hotel_id, branch_id);
    res.status(200).json(item);
  });
}
```

### 3.4 `validation/Feedback.validation.ts` (Zod, wrapped under `body`/`params`, `.strip()`)

```ts
import { z } from "zod";

const createBody = z.object({
  title: z.string().min(1, { message: "Title is required" }).max(255),
  body: z.string().min(1, { message: "Description is required" }),
  category: z.number().int().min(1).max(4).optional(),   // defaults to 1 in service
  priority: z.number().int().min(1).max(3).optional(),   // defaults to 2 in service
}).strip();

const create = z.object({ body: createBody });
const getOne = z.object({
  params: z.object({
    id: z.string({ required_error: "Feedback ID is required" }).regex(/^\d+$/, { message: "Feedback ID must be a positive integer" }),
  }),
});

export const FeedbackValidation = { create, getOne };
```

> **`getOne` param refine (resolved):** the `id` param is validated as `^\d+$` so a non-numeric id is rejected at the validation layer with a 400 instead of falling through to `parseInt → NaN → findById([NaN,…]) → 404`. This matches the controller's `parseInt(req.params.id, 10)`.

> **`validateReq` wipes unlisted keys (documented):** `ValidateReq.ts` reassigns `req.query = parsedData.query` and `req.params = parsedData.params` **unconditionally**. The `getOne` schema declares only `{ params }`, so applying it sets `req.query = undefined`. `getById` does not read `req.query`, so this is harmless **today** — but any future query filter added to the `GET /:id` route must also add a `query` block to this schema, or it will be silently wiped. The list route (`GET /`) intentionally has **no** `validateReq` (see §3.1) so its `req.query` survives for `findAll`'s allowlist.

### 3.5 `routes/feedback.routes.ts` + barrel

```ts
// routes/feedback.routes.ts
import { Router } from "express";
import { FeedbackService } from "../services/Feedback.service";
import { FeedbackController } from "../controllers/Feedback.controller";
import validateReq from "../../../../utils/ValidateReq";
import { FeedbackValidation } from "../validation/Feedback.validation";
import { authorize } from "../../middlewares/authorize";

const router = Router();
const controller = new FeedbackController(new FeedbackService());

router.post("/", authorize("hotel_feedback.create"), validateReq(FeedbackValidation.create), controller.create.bind(controller));
router.get("/", authorize("hotel_feedback.read"), controller.getAll.bind(controller));            // no validateReq -> req.query preserved
router.get("/:id", authorize("hotel_feedback.read"), validateReq(FeedbackValidation.getOne), controller.getById.bind(controller));

export { router as feedbackRoutes };
```

```ts
// index.ts (barrel)
import { feedbackRoutes } from "./routes/feedback.routes";
export { feedbackRoutes };
```

> `authorize()` runs **before** `validateReq` and returns its own ad-hoc `{ message }` 401/403 (not the global envelope) — so the permission strings in §3.8 must already be seeded and role-mapped or every request 403s.

### 3.6 Endpoints (clients realm) → base path `/api/clients/feedback`

- `POST /api/clients/feedback` — submit
- `GET  /api/clients/feedback` — list-own (supports `?status=&category=&priority=`, all optional, ignored if non-numeric)
- `GET  /api/clients/feedback/:id` — get-one (own)

### 3.7 The ONE required router-registration edit — `src/modules/clients/index.ts`

Add the import alongside the others (next to `quotations`, ~line 18) and mount with auth (~line 33):

```ts
import { quotationRoutes } from "./quotations";
import { feedbackRoutes } from "./feedback";        // ADD

// ...
router.use("/quotations", AuthMiddleware.ensureAuthenticated, quotationRoutes);
router.use("/feedback",   AuthMiddleware.ensureAuthenticated, feedbackRoutes);   // ADD
```

> `feedbackRoutes` is a **named** export — do **not** add a default export to the module's `index.ts`.

### 3.8 Permissions — seed array (fresh installs) + grant migration `0004` (existing DBs)

`hotel_feedback.create` / `hotel_feedback.read` are checked at runtime by `authorize()` against `hotel_permissions` + `hotel_role_permissions`. Ship them in **two** places, mirroring exactly how `hotel_quotations.*` was shipped (`0002_grant_quotation_permissions.sql`):

1. **Fresh installs — add to `hotelPermissions[]`** in `src/modules/platform/seed.ts` (next to the `hotel_quotations.*` block) so `GET /platform_seed` provisions the definitions:
   ```ts
   { name: "hotel_feedback.create", description: "Create hotel feedback" },
   { name: "hotel_feedback.read",   description: "Read hotel feedback" },
   ```
2. **Existing DBs — grant migration.** Seeding the *definitions* is not enough: an admin role's permissions are a one-time snapshot taken at hotel registration, and `authorize()` checks `hotel_role_permissions` **live** (no owner bypass), so admin roles created earlier won't have the new perms. They must be **linked** to every hotel's `admin` role. This ships as `src/config/migrations/0004_grant_feedback_permissions.sql` (idempotent: `ON CONFLICT` on the unique name + `NOT EXISTS` guard on the links — `hotel_role_permissions` has no unique key), applied by `npm run db:migrate`. The **hotel half** (the same file also carries the platform half from §4.8):

   ```sql
   -- HOTEL side — definitions (no-op if /platform_seed ran) + link to every hotel's "admin" role.
   INSERT INTO "hotel_permissions" ("name", "description") VALUES
     ('hotel_feedback.create', 'Create hotel feedback'),
     ('hotel_feedback.read',   'Read hotel feedback')
   ON CONFLICT ("name") DO NOTHING;

   INSERT INTO "hotel_role_permissions" ("role_id", "permission_id")
   SELECT r."id", p."id"
   FROM "hotel_roles" r
   CROSS JOIN "hotel_permissions" p
   WHERE r."name" = 'admin'
     AND r."deleted_at" IS NULL
     AND p."deleted_at" IS NULL
     AND p."name" IN ('hotel_feedback.create', 'hotel_feedback.read')
     AND NOT EXISTS (
       SELECT 1 FROM "hotel_role_permissions" rp
       WHERE rp."role_id" = r."id" AND rp."permission_id" = p."id" AND rp."deleted_at" IS NULL
     );
   ```

> Until `npm run db:migrate` has run (linking the perms to the `admin` role), every clients feedback request 403s via `authorize()`.

---

## 4. Backend — admin API (`modules/platform`)

Mirror the **layered, OOP-model** template (`modules/clients/quotations` — which *does* have a real `models/*.model.ts` class — adapted to the platform realm's depth and auth), but respond with **raw JSON** (`res.status().json(entity)`), **not** the `ResponseSender` envelope. So this module is: **model → service → controller → routes + validation**, raw-JSON out, cross-tenant (sees all hotels), with the `PATCH /:id/status` action and the atomic terminal guard inside the model's UPDATE.

> **Hard prerequisite — run the `0003` migration first (§1.2).** This module's SQL is dead-on-arrival without the table: `hotel_feedback` does **not** exist in the schema yet. Migration `src/config/migrations/0003_hotel_feedback.sql` (§1.2) creates it with *exactly* the columns this model reads — `id, hotel_id, branch_id, title, body, category, priority, status, admin_note, reviewed_by, reviewed_at, created_at, created_by, deleted_at` — applied by **`npm run db:migrate`**; until then every query below 500s at runtime. The `LEFT JOIN hotel_hotels h ON h.id = f.hotel_id … h.name AS hotel_name` is safe: `hotel_hotels.name VARCHAR(255) NOT NULL` is confirmed present at `Hotel_Pro.sql:99`. See the **DDL contract** block at the end of §4.8 for the exact column list this layer depends on.

> **Why raw-JSON and not `ResponseSender`:** the user explicitly wants the layered style with `res.json(entity)`. The `platform/users`/`hotels`/`dashboard` controllers use `ResponseSender`; we deliberately diverge. The only platform controllers that already return raw JSON are `auth` and the middlewares — so raw-JSON is an *established* platform pattern, just not the CRUD-module default. Doing this removes the previous clients-vs-platform envelope asymmetry (both realms now raw-JSON) — see §7.

> **Why a model layer here when `platform/users` has none:** `platform/users/models/User.ts` is a **0-byte decoy** (verified `wc -c` = 0; its SQL lives in the service). The user wants a genuine model layer, so we copy the **`clients/quotations` OOP-model shape** (a real `Feedback` class holding the SQL) and place it at the platform depth. Import specifiers below are the *platform* `feedback` depths (same folder depth as `platform/users`), **not** the `quotations` ones.

### 4.1 Folder structure

`platform/feedback` sits at the **same depth as `platform/users`** (`src/modules/platform/feedback/`). `platform/users` uses a **flat `routes.ts`** (not a `routes/` subdir), so we do too — but unlike `users` we add a real `models/` folder:

```
hotelsPro--server/src/modules/platform/feedback/
  models/Feedback.model.ts           # OOP class, raw SQL, cross-tenant, atomic guard
  services/Feedback.service.ts       # business logic, throws ApiError(404/409)
  controllers/Feedback.controller.ts # RAW-JSON responses (res.status().json(...))
  validation/Feedback.validation.ts  # Zod
  routes.ts                          # default-exported Router (matches platform/users)
```

> No `index.ts` barrel: `platform/users` has an **empty** `index.ts` (verified 0 bytes) and `platform/index.ts` imports its routes file **directly** (`import platformUserRoutes from "./users/routes"`). We mirror that — `platform/index.ts` imports `./feedback/routes` directly (§4.7). Do **not** add a barrel.

### 4.2 `models/Feedback.model.ts` (OOP class — cross-tenant, NOT hotel-scoped)

Mirrors the `clients/quotations/models/Quotation.model.ts` *shape* (a `Feedback` class wrapping raw parameterized SQL, `static tableName`, `import query from "../../../../config"`), but is **not** tenant-scoped: every method spans all hotels and `LEFT JOIN hotel_hotels` for `hotel_name`. The terminal guard lives **inside** `setStatus`'s `UPDATE … WHERE … status = 1`.

> **Import depth (subfolder file, `…/feedback/models/Feedback.model.ts`):** `query` → **four** `../` (`../../../../config`, default export — verified `src/config/index.ts:3 export default query`); `ApiError` is **not** imported here (the model returns `null`/rows; the **service** throws `ApiError`).

```ts
import query from "../../../../config";

export interface FeedbackAttributes {
  id: number;
  hotel_id: number | null;
  branch_id: number | null;
  title: string;
  body: string;
  category: number;   // 1=feature_request,2=bug,3=improvement,4=other
  priority: number;   // 1=low,2=medium,3=high
  status: number;     // 1=pending,2=accepted,3=rejected
  admin_note: string | null;
  reviewed_by: number | null;
  reviewed_at: Date | null;
  created_at: Date;
  created_by: number | null;
  hotel_name?: string | null;   // from LEFT JOIN hotel_hotels (admin surface only)
}

export interface FeedbackFilters {
  status?: number;
  category?: number;
}

export class Feedback {
  static tableName = "hotel_feedback";

  // Shared WHERE builder — cross-tenant (no hotel_id scope), soft-delete filtered.
  private static buildWhere(filters: FeedbackFilters, startIdx = 1) {
    const where: string[] = ["f.deleted_at IS NULL"];
    const values: any[] = [];
    let i = startIdx;
    if (Number.isInteger(filters.status))   { where.push(`f.status = $${i++}`);   values.push(filters.status); }
    if (Number.isInteger(filters.category)) { where.push(`f.category = $${i++}`); values.push(filters.category); }
    return { whereSql: `WHERE ${where.join(" AND ")}`, values, nextIdx: i };
  }

  // Page of rows, newest first, with hotel_name. Returns bare rows[].
  static async findAll(filters: FeedbackFilters, page: number, limit: number): Promise<FeedbackAttributes[]> {
    const { whereSql, values, nextIdx } = this.buildWhere(filters);
    const offset = (page - 1) * limit;
    const sql = `
      SELECT f.id, f.hotel_id, f.branch_id, f.title, f.body, f.category, f.priority, f.status,
             f.admin_note, f.reviewed_by, f.reviewed_at, f.created_at, f.created_by,
             h.name AS hotel_name
        FROM ${this.tableName} f
        LEFT JOIN hotel_hotels h ON h.id = f.hotel_id
        ${whereSql}
        ORDER BY f.created_at DESC
        LIMIT $${nextIdx} OFFSET $${nextIdx + 1};`;
    const { rows } = await query(sql, [...values, limit, offset]);
    return rows;
  }

  // Total matching the same filters (for pagination meta).
  static async count(filters: FeedbackFilters): Promise<number> {
    const { whereSql, values } = this.buildWhere(filters);
    const { rows } = await query(`SELECT COUNT(*)::int AS total FROM ${this.tableName} f ${whereSql};`, values);
    return rows[0].total;
  }

  static async findById(id: number): Promise<FeedbackAttributes | null> {
    const { rows } = await query(
      `SELECT f.*, h.name AS hotel_name
         FROM ${this.tableName} f
         LEFT JOIN hotel_hotels h ON h.id = f.hotel_id
        WHERE f.id = $1 AND f.deleted_at IS NULL
        LIMIT 1;`,
      [id]
    );
    return rows.length ? rows[0] : null;
  }

  // ATOMIC terminal transition. The guard (`status = 1`) is IN the UPDATE WHERE,
  // so two concurrent admins can't both flip the same pending row.
  // Returns the updated row on success, or null if nothing matched
  // (missing / soft-deleted / already-terminal — the SERVICE disambiguates).
  // Mirrors the quotations model's UPDATE…RETURNING + 0-matched-rows -> null
  // convention (the repo's Quotation.model expresses it as rowCount===0;
  // rows.length===0 is the exact equivalent for RETURNING-*).
  static async setStatus(
    id: number, status: number, adminId: number, note?: string | null
  ): Promise<FeedbackAttributes | null> {
    const { rows } = await query(
      `UPDATE ${this.tableName}
          SET status = $1, admin_note = $2, reviewed_by = $3, reviewed_at = NOW()
        WHERE id = $4 AND status = 1 AND deleted_at IS NULL
        RETURNING *;`,
      [status, note ?? null, adminId, id]
    );
    return rows.length ? rows[0] : null;
  }

  // One follow-up read so the service can tell 404 (missing) from 409 (terminal).
  static async currentStatus(id: number): Promise<number | null> {
    const { rows } = await query(
      `SELECT status FROM ${this.tableName} WHERE id = $1 AND deleted_at IS NULL LIMIT 1;`,
      [id]
    );
    return rows.length ? rows[0].status : null;
  }
}
```

> The model never throws — `findById`/`setStatus`/`currentStatus` return `null`; the **service** owns all `ApiError`s (matches the quotations layering where the model is data-only and the service is the policy layer).

### 4.3 `services/Feedback.service.ts` (business logic — throws `ApiError`)

> **Import depth (`…/feedback/services/Feedback.service.ts`):** `ApiError` → **three** `../` (`../../../shared/errors/ApiError`, default export — verified `ApiError.ts:11 export default ApiError`); the model is a sibling-of-sibling (`../models/Feedback.model`).

```ts
import ApiError from "../../../shared/errors/ApiError";
import { Feedback, FeedbackAttributes, FeedbackFilters } from "../models/Feedback.model";

export interface FeedbackListResult {
  items: FeedbackAttributes[];
  total: number;
  page: number;
  limit: number;
}

class PlatformFeedbackService {
  // Returns the SHAPE that survives adminGet's `body?.data ?? body` unwrap (§4.6):
  // rows are under `items`, NOT `data`.
  async list(filters: FeedbackFilters, page: number, limit: number): Promise<FeedbackListResult> {
    const [items, total] = await Promise.all([
      Feedback.findAll(filters, page, limit),
      Feedback.count(filters),
    ]);
    return { items, total, page, limit };
  }

  async getById(id: number): Promise<FeedbackAttributes> {
    const item = await Feedback.findById(id);
    if (!item) throw new ApiError(404, "Feedback not found");
    return item;
  }

  async setStatus(id: number, status: number, adminId: number, note?: string | null): Promise<FeedbackAttributes> {
    const updated = await Feedback.setStatus(id, status, adminId, note);
    if (updated) return updated;

    // UPDATE matched 0 rows -> disambiguate with one read.
    const cur = await Feedback.currentStatus(id);
    if (cur === null) throw new ApiError(404, "Feedback not found");
    throw new ApiError(409, `Feedback already ${cur === 2 ? "accepted" : "rejected"}`);
  }
}

export { PlatformFeedbackService };
```

> Named export (`export { PlatformFeedbackService };`) — matches `platform/users`' `export { PlatformUserService };`.

### 4.4 `controllers/Feedback.controller.ts` (RAW-JSON responses)

Class with constructor-injected service, `catchAsync`-wrapped arrow-field methods (same as `UserController`), **but** every response is `res.status(...).json(...)` of the **bare entity / list object** — no `ResponseSender`. Admin id comes off the platform token (`req.user.id`). Query params are coerced with a `NaN` guard and `limit` is clamped.

> **Import depth (`…/feedback/controllers/Feedback.controller.ts`):** `catchAsync` → **four** `../` (`../../../../utils/catchAsync`); the service is a sibling-of-sibling (`../services/Feedback.service`). **No `ResponseSender` import** — that's the whole point of the raw-JSON switch.

```ts
import { Request, Response } from "express";
import catchAsync from "../../../../utils/catchAsync";
import { PlatformFeedbackService } from "../services/Feedback.service";

const toInt = (v: unknown): number | undefined => {
  const n = Number(v);
  return Number.isInteger(n) ? n : undefined;
};

class PlatformFeedbackController {
  private readonly service: PlatformFeedbackService;
  constructor(service: PlatformFeedbackService) { this.service = service; }

  // GET /  -> { items, total, page, limit }   (raw JSON; see §4.6 for the unwrap proof)
  list = catchAsync(async (req: Request, res: Response) => {
    const page  = Math.max(toInt(req.query.page) ?? 1, 1);
    const limit = Math.min(Math.max(toInt(req.query.limit) ?? 20, 1), 100); // clamp 1..100
    const result = await this.service.list(
      { status: toInt(req.query.status), category: toInt(req.query.category) },
      page, limit
    );
    res.status(200).json(result);
  });

  // GET /:id -> raw entity
  get = catchAsync(async (req: Request, res: Response) => {
    const item = await this.service.getById(Number(req.params.id));
    res.status(200).json(item);
  });

  // PATCH /:id/status -> raw updated entity
  setStatus = catchAsync(async (req: Request, res: Response) => {
    const adminId = (req.user as { id: number }).id;
    const { status, admin_note } = req.body; // status: 2|3 — validated in §4.5
    const item = await this.service.setStatus(Number(req.params.id), status, adminId, admin_note);
    res.status(200).json(item);
  });
}

export { PlatformFeedbackController };
```

> `req.user` is populated by `PlatformAuthMiddleware.ensurePlatformAuth` (the `platform-jwt` strategy) at mount time (§4.7); `req.user.id` is the acting admin's `platform_users.id`, persisted as `reviewed_by`. `catchAsync` forwards any thrown `ApiError` to `globalErrorHandler`, which still emits the **error** envelope (`{success:false,message,errorDetails}`) — only the **success** path is raw JSON. That's fine: `adminSend`/`adminGet`'s `extractErrorMessage` already parses the error envelope.

### 4.5 `validation/Feedback.validation.ts` (Zod)

Only `PATCH /:id/status` is `validateReq`-wrapped (it has a body with a strict status literal). `GET /` and `GET /:id` are unwrapped — the controller's `toInt` guards + `Number(req.params.id)` handle coercion (a non-numeric `:id` falls through to the service's 404).

> **Import depth:** none needed here beyond `zod`.

```ts
import { z } from "zod";

const status = z.object({
  params: z.object({
    id: z.string().regex(/^\d+$/, { message: "Feedback ID must be a positive integer" }),
  }),
  body: z.object({
    status: z.union([z.literal(2), z.literal(3)], {
      errorMap: () => ({ message: "status must be 2 (accept) or 3 (reject)" }),
    }),
    admin_note: z.string().max(2000).optional().nullable(),
  }).strip(),
});

export const PlatformFeedbackValidation = { status };
```

> **`validateReq` query-wipe caveat (load-bearing — do not regress):** `ValidateReq.ts` reassigns `req.query`/`req.params` unconditionally from the parsed schema. The `status` schema declares only `{ params, body }` (**no `query` key**), so after `validateReq` runs on the PATCH route, `req.query` becomes `undefined`. This is **safe today** because `setStatus` never reads `req.query` — and it's the *identical* pattern already live in `clients/quotations` `update`/`convert` (also no `query` key). **If you ever add a query-param read to the `setStatus` controller, you MUST first add a `query` passthrough to this schema**, or it will read `undefined`. The list route is **not** `validateReq`-wrapped, so its `req.query` survives for the controller's `toInt` coercion.

### 4.6 `routes.ts` (flat, default-exported — mirrors `platform/users/routes.ts`)

> **Import depths (flat `routes.ts`, one level shallower than the subfolders):** `validateReq` → **three** `../` (`../../../utils/ValidateReq`, capital V, default export — verified `ValidateReq.ts:21`); `PlatformAuthMiddleware` → **two** `../` (`../middlewares/Auth`, named import). Controller/service/validation are local (`./...`). These match `platform/users/routes.ts` one-for-one — they are **not** the `clients/quotations` depths (that routes file sits one level deeper in a `routes/` subdir).

```ts
import { Router } from "express";
import { PlatformFeedbackController } from "./controllers/Feedback.controller";
import { PlatformFeedbackService } from "./services/Feedback.service";
import { PlatformFeedbackValidation } from "./validation/Feedback.validation";
import { PlatformAuthMiddleware } from "../middlewares/Auth";
import validateReq from "../../../utils/ValidateReq";

const router = Router();
const controller = new PlatformFeedbackController(new PlatformFeedbackService());
const { authorize } = PlatformAuthMiddleware;

router.get("/",    authorize("platform_feedback.read"), controller.list);
router.get("/:id", authorize("platform_feedback.read"), controller.get);
router.patch(
  "/:id/status",
  authorize("platform_feedback.update"),
  validateReq(PlatformFeedbackValidation.status),
  controller.setStatus
);

export default router;
```

> Accept and reject are **one** route (`PATCH /:id/status` with `status: 2|3`). The frontend exposes two buttons that both hit it. `authorize(perm)` checks `req.user.permissions.includes(perm)` → 403 otherwise; `ensurePlatformAuth` (added at the mount in §4.7) runs the `platform-jwt` strategy → 401 if unauthenticated. `export default router` — matches `platform/users/routes.ts`.

#### Endpoints (platform realm) → base `/api/platform/feedback`

- `GET   /api/platform/feedback` — list-all (`?status=&category=&page=&limit=`) → `{ items, total, page, limit }`
- `GET   /api/platform/feedback/:id` — get-one → raw entity
- `PATCH /api/platform/feedback/:id/status` — accept `{status:2, admin_note?}` / reject `{status:3, admin_note?}` → raw updated entity

### 4.7 Platform router-registration edit — `src/modules/platform/index.ts`

`platform/index.ts` imports each module's routes file **directly** (default import) and mounts it wrapped in `ensurePlatformAuth` — e.g. `import platformUserRoutes from "./users/routes";` then `router.use("/users", PlatformAuthMiddleware.ensurePlatformAuth, platformUserRoutes);` (verified at `platform/index.ts:5` + the `/users` mount). Mirror that exactly (this is a **default** import — the routes file `export default router`s):

```ts
import platformFeedbackRoutes from "./feedback/routes";          // ADD (default import, next to ./users/routes)

// ...next to the /users, /hotels, /dashboard mounts:
router.use(
  "/feedback",
  PlatformAuthMiddleware.ensurePlatformAuth,
  platformFeedbackRoutes
);                                                               // ADD
```

> Default import, **not** named — matches `platformUserRoutes`. `ensurePlatformAuth` at the mount is what populates `req.user` for the controller's `req.user.id`.

### 4.8 Platform permissions — seed array (fresh installs) + grant migration `0004`

`platformPermissions[]` (verified at `seed.ts:5–17`) is inserted into `platform_permissions` by `seedPlatformAdmin()`, whose **step 3** (`seed.ts:446–450`) auto-links every `platform_%` permission to the `super_admin` role (`WHERE p.name LIKE 'platform_%'`). Two places again:

1. **Fresh installs — add to `platformPermissions[]`** (auto-linked to `super_admin` on `GET /platform_seed`):
   ```ts
   { module: "feedback", name: "platform_feedback.read",   description: "Read hotel feedback" },
   { module: "feedback", name: "platform_feedback.update", description: "Review (accept/reject) hotel feedback" },
   ```
2. **Existing DBs — grant migration.** The **platform half** of `0004_grant_feedback_permissions.sql` (same file as §3.8's hotel half), idempotent, applied by `npm run db:migrate`. Verified columns: `platform_permissions(module, name UNIQUE, description, …)`, `platform_role_permissions(role_id, permission_id, …)` (no unique key → `NOT EXISTS` guard), `platform_roles.name = 'super_admin'`:

   ```sql
   -- PLATFORM side — definitions + link to super_admin.
   INSERT INTO "platform_permissions" ("module", "name", "description") VALUES
     ('feedback', 'platform_feedback.read',   'Read hotel feedback'),
     ('feedback', 'platform_feedback.update', 'Review (accept/reject) hotel feedback')
   ON CONFLICT ("name") DO NOTHING;

   INSERT INTO "platform_role_permissions" ("role_id", "permission_id")
   SELECT r."id", p."id"
   FROM "platform_roles" r
   CROSS JOIN "platform_permissions" p
   WHERE r."name" = 'super_admin'
     AND r."deleted_at" IS NULL
     AND p."deleted_at" IS NULL
     AND p."name" IN ('platform_feedback.read', 'platform_feedback.update')
     AND NOT EXISTS (
       SELECT 1 FROM "platform_role_permissions" rp
       WHERE rp."role_id" = r."id" AND rp."permission_id" = p."id" AND rp."deleted_at" IS NULL
     );
   ```

> `authorize("platform_feedback.*")` checks loaded permissions at runtime — until `npm run db:migrate` links them to `super_admin`, every platform-feedback request 403s. **One command (`npm run db:migrate`) covers both fresh and existing DBs** — no manual prod SQL, no destructive re-migrate.

#### DDL contract (what this layer requires from migration `0003` / §1.2)

Migration `0003` (§1.2) owns creating `hotel_feedback`, but for traceability the **exact** columns this `§4` layer reads/writes are:

| Column | Used by | Notes |
|---|---|---|
| `id` (PK) | all | `SERIAL PRIMARY KEY` |
| `hotel_id` | `findAll`/`findById` JOIN + select | nullable; FK → `hotel_hotels(id)` |
| `branch_id` | select | nullable; FK → `hotel_branches(id)` |
| `title`, `body` | select | text |
| `category` | filter + select | int (1–4) |
| `priority` | select | int (1–3) |
| `status` | **guard** + filter + select | int (1=pending,2=accepted,3=rejected); the `WHERE status = 1` guard depends on this |
| `admin_note` | `setStatus` write + select | nullable text |
| `reviewed_by` | `setStatus` write | nullable; FK → `platform_users(id)` (the acting admin) |
| `reviewed_at` | `setStatus` write (`NOW()`) | nullable timestamp |
| `created_at` | `ORDER BY` + select | timestamp default `CURRENT_TIMESTAMP` |
| `created_by` | select | nullable (originating hotel user) |
| `deleted_at` | **every** `WHERE … deleted_at IS NULL` | nullable timestamp (soft delete) |

Plus the JOIN dependency `hotel_hotels.name` (confirmed present, `Hotel_Pro.sql:99`). If `0003`'s DDL omits any of these, the corresponding query 500s — keep migration `0003` (§1.2) and this layer in lockstep.

### Chosen RAW-JSON LIST SHAPE (explicit) — and why it survives `adminGet` with no helper edit

**Shape returned by `GET /api/platform/feedback`:**

```jsonc
{ "items": [ /* feedback rows, each with hotel_name */ ], "total": 123, "page": 1, "limit": 20 }
```

`adminGet`/`adminSend` unwrap with **`body?.data ?? body`** (nullish coalescing). Because this object has **no top-level `data` key**, `body?.data` is `undefined`, so the unwrap **falls through to `body`** and the action receives the **whole object** — `items` *and* `total`/`page`/`limit` all survive. This is exactly the safe shape from the frontend audit.

- **Why not a bare array** (like `getTenants`): a bare array would also survive the unwrap, but then `total`/`page`/`limit` have nowhere to live, killing server-side pagination meta. We keep pagination, so we use the object form.
- **Why the rows key must NOT be `data`:** the trap shape `{ data:[...], total }` would make `body?.data` truthy, so `adminGet` returns **only the inner array** and silently **drops `total`**. Naming it `items` avoids that. The single-entity `GET /:id` and `PATCH …/status` return **bare entities** (no `data` key, no `items` key) → `body?.data ?? body` → the entity, unchanged. **No edit to `_shared.ts` is required.**

---

## 5. Frontend — hotel-owner UI

New route `/dashboard/feedback`, mirroring `quotations/` end-to-end. Both `page.tsx` and `layout.tsx` are mandatory.

### 5.1 Files

```
ideal/src/app/[lang]/(hotelOwner)/dashboard/feedback/page.tsx
ideal/src/app/[lang]/(hotelOwner)/dashboard/feedback/layout.tsx
ideal/src/actions/features/feedback/index.ts
ideal/src/types/hotelOwner/Feedback.ts
ideal/src/contexts/HotelOwners/FeedbackContext.tsx
ideal/src/components/hotelOwner/feedback/FeedbackComponent.tsx
ideal/src/components/hotelOwner/feedback/FeedbackForm.tsx
```

### 5.2 `types/hotelOwner/Feedback.ts` (zod + inferred types — mirror `Quotations.ts`)

The read type includes **all** columns the owner list/detail may surface (so the table accessors and any future detail view are fully typed against the clients SQL `SELECT *`).

```ts
import { z } from "zod";

export const Feedback = z.object({
  title: z.string().min(1, { message: "Title is required" }),
  body: z.string().min(1, { message: "Description is required" }),
  category: z.number().int().min(1).max(4).default(1),
  priority: z.number().int().min(1).max(3).default(2),
});

export type TFeedback = z.infer<typeof Feedback>;
export type TFeedbackWithId = TFeedback & {
  id: number;
  status: number;                 // 1=pending,2=accepted,3=rejected
  admin_note?: string | null;
  reviewed_by?: number | null;
  reviewed_at?: string | null;
  created_at: string;
};
```

### 5.3 `config/endpoints.ts` — add the clients block (~line 107, near `quotations`)

```ts
feedback: {
  create: `${clients}/feedback`,
  getAll: `${clients}/feedback`,
  get: (id: number) => `${clients}/feedback/${id}`,
},
```

### 5.4 `actions/features/feedback/index.ts` (`"use server"`, ActionResult style)

`getAllFeedback` returns `TFeedbackWithId[] | null` with `next: { tags: ["feedback"] }`; `createFeedback` returns `ActionResult<TFeedbackWithId>`. Cookie `accessToken` + `Authorization: Bearer`. Unwrap `raw?.data ?? raw` defensively (the clients endpoint returns a raw array/entity).

```ts
"use server";
import { cookies } from "next/headers";
import endpoints from "@/config/endpoints";
import { extractErrorMessage, type ActionResult } from "@/actions/_shared";
import type { TFeedback, TFeedbackWithId } from "@/types/hotelOwner/Feedback";

export async function getAllFeedback(): Promise<TFeedbackWithId[] | null> {
  try {
    const token = (await cookies()).get("accessToken")?.value;
    if (!token) throw new Error("Access token not found");
    const res = await fetch(endpoints.feedback.getAll, {
      headers: { Authorization: `Bearer ${token}` },
      next: { tags: ["feedback"] },
    });
    if (!res.ok) throw new Error(await extractErrorMessage(res));
    const raw = await res.json();
    return raw?.data ?? raw;
  } catch (e) { console.error(e); return null; }
}

export async function createFeedback(input: TFeedback): Promise<ActionResult<TFeedbackWithId>> {
  try {
    const token = (await cookies()).get("accessToken")?.value;
    if (!token) return { ok: false, error: "Access token not found" };
    const res = await fetch(endpoints.feedback.create, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
      body: JSON.stringify(input),
    });
    if (!res.ok) return { ok: false, error: await extractErrorMessage(res) };
    const raw = await res.json();
    return { ok: true, data: raw?.data ?? raw };
  } catch (e) { return { ok: false, error: e instanceof Error ? e.message : "Unknown error" }; }
}
```

### 5.5 `contexts/HotelOwners/FeedbackContext.tsx` (Provider + `useFeedback`)

Seeds list from `initialFeedback`; exposes `addFeedback` (prepend + `clearCache("/dashboard/feedback")`).

```tsx
"use client";
// FeedbackProvider({ initialFeedback, children }) -> useState seeded; addFeedback prepends + clearCache("/dashboard/feedback")
// export function useFeedback() {
//   const c = useContext(Ctx);
//   if (!c) throw new Error("useFeedback must be used within a FeedbackProvider");
//   return c;
// }
```

### 5.6 `layout.tsx` (async server component)

> **BUG FIX (critic):** do **NOT** wrap the list in `removeAttributes(...)`. Verified: `removeAttributes` maps every element to `obj.attributes`, but the clients feedback endpoint returns raw rows (`RETURNING *` / `SELECT *`) with **no `.attributes` wrapper** — so `removeAttributes(feedback || [])` would yield an array of `undefined`. The canonical quotations layout passes `quotations || []` **directly** to its provider (only `apartments?.data` / `apartmentTypes`, which *do* carry `.attributes`, are wrapped). The feedback layout therefore passes `feedback || []` **raw**.

```tsx
import { getAllFeedback } from "@/actions/features/feedback";
import { FeedbackProvider } from "@/contexts/HotelOwners/FeedbackContext";

export default async function FeedbackLayout({ children }: { children: React.ReactNode }) {
  const [feedback] = await Promise.all([getAllFeedback()]);   // null on fetch error -> [] (see §5.8 for error surfacing)
  return (
    <FeedbackProvider initialFeedback={feedback || []}>
      {children}
    </FeedbackProvider>
  );
}
```

### 5.7 `page.tsx` (copy quotations verbatim, swap the import)

```tsx
import { Suspense } from "react";
import FeedbackComponent from "@/components/hotelOwner/feedback/FeedbackComponent";

export default function FeedbackPage() {
  return (
    <div className="w-full p-6">
      <Suspense>
        <FeedbackComponent />
      </Suspense>
    </div>
  );
}
```

### 5.8 `FeedbackComponent.tsx` ("use client" — DataTable + PageHeader + form)

Reads `useFeedback()`, renders `PageHeader` (with `+` to open the `FeedbackForm` modal) and a `DataTable<TFeedbackWithId>` of my-submissions. A **status column** renders a read-only badge from the int (owner can't change it); an `admin_note` column shows the admin's reason once present.

**Empty / error / loading states (fully specified):**
- **Loading:** the list is provided synchronously by the context (seeded in the server `layout.tsx`), so there is **no client-side fetch and no spinner** on first paint — this matches quotations. (If a future refresh action is added, gate it behind a local `isLoading` and render `DataTable`'s built-in empty/loading affordance.)
- **Empty:** `DataTable` `emptyMessage={t.list.empty}` ("You haven't submitted any feedback yet").
- **Error:** `getAllFeedback()` returns `null` on failure and the layout coerces it to `[]`, which would otherwise be **indistinguishable from "no feedback yet."** To surface fetch failure, the layout passes the raw result through and the provider stores a `loadFailed` boolean (`initialFeedback === null`); `FeedbackComponent` renders an inline error banner (`t.list.error`, palette-token styled, not a toast) above the table when `loadFailed` is true, and still renders the empty table beneath it. Concretely: change the provider prop to accept `TFeedbackWithId[] | null`, store `loadFailed = initialFeedback === null`, and seed the list from `initialFeedback ?? []`. (The layout then passes `feedback` directly — still **no `removeAttributes`**.)

```tsx
const { dict } = useDictionary();
const t = dict.dashboard.feedback;

const statusTone = (s: number) => (s === 2 ? "green" : s === 3 ? "red" : "yellow");

const columns: Column<TFeedbackWithId>[] = [
  { key: "id",        header: dict.common.id,    accessor: (r) => `#${r.id}` },
  { key: "title",     header: t.table.title,     accessor: (r) => r.title },
  { key: "category",  header: t.table.category,  accessor: (r) => t.category[r.category] },
  { key: "status",    header: t.table.status,
    accessor: (r) => <StatusBadge label={t.status[r.status]} tone={statusTone(r.status)} /> },
  { key: "adminNote", header: t.table.adminNote, accessor: (r) => r.admin_note || "—" },
];
// {loadFailed && <p className="text-x-red-500 bg-x-red-50 rounded-md px-3 py-2 mb-3">{t.list.error}</p>}
// <DataTable columns={columns} data={feedback} getRowKey={(r) => r.id.toString()} emptyMessage={t.list.empty} />
```

On submit: `createFeedback(values)` → on `ok`: `addFeedback(result.data)` → `clearCache("/dashboard/feedback")` → `toast.success(t.toasts.created)` → close form; else `toast.error(result.error)`.

> `StatusBadge` is the shared default export from `components/hotelOwner/general/statusBadge.tsx`; tone union is `yellow|red|purple|cyan|green|blue|gray`. Map pending→`yellow`, accepted→`green`, rejected→`red`.

> **No owner-side pagination/filtering UI in v1.** The clients `GET /` supports `?status/category/priority` server-side, but the owner table renders the full own-list (a single owner's feedback is small). This is intentional; pagination/filter controls are a Phase-2 item (§9) if volumes grow.

### 5.9 `FeedbackForm.tsx` (react-hook-form + zodResolver(Feedback))

Modal with `title`, `body` (textarea), `category` + `priority` `<select>`s. The resolver cast is required: `resolver: zodResolver(Feedback) as unknown as Resolver<TFeedback>`. Use `inputCls/labelCls/errorCls` consts, `{...register("title")}`, errors as `<p className={errorCls}>{errors.title?.message}</p>`, and submit via `onSubmit={(e) => { e.stopPropagation(); handleSubmit(submit)(e); }}`. `<select>` values must be registered with `valueAsNumber` (or coerced) so `category`/`priority` reach the action as numbers, not strings.

### 5.10 Nav entry — `components/hotelOwner/general/Navbar.tsx`

> **Verified placement:** `navGroups` is a **grouped** array (not flat). `quotations` lives in the `frontOffice` group (line ~84). Add the feedback item **as a sibling of `quotations` in the `frontOffice` group** (it is owner-facing front-office capability):

```ts
// inside the nav.groups.frontOffice items array, after quotations:
{ label: nav.feedback, href: `/${lang}/dashboard/feedback` },
```

> **Nav link is NOT permission-filtered (corrected from the original plan).** The hotel-owner `Navbar.tsx` renders `navGroups` **directly with zero permission filtering** — `getPermissableLinks` is only used in `dashboard/setting/[settingType]/layout.tsx`. So the `protectedRoutes` entry in §5.11 gates the **page** (via `proxy.ts`) but does **not** hide this nav link. An owner lacking `hotel_feedback.read` will still see the link; clicking it redirects per the `redirect` below. This is the accepted behavior (consistent with how the other ungated-but-visible owner links behave); do **not** claim the entry hides the link.

### 5.11 Route-permission gate — `src/lib/constants/auth/protectedRoutes.ts`

Add the entry so `proxy.ts` gates the page. **Verified:** the existing `quotations` entry carries a `redirect`, so the feedback entry must too (convention):

```ts
// next to the Quotations block:
{
  path: "/dashboard/feedback",
  permission: ["hotel_feedback.read"],
  redirect: "dashboard",
},
```

The `path` is the locale-stripped route and must match exactly. `proxy.ts` needs no edit — its matcher already covers `/:locale/dashboard/:path*`.

### 5.12 i18n — `dashboard.feedback` + `dashboard.nav.feedback` in BOTH `en.json` and `ar.json`

Mirror the `dashboard.quotations` shape; keep the two files **structurally identical** (the `Dictionary` type is inferred from `en.json` only — a missing `ar.json` key renders `undefined`, not a type error). Include the `list.error` key required by §5.8.

```json
"feedback": {
  "title": "Feedback & Wishlist",
  "add": "New request",
  "table": { "title": "Title", "category": "Category", "status": "Status", "adminNote": "Admin note", "action": "Action" },
  "category": { "1": "Feature request", "2": "Bug", "3": "Improvement", "4": "Other" },
  "priority": { "1": "Low", "2": "Medium", "3": "High" },
  "status":   { "1": "Pending", "2": "Accepted", "3": "Rejected" },
  "form": { "title": "Title", "body": "Description", "category": "Category", "priority": "Priority", "submit": "Submit" },
  "toasts": { "created": "Your request was submitted" },
  "list": { "empty": "You haven't submitted any feedback yet", "error": "Couldn't load your feedback. Please refresh." }
}
```

and under `dashboard.nav`: `"feedback": "Feedback"` (added to **both** files; this is the `nav.feedback` referenced in §5.10).

---

## 6. Frontend — admin UI

New route `/admin/feedback`, mirroring the **tenants** feature. Inherits the `(dashboard)/layout.tsx` (AdminAuthProvider + AdminNavbar) — no per-route layout.

### 6.1 Files

```
ideal/src/app/[lang]/(platformAdmin)/admin/(dashboard)/feedback/page.tsx
ideal/src/components/platform/feedback/FeedbackComponent.tsx
ideal/src/actions/features/admin/feedback.ts
ideal/src/types/platform/Feedback.ts
```

### 6.2 `types/platform/Feedback.ts`

```ts
export interface TFeedback {
  id: number;
  hotel_id: number;
  branch_id: number;
  hotel_name: string | null;   // LEFT JOIN -> may be null if hotel row missing
  title: string;
  body: string;
  category: number;   // 1..4
  priority: number;   // 1..3
  status: number;     // 1=pending,2=accepted,3=rejected
  admin_note: string | null;
  reviewed_by: number | null;
  reviewed_at: string | null;
  created_at: string;
}
```

### 6.3 `config/endpoints.ts` — add INSIDE the existing `admin:` object (uses `${platform}`)

```ts
feedback: {
  getAll: `${platform}/feedback`,                              // STRING (so `${getAll}?${qs}` concatenation works)
  get:    (id: number | string) => `${platform}/feedback/${id}`,
  status: (id: number | string) => `${platform}/feedback/${id}/status`,
},
```

### 6.4 `actions/features/admin/feedback.ts` (adminGet/adminSend — `adminAccessToken` realm)

> **All net-new** — there is no existing admin feedback action, `TFeedback` type, `TPaginated` type, or `endpoints.admin.feedback` block today. Create them. The list endpoint returns the **raw object** `{ items, total, page, limit }` (§4 list shape), keyed under `items` (**not** `data`), so `adminGet`'s `body?.data ?? body` unwrap passes the whole object through and `total`/`page`/`limit` survive — **no `_shared.ts` edit needed**.

```ts
"use server";
import endpoints from "@/config/endpoints";
import type { ActionResult } from "@/actions/_shared";
import { adminGet, adminSend } from "@/actions/features/admin/_shared";
import type { TFeedback } from "@/types/platform/Feedback";

export interface TPaginated<T> { items: T[]; total: number; page: number; limit: number; }

const getFeedback = async (
  query?: { status?: number; page?: number; limit?: number }
): Promise<TPaginated<TFeedback> | null> => {
  const p = new URLSearchParams();
  if (query?.status) p.set("status", String(query.status));
  if (query?.page)   p.set("page",   String(query.page));
  if (query?.limit)  p.set("limit",  String(query.limit));
  const qs = p.toString();
  return adminGet<TPaginated<TFeedback>>(
    qs ? `${endpoints.admin.feedback.getAll}?${qs}` : endpoints.admin.feedback.getAll
  );
};

const acceptFeedback = async (id: number, admin_note?: string): Promise<ActionResult<TFeedback>> =>
  adminSend<TFeedback>(endpoints.admin.feedback.status(id), "PATCH", { status: 2, admin_note });

const rejectFeedback = async (id: number, admin_note?: string): Promise<ActionResult<TFeedback>> =>
  adminSend<TFeedback>(endpoints.admin.feedback.status(id), "PATCH", { status: 3, admin_note });

export { getFeedback, acceptFeedback, rejectFeedback };
```

> Use `adminGet/adminSend` (`adminAccessToken` cookie + `/api/platform`) — **never** the hotel-owner `authedGet`. `adminGet` is `cache:"no-store"` and unwraps `body?.data ?? body`; the success bodies are now **raw JSON** (no envelope), so the unwrap is a **pass-through** — `getFeedback` receives the `{items,total,page,limit}` object verbatim and accept/reject receive the raw updated entity. Read `total`/`page`/`limit` directly off the result (`result.total`, …) — top-level siblings of `items`, **not** buried in any `ResponseSender.meta`. The numeric `status: 2|3` payload matches the backend Zod literal (§4.5); **do not** send the old string `"accepted"/"rejected"` shape — the backend rejects it.

### 6.5 `page.tsx` (thin Suspense wrapper, server component)

```tsx
import { Suspense } from "react";
import FeedbackComponent from "@/components/platform/feedback/FeedbackComponent";
export default function AdminFeedbackPage() {
  return (<Suspense><FeedbackComponent /></Suspense>);
}
```

### 6.6 `FeedbackComponent.tsx` (copy `TenantsComponent.tsx`; swap delete/status for accept/reject)

`"use client"`, local list state + `useEffect` load via `getFeedback()`, `PageHeader`, an optional status filter `<select>`, and a `DataTable<TFeedback>`.

**Loading / error / empty states (fully specified — admin reads are client-side and no-store):**
- **Loading:** local `loading` boolean set `true` before the `getFeedback()` call and `false` in `finally`. While `loading`, render the shared table-skeleton / a centered spinner (same affordance `TenantsComponent` uses) instead of the table.
- **Error vs empty (disambiguated):** `getFeedback()` returns `TPaginated<TFeedback> | null` — `null` on fetch failure, or `{ items: [], total: 0, … }` on a genuinely empty queue. Store the whole result; set `loadError = result === null` and `setFeedback(result?.items ?? [])`. If `loadError`, render an inline error banner (`t.errors.load`, palette-token styled) with a "Retry" button that re-invokes `reload()`. If `result.items.length === 0`, render the `DataTable` with `emptyMessage={t.empty}`. Never collapse `null` into `[]` — they mean different things.
- **Pagination wirable:** `total`/`page`/`limit` are top-level on the action result. The v1 `DataTable` has no pager (render a single page), but a pager can read `result.total`/`result.page`/`result.limit` directly — no envelope drilling.
- **Reload after mutation:** mutations don't auto-revalidate (admin GET is no-store), so each successful accept/reject calls a local `reload()` (re-runs `getFeedback(filter)`) then a sonner toast.

Columns:

```tsx
const t = dict.admin.feedback;
const tone = (s: number) => (s === 2 ? "green" : s === 3 ? "red" : "yellow");

const columns: Column<TFeedback>[] = [
  { key: "hotel",  header: t.columns.hotel,  accessor: (r) => r.hotel_name ?? "—" },
  { key: "title",  header: t.columns.title,  accessor: (r) => r.title },
  { key: "category", header: t.columns.category, accessor: (r) => t.categories[r.category] },
  { key: "status", header: t.columns.status, accessor: (r) => <StatusBadge label={t.statuses[r.status]} tone={tone(r.status)} /> },
  { key: "actions", header: t.columns.actions, align: "end", accessor: (r) => (
      <div className="flex items-center justify-end gap-2">
        <button disabled={r.status !== 1} onClick={() => openAccept(r)} className="text-x-green-600 hover:bg-x-green-50 disabled:opacity-40 rounded p-1"><Check className="w-4 h-4" /></button>
        <button disabled={r.status !== 1} onClick={() => openReject(r)} className="text-x-red-500 hover:bg-x-red-50 disabled:opacity-40 rounded p-1"><X className="w-4 h-4" /></button>
      </div>
  ) },
];
```

**Note-textarea state hygiene (critic — avoid stale note leaking between dialogs).** Use a **single shared `note` state** but **reset it on every dialog open and on cancel**, not only on the accept-success path. `openAccept`/`openReject` set the pending row **and** clear `note`; `onCancel` clears the pending row **and** `note`; both success paths also clear `note`. This guarantees a note typed into the accept dialog can never leak into a subsequent reject (or vice-versa).

```tsx
const openAccept = (r: TFeedback) => { setNote(""); setPendingAccept(r); };
const openReject = (r: TFeedback) => { setNote(""); setPendingReject(r); };
const cancel    = () => { setNote(""); setPendingAccept(null); setPendingReject(null); };

const handleAccept = async () => {
  if (!pendingAccept) return;
  setActing(true);
  const res = await acceptFeedback(pendingAccept.id, note || undefined);
  setActing(false);
  if (res.ok) { toast.success(t.toasts.accepted); cancel(); await reload(); }
  else toast.error(res.error);
};
const handleReject = async () => {
  if (!pendingReject) return;
  setActing(true);
  const res = await rejectFeedback(pendingReject.id, note || undefined);
  setActing(false);
  if (res.ok) { toast.success(t.toasts.rejected); cancel(); await reload(); }
  else toast.error(res.error);
};
```

Two `ConfirmDialog`s (accept: `confirmTone="primary"`, reject: `confirmTone="danger"`), each rendering the optional admin-note `<textarea>` bound to the shared `note`, `open={pendingAccept !== null}` / `open={pendingReject !== null}`, `isLoading={acting}`, `onCancel={cancel}`. Disable the row buttons when `row.status !== 1` (terminal) — and the backend's atomic guard (§4.1) is the authoritative backstop if a stale row is acted on, returning a 409 that surfaces via `toast.error(res.error)`.

### 6.7 Admin nav — `components/platform/general/AdminNavbar.tsx`

Import a lucide icon and add **one** entry to the inline `links` array (it is rendered in both the desktop and mobile `.map()` — a single entry covers both; do **not** duplicate it):

```ts
import { MessageSquare } from "lucide-react";
// ...
{ label: t.nav.feedback, href: `/${lang}/admin/feedback`, icon: MessageSquare },
```

### 6.8 Admin permission gate — `src/lib/constants/auth/adminProtectedRoutes.ts` (REQUIRED, not optional)

> **Corrected from "optional":** because `platform_feedback.read` is seeded and mapped to `super_admin` **only** (§4.7), omitting this entry would leave the page loadable by any authenticated admin while the API 403s — a confusing half-gated state. For consistency with `tenants`/`users`, **include** the gate:

```ts
{
  path: "/admin/feedback",
  permission: ["platform_feedback.read"],
  redirect: "admin/overview",
},
```

`proxy.ts` reads this automatically; no `proxy.ts` edit needed.

### 6.9 i18n — add `admin.nav.feedback` + the `admin.feedback` block to BOTH files

Include the `categories`, `empty`, and `errors.load` keys referenced by §6.6, and keep `en.json`/`ar.json` structurally identical.

```json
"feedback": {
  "title": "Feedback review",
  "subtitle": "Accept or reject hotel feature requests",
  "columns": { "hotel": "Hotel", "title": "Title", "category": "Category", "status": "Status", "actions": "Actions" },
  "categories": { "1": "Feature request", "2": "Bug", "3": "Improvement", "4": "Other" },
  "statuses": { "1": "Pending", "2": "Accepted", "3": "Rejected" },
  "filters": { "all": "All", "pending": "Pending", "accepted": "Accepted", "rejected": "Rejected" },
  "actions": { "accept": "Accept", "reject": "Reject" },
  "note": { "label": "Admin note (optional)", "placeholder": "Reason shown to the hotel…" },
  "confirmAccept": { "title": "Accept request", "message": "Accept this feature request?" },
  "confirmReject": { "title": "Reject request", "message": "Reject this feature request?" },
  "toasts": { "accepted": "Request accepted", "rejected": "Request rejected" },
  "empty": "No feedback to review",
  "errors": { "load": "Couldn't load feedback. Please retry." }
}
```

and `"feedback": "Feedback"` under `admin.nav` (both files).

---

## 7. Cross-cutting

- **Shared TS types:** there is no shared package — each realm owns its file (`types/hotelOwner/Feedback.ts` for owners, `types/platform/Feedback.ts` for admin). They describe the **same** `hotel_feedback` row but differ in surface (owner is a zod input + read type; admin is a read interface with `hotel_name`). The status/category/priority ints (§1.3) are the contract keeping them aligned — keep the enum maps identical.
- **API client / base-URL:** one base URL (`NEXT_PUBLIC_API_URL || https://server.pioneerhms.com`). Owner actions hit `${clients}` (`/api/clients`) with the **`accessToken`** cookie; admin actions hit `${platform}` (`/api/platform`, under `endpoints.admin.*`) with the **`adminAccessToken`** cookie via `adminGet/adminSend`. Never cross realms. All URLs go in `config/endpoints.ts` — no inline URLs in actions.
- **i18n namespaces:** owner copy under `dashboard.feedback` + `dashboard.nav.feedback`; admin copy under `admin.feedback` + `admin.nav.feedback`. `en.json` and `ar.json` must stay **structurally identical** (the `Dictionary` type is inferred from `en.json` only — a missing `ar.json` key renders `undefined`, not a type error).
- **Theme tokens / no hex:** badges, banners, and buttons use palette tokens only (`text-x-green-600`, `bg-x-red-50`, `text-x-red-500`, `bg-card`, `text-midnight-950`) — never `[#hex]` (project memory). Dark mode is handled by next-themes `.dark`; light stays pixel-identical, only add `.dark`-scoped token overrides if needed.
- **RTL:** logical CSS props throughout (`ps/pe/start/end`, `text-start/text-end`, `align: "end"` columns), never `left/right`. Number formatting via `useDictionary().intl`.
- **Proxy/middleware:** middleware is `src/proxy.ts` (not `middleware.ts`); its matcher already covers `/:locale/dashboard/:path*` and `/:locale/admin/:path*`, so the new routes are gated automatically once the `protectedRoutes` (§5.11) / `adminProtectedRoutes` (§6.8) entries exist. **No `proxy.ts` edit needed.**
- **Owner cache staleness after an admin action (known limitation, documented):** the owner list is cached under the `["feedback"]` tag and is only revalidated by **owner** mutations (`createFeedback` → `clearCache("/dashboard/feedback")`). When an **admin** accepts/rejects, nothing revalidates the owner tag, so the owner sees the new `status` / `admin_note` only on a cold fetch (next full page load / cache expiry), not instantly. This is acceptable for v1; the clean fix is the §9 notification hook (which can also call a server-side tag revalidation for the affected hotel). Do **not** assume the owner sees the transition in real time.
- **Backend response shape (now uniform raw-JSON across both realms):** clients endpoints return **raw JSON** (quotations style) and the platform **feedback** endpoints **also** return **raw JSON** (`res.status().json(entity)`; the list returns `{items,total,page,limit}`) — the `ResponseSender` envelope is **not** used for this module. Frontend actions still unwrap defensively (`raw?.data ?? raw`; `adminGet/adminSend` use `body?.data ?? body`), now a **no-op pass-through** on success (no `data` wrapper exists); the platform list deliberately keys rows under `items` (not `data`) so the sibling `total`/`page`/`limit` survive that unwrap. The **error** envelope (`{success:false, message, errorDetails:[{path,message}]}`) is still centrally produced by `globalErrorHandler` on both realms — you only `throw new ApiError(code, msg)`. `authorize()`/`PlatformAuthMiddleware.authorize()` return their own ad-hoc `{ message }`/`{ error }` on 401/403; `extractErrorMessage` handles all shapes. *(The other platform modules — `users`/`hotels`/`dashboard` — still use `ResponseSender`; this is the **feedback** module aligning with the clients realm, not a repo-wide migration.)*

---

## 8. Build / verify

Per the project verification convention (full lint has ~90 pre-existing errors — scope to tsc + new files; there are no tests):

**Backend (`hotelsPro--server`):**
> **Corrected command:** there is **no `build:dev` script** (verified against `HOTELSPRO-SERVER.md`: only `dev`, `build`, `build:prod`, `start`, `init` exist). Use:
```bash
npm run build          # tsc -> dist/ ; must compile clean for the new feedback modules
# or, faster, no emit:
npx tsc --noEmit
```
Then exercise the endpoints against a running dev server (`npm run dev`, HTTP `:8000`):
1. **Apply migrations (creates the table + grants the perms):** run **`npm run db:migrate`** — applies `0003_hotel_feedback.sql` and `0004_grant_feedback_permissions.sql` (additive, transactional, ledger-tracked; re-run is a no-op). Confirm with `\d hotel_feedback` and that the four permission rows are linked to their roles. **Never** use the destructive `POST /migrate` on a populated DB.
2. Clients: with a valid `accessToken` cookie + a role mapped to `hotel_feedback.*`, `POST` then `GET /api/clients/feedback` and `GET /api/clients/feedback/:id`.
3. Platform: with a valid `adminAccessToken` cookie (`super_admin`), `GET /api/platform/feedback` (expect a **raw `{ items:[…], total, page, limit }` object**, *not* a `{success,data}` envelope), then `PATCH /api/platform/feedback/:id/status` with `{status:2}` (expect the **raw updated entity**) and re-PATCH the same row to confirm the **409 atomic terminal guard**.

(All DB changes ship as additive migration files under `src/config/migrations/` and apply via `npm run db:migrate`; the destructive `POST /migrate` is only for a fresh-install bootstrap and must never run against a populated DB.)

**Frontend (`ideal`):**
```bash
npx tsc --noEmit                                   # typecheck (en.json infers the Dictionary type)
npx eslint src/components/hotelOwner/feedback src/components/platform/feedback \
           src/actions/features/feedback src/actions/features/admin/feedback.ts \
           src/types/hotelOwner/Feedback.ts src/types/platform/Feedback.ts   # scoped lint only
```
Confirm `en.json` and `ar.json` parse and are **key-identical** (a missing `ar.json` key won't fail tsc but breaks the Arabic UI). Spot-check the new keys: `dashboard.feedback.list.error`, `dashboard.nav.feedback`, `admin.feedback.categories`, `admin.feedback.empty`, `admin.feedback.errors.load`, `admin.nav.feedback`.

---

## 9. Optional / Phase-2 enhancements

- **Notifications + owner-cache invalidation:** on `setStatus`, write a `platform_notifications` / hotel-side message row (the `sent_whatsapp_at` column exists but has no sender) so the owner is pinged on accept/reject, and trigger a server-side revalidation of the owner's `["feedback"]` tag for the affected hotel — fixing the §7 staleness in one hook inside `PlatformFeedbackService.setStatus`.
- **Owner edit/withdraw while pending:** add `update`/`softDelete` to the clients `Feedback.model.ts` + service guarded by `status === 1`, add `hotel_feedback.update/delete` to `hotelPermissions[]` (§3.8) + the role mappings, and add `PUT/DELETE` routes — straight quotations parity.
- **Owner pagination/filtering UI:** add `?status/category/priority` filter controls (and a pager if volumes grow) to the owner `FeedbackComponent` — the backend already supports the filters.
- **Voting / upvotes:** new `hotel_feedback_votes (feedback_id, hotel_id, created_by)` with a unique `(feedback_id, created_by)` constraint; surface a `votes_count` subquery in both list SQLs and a vote button on the owner UI to prioritize the admin queue.
- **Comments / threaded discussion:** `hotel_feedback_comments (feedback_id, author_realm, author_id, body, created_at)` to let admin and owner converse beyond the single `admin_note`.
- **Audit trail:** `hotel_feedback_status_history (feedback_id, from_status, to_status, changed_by, note, changed_at)` written inside the `setStatus` transaction (`pool.connect()` + `BEGIN/COMMIT/ROLLBACK`, passing `client` into the insert) — mirrors the quotations transactional `convert()`.
- **Admin pagination UI:** the platform list already returns `{items,total,page,limit}` (top-level, not in an envelope `meta`); wire a pager in the admin `FeedbackComponent.tsx` reading `result.total`/`result.page`/`result.limit` and pass `?page=&limit=` through `getFeedback`.

---

## Appendix — file/edit checklist

**Backend (`hotelsPro--server`):**
- ADD `src/config/migrations/0003_hotel_feedback.sql` — `CREATE TABLE IF NOT EXISTS hotel_feedback` + **inline** FKs + index (§1.2); **also** append the identical block to `src/config/Hotel_Pro.sql` for fresh installs
- ADD `src/modules/clients/feedback/{models,controllers,services,routes,validation}` + `index.ts` (§3.1–3.5)
- EDIT `src/modules/clients/index.ts` — import + mount `feedbackRoutes` (§3.7)
- ADD `src/modules/platform/feedback/{models,controllers,services,validation,routes.ts}` (§4.1–4.6) — **`models/` is new**: a real cross-tenant OOP `Feedback.model.ts`
- EDIT `src/modules/platform/index.ts` — **default** import + mount `platformFeedbackRoutes` (§4.7)
- ADD `src/config/migrations/0004_grant_feedback_permissions.sql` — idempotent grants: `hotel_feedback.*` → every hotel `admin` role (mirrors `0002`) **and** `platform_feedback.*` → `super_admin` (§3.8 / §4.8)
- EDIT `src/modules/platform/seed.ts` (fresh installs) — add `hotel_feedback.create/read` to `hotelPermissions[]` (§3.8) **and** `platform_feedback.read/update` to `platformPermissions[]` (§4.8)
- RUN `npm run db:migrate` — applies `0003` + `0004` (additive, non-destructive); covers both fresh and existing DBs. **Never** `POST /migrate` on a populated DB.

**Frontend (`ideal`):**
- ADD `src/types/hotelOwner/Feedback.ts` (§5.2), `src/types/platform/Feedback.ts` (§6.2)
- EDIT `src/config/endpoints.ts` — clients `feedback` block (§5.3) + `admin.feedback` block (§6.3)
- ADD `src/actions/features/feedback/index.ts` (§5.4), `src/actions/features/admin/feedback.ts` (§6.4)
- ADD `src/contexts/HotelOwners/FeedbackContext.tsx` (§5.5)
- ADD owner route `src/app/[lang]/(hotelOwner)/dashboard/feedback/{layout.tsx,page.tsx}` (§5.6–5.7)
- ADD `src/components/hotelOwner/feedback/{FeedbackComponent.tsx,FeedbackForm.tsx}` (§5.8–5.9)
- ADD admin route `src/app/[lang]/(platformAdmin)/admin/(dashboard)/feedback/page.tsx` (§6.5)
- ADD `src/components/platform/feedback/FeedbackComponent.tsx` (§6.6)
- EDIT `src/components/hotelOwner/general/Navbar.tsx` — add item to `frontOffice` group (§5.10)
- EDIT `src/components/platform/general/AdminNavbar.tsx` — add `links` entry + icon import (§6.7)
- EDIT `src/lib/constants/auth/protectedRoutes.ts` — `/dashboard/feedback` entry w/ `redirect` (§5.11)
- EDIT `src/lib/constants/auth/adminProtectedRoutes.ts` — `/admin/feedback` entry (§6.8)
- EDIT `src/dictionaries/en.json` + `src/dictionaries/ar.json` — `dashboard.feedback`, `dashboard.nav.feedback`, `admin.feedback`, `admin.nav.feedback` (§5.12, §6.9)

**Plan file location:** Per the repo's "Rule 1 — Save all plans," save this plan to `hotelsPro--server/.claude/plans/feedback-feature-wishlist.md` before implementation begins.
