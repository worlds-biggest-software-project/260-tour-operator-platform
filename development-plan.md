# Tour Operator Platform — Phased Development Plan

> Project: 260-tour-operator-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into an implementable, phased build. The schema is based primarily on **Data Model Suggestion 3 (Hybrid Relational + JSONB)** — a relational backbone for universal entities plus typed JSONB for jurisdiction-specific and rapidly-evolving fields — augmented with the **segregated-fund ledger** and **audit-log** patterns from Suggestion 1. This combination keeps the table count manageable for an MVP, avoids migrations for custom/jurisdiction fields, and stays OCTO-native.

---

## Core Requirements Summary

**What it does:** An AI-native, multi-tenant booking, itinerary, and operations platform for multi-day and multi-destination tour operators, activity providers, and destination management companies (DMCs). Combines an OCTO-native booking engine, a multi-day itinerary builder, OTA channel distribution, supplier/guide management, payment/deposit handling with bonding-compliant fund segregation, and AI-driven itinerary generation and real-time disruption adaptation.

**Primary personas:** tour operator founders/ops managers (multi-day packages); activity operators (rafting, cycling, food tours) wanting online booking + OTA distribution; DMCs building FIT/group programmes; travel advisors assembling bespoke luxury itineraries; adventure-travel companies managing guides and logistics.

**Key differentiators (AI-native):** itinerary generation from a client brief in minutes; dynamic disruption adaptation (weather/transport/venue) with pre-negotiated supplier alternatives; ML guide matching (language/specialisation/availability); guest personalisation; supplier-relationship intelligence and scoring.

**Deployment model:** Cloud-hosted SaaS, multi-tenant, with a self-host option (Docker Compose). API-first; an operator dashboard (web SPA) and a guest itinerary web app are first-party clients.

**Integration surface:** OCTO v1 (primary channel/reseller standard); OTA/OpenTravel XML bridge (legacy wholesale); NDC aggregator (Duffel/Verteil) for air-inclusive packages (roadmap); Stripe/Adyen for payments (PCI tokenisation); LLM provider (OpenAI/Anthropic) for AI features; weather/transport data feeds for disruption monitoring; email/SMS providers for communications.

**Standards to implement:** OCTO v1; OpenAPI 3.1 + JSON Schema 2020-12; RFC 7231 HTTP semantics; OAuth 2.0 (RFC 6749/6750) + OpenID Connect; RFC 4122 UUID (v7); ISO 3166-1/-2, ISO 4217, ISO 8601, BCP 47; Schema.org JSON-LD (TouristTrip/TouristAttraction); PCI DSS v4.0 (SAQ A scope via tokenisation); GDPR/CCPA; EU Package Travel Directive 2015/2302; ATOL/ABTA fund segregation; OWASP API Security Top 10.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **TypeScript (Node 22 LTS)** | The product is API-and-frontend-heavy (booking engine, OCTO endpoints, operator dashboard, guest app). One language across backend + both web clients reduces context-switching. The AI features are LLM-orchestration (prompting, tool-calling), not in-house model training, so Python's ML edge is not needed. |
| API framework | **Fastify + `@fastify/swagger`** | High throughput for booking/availability traffic, first-class JSON Schema validation (aligns with JSON Schema 2020-12 used by OCTO and OpenAPI 3.1), and automatic OpenAPI 3.1 generation from route schemas. |
| Runtime validation / types | **Zod + `zod-to-json-schema`** | Single source of truth for request/response shapes; Zod schemas compile to JSON Schema for OpenAPI and to TypeScript types, satisfying the "schema-on-read" validation discipline the hybrid JSONB model requires. |
| Database | **PostgreSQL 16** | The hybrid relational+JSONB model (Suggestion 3) needs both strong relational integrity and JSONB with GIN indexes. Postgres Row-Level Security enforces multi-tenant isolation. `gen_random_uuid()` / UUID v7 for RFC 4122 IDs. |
| ORM / query layer | **Drizzle ORM** | Type-safe SQL-first ORM with first-class JSONB column typing and a lightweight migration tool (`drizzle-kit`). Closer to SQL than Prisma, which matters for the fund-ledger and reporting queries. |
| Migrations | **drizzle-kit** | Generates and applies versioned SQL migrations; required for the relational backbone. JSONB fields evolve without migrations (the hybrid model's core benefit). |
| Task queue / async | **BullMQ on Redis** | Webhooks (OCTO/OTA), channel sync, LLM calls, disruption polling, and email/SMS are async and retryable. BullMQ gives delayed jobs, retries with backoff, and repeatable jobs (cron) for disruption polling and data-retention purges. |
| Cache / locks | **Redis 7** | Availability hold locks, rate limiting, idempotency keys, and BullMQ backing store. |
| AI orchestration | **Vercel AI SDK + provider SDKs (OpenAI / Anthropic)** | Unified streaming, structured output (JSON Schema → typed objects), and tool-calling across providers. Used for itinerary generation, disruption adaptation reasoning, and content generation. Provider abstracted behind an `LlmClient` interface so the model is swappable. |
| Frontend (operator dashboard) | **Next.js 16 (App Router) + React + shadcn/ui + Tailwind** | Server components for fast dashboards, shadcn for accessible primitives, Tailwind for speed. Talks to the Fastify API. |
| Frontend (guest itinerary app) | **Next.js 16 (separate app in monorepo)** | Public, mobile-first, PWA-installable dynamic itinerary viewer that receives real-time disruption updates (SSE/WebSocket). |
| Monorepo | **pnpm workspaces + Turborepo** | Shared `@repo/db`, `@repo/schemas` (Zod/OCTO types), `@repo/llm` packages across API and two web apps; cached builds. |
| Auth | **Lucia-style sessions for operator users + OAuth 2.0 / OIDC** | OIDC for operator SSO and agent-on-behalf-of login; signed magic links for guest itinerary access (no password). OAuth 2.0 client-credentials for outbound channel APIs (Amadeus, GetYourGuide). |
| Payments | **Stripe (PaymentIntents) primary; Adyen adapter behind a `PaymentGateway` interface** | Tokenisation keeps PCI scope at SAQ A. Supports deposits, instalments, and refunds; webhook-driven status. |
| Email / SMS | **Resend (email) + Twilio (SMS) behind a `Notifier` interface** | Pluggable; templated confirmations and disruption alerts. |
| Containerisation | **Docker + docker-compose** | Self-host story (README) and reproducible dev: api, postgres, redis, two web apps, worker. |
| Testing | **Vitest (unit/integration) + Supertest (HTTP) + Playwright (E2E web)** | Vitest is fast and TS-native; Playwright covers dashboard and guest-app flows. |
| Mocking | **`msw` (HTTP), `testcontainers` (real Postgres/Redis in integration tests)** | msw mocks OTA/LLM/payment HTTP; testcontainers gives real Postgres for repository tests. |
| Code quality | **Biome (lint + format) + `tsc --noEmit` (type check)** | Single fast tool for lint/format across the monorepo; strict TypeScript. |
| Package manager | **pnpm** | Fast, disk-efficient, first-class workspace support. |
| Observability | **Pino (structured logs) + OpenTelemetry traces** | Structured logs feed the audit story; OTel for request tracing across api → worker → external APIs. |
| Key libraries | `octo` types (hand-modelled to spec), `xml2js`/`fast-xml-parser` (OTA XML bridge), `ulid`/`uuidv7`, `ical-generator` (itinerary calendar export), `@js-temporal/polyfill` (ISO 8601 durations/timezones), `i18n` for locale-aware documents. | Domain-specific needs from standards.md. |

### Project Structure

```
tour-operator-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml                # postgres, redis, api, worker, dashboard, guest-app
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── db/                           # @repo/db — Drizzle schema, migrations, client, RLS policies
│   │   ├── src/
│   │   │   ├── schema/               # one file per domain (operator, product, booking, ...)
│   │   │   ├── client.ts             # tenant-scoped connection factory
│   │   │   ├── rls.sql               # row-level security policies
│   │   │   └── seed/                 # ISO reference data (countries, currencies, languages)
│   │   ├── drizzle.config.ts
│   │   └── migrations/
│   ├── schemas/                      # @repo/schemas — Zod models, OCTO types, JSON Schema export
│   │   └── src/
│   │       ├── octo/                 # OCTO v1 Product/Option/Availability/Booking shapes
│   │       ├── domain/               # internal domain DTOs
│   │       └── jsonb/                # JSON Schemas for every JSONB column
│   ├── llm/                          # @repo/llm — LlmClient interface, prompt templates, structured-output helpers
│   │   └── src/
│   │       ├── client.ts
│   │       └── prompts/
│   ├── integrations/                 # @repo/integrations — channel adapters, payment, notifier, data feeds
│   │   └── src/
│   │       ├── channels/             # octo/, ota-xml/, base ChannelAdapter
│   │       ├── payments/             # stripe/, adyen/, PaymentGateway
│   │       ├── notify/               # resend/, twilio/, Notifier
│   │       └── feeds/                # weather/, transport/ DisruptionSource
│   └── core/                         # @repo/core — domain services shared by api + worker
│       └── src/
│           ├── booking/
│           ├── availability/
│           ├── itinerary/
│           ├── pricing/
│           ├── funds/
│           ├── compliance/
│           └── audit/
├── apps/
│   ├── api/                          # Fastify app: REST + OCTO endpoints, webhooks, OpenAPI
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/              # auth, rls-context, error-handler, rate-limit, idempotency
│   │       ├── routes/               # /v1/products, /v1/bookings, /octo/*, /webhooks/*
│   │       └── openapi.ts
│   ├── worker/                       # BullMQ workers: channel sync, llm jobs, disruption poll, notifications, purge
│   │   └── src/
│   │       ├── queues.ts
│   │       └── jobs/
│   ├── dashboard/                    # Next.js operator dashboard
│   └── guest-app/                    # Next.js guest itinerary PWA
└── tests/
    ├── fixtures/                     # sample OCTO payloads, OTA XML, LLM responses, briefs
    └── e2e/                          # Playwright specs
```

The structure is grouped by concern, not by phase. Every later phase adds files within these directories without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Multi-Tenancy, Auth

### Purpose
Stand up the monorepo, the PostgreSQL schema backbone with ISO reference data, multi-tenant isolation via Row-Level Security, and authentication. After this phase, an operator can be created, users can log in, and every subsequent feature has a tenant-scoped, audited, type-safe data layer to build on.

### Tasks

#### 1.1 — Monorepo & tooling skeleton

**What:** Initialise the pnpm/Turborepo workspace with the package and app directories, shared TS config, Biome, and Docker Compose for Postgres + Redis.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `tsconfig.base.json`: `strict: true`, `moduleResolution: "bundler"`, `target: "ES2023"`, path aliases `@repo/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test`, each with `dependsOn: ["^build"]`.
- `docker-compose.yml` services: `postgres` (postgres:16, healthcheck `pg_isready`), `redis` (redis:7). Named volumes for data.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `SESSION_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `LLM_PROVIDER`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `RESEND_API_KEY`, `TWILIO_*`, `APP_BASE_URL`, `GUEST_APP_BASE_URL`.

**Testing**:
- `Unit: pnpm install + pnpm -w typecheck` → exits 0 with empty packages.
- `Integration: docker compose up postgres redis` → both healthy; `pg_isready` returns 0.
- `CI smoke: turbo run lint` → passes on empty scaffold.

#### 1.2 — Reference data tables & seed

**What:** Create ISO reference tables (`country`, `subdivision`, `currency`, `language`) and a seed loader.

**Design** (Drizzle schema → SQL):
```sql
CREATE TABLE country (
  code CHAR(2) PRIMARY KEY,            -- ISO 3166-1 alpha-2
  name VARCHAR(100) NOT NULL,
  alpha3 CHAR(3) NOT NULL,
  numeric_code CHAR(3)
);
CREATE TABLE subdivision (
  code VARCHAR(6) PRIMARY KEY,         -- ISO 3166-2 e.g. US-CA
  country_code CHAR(2) NOT NULL REFERENCES country(code),
  name VARCHAR(200) NOT NULL,
  type VARCHAR(50)
);
CREATE TABLE currency (
  code CHAR(3) PRIMARY KEY,            -- ISO 4217
  name VARCHAR(100) NOT NULL,
  symbol VARCHAR(5),
  minor_units SMALLINT NOT NULL DEFAULT 2
);
CREATE TABLE language (
  code VARCHAR(10) PRIMARY KEY,        -- BCP 47
  name VARCHAR(100) NOT NULL
);
```
- Seed loader reads bundled JSON (ISO datasets) and upserts. Idempotent (`ON CONFLICT DO NOTHING`).

**Testing**:
- `Integration (testcontainers): run seed twice → row counts stable; ISO 4217 'USD' minor_units = 2; 'JPY' minor_units = 0.`
- `Unit: loader rejects malformed alpha-2 code → throws with code value in message.`

#### 1.3 — Tenant & identity schema

**What:** Create `operator`, `platform_user`, `operator_membership` with RBAC roles, and `compliance_profile`.

**Design** (from Suggestion 1, with JSONB settings per Suggestion 3):
```sql
CREATE TABLE operator (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) NOT NULL UNIQUE,
  legal_name VARCHAR(255),
  country_code CHAR(2) NOT NULL REFERENCES country(code),
  currency_code CHAR(3) NOT NULL DEFAULT 'USD' REFERENCES currency(code),
  timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
  iata_number VARCHAR(20), atol_number VARCHAR(20), abta_number VARCHAR(20),
  compliance_profile_id UUID REFERENCES compliance_profile(id),
  subscription_tier VARCHAR(20) NOT NULL DEFAULT 'foundation',
  settings JSONB NOT NULL DEFAULT '{}',        -- branding, data_retention_policy, feature flags
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE platform_user (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255),
  full_name VARCHAR(255) NOT NULL,
  auth_provider VARCHAR(20) DEFAULT 'local',   -- local, google, microsoft (OIDC)
  auth_provider_id VARCHAR(255),
  email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE operator_membership (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES platform_user(id) ON DELETE CASCADE,
  role VARCHAR(30) NOT NULL DEFAULT 'agent',   -- owner, admin, agent, guide, viewer
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  UNIQUE(operator_id, user_id)
);
CREATE TABLE compliance_profile (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  jurisdiction_code CHAR(2) NOT NULL REFERENCES country(code),
  requires_bonding BOOLEAN NOT NULL DEFAULT FALSE,
  requires_fund_segregation BOOLEAN NOT NULL DEFAULT FALSE,
  rules JSONB NOT NULL DEFAULT '{}'            -- cancellation/refund templates, pre-booking info
);
```
- Password hashing: `argon2id`. Role hierarchy enforced in an `authorize(role, action)` helper in `@repo/core`.

**Testing**:
- `Unit: argon2 hash/verify round-trip.`
- `Integration: insert operator with bad country_code → FK violation.`
- `Unit: authorize('viewer', 'booking.create') → false; authorize('owner', 'booking.create') → true.`

#### 1.4 — Row-Level Security & tenant context

**What:** Enable RLS on all tenant tables; the API sets the tenant via a per-request Postgres session variable.

**Design**:
- Every tenant table has `operator_id UUID NOT NULL`.
- `rls.sql`: `ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;` plus a policy `USING (operator_id = current_setting('app.operator_id')::uuid)`.
- `@repo/db` exposes `withTenant(operatorId, fn)` that runs `SET LOCAL app.operator_id = $1` inside a transaction before executing `fn`.
- The API uses a non-superuser role so RLS is enforced (superusers bypass RLS).

**Testing**:
- `Integration: two operators each insert a product; querying as operator A under withTenant(A) returns only A's rows.`
- `Integration: attempt to read operator B's booking by id while tenant = A → 0 rows.`
- `Integration (security): bypass attempt without SET LOCAL → query returns 0 rows (policy denies).`

#### 1.5 — Authentication & session API

**What:** Email/password login, OIDC login, session issuance, and a guest magic-link issuer.

**Design** (Fastify routes; RFC 7231 status codes):
- `POST /v1/auth/register` → `{ email, password, fullName, operatorName, countryCode }` → creates operator + owner membership; 201.
- `POST /v1/auth/login` → `{ email, password }` → sets `Set-Cookie` session (httpOnly, Secure, SameSite=Lax); 200 with `{ user, memberships }`.
- `GET /v1/auth/oidc/:provider` and `/callback` → OIDC code flow (OpenID Connect Core).
- `POST /v1/auth/guest-link` (operator-auth) → `{ bookingId }` → returns signed, expiring URL (`HMAC-SHA256`, 14-day TTL) for guest itinerary access; no password.
- Auth plugin populates `req.user`, `req.operatorId`, `req.role`; downstream routes wrap DB calls in `withTenant`.
- OWASP: rate-limit login (5/min/IP), constant-time compare, generic error on bad creds.

**Testing**:
- `Integration (mocked OIDC): valid code → session cookie set, user upserted.`
- `Integration: 6th login attempt within a minute → 429.`
- `Unit: guest-link signature tamper → verification fails.`
- `Integration: expired guest link → 401.`

---

## Phase 2: Product Catalogue & Pricing (OCTO-aligned)

### Purpose
Implement the OCTO object hierarchy (Product → Option → UnitType) plus pricing and sustainability tagging. This is the foundation the booking engine, availability, channels, and AI itinerary generation all read from. After this phase an operator can build a fully-described, priced, SEO-ready product catalogue.

### Tasks

#### 2.1 — Product, Option, UnitType schema

**What:** Relational backbone with JSONB for variable OCTO capability data.

**Design**:
```sql
CREATE TABLE product (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(150),
  description TEXT, short_description VARCHAR(500),
  product_type VARCHAR(30) NOT NULL,          -- day_tour, multi_day, activity, transfer, package
  duration_iso VARCHAR(40),                    -- ISO 8601 duration e.g. PT3H30M / P5D
  duration_days INTEGER,
  country_code CHAR(2) REFERENCES country(code),
  destination VARCHAR(255), meeting_point TEXT,
  latitude DECIMAL(10,7), longitude DECIMAL(10,7),
  min_participants INTEGER DEFAULT 1, max_participants INTEGER,
  difficulty_level VARCHAR(20), minimum_age INTEGER,
  schema_org_type VARCHAR(50) DEFAULT 'TouristTrip',
  octo_data JSONB NOT NULL DEFAULT '{}',       -- OCTO capability fields (content, pickups, pricing flags)
  status VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, active, archived
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_operator ON product(operator_id);
CREATE INDEX idx_product_octo_gin ON product USING GIN (octo_data);

CREATE TABLE product_option (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL, description TEXT,
  duration_iso VARCHAR(40),
  sort_order INTEGER NOT NULL DEFAULT 0,
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
  required_contact_fields VARCHAR(50)[] DEFAULT '{}'  -- OCTO: fullName, emailAddress, phoneNumber
);
CREATE TABLE unit_type (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  option_id UUID NOT NULL REFERENCES product_option(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,                  -- adult, child, infant, senior
  reference VARCHAR(50), min_age INTEGER, max_age INTEGER,
  is_accompanied BOOLEAN DEFAULT FALSE, sort_order INTEGER NOT NULL DEFAULT 0
);
```
- Zod schemas in `@repo/schemas/domain` mirror these; `octo_data` validated against a JSON Schema in `@repo/schemas/jsonb`.

**Testing**:
- `Unit: Zod product schema rejects unknown product_type → ValidationError naming product_type.`
- `Integration: create product with two options each with adult/child units → hierarchy reads back intact.`
- `Integration: GIN query octo_data @> {"capabilities":["pickups"]} returns the product.`

#### 2.2 — Pricing & seasons

**What:** Per-unit pricing with validity windows and seasonal modifiers.

**Design**:
```sql
CREATE TABLE pricing (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  unit_type_id UUID NOT NULL REFERENCES unit_type(id) ON DELETE CASCADE,
  currency_code CHAR(3) NOT NULL REFERENCES currency(code),
  retail_price NUMERIC(12,2) NOT NULL,
  net_price NUMERIC(12,2), commission_pct DECIMAL(5,2),
  tax_included BOOLEAN NOT NULL DEFAULT TRUE, tax_rate DECIMAL(5,4),
  valid_from DATE NOT NULL, valid_to DATE
);
CREATE TABLE pricing_season (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL, start_date DATE NOT NULL, end_date DATE NOT NULL,
  price_modifier DECIMAL(5,2) NOT NULL          -- multiplier, e.g. 1.25
);
```
- `priceFor(unitTypeId, date, currency)` in `@repo/core/pricing`: selects active pricing row for date, applies any overlapping season modifier, rounds to currency `minor_units`.

**Testing**:
- `Unit: priceFor with no active row for date → throws PricingUnavailable.`
- `Unit: base 100.00 + season modifier 1.25 → 125.00.`
- `Unit: JPY price rounds to 0 decimals (minor_units=0).`

#### 2.3 — Product CRUD API + Schema.org JSON-LD

**What:** REST endpoints for catalogue management and a public JSON-LD emitter for SEO.

**Design**:
- `POST /v1/products`, `GET /v1/products`, `GET /v1/products/:id`, `PATCH /v1/products/:id`, `POST /v1/products/:id/publish` (draft→active), `DELETE` (→archived, soft).
- Nested writes: option and unit creation under `POST /v1/products/:id/options`.
- `GET /v1/products/:id/jsonld` (public) → `application/ld+json` using `schema_org_type` (TouristTrip / TouristAttraction), name, description, offers (lowest price), geo.
- All response/request schemas registered with `@fastify/swagger` → appear in OpenAPI 3.1.

**Testing**:
- `Integration: POST then GET product → 201 then 200 with same payload.`
- `Integration: publish empty product (no options) → 422 with reason.`
- `Unit: JSON-LD output validates against schema.org TouristTrip required fields.`
- `Contract: generated OpenAPI includes /v1/products with 200/201/422 responses.`

---

## Phase 3: Availability, Inventory & the Booking Engine

### Purpose
The heart of the product. Implement availability slots/rules, capacity holds, and the full booking lifecycle (create → confirm → cancel) with idempotency and concurrency safety. After this phase the platform can take real bookings against real inventory.

### Tasks

#### 3.1 — Availability schema & generation

**What:** Availability slots plus recurring rules that materialise slots.

**Design**:
```sql
CREATE TABLE availability (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  product_id UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  option_id UUID NOT NULL REFERENCES product_option(id),
  local_date DATE NOT NULL, start_time TIME, end_time TIME,
  total_capacity INTEGER NOT NULL,
  remaining_capacity INTEGER NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'available', -- available, limited, sold_out, closed
  cutoff_minutes INTEGER DEFAULT 60,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT chk_remaining CHECK (remaining_capacity >= 0)
);
CREATE UNIQUE INDEX uq_avail_slot ON availability(option_id, local_date, start_time);
CREATE INDEX idx_avail_product_date ON availability(product_id, local_date);

CREATE TABLE availability_rule (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  option_id UUID NOT NULL REFERENCES product_option(id),
  day_of_week SMALLINT[], start_time TIME NOT NULL, end_time TIME,
  capacity INTEGER NOT NULL, valid_from DATE NOT NULL, valid_to DATE,
  is_active BOOLEAN NOT NULL DEFAULT TRUE
);
```
- Worker job `materialiseAvailability` (BullMQ repeatable, daily) expands rules into `availability` rows for a rolling horizon (default 365 days), upserting on `uq_avail_slot`.
- `status` derived: `remaining=0 → sold_out`; `remaining <= 20% → limited`.

**Testing**:
- `Integration: rule MON/WED, capacity 12, 30-day horizon → correct count of slots only on those weekdays.`
- `Unit: status derivation thresholds.`
- `Integration: re-run materialise → no duplicate slots (upsert).`

#### 3.2 — Capacity holds & idempotency

**What:** Short-lived inventory holds and request idempotency to prevent double-booking and duplicate charges.

**Design**:
- `holdCapacity(availabilityId, units)`: `UPDATE availability SET remaining_capacity = remaining_capacity - $units WHERE id=$1 AND remaining_capacity >= $units RETURNING id` — atomic; the CHECK + `>=` guard prevents oversell. Place a Redis key `hold:{token}` (TTL 15 min) recording the decrement for rollback on abandonment.
- Idempotency plugin: clients send `Idempotency-Key`; first response cached in Redis (24h) and replayed for repeats (OWASP-safe, prevents duplicate bookings).

**Testing**:
- `Integration (concurrency): 20 parallel holds of 1 unit on capacity-12 slot → exactly 12 succeed, 8 fail with CapacityUnavailable.`
- `Integration: hold then let TTL expire (mock clock) → remaining_capacity restored.`
- `Integration: same Idempotency-Key twice → identical response, one booking created.`

#### 3.3 — Booking lifecycle schema

**What:** Booking, unit items, and custom field responses.

**Design** (Suggestion 1 + JSONB custom fields):
```sql
CREATE TABLE booking (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  product_id UUID NOT NULL REFERENCES product(id),
  option_id UUID NOT NULL REFERENCES product_option(id),
  availability_id UUID REFERENCES availability(id),
  booking_number VARCHAR(20) NOT NULL UNIQUE,   -- OP-2026-001234
  reseller_reference VARCHAR(100),
  channel_id UUID REFERENCES distribution_channel(id),
  status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, confirmed, cancelled, completed, no_show
  contact_name VARCHAR(255), contact_email VARCHAR(255), contact_phone VARCHAR(50),
  contact_country CHAR(2) REFERENCES country(code), contact_locale VARCHAR(10),
  total_amount NUMERIC(12,2) NOT NULL, amount_paid NUMERIC(12,2) NOT NULL DEFAULT 0,
  tax_amount NUMERIC(12,2) DEFAULT 0, currency_code CHAR(3) NOT NULL REFERENCES currency(code),
  travel_date DATE NOT NULL,
  custom_fields JSONB NOT NULL DEFAULT '{}',     -- per-product custom question answers
  booked_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  confirmed_at TIMESTAMPTZ, cancelled_at TIMESTAMPTZ, cancellation_reason TEXT,
  created_by UUID REFERENCES platform_user(id)
);
CREATE TABLE booking_unit_item (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
  unit_type_id UUID NOT NULL REFERENCES unit_type(id),
  full_name VARCHAR(255), age INTEGER,
  dietary_requirements TEXT, special_requests TEXT,
  unit_price NUMERIC(12,2) NOT NULL
);
```
- State machine: `pending → confirmed → completed`; `pending|confirmed → cancelled`; `confirmed → no_show`. Enforced in `@repo/core/booking/transition.ts`; illegal transitions throw `InvalidTransition`.
- `booking_number` generated `OP-{year}-{6-digit sequence}` per operator via a `booking_sequence` table.

**Testing**:
- `Unit: transition(pending → completed) → InvalidTransition.`
- `Unit: booking_number format and per-operator sequence increments.`
- `Integration: cancel a confirmed booking → status cancelled, cancelled_at set, hold released.`

#### 3.4 — Booking API (internal + OCTO booking flow)

**What:** Internal booking endpoints and an OCTO-compatible reservation/confirm/cancel flow.

**Design**:
- Internal: `POST /v1/bookings` (validates availability via hold, prices via `priceFor`, totals, writes booking + unit items), `GET /v1/bookings/:id`, `POST /v1/bookings/:id/confirm`, `POST /v1/bookings/:id/cancel`.
- OCTO reseller flow under `/octo`: `POST /octo/bookings` (reservation, status `ON_HOLD`), `POST /octo/bookings/:uuid/confirm`, `DELETE /octo/bookings/:uuid` (cancel) — OCTO field names (`uuid`, `resellerReference`, `unitItems`, `availabilityId`).
- Booking creation runs inside `withTenant` transaction: hold → insert → response, all-or-nothing.

**Testing**:
- `Integration: POST booking for sold-out slot → 409 CapacityUnavailable, no row written.`
- `Integration (OCTO contract): reservation → confirm → GET returns CONFIRMED with resellerReference echoed.`
- `Integration: cancel after confirm → capacity restored.`
- `Contract: /octo endpoints validate against OCTO request/response JSON Schemas in fixtures.`

---

## Phase 4: Payments, Deposits & Bonding-Compliant Fund Segregation

### Purpose
Take money safely (PCI SAQ A via tokenisation), support deposits and balances, and maintain an immutable segregated-fund ledger that produces ATOL/ABTA-audit-ready reports. After this phase the booking flow is monetised and EU/UK compliance is evidenced.

### Tasks

#### 4.1 — Payment schema & gateway interface

**What:** `payment` table and a `PaymentGateway` interface with a Stripe implementation.

**Design**:
```sql
CREATE TABLE payment (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES booking(id),
  operator_id UUID NOT NULL REFERENCES operator(id),
  amount NUMERIC(12,2) NOT NULL, currency_code CHAR(3) NOT NULL REFERENCES currency(code),
  payment_type VARCHAR(20) NOT NULL,            -- deposit, balance, refund
  payment_method VARCHAR(20) NOT NULL,          -- card, bank_transfer, voucher
  payment_method_token VARCHAR(255),            -- Stripe/Adyen token only; NO raw card data (PCI DSS)
  gateway VARCHAR(30), gateway_reference VARCHAR(255),
  status VARCHAR(20) NOT NULL DEFAULT 'pending',-- pending, completed, failed, refunded
  paid_at TIMESTAMPTZ
);
```
```typescript
interface PaymentGateway {
  createIntent(input: { amount: number; currency: string; bookingId: string;
    metadata: Record<string,string> }): Promise<{ intentId: string; clientSecret: string }>;
  refund(input: { gatewayReference: string; amount: number }): Promise<{ refundId: string }>;
  verifyWebhook(rawBody: Buffer, signature: string): GatewayEvent; // throws on bad signature
}
```
- Card data never touches the server: client confirms PaymentIntent with `clientSecret`; status flows via webhook.

**Testing**:
- `Integration (msw Stripe): createIntent returns clientSecret; payment row pending.`
- `Unit: verifyWebhook with tampered signature → throws.`
- `Unit: no code path stores PAN/CVV (static assertion in schema; lint rule).`

#### 4.2 — Deposit & instalment workflow

**What:** Deposit policy enforcement and balance-due tracking.

**Design**:
- `operator.settings.depositPolicy` JSONB: `{ type: "percentage"|"fixed", value, balanceDueDaysBeforeTravel }`.
- On booking: compute deposit; create deposit PaymentIntent. On webhook `completed`: `booking.amount_paid += amount`; if `amount_paid >= total_amount` → eligible for confirm.
- Worker repeatable job `balanceReminders`: finds bookings with outstanding balance within `balanceDueDaysBeforeTravel` and enqueues reminder notifications (Phase 7).

**Testing**:
- `Unit: 30% deposit on 1000.00 → 300.00 due now, 700.00 balance.`
- `Integration: deposit paid → booking remains pending until balance; balance paid → confirmable.`
- `Integration: balanceReminders selects only bookings inside the window with outstanding balance.`

#### 4.3 — Segregated fund ledger

**What:** Append-only client-money ledger (ATOL/ABTA) with running balance and audit report.

**Design** (from Suggestion 1):
```sql
CREATE TABLE segregated_fund_ledger (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  booking_id UUID REFERENCES booking(id), payment_id UUID REFERENCES payment(id),
  entry_type VARCHAR(20) NOT NULL,              -- deposit_in, balance_in, supplier_payout, refund_out, commission_release
  amount NUMERIC(12,2) NOT NULL,               -- positive=in, negative=out
  currency_code CHAR(3) NOT NULL REFERENCES currency(code),
  running_balance NUMERIC(14,2) NOT NULL,
  description TEXT, recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Ledger writes are append-only (no UPDATE/DELETE; enforced by a DB trigger raising on UPDATE/DELETE). `running_balance` computed under a per-operator advisory lock to serialise writes.
- `GET /v1/reports/fund-segregation?from&to` → CSV + JSON audit report for bonding inspections.

**Testing**:
- `Integration: deposit_in 300 then supplier_payout -200 → running_balance 100.`
- `Integration (security): UPDATE on ledger row → trigger raises.`
- `Integration: report endpoint sums match final running_balance for the period.`

---

## Phase 5: Multi-Day Itinerary Builder & Guest App

### Purpose
Deliver the differentiator that incumbents fragment: a first-class multi-day, multi-destination itinerary builder plus a dynamic, mobile-first guest itinerary app. After this phase operators can build rich itineraries and guests can view them live via magic link.

### Tasks

#### 5.1 — Itinerary schema

**What:** Itinerary, day, and activity tables (template and booking-linked).

**Design** (Suggestion 1):
```sql
CREATE TABLE itinerary (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  product_id UUID REFERENCES product(id),       -- NULL = bespoke
  booking_id UUID REFERENCES booking(id),       -- NULL = template
  name VARCHAR(255) NOT NULL, description TEXT,
  total_days INTEGER NOT NULL, start_date DATE, end_date DATE,
  status VARCHAR(20) NOT NULL DEFAULT 'draft',   -- draft, published, active, completed
  version INTEGER NOT NULL DEFAULT 1
);
CREATE TABLE itinerary_day (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  itinerary_id UUID NOT NULL REFERENCES itinerary(id) ON DELETE CASCADE,
  day_number INTEGER NOT NULL, title VARCHAR(255), description TEXT,
  destination VARCHAR(255), country_code CHAR(2) REFERENCES country(code),
  accommodation_id UUID REFERENCES supplier(id),
  meal_plan VARCHAR(30)
);
CREATE TABLE itinerary_activity (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  itinerary_day_id UUID NOT NULL REFERENCES itinerary_day(id) ON DELETE CASCADE,
  product_id UUID REFERENCES product(id), supplier_id UUID REFERENCES supplier(id),
  title VARCHAR(255) NOT NULL, description TEXT,
  start_time TIME, end_time TIME, duration_iso VARCHAR(40),
  activity_type VARCHAR(30),                      -- tour, transfer, meal, free_time, accommodation, flight
  location VARCHAR(255), latitude DECIMAL(10,7), longitude DECIMAL(10,7),
  cost NUMERIC(12,2), currency_code CHAR(3) REFERENCES currency(code),
  sort_order INTEGER NOT NULL DEFAULT 0, notes TEXT
);
```
- Suppliers table introduced here (subset; full version in Phase 6): minimal `supplier(id, operator_id, name, supplier_type, ...)` so itinerary FKs resolve.

**Testing**:
- `Integration: create 5-day itinerary with activities → day/activity counts correct, ordered by sort_order.`
- `Unit: total_days must equal max(day_number); mismatch → validation error.`

#### 5.2 — Itinerary API & versioning + calendar export

**What:** CRUD, publish, reorder, version bump on edit, and iCal export.

**Design**:
- `POST /v1/itineraries`, nested day/activity endpoints, `POST /v1/itineraries/:id/publish`, `POST /v1/itineraries/:id/reorder` (day/activity sort).
- Editing a `published` itinerary bumps `version` and records prior state to `audit_log` (Phase 8 dependency satisfied minimally now).
- `GET /v1/itineraries/:id/ical` → `.ics` with VEVENT per activity (using `ical-generator`, ISO 8601 times in operator timezone).

**Testing**:
- `Integration: reorder activities → sort_order persisted.`
- `Integration: edit published itinerary → version 1→2.`
- `Unit: iCal output parses and contains one VEVENT per timed activity.`

#### 5.3 — Guest itinerary app (Next.js PWA)

**What:** Public, magic-link-gated, mobile-first dynamic itinerary viewer with live updates.

**Design**:
- Route `/i/:token` validates the signed guest link (Phase 1.5) → fetches itinerary read model.
- Day-by-day timeline UI (shadcn), map pins, accommodation, meals, guide details.
- Live updates: SSE stream `/v1/guest/:token/stream` pushes `itinerary.updated` events (sets up Phase 9 disruption pushes). PWA manifest + service worker for installability and offline last-known itinerary.

**Testing**:
- `E2E (Playwright): open valid magic link → itinerary renders all days.`
- `E2E: tampered token → access denied page.`
- `E2E: server emits itinerary.updated → UI reflects change without reload.`

---

## Phase 6: Suppliers, Guides & Guide Assignment

### Purpose
Complete back-office operations: full supplier records, guide profiles with languages/specialisations, and guide assignment to bookings and itinerary days. Sets up the structured data the AI guide-matcher (Phase 9) will consume.

### Tasks

#### 6.1 — Supplier & guide schema

**What:** Expand `supplier`; add `guide`, `guide_language`, `guide_assignment`.

**Design** (Suggestion 1):
```sql
ALTER TABLE supplier ADD COLUMN contact_email VARCHAR(255), ADD COLUMN payment_terms VARCHAR(50),
  ADD COLUMN commission_pct DECIMAL(5,2), ADD COLUMN rating DECIMAL(3,2),
  ADD COLUMN is_preferred BOOLEAN NOT NULL DEFAULT FALSE, ADD COLUMN metadata JSONB NOT NULL DEFAULT '{}';
CREATE TABLE guide (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  user_id UUID REFERENCES platform_user(id),
  full_name VARCHAR(255) NOT NULL, email VARCHAR(255), phone VARCHAR(50),
  bio TEXT, photo_url TEXT,
  certifications TEXT[], specialisations TEXT[],
  daily_rate NUMERIC(10,2), currency_code CHAR(3) REFERENCES currency(code),
  rating DECIMAL(3,2), is_active BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE TABLE guide_language (
  guide_id UUID NOT NULL REFERENCES guide(id) ON DELETE CASCADE,
  language_code VARCHAR(10) NOT NULL REFERENCES language(code),
  proficiency VARCHAR(20) NOT NULL DEFAULT 'fluent',
  PRIMARY KEY (guide_id, language_code)
);
CREATE TABLE guide_assignment (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  guide_id UUID NOT NULL REFERENCES guide(id),
  booking_id UUID REFERENCES booking(id),
  itinerary_day_id UUID REFERENCES itinerary_day(id),
  assignment_date DATE NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'tentative', -- tentative, confirmed, declined, completed
  match_score DECIMAL(4,3), notes TEXT
);
```

**Testing**:
- `Integration: assign guide already booked same date → conflict warning surfaced (overlap check).`
- `Unit: guide with no language for required tour → flagged ineligible.`

#### 6.2 — Supplier/guide CRUD API & availability check

**What:** Management endpoints and a guide availability/conflict checker.

**Design**:
- `POST/GET/PATCH /v1/suppliers`, `/v1/guides`, `/v1/guides/:id/assignments`.
- `guideConflicts(guideId, date)` returns existing non-declined assignments on that date.
- `GET /v1/guides/available?date&language&specialisation` filters active guides by language/specialisation with no conflict.

**Testing**:
- `Integration: available-guides excludes a guide with a confirmed assignment that day.`
- `Integration: filter by language=es returns only Spanish-speaking guides.`

---

## Phase 7: Channel Distribution, Communications & Reporting (MVP completion)

### Purpose
Complete the MVP feature set from features.md: OTA channel distribution (OCTO push/pull), customer communications/confirmations, and core reporting. After this phase the platform is a complete, sellable booking-and-ops product.

### Tasks

#### 7.1 — Distribution channel schema & OCTO inbound

**What:** Channel records and OCTO supplier-side endpoints so OTAs can read inventory and push bookings.

**Design**:
```sql
CREATE TABLE distribution_channel (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  name VARCHAR(100) NOT NULL, channel_type VARCHAR(30) NOT NULL, -- ota, direct, agent, api
  protocol VARCHAR(20),                          -- octo, ota_xml, custom_api, manual
  api_endpoint TEXT, api_key_hash VARCHAR(255), commission_pct DECIMAL(5,2),
  is_active BOOLEAN NOT NULL DEFAULT TRUE, config JSONB NOT NULL DEFAULT '{}'
);
CREATE TABLE channel_product (
  channel_id UUID NOT NULL REFERENCES distribution_channel(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  external_product_id VARCHAR(255), is_active BOOLEAN NOT NULL DEFAULT TRUE,
  last_synced_at TIMESTAMPTZ, PRIMARY KEY (channel_id, product_id)
);
```
- OCTO supplier endpoints (consumed by resellers): `GET /octo/products`, `GET /octo/products/:id`, `POST /octo/availability` (availability check), then the booking flow from Phase 3.4. Auth: API key per channel (`api_key_hash`).
- `ChannelAdapter` interface in `@repo/integrations/channels`: `pushProduct`, `pushAvailability`, `acknowledgeBooking`. OCTO and an OTA-XML stub adapter implement it.

**Testing**:
- `Integration (OCTO contract): GET /octo/products with channel key returns only channel-mapped active products.`
- `Integration: invalid channel API key → 401.`
- `Integration: OTA-XML adapter serialises a product to valid OpenTravel XML (schema-validated fixture).`

#### 7.2 — Outbound channel sync (worker)

**What:** Push availability/price changes to OTAs and ingest bookings.

**Design**:
- BullMQ jobs: `syncAvailability(channelId, productId)` (debounced on availability update), `ingestChannelBooking` (from inbound webhook/poll). Retry with exponential backoff; dead-letter after 5 attempts; failures logged and surfaced in dashboard.
- Webhook receiver `POST /webhooks/channels/:channelId` validates signature (HMAC for Bókun-style, API key for Rezdy-style) then enqueues `ingestChannelBooking`.

**Testing**:
- `Integration (msw): availability update triggers syncAvailability with correct external_product_id.`
- `Integration: inbound channel booking webhook (valid sig) → booking created with channel_id and reseller_reference; invalid sig → 401, nothing enqueued.`

#### 7.3 — Communications & confirmations

**What:** Templated email/SMS for confirmations, reminders, and disruption alerts.

**Design**:
```sql
CREATE TABLE communication (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id),
  booking_id UUID REFERENCES booking(id), guest_id UUID,
  channel VARCHAR(20) NOT NULL,                  -- email, sms, whatsapp, in_app
  direction VARCHAR(10) NOT NULL DEFAULT 'outbound',
  template_name VARCHAR(100), subject VARCHAR(255), body TEXT,
  status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, sent, delivered, failed, opened
  sent_at TIMESTAMPTZ
);
```
- `Notifier` interface (`send(channel, to, template, vars)`) with Resend + Twilio impls. Locale-aware templates (BCP 47).
- Event hooks: `booking.confirmed` → confirmation; `balanceReminders` job → reminder; `itinerary.updated` (Phase 9) → disruption alert. All sends are worker jobs writing a `communication` row.

**Testing**:
- `Integration (msw Resend): booking.confirmed enqueues + sends confirmation; communication row status sent.`
- `Unit: template renders localised content for fr-FR vs en-US.`
- `Integration: send failure → status failed, retried.`

#### 7.4 — Core reporting

**What:** Bookings, revenue, and occupancy reports.

**Design**:
- `GET /v1/reports/bookings?from&to&status` → counts, totals by product/channel.
- `GET /v1/reports/revenue?from&to&groupBy=product|channel|month` → gross, net, commission, tax.
- `GET /v1/reports/occupancy?productId&from&to` → capacity vs booked per slot. All support CSV export.
- Read-heavy queries use a few materialised views refreshed by a worker job; align with ISO 9001 evidence (timestamped, reproducible).

**Testing**:
- `Integration: revenue report groupBy=month sums match raw payment totals.`
- `Integration: occupancy report = total_capacity - remaining_capacity per slot.`
- `Unit: CSV export escapes commas/quotes correctly.`

---

## Phase 8: Audit, Compliance & Data Protection (GDPR/CCPA)

### Purpose
Make the platform enterprise- and EU-ready: comprehensive audit trail, GDPR/CCPA data subject workflows, compliance document generation (EU Package Travel Directive), and OWASP API hardening. Required before serving European operators and corporate buyers (ISO 27001 expectations).

### Tasks

#### 8.1 — Audit log & change tracking

**What:** Centralised, queryable audit trail across all mutations.

**Design** (Suggestion 1):
```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operator(id), user_id UUID REFERENCES platform_user(id),
  entity_type VARCHAR(50) NOT NULL, entity_id UUID NOT NULL,
  action VARCHAR(20) NOT NULL,                   -- create, update, delete, status_change
  old_values JSONB, new_values JSONB,
  ip_address INET, user_agent TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```
- `withAudit(ctx, entityType, entityId, action, before, after)` helper called by core services; computes a JSONB diff. A Drizzle middleware auto-captures mutations on flagged tables.

**Testing**:
- `Integration: update booking status → audit row with old/new status diff and acting user.`
- `Integration: audit query returns full lifecycle for a booking in order.`

#### 8.2 — GDPR/CCPA data subject workflows

**What:** Right-to-access (export), right-to-erasure, consent management, and retention purge.

**Design**:
- `guest` table (introduced minimally in Phase 5 via bookings) extended with `consent_marketing`, `consent_data_processing`, `data_retention_until`, and app-level encryption of `passport_number`.
- `GET /v1/privacy/export?guestId` → JSON bundle of all PII (booking history, communications). `POST /v1/privacy/erase` → pseudonymises PII (name/email/phone → tombstone), preserves financial/audit rows for bonding obligations (legal-basis override).
- Worker repeatable `retentionPurge`: erases guests past `data_retention_until`.

**Testing**:
- `Integration: erase request → guest PII pseudonymised; booking financial totals intact; audit records the erasure.`
- `Integration: export returns all PII categories for the guest.`
- `Integration: retentionPurge erases only past-retention guests.`

#### 8.3 — Compliance documents & profiles

**What:** Generate jurisdiction-specific documents (EU Package Travel pre-contractual info, ATOL certificate) from `compliance_profile.rules`.

**Design**:
- `GET /v1/bookings/:id/documents/:type` where type ∈ `pre_contractual_info`, `confirmation`, `atol_certificate`. Renders from JSONB rule templates + booking data to PDF (locale-aware).
- Compliance profile selected by operator jurisdiction at registration; drives which document types are available and whether fund segregation is mandatory.

**Testing**:
- `Integration: EU operator booking → pre_contractual_info document available; US operator → not offered.`
- `Unit: document renders required EU Package Travel fields (cancellation rights, insolvency protection statement).`

#### 8.4 — OWASP API hardening

**What:** Audit and close OWASP API Top 10 gaps.

**Design**:
- BOLA: every object fetch is tenant-scoped via RLS + explicit `operator_id` check; add automated test that operator A cannot fetch B's objects by ID across all `GET /:id` routes.
- Mass assignment: Zod schemas use strict allow-lists (`.strict()`); reject unknown fields.
- Rate limiting per route group; security headers (`helmet`-equivalent); request size limits.

**Testing**:
- `Integration (matrix): for every /:id GET, operator A requesting B's id → 404/403, never data.`
- `Integration: POST with extra unexpected field → 422 (mass-assignment blocked).`
- `Security scan: zap baseline against running api → no high-severity findings.`

---

## Phase 9: AI Itinerary Generation & Dynamic Disruption Adaptation

### Purpose
Ship the headline AI-native differentiators: generate a fully-costed, supplier-linked itinerary from a client brief in minutes, and adapt live itineraries to real-world disruptions with pre-negotiated supplier alternatives. This is the competitive frontier identified across research.md and features.md.

### Tasks

#### 9.1 — LLM client & structured-output contracts

**What:** Provider-agnostic `LlmClient` with structured (JSON-Schema-constrained) generation.

**Design**:
```typescript
interface LlmClient {
  generateObject<T>(input: { system: string; prompt: string;
    schema: ZodSchema<T>; model?: string }): Promise<T>;
  generateText(input: { system: string; prompt: string; stream?: boolean }): Promise<string>;
}
```
- Built on the Vercel AI SDK; provider chosen by `LLM_PROVIDER`. All itinerary outputs constrained by Zod schemas so results are validated, not free text. Token usage logged per operator for cost tracking.

**Testing**:
- `Unit (mocked provider): generateObject returns schema-valid object; invalid model output → retried then throws after N attempts.`

#### 9.2 — AI itinerary generation from brief

**What:** `POST /v1/ai/itineraries/generate` turns a client brief into a draft itinerary built from the operator's catalogue and suppliers.

**Design**:
- Input: `{ destination, durationDays, budget, currency, interests[], fitnessLevel, partySize, dates }`.
- Pipeline: (1) retrieve candidate products/suppliers for the destination (SQL + filters); (2) call `generateObject` with a system prompt instructing the model to assemble a day-by-day plan **only from supplied catalogue items** (passed as tool context) within budget; (3) validate output against itinerary Zod schema; (4) cost each activity via `priceFor`/supplier rates; (5) persist as a `draft` itinerary.
- System prompt template (`@repo/llm/prompts/itinerary-generate.ts`): role = expert tour planner; constraints = use only provided catalogue IDs, respect budget/fitness/duration, balance pace, return structured days/activities with `productId`/`supplierId` references.
- Guardrail: any activity referencing an ID not in the supplied set is rejected and regenerated.

**Testing**:
- `Integration (mocked LLM, real DB): brief for 5-day Kenya safari → draft itinerary with 5 days, all activity IDs exist in catalogue, total ≤ budget.`
- `Unit: output referencing a hallucinated product ID → rejected, regeneration triggered.`
- `Integration: generated itinerary persists as draft and is editable via Phase 5 API.`

#### 9.3 — Disruption monitoring & adaptation

**What:** Poll weather/transport feeds; when an active itinerary is impacted, propose an adapted plan with supplier alternatives.

**Design**:
- `DisruptionSource` interface: `check(location, date) → Disruption | null` (weather, transport). Worker repeatable `pollDisruptions` scans active itineraries' upcoming days.
- On disruption: build context (impacted activity, guest preferences, pre-negotiated alternative suppliers flagged `is_preferred` for indoor/wet-weather fallback) → `generateObject` proposes a swap → create a **proposed** `ItineraryAdaptation` record (do not auto-apply unless operator opted in).
- `POST /v1/itineraries/:id/adaptations/:id/apply` applies the swap, bumps `version`, writes audit, emits `itinerary.updated` (→ guest SSE + alert).
```sql
CREATE TABLE itinerary_adaptation (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  itinerary_id UUID NOT NULL REFERENCES itinerary(id) ON DELETE CASCADE,
  day_number INTEGER NOT NULL,
  reason VARCHAR(50) NOT NULL,                   -- weather_disruption, transport_disruption, venue_closure
  disruption JSONB NOT NULL, proposal JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'proposed',-- proposed, applied, dismissed
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), applied_at TIMESTAMPTZ
);
```

**Testing**:
- `Integration (mocked weather feed amber warning): pollDisruptions creates a proposed adaptation for the impacted day with a preferred indoor alternative.`
- `Integration: apply adaptation → itinerary day activity swapped, version bumped, itinerary.updated emitted, guest comm enqueued.`
- `Integration: no disruption → no adaptation created.`

#### 9.4 — AI content generation

**What:** Generate product descriptions and SEO copy from structured product data.

**Design**:
- `POST /v1/ai/products/:id/describe` → `generateText` produces description + short_description + suggested Schema.org keywords; operator reviews before publish (human-in-the-loop).

**Testing**:
- `Integration (mocked LLM): describe returns non-empty description/short_description; not auto-applied (stored as suggestion).`

---

## Phase 10: AI Intelligence — Guide Matching, Personalisation & Supplier Scoring

### Purpose
Round out the AI-native intelligence layer with the remaining differentiators: ML-assisted guide matching, guest personalisation/preference learning, and supplier-relationship scoring. These build on the structured data accumulated in Phases 6–9.

### Tasks

#### 10.1 — Guide matching engine

**What:** Score and rank guides for a booking/day by language, specialisation, availability, and past rating.

**Design**:
- `matchGuides(bookingId | itineraryDayId)`: candidate set = active guides with no conflict (Phase 6.2). Score = weighted sum: language match (0.4), specialisation overlap with product tags (0.3), past guest rating (0.2), availability confidence (0.1). Returns ranked list with `match_score`; writing an assignment stores the score.
- Endpoint `GET /v1/guides/match?bookingId` returns ranked candidates with factor breakdown (mirrors the `GuideAssigned.matchFactors` shape from Suggestion 2).

**Testing**:
- `Unit: guide with required language + specialisation + 4.8 rating outranks a guide missing the language.`
- `Integration: match excludes conflicted guides; top result is highest weighted score.`

#### 10.2 — Guest personalisation & preference learning

**What:** Build a guest preference profile from booking history and feedback; surface tailored upgrade/repeat-booking suggestions.

**Design**:
- `guest_preference` derived store: aggregates past product types, destinations, activity tags, price band, dietary/medical flags. Updated by a worker job on `booking.completed` and on review submission.
- `GET /v1/guests/:id/recommendations` → products similar to past enjoyed items (tag/destination similarity + collaborative signal), filtered to upcoming availability.

**Testing**:
- `Integration: guest with two culinary tours → recommendations rank culinary/food-tag products first.`
- `Unit: preference aggregation weights recent completed trips above old ones.`

#### 10.3 — Supplier relationship intelligence

**What:** Supplier scorecard aggregating booking volume, lead times, cancellation rate, and guest ratings; flag preferred/underperforming partners.

**Design** (read model from Suggestion 2's `rm_supplier_scorecard`):
```sql
CREATE TABLE supplier_scorecard (
  supplier_id UUID PRIMARY KEY REFERENCES supplier(id),
  operator_id UUID NOT NULL REFERENCES operator(id),
  total_bookings INTEGER NOT NULL DEFAULT 0,
  cancellation_count INTEGER NOT NULL DEFAULT 0, cancellation_rate DECIMAL(5,4) DEFAULT 0,
  avg_lead_time_days DECIMAL(5,1), avg_guest_rating DECIMAL(3,2),
  total_revenue NUMERIC(14,2) DEFAULT 0, currency_code CHAR(3),
  computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Worker `recomputeSupplierScores` (daily) rebuilds from bookings/payments/reviews. `GET /v1/reports/suppliers` ranks; flags `cancellation_rate > threshold` as at-risk and high-volume/high-rating as preferred.

**Testing**:
- `Integration: supplier with 3/10 cancellations → cancellation_rate 0.30, flagged at-risk.`
- `Integration: scorecard avg_guest_rating matches mean of review ratings.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, RLS, auth)        ─── required by everything
    │
Phase 2: Product Catalogue & Pricing                 ─── requires P1
    │
Phase 3: Availability & Booking Engine               ─── requires P2  (core value ships here)
    │
    ├── Phase 4: Payments & Fund Segregation          ─── requires P3
    │
    ├── Phase 5: Itinerary Builder & Guest App        ─── requires P2 (P3 for booking-linked itineraries)
    │       │
    │       └── Phase 6: Suppliers & Guides           ─── requires P5 (itinerary FKs) ; can parallel P4
    │
    └── Phase 7: Channels, Comms & Reporting (MVP done)─── requires P3, P4 (revenue reports), P5 (comms hooks)
            │
            ├── Phase 8: Audit, Compliance, GDPR       ─── requires P3–P7 ; can parallel P9
            │
            └── Phase 9: AI Generation & Disruption    ─── requires P5, P6 (catalogue+suppliers+guides)
                    │
                    └── Phase 10: AI Intelligence       ─── requires P6, P9 (data accumulated)
```

Parallelism opportunities:
- **Phase 4 (Payments)** and **Phase 5 (Itinerary/Guest App)** can be built concurrently after Phase 3.
- **Phase 6 (Suppliers/Guides)** can parallel **Phase 4** once Phase 5 lands the supplier stub.
- **Phase 8 (Compliance)** and **Phase 9 (AI Generation)** can be developed concurrently after Phase 7.
- The **dashboard** and **guest-app** front-ends can be built incrementally alongside their backing APIs from Phase 2 onward.

MVP boundary: **Phases 1–7** deliver the complete must-have feature set from features.md (booking engine, channel distribution, multi-day itineraries, availability sync, communications, reporting, supplier/guide management). **Phases 8–10** deliver should-have/nice-to-have and the AI-native differentiators.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented as specified.
2. All unit and integration tests pass (`turbo run test`).
3. Biome lint + format clean (`turbo run lint`).
4. Type check passes (`tsc --noEmit` across the workspace).
5. Docker build succeeds and `docker compose up` brings the stack healthy.
6. New/changed REST and OCTO endpoints appear in the generated OpenAPI 3.1 spec and validate against their JSON Schemas.
7. Drizzle migrations generated, applied cleanly to a fresh database, and reversible where applicable.
8. RLS verified for any new tenant tables (cross-tenant access test passes).
9. New config/env keys added to `.env.example` and documented in the README.
10. The phase's headline capability demonstrated end-to-end (HTTP for API phases, Playwright for web phases).
11. OWASP-relevant routes covered by the BOLA/mass-assignment test matrix (from Phase 8 onward, applied retroactively to earlier routes).
12. Audit entries written for all new mutating operations (from Phase 8 onward).
```
