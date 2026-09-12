# ELMA+ Conseil: Tech stack & Architecture Design

## 1. Scope

**MVP = Module 1 (client management) + Module 2 (auth & access).**
Screens: login, dashboard/home, client list, client detail, client create/edit.
Later modules: 3 finance, 4 operations, 5 planning, 6 user management.

---

## 2. Tech stack

**Frontend** — React 19 + TypeScript + Vite · (React Router · TanStack Query · react-hook-form + Zod · Vitest + RTL · Playwright)

**Backend** — Java 21 + Spring Boot 3 + Maven · Spring Web / Data JPA / Security · PostgreSQL 16 · Flyway · JUnit 5 + MockMvc + Testcontainers

**Infra** — Contabo VPS (Germany) · Dokploy (Traefik, auto HTTPS) · Docker · GitHub Actions

**Repo** — monorepo, two Dokploy apps with separate watch paths

---

## 3. Architecture

**Backend**

- **Modular monolith.** One deployable. Packages by business module (`client`, `access`, then `finance`, `operations`…), not by technical layer. Modules communicate through services, never across repositories.
- **Same origin.** One host: `/` → frontend, `/api` → backend via Traefik path routing. No CORS.
- **DTOs are never JPA entities.** Entities stay out of the JSON layer.
- **Soft delete everywhere** (`archived` flag) + audit columns (`created_at/by`, `updated_at/by`).
- **Auth**: JWT in an HttpOnly cookie. Stateless backend.

**Frontend**

---

## 4. Domain model

```
Client (1)
      ── LegalEntity (1..*)
              ── Contact (0..*)
              ── Address (0..*)
              ── Invoice (0..*)   [Module 3]
```

| Entity          | Fields                                                                                                           |
| --------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Client**      | `name`, `status` (PROSPECT \| CLIENT \| INACTIVE), `notes`                                                       |
| **LegalEntity** | `client_id`, `name`, `type` (STANDALONE \| SUBSIDIARY \| BRANCH), `is_principal`, `siren`, `siret`, `vat_number` |
| **Address**     | `legal_entity_id`, `line1`, `line2`, `postal_code`, `city`, `country`, `type` (HEAD_OFFICE \| BILLING \| SITE)   |
| **Contact**     | `legal_entity_id`, `first_name`, `last_name`, `job_title`, `email`, `phone`                                      |
| **User**        | `email`, `password_hash`, `first_name`, `last_name`, `role`, `active`, `last_login_at`                           |

Rules:

- Every client has **at least one** legal entity; the legal entity is the invoiced party.
- Exactly **one principal entity** per client.
- Only `name` is required on an entity at creation; SIREN/SIRET/VAT stay optional until Module 3.

---

## 5. Feature flows

Format: Load · Fields / Layout · Actions · Edge cases.

### 5.1 Authentication

**Load**

- Browse to application link → render login. (If authenticated → render app).

**Fields**

- Email, password.

**Actions**

- `POST /api/auth/login` → backend validates email, verifies password, sets JWT cookie, returns user → redirect to intended route or dashboard.
- Authenticated requests: nothing attached, browser sends the cookie.
- `POST /api/auth/logout` → clears cookie, frontend clears the whole cache.
- Any 401 → clear cache, redirect to login.

**Edge cases**

1. No user: wrong password / unknown email / deactivated → all return the same 401.
2. Nested link while logged out → capture target route, restore after login.
3. Brute force → throttle or lock after N failed attempts per email.
4. First admin seeded by Flyway (password shared with user for tests).
5. Revocation lag: a deactivated user stays valid until expiry.
6. Two tabs share the cookie; the second finds out on its next request.

### 5.2 Client list

**Load**

- Route `/clients`. Request `GET /api/clients` — full list in one request. Lean DTO: `id`, `name`, `status`, principal entity name, city.
- Cached under frontend `['clients']`. Search/filter/sort run **in the browser**.

**Layout**

- Table, search box, status filter, "New client" button.

**Actions**

- Search / filter / sort → in memory, no request.
- Row click → `/clients/:id`.

**Edge cases**

1. Empty state on day one: display "No clients yet" message.
2. Skeleton on first load, retry on error.
3. Accent- and case-insensitive search ("Dupré" matches "dupre").
4. Any modification must invalidate `['clients']`.

### 5.3 Create client

**Load**

- Route `/clients/new` (not a modal: deep-linkable, survives reload).

**Fields**

- Client: `name` (required), `status` (defaults PROSPECT), `notes`.
- Initial legal entity: `name` (required), `type` (defaults STANDALONE).
- No "set as principal" checkbox — the first entity is principal by definition.

**Actions**

- `POST /api/clients` with the entity nested → backend creates both (client, legal entity) **in one transaction**, forces `is_principal = true`.
- Invalidate `['clients']`, seed `['client', id]`, navigate to `/clients/:id`.

**Edge cases**

1. Double submit → disable button.
2. Duplicate client name → warn with a link to the existing one, don't block.
3. Nested validation errors map to the right input (`legalEntity.name`), not a generic toast.
4. Unsaved-changes prompt on navigate away.
5. Trim and case-fold names before duplicate detection.

## 6. Caching & persistence

**Cookie** — the JWT only. HttpOnly, Secure, SameSite=Lax, 8h. Invisible to frontend code.

**localStorage** — UI preferences only: last status filter, table sort, login email. No business data, no auth state.

**In-memory**

| Key              | staleTime | Notes                                                 |
| ---------------- | --------- | ----------------------------------------------------- |
| `['me']`         | Infinity  | Set on login, refetched on refresh, cleared on logout |
| `['clients']`    | ~2 min    | Refetch on window focus                               |
| `['client', id]` | ~2 min    | Invalidated by any modification on that client.       |

**Always fresh from the backend**: authorisation. The cached role drives what the UI _shows_; it never decides what is _allowed_. The backend re-checks every request.

**Edge cases**

1. Logout must clear the entire cache, or the next user sees the previous list before the 401.
2. Two tabs diverge: maybe window-focus refetch.
3. Back button after archiving → invalidate on archive.
4. A saved filter pointing at deleted data must fail soft.

---

## 7. Testing

**Backend unit** (no Spring context)

- SIREN/SIRET Luhn validation.
- Email normalisation, duplication, non existing entities.

**Backend integration** (Testcontainers + real Postgres)

- Flyway migrations run clean on an empty database.
- Unique constraints fire: one principal per client, unique SIRET.
- Create client + entity rolls back fully when the entity is invalid.

**Backend API** (MockMvc + Spring Security)

- 401 without a cookie, 200 with one, on every endpoint.
- Login: success sets the cookie; wrong password / unknown email / deactivated: all return the same 401.
- Validation errors return field-level messages, not a 500.

**Frontend unit** (Vitest)

- Search / filter / sort?

**Frontend component** (RTL)

- Client list: loading, empty, error, populated.
- Create form: validation, double-submit disabled, server errors mapped to the right field.
- 401 triggers redirect to login.

**E2E** (Playwright) — three only, they are the expensive ones to maintain:

1. Login → dashboard.
2. Create client with entity → appears in list → opens on detail.
3. Bad password → error, no redirect.

**CI** — GitHub Actions on every push: backend tests, frontend tests, both builds. E2E on main only.

**Deliberately not automated in the MVP**: visual regression, load testing, exhaustive E2E.

---

## 8. Open decisions

1. **Frontend architecture and design**.
2. **Frontend testing**.
3. **Styling & component library** — mentor's call (Tailwind, shadcn/ui, other, scss ...).
