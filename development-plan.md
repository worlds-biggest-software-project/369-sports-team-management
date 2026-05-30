# Sports Team Management — Phased Development Plan

> Project: 369-sports-team-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model proposals. The database is built on **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** — chosen because the project targets a production, multi-club, multi-sport platform where conflict-detecting cross-team scheduling, cross-season analytics, and GDPR/COPPA per-entity audit compliance are first-class requirements. The append-only audit log and offline-sync ideas from Suggestion 3, and the materialised read-model concept for season analytics, are folded in where they add value.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **TypeScript** (Node 22 LTS) end-to-end | The product is API + PWA + native-shell heavy; one language across server, client, and shared validation schemas (`zod`) eliminates type drift on a large multi-module surface. The AI surface (LLM calls, MCP server) has first-class TS SDKs. |
| Monorepo tooling | **pnpm workspaces + Turborepo** | Shared `@stm/schemas`, `@stm/db`, `@stm/sdk` packages consumed by API, web, and MCP server. Turborepo caches builds/tests across the workspace. |
| API framework | **Fastify 5 + `@fastify/swagger`** | High throughput, native JSON-schema validation per route, auto-generates the OpenAPI 3.1 spec required by `standards.md`. Plugin model maps cleanly onto bounded modules (auth, roster, schedule, stats…). |
| API contract | **OpenAPI 3.1.0**, generated from route schemas; **RFC 9457 Problem Details** error envelope; **RFC 8288** `Link` pagination headers | Mandated by `standards.md`. Contract-first keeps the PWA, native shell, and partner integrations in sync. |
| Database | **PostgreSQL 16** | Suggestion 1 relies on FK integrity, JSONB stat templates, GIN indexes, partial indexes, and range-partitioned audit tables — all native to Postgres. JSONB gives multi-sport flexibility inside a relational frame. |
| Migrations & query layer | **Drizzle ORM + drizzle-kit** | Typed schema in TS that doubles as the source of truth for `@stm/schemas`; generates SQL migrations; thin enough to drop to raw SQL for the analytics aggregations and conflict-detection queries. |
| Cache / queue | **Redis 7 + BullMQ** | Async workloads: LLM generation, video processing, calendar sync, notification fan-out, Stripe webhook handling. Redis also backs rate limiting and SSE pub/sub. |
| Object storage | **S3-compatible (AWS S3 / MinIO self-hosted)** | Video, documents (medical clearances, contracts), avatars. MinIO satisfies the self-hosted deployment mode. Presigned uploads keep large files off the API. |
| Auth | **OAuth 2.1 + OIDC** (Google/Apple/Microsoft) **+ email/password**, **JWT (RFC 7519)** access tokens, **PKCE (RFC 7636)** for mobile, **WebAuthn L3** passkeys for admins/coaches | `standards.md` security section. JWT access + rotating refresh tokens; argon2id password hashing per NIST SP 800-63B. |
| LLM provider | **Provider-abstracted via Vercel AI SDK** (default Anthropic Claude; pluggable OpenAI/local) | Recaps, parent emails, NL stat queries, lineup suggestions. Abstraction lets self-hosted clubs point at a local model. |
| Vision (auto-tagging) | **Open vision models** (e.g. YOLO-family + a CLIP/embedding model) run in a Python sidecar worker | `features.md` and the IP summary require avoiding patented auto-camera/tagging mechanisms; open models in an isolated worker keep the licence clean and the heavy deps out of the Node API. |
| Embeddings / similarity | **pgvector extension** | Scouting similarity search and NL-query retrieval over stat/video embeddings without a separate vector DB. |
| MCP server | **`@modelcontextprotocol/typescript-sdk`** | Exposes `get_roster`, `query_stats`, `draft_match_recap`, `find_schedule_conflicts` per `standards.md`. Reuses `@stm/db` and service layer. |
| Payments | **Stripe Connect** (Node SDK) | Registration fees, dues, kit orders. Delegating card handling keeps PCI DSS scope minimal (SAQ-A). |
| Calendar | **iCalendar (RFC 5545) export**, **CalDAV (RFC 4791) two-way**, Google Calendar + Microsoft Graph push sync | Schedule sync per `standards.md`. |
| Notifications | **Web Push (VAPID) + email (Resend/SES) + SMS (Twilio)** | Multi-channel game-day reminders with per-user `notification_prefs`. |
| Realtime | **Server-Sent Events** for live scores; **AsyncAPI 3.0**-described channels; **CloudEvents 1.0** webhook envelopes | Live scoring transport per `standards.md` notes (SSE preferred over polling). |
| Frontend | **Next.js 15 (App Router) PWA** + **Workbox** service worker; **Capacitor** native shell (iOS/Android) | One codebase for PWA + native shell per README. Workbox + IndexedDB power offline match-day mode. WCAG 2.2 AA target. |
| UI components | **Tailwind CSS + shadcn/ui + Radix** | Accessible primitives (WCAG 2.2), fast to build data-dense coach dashboards and parent-friendly views. |
| Offline store (client) | **IndexedDB via Dexie + an outbox of intent events** | Match-day actions buffered as ordered intents, replayed to the API on reconnect (Suggestion 3's offline-sync idea at the client layer). |
| Testing | **Vitest** (unit/integration) + **Supertest**/Fastify `inject` (API) + **Playwright** (E2E/PWA) + **Testcontainers** (real Postgres/Redis) | Standard TS stack; Testcontainers gives real-DB integration tests without external services. |
| Quality | **ESLint + Prettier + `tsc --noEmit`** type checks; **Drizzle migration check** in CI | Enforced per phase Definition of Done. |
| Packaging | **Docker + docker-compose** (api, worker, vision-worker, web, postgres, redis, minio) | Supports both managed cloud and self-hosted installs per README. |
| Observability | **OpenTelemetry traces/metrics**, **pino** structured logs | Per-request tracing across API → queue → worker; audit-grade logging for minor-data access. |

### Project Structure

```
sports-team-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── schemas/                 # @stm/schemas — zod + JSON Schema, shared types
│   │   └── src/
│   │       ├── stat-template.ts # JSON Schema 2020-12 for sport stat templates
│   │       ├── entities.ts      # zod models mirroring DB rows
│   │       ├── api/             # request/response DTOs per module
│   │       └── events.ts        # CloudEvents envelope + domain event types
│   ├── db/                      # @stm/db — drizzle schema, migrations, repos
│   │   └── src/
│   │       ├── schema/          # one file per table group
│   │       ├── migrations/
│   │       ├── client.ts
│   │       └── repositories/    # typed data-access (clubs, users, teams…)
│   ├── core/                    # @stm/core — domain services (no HTTP/transport)
│   │   └── src/
│   │       ├── auth/            # tokens, password, passkey, consent rules
│   │       ├── roster/
│   │       ├── scheduling/      # conflict detection engine
│   │       ├── stats/           # aggregation + benchmarking
│   │       ├── payments/
│   │       ├── injuries/
│   │       ├── scouting/
│   │       ├── ai/             # llm provider, prompt templates, embeddings
│   │       ├── notifications/
│   │       ├── calendar/        # ical/caldav/google/graph
│   │       └── audit/           # append-only audit writer + minor-access log
│   └── sdk/                     # @stm/sdk — generated TS client from OpenAPI
├── apps/
│   ├── api/                     # Fastify server
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/         # auth, db, redis, swagger, errors, rate-limit
│   │       ├── modules/         # route registrations per bounded module
│   │       └── sse/             # live-score SSE handlers
│   ├── worker/                  # BullMQ processors
│   │   └── src/
│   │       ├── queues.ts
│   │       └── processors/      # llm, video, calendar-sync, notify, stripe, projections
│   ├── vision-worker/           # Python sidecar (FastAPI) for auto-tagging
│   │   ├── app.py
│   │   ├── requirements.txt
│   │   └── pipeline/
│   ├── mcp/                     # @modelcontextprotocol MCP server
│   │   └── src/index.ts
│   └── web/                     # Next.js PWA + Capacitor shell
│       └── src/
│           ├── app/
│           ├── components/
│           ├── lib/offline/     # Dexie outbox + sync
│           └── service-worker/
├── infra/
│   ├── openapi.json             # generated, committed for SDK + partners
│   └── asyncapi.yaml            # live-score channel descriptions
└── tests/
    ├── fixtures/                # sample clubs, soccer/basketball stat templates, diffs
    └── e2e/                     # Playwright specs
```

---

## Phase 1: Foundation — Monorepo, Schemas, Database, Auth

### Purpose
Establish the workspace, the shared validation/type layer, the full database schema with migrations, and authentication. Everything downstream depends on the schema package and a working auth context. After this phase the API boots, can authenticate a user, and the database is fully migrated with seed data.

### Tasks

#### 1.1 — Workspace, tooling, and Docker scaffold

**What**: Initialise the pnpm/Turborepo monorepo, base packages, and a `docker-compose` dev stack.

**Design**:
- `pnpm-workspace.yaml` declares `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build` (depends on `^build`), `lint`, `typecheck`, `test`.
- `docker-compose.yml` services: `postgres:16` (with `pgvector` image `pgvector/pgvector:pg16`), `redis:7`, `minio`, plus app services added later.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`/`S3_BUCKET`/`S3_KEY`/`S3_SECRET`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `LLM_PROVIDER`, `LLM_API_KEY`, `GOOGLE_OAUTH_*`, `APPLE_OAUTH_*`, `MICROSOFT_OAUTH_*`, `TWILIO_*`, `VAPID_PUBLIC`/`VAPID_PRIVATE`, `RESEND_API_KEY`.
- Root scripts: `dev`, `build`, `lint`, `typecheck`, `test`, `db:migrate`, `db:seed`.

**Testing**:
- `Unit: pnpm -w typecheck → exits 0 on empty packages`.
- `Integration: docker compose up postgres redis minio → all healthchecks pass`.
- `Smoke: pnpm build → turbo reports all packages built, cache populated`.

#### 1.2 — Shared schema package (`@stm/schemas`)

**What**: zod models and JSON Schemas for all entities, the sport stat-template format, and the CloudEvents envelope.

**Design**:
- Stat template (JSON Schema 2020-12), validated at season creation:
```ts
export const StatField = z.object({
  key: z.string().regex(/^[a-z][a-z0-9_]*$/),
  label: z.string(),
  type: z.enum(["integer", "decimal", "boolean", "duration_seconds"]),
  aggregate: z.enum(["sum", "avg", "max", "min", "none"]).default("sum"),
  benchmarkable: z.boolean().default(false),
});
export const StatTemplate = z.object({
  sport: z.string(),
  fields: z.array(StatField).min(1),
});
```
- Entity zod models mirror the DB rows (Club, User, Team, TeamMember, Season, Event, EventRsvp, PlayerStats, Video, Prospect, Injury, Payment, AiSuggestion).
- CloudEvents envelope:
```ts
export const CloudEvent = z.object({
  specversion: z.literal("1.0"),
  id: z.string().uuid(),
  source: z.string(),            // e.g. "/clubs/{id}/events"
  type: z.string(),              // e.g. "com.stm.rsvp.submitted"
  time: z.string().datetime(),   // ISO 8601
  subject: z.string().optional(),
  data: z.unknown(),
});
```
- Ship `roles`, `event_type`, `injury_severity`, etc. as exported const enums reused by DB CHECK constraints and API validation.

**Testing**:
- `Unit: valid soccer stat template → parses; unknown field type → ZodError naming the field`.
- `Unit: StatTemplate with empty fields[] → ZodError "fields must contain at least 1"`.
- `Unit: CloudEvent missing time → ZodError; non-ISO time → ZodError`.

#### 1.3 — Database schema & migrations (`@stm/db`)

**What**: Drizzle schema implementing the 14 tables of Suggestion 1, plus `refresh_tokens` and `webauthn_credentials`, plus the partitioned `audit_log`.

**Design**:
- Translate Suggestion 1 DDL verbatim into Drizzle (`clubs`, `users`, `teams`, `team_members`, `seasons`, `events`, `event_rsvps`, `player_stats`, `videos`, `prospects`, `injuries`, `payments`, `ai_suggestions`, `audit_log`). Enable extensions `pgcrypto`, `pgvector`.
- Add auth tables:
```sql
CREATE TABLE refresh_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash TEXT NOT NULL,           -- sha256 of token
  family_id UUID NOT NULL,            -- rotation family
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_refresh_user ON refresh_tokens (user_id) WHERE revoked_at IS NULL;

CREATE TABLE webauthn_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  credential_id TEXT UNIQUE NOT NULL,
  public_key BYTEA NOT NULL,
  counter BIGINT NOT NULL DEFAULT 0,
  transports TEXT[],
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Add a pgvector column for later phases: `ALTER TABLE prospects ADD COLUMN attribute_embedding vector(384);` (nullable now, populated in Phase 9).
- `audit_log` created as a range-partitioned parent with a monthly-partition helper and a default partition.
- `repositories/` expose typed methods (`clubs.byId`, `users.byEmail`, `teamMembers.forTeam`, …) so `@stm/core` never imports drizzle directly.

**Testing** (Testcontainers Postgres):
- `Integration: run all migrations on empty DB → all 16 tables + partitions exist; rollback to zero leaves no tables`.
- `Integration: insert user with role 'banana' → CHECK constraint violation`.
- `Integration: insert team_member duplicate (team_id,user_id) → unique violation`.
- `Integration: GIN index on player_stats.stats_json present (query pg_indexes)`.

#### 1.4 — Seed data & fixtures

**What**: Deterministic seed: one club, soccer + basketball teams, ~25 users (admins, coaches, players incl. minors with parents, parents, medical staff), one active season each with stat templates, and a handful of events with RSVPs and stats.

**Design**: `db:seed` script using fixed UUIDs from `tests/fixtures/`. Soccer and basketball `stat_template_json` exactly match the examples in Suggestion 1. Include at least 3 minors with `parent_user_id` and consent timestamps to exercise compliance flows.

**Testing**:
- `Integration: db:seed twice → idempotent (upsert by fixed UUID, no duplicate-key errors)`.
- `Integration: seeded minor has parent_user_id set and parental_consent_at non-null`.

#### 1.5 — Authentication & session management

**What**: Email/password + OIDC social login, JWT access + rotating refresh tokens, passkey registration, and the request auth context.

**Design**:
- Password hashing: argon2id (NIST SP 800-63B params). Email/password login issues a 15-min access JWT (`sub`, `club_id`, `role`, `is_minor`) and a 30-day refresh token (hashed, rotation family). Refresh rotation: presenting a used refresh token revokes the whole family (reuse detection).
- OIDC: Authorization Code + PKCE for Google/Apple/Microsoft; on first login create/link `users` row by verified email.
- WebAuthn L3 registration/authentication for `admin`/`coach` via `@simplewebauthn/server`.
- Fastify `authPlugin` decorates `request.auth = { userId, clubId, role, isMinor }`; `requireRole(...roles)` and `requireSameClub(resourceClubId)` guards.
- Endpoints (OpenAPI-described): `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/oidc/:provider/start`, `GET /auth/oidc/:provider/callback`, `POST /auth/passkey/register/(options|verify)`, `POST /auth/passkey/login/(options|verify)`, `GET /auth/me`.

**Testing**:
- `Unit: argon2id verify round-trip; wrong password → false`.
- `Unit: refresh rotation — old token reuse → all family tokens revoked`.
- `Integration (real DB): login with seeded coach → 200 + access/refresh; bad password → 401 RFC 9457 body`.
- `Integration: access /auth/me with expired JWT → 401; with valid → user profile`.
- `Integration: passkey register options → verify with @simplewebauthn test authenticator → credential stored`.

---

## Phase 2: Core Org Model — Clubs, Users, Teams, Roster, RBAC

### Purpose
Deliver the administrative backbone: managing clubs, inviting users, creating teams, and building rosters with role-based access. This is the foundation every other module reads from. After this phase a club admin can stand up a team and roster end-to-end through the API.

### Tasks

#### 2.1 — Club & user management

**What**: CRUD for clubs and users with RBAC and club-scoping.

**Design**:
- Endpoints: `GET/PATCH /clubs/:id`, `POST /clubs` (signup-time), `GET /clubs/:id/users` (paginated, RFC 8288 `Link`), `POST /clubs/:id/users/invite`, `GET/PATCH/DELETE /users/:id`.
- Invite flow: admin creates a pending user with email + role; invite email contains a single-use token; invitee completes via `POST /auth/register?invite=…`.
- Every mutation routes through `audit/` writer (see 2.4). Cross-club access blocked by `requireSameClub`.
- `DELETE /users/:id` is a soft delete (`is_active=false`) unless a GDPR erasure (Phase 8) hard-anonymises.

**Testing**:
- `Integration: coach lists club users → 200; coach from club B requests club A users → 403`.
- `Integration: invite creates pending user + audit row action='user.invited'`.
- `Unit: pagination cursor encode/decode round-trip; Link header has rel="next" when more rows`.

#### 2.2 — Team management

**What**: CRUD for teams, scoped to a club, with sport/level/age metadata.

**Design**:
- Endpoints: `POST /clubs/:id/teams`, `GET /clubs/:id/teams`, `GET/PATCH/DELETE /teams/:id`.
- `sport` constrained to the enum from Suggestion 1; `color_*` validated as hex.
- Authorization: `admin` creates/deletes; `coach` assigned to a team may PATCH metadata of that team only.

**Testing**:
- `Integration: admin creates soccer team → 201 with id; non-admin → 403`.
- `Integration: create team with sport='quidditch' → 422 Problem Details, field "sport"`.

#### 2.3 — Roster (team_members) & documents

**What**: Add/remove members, assign jersey/position/captaincy, manage eligibility, and store roster documents.

**Design**:
- Endpoints: `POST /teams/:id/members`, `GET /teams/:id/members`, `PATCH /teams/:id/members/:memberId` (**RFC 6902 JSON Patch** for partial roster/lineup updates per `standards.md`), `DELETE /teams/:id/members/:memberId` (sets `left_at`, `is_active=false`).
- Document upload: `POST /teams/:id/members/:memberId/documents` returns an S3 presigned PUT URL; client uploads directly; `documents_json` records `{type, url, expires_at}`. Allowed types: `medical_clearance`, `contract`, `id_verification`, `consent_form`.
- Validation: a player who is a minor must have `parental_consent_at` set on their user before becoming `eligible` (compliance guard from Phase 8 wired here as a pre-check).
- Unique `(team_id, jersey_number)` among active players enforced at service layer.

**Testing**:
- `Integration: add player → roster lists them with jersey/position`.
- `Integration: add second active player with same jersey → 409 Problem Details`.
- `Unit: JSON Patch [{op:"replace",path:"/position",value:"forward"}] applied → member.position == "forward"`.
- `Integration: presigned URL issued; PUT to MinIO succeeds; documents_json updated on confirm`.
- `Integration: set minor player eligible without parental consent → 409 "parental_consent_required"`.

#### 2.4 — Audit logging service

**What**: Centralised append-only audit writer with a `involves_minor` flag, used by every mutating service.

**Design**:
- `audit/writer.ts`: `recordAudit({clubId, userId, actorType, action, entityType, entityId, involvesMinor, changes, ip})` inserts into the partitioned `audit_log`. `actorType ∈ {user, system, ai, stripe_webhook, calendar_sync, offline_sync}`.
- A Fastify `onResponse` hook captures `ip_address`; mutating services pass `before`/`after` diffs as `changes_json`.
- `involves_minor` is computed by checking whether the affected user/member is a minor (helper in `auth/consent`).
- Read endpoint (admin only): `GET /clubs/:id/audit?involvesMinor=&from=&to=&entityType=` — the GDPR minor-access report.

**Testing**:
- `Unit: recordAudit on minor entity → involves_minor=true`.
- `Integration: PATCH a minor's profile → audit row with involves_minor=true, changes_json has before/after`.
- `Integration: GET audit filtered involvesMinor=true returns only minor-related rows; non-admin → 403`.

---

## Phase 3: Scheduling Engine & Conflict Detection

### Purpose
Build the calendar core and the headline differentiator: a scheduling engine that detects and helps resolve conflicts across venues, officials, and opponent/team availability. After this phase clubs can plan a season's events and trust the system to surface clashes. This is the "heart" of the product shipping early per the phase-design principles.

### Tasks

#### 3.1 — Seasons & events CRUD

**What**: Manage seasons (with stat templates) and the events that hang off them.

**Design**:
- Endpoints: `POST /teams/:id/seasons`, `GET/PATCH /seasons/:id`; `POST /teams/:id/events`, `GET /teams/:id/events?from=&to=&type=&status=`, `GET/PATCH/DELETE /events/:id`.
- Season creation validates `stat_template_json` against the `StatTemplate` schema (1.2).
- Event timestamps stored as `TIMESTAMPTZ` (ISO 8601 with tz per `standards.md`); `duration_minutes` defaults from `end_at - start_at`.
- Recurrence: `POST /teams/:id/events:recurring` accepts an RRULE (RFC 5545) and expands to concrete events (cap 200).

**Testing**:
- `Unit: RRULE "FREQ=WEEKLY;COUNT=10" → 10 events on correct weekdays`.
- `Integration: create season with invalid stat template → 422 naming the bad field`.
- `Integration: list events with from/to filters → only events in window, ordered by start_at`.

#### 3.2 — Conflict detection engine

**What**: A pure service that, given a proposed or existing event, returns all conflicts across four dimensions.

**Design**:
```ts
type ConflictKind = "venue" | "official" | "team_double_book" | "player_overlap";
interface Conflict {
  kind: ConflictKind;
  eventId: string;        // the conflicting existing event
  detail: string;         // human-readable
  severity: "hard" | "soft";
}
function detectConflicts(input: {
  clubId: string;
  teamId: string;
  startAt: Date; endAt: Date;
  venue?: string;
  officials?: string[];
  excludeEventId?: string;
}): Promise<Conflict[]>;
```
- **venue (hard)**: same `venue` (normalised, case-insensitive) with overlapping `[start_at, end_at)` in the same club.
- **team_double_book (hard)**: same `team_id` overlapping another event.
- **player_overlap (soft)**: players rostered to this team also rostered to another team that has an overlapping event (cross-team via `team_members`).
- **official (hard)**: officials stored in `events.lineup_json`-adjacent `officials` array (add `officials JSONB` to events via migration) overlapping.
- SQL uses range overlap `tstzrange(start_at, end_at) && tstzrange($1,$2)`; an exclusion-constraint-style index supports it.
- Exposed at `POST /events:check-conflicts` (dry run) and run automatically on `POST/PATCH /events`, returning conflicts in the response `meta.conflicts` (hard conflicts → 409 unless `?force=true`).

**Testing**:
- `Unit: two events same venue overlapping → venue hard conflict; back-to-back non-overlapping → none`.
- `Unit: player on U15 and U17, both have overlapping events → player_overlap soft conflict listing player`.
- `Integration: POST event causing venue clash → 409 with meta.conflicts; with ?force=true → 201`.
- `Integration: PATCH event time to clear a clash → re-check returns no conflicts`.

#### 3.3 — Calendar export & sync

**What**: iCalendar export, CalDAV two-way sync, and push sync to Google Calendar / Microsoft Graph.

**Design**:
- `GET /teams/:id/calendar.ics` and `GET /users/:id/calendar.ics` → RFC 5545 `VCALENDAR` with one `VEVENT` per event (UID = event id, `DTSTART`/`DTEND` in UTC, `RRULE` preserved, RFC 7986 `COLOR`/`IMAGE`). Token-authenticated feed URL.
- CalDAV (RFC 4791) endpoint for two-way sync of personal feeds.
- Google Calendar + Microsoft Graph: OAuth-connected per user; a `calendar-sync` worker pushes create/update/cancel as events change (enqueued on event mutation). Sync actions audited with `actor_type='calendar_sync'`.

**Testing**:
- `Unit: event → VEVENT serialisation parses with an ical library; DTSTART in UTC`.
- `Integration: .ics feed for team → contains all scheduled events; cancelled events have STATUS:CANCELLED`.
- `Integration (mocked Google API): event created → sync job enqueued → mocked client receives insert with matching iCalUID`.

---

## Phase 4: Communication, RSVP & Notifications

### Purpose
Add the team communication hub — announcements, messaging, RSVP availability — and the multi-channel notification system that drives game-day engagement. After this phase players/parents receive reminders and respond to events, and coaches see availability.

### Tasks

#### 4.1 — RSVP & availability

**What**: Per-player, per-event availability tracking with summaries.

**Design**:
- Endpoints: `PUT /events/:id/rsvp` (self, body `{status, reason?}`), `GET /events/:id/rsvps` (coach view + counts), `POST /events/:id/rsvps/bulk-remind` (re-notify `no_response`).
- `event_rsvps` upsert on `(event_id, user_id)`; parents may RSVP on behalf of linked minors.
- Response includes `summary: {available, unavailable, maybe, no_response}`.

**Testing**:
- `Integration: player PUT rsvp available → row upserted, summary increments`.
- `Integration: parent RSVPs for linked minor → allowed; for unlinked minor → 403`.
- `Integration: bulk-remind enqueues notifications only for no_response users`.

#### 4.2 — Announcements & messaging

**What**: Team announcements and group/direct messaging with delivery tracking.

**Design**:
- `messages` and `message_deliveries` tables (added this phase):
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  club_id UUID NOT NULL REFERENCES clubs(id),
  team_id UUID REFERENCES teams(id),
  sender_id UUID NOT NULL REFERENCES users(id),
  kind TEXT NOT NULL CHECK (kind IN ('announcement','group','direct')),
  subject TEXT, body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE message_deliveries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  recipient_id UUID NOT NULL REFERENCES users(id),
  delivered_at TIMESTAMPTZ, read_at TIMESTAMPTZ,
  channel TEXT NOT NULL CHECK (channel IN ('in_app','push','email','sms')),
  UNIQUE (message_id, recipient_id, channel)
);
```
- Endpoints: `POST /teams/:id/announcements`, `GET /teams/:id/messages`, `POST /messages` (direct/group), `POST /messages/:id/read`.
- Minor-safety rule: direct messaging to a minor must include the linked parent as a recipient (configurable per club, default on, SafeSport-aligned).

**Testing**:
- `Integration: announcement to team → message_deliveries row per active member`.
- `Integration: direct message to minor without parent copy (club requires it) → 422`.
- `Integration: mark read → read_at set; GET messages shows unread count`.

#### 4.3 — Notification dispatch (Web Push / email / SMS)

**What**: A worker-driven dispatcher honouring per-user channel preferences.

**Design**:
- `notify` queue jobs: `{userId, template, data, channels?}`. Dispatcher resolves channels from `users.notification_prefs` intersected with requested channels.
- Providers: Web Push (VAPID), email (Resend/SES), SMS (Twilio). Each writes a `message_deliveries`-style record for game-day reminders.
- Scheduled reminders: on event create, enqueue delayed jobs at T-24h and T-2h (BullMQ delayed jobs).
- Quiet-hours and minor-specific rules respected (no SMS to minors by default).

**Testing**:
- `Unit: prefs {push:true,email:false} + requested [push,email] → only push dispatched`.
- `Integration (mocked Twilio/Resend/webpush): reminder job → each enabled provider called once`.
- `Integration: event created → two delayed reminder jobs scheduled at correct offsets`.

---

## Phase 5: Match Day — Stats, Live Scoring & Offline Mode

### Purpose
Enable the core in-game workflow: recording lineups, live scores, and per-player statistics — including offline match-day capture that syncs on reconnect. After this phase a coach can run a match end-to-end on a flaky venue connection and have data arrive intact.

### Tasks

#### 5.1 — Lineups & player stats

**What**: Set lineups and record per-player match/session statistics against the season's stat template.

**Design**:
- `PUT /events/:id/lineup` writes `events.lineup_json` (validated: jersey numbers unique, players belong to team).
- `PUT /events/:id/stats/:userId` upserts `player_stats` (unique `(event_id,user_id)`); `stats_json` validated against the season's `stat_template_json` (keys/types must match template; unknown keys rejected).
- `PATCH /events/:id` sets `score_home/away`, `result`, `status='completed'`.

**Testing**:
- `Unit: stats with key not in template → 422 listing unknown key; wrong type (string for integer) → 422`.
- `Integration: record stats for non-roster player → 422`.
- `Integration: complete match → result derived consistently with scores`.

#### 5.2 — Live scoring via SSE

**What**: Stream incremental score/stat updates to spectators.

**Design**:
- `GET /events/:id/live` → SSE stream; each update is a **CloudEvents 1.0** envelope (`type: com.stm.score.updated` / `com.stm.match.completed`). Channels described in `infra/asyncapi.yaml` (**AsyncAPI 3.0**).
- Score updates published to Redis pub/sub; SSE handler subscribes per event id. `Last-Event-ID` resumes from the last sequence.
- `POST /events/:id/score` (coach) appends an incremental score change, publishes to the channel.

**Testing**:
- `Integration: two SSE clients on same event → both receive a published score update`.
- `Integration: reconnect with Last-Event-ID → missed events replayed in order`.
- `Unit: score event serialises as valid CloudEvent`.

#### 5.3 — Offline match-day outbox & sync

**What**: Client buffers match-day intents offline; the API ingests a replayed batch idempotently.

**Design**:
- Client (web app, Phase 11) records intents in a Dexie outbox: `{clientIntentId (uuid), type, payload, recordedAt}` for `rsvp`, `lineup_set`, `score_update`, `stats_recorded`.
- Sync endpoint: `POST /sync/replay` accepts an ordered batch; server processes each intent idempotently keyed on `clientIntentId` (dedupe table `sync_intents(client_intent_id PK, processed_at)`); conflicts resolved last-write-wins per field with the original `recordedAt` preserved; all writes audited `actor_type='offline_sync'` with `metadata.recorded_at`.
- Events created offline carry `is_offline_created=true`.

**Testing**:
- `Unit: replaying the same batch twice → second is a no-op (idempotent on clientIntentId)`.
- `Integration: offline batch [score 1-0, score 2-1, completed] → final event state 2-1 completed, audit rows actor_type=offline_sync`.
- `Integration: out-of-order recordedAt resolved by timestamp, not arrival order`.

---

## Phase 6: Performance Analytics & Benchmarking

### Purpose
Turn raw stats into insight: season aggregates, trend analysis, and positional benchmarking — the mid-market differentiator over consumer apps. After this phase coaches see player/team trends and comparisons against positional baselines.

### Tasks

#### 6.1 — Aggregation & trend queries

**What**: Season and career stat aggregations per player and team.

**Design**:
- Service `stats/aggregate.ts` runs the Suggestion 1 aggregation queries (e.g. season totals/averages over `player_stats.stats_json`), respecting each field's `aggregate` directive from the stat template (sum vs avg vs max).
- Endpoints: `GET /seasons/:id/leaderboard?metric=`, `GET /users/:id/stats?seasonId=&teamId=`, `GET /users/:id/stats/trend?metric=&window=` (per-event series for charting).
- Generic JSONB extraction casts by template field type; results cached in Redis keyed on `(seasonId, version)` and invalidated on stat writes.

**Testing**:
- `Unit: aggregate respects directive — minutes(sum) vs rating(avg) vs top_speed(max)`.
- `Integration (seeded): leaderboard by goals → ordered desc, games_played correct`.
- `Integration: recording new stats invalidates the cached season aggregate`.

#### 6.2 — Positional benchmarking

**What**: Compare a player to positional baselines within the season/club.

**Design**:
- Baseline = distribution of a metric across all players sharing `position` in the season (or club-wide if sample < N). Return player value, percentile, median, and p75/p90.
- `GET /users/:id/benchmark?seasonId=&metric=` → `{value, percentile, cohortSize, median, p75, p90}`.
- Only `benchmarkable: true` template fields are eligible.

**Testing**:
- `Unit: percentile of known distribution computed correctly (e.g. value at p90)`.
- `Integration: cohort smaller than threshold falls back to club-wide cohort, flagged in response`.
- `Unit: non-benchmarkable metric → 422`.

#### 6.3 — Season analytics read model (optional projection)

**What**: A precomputed `rm_season_analytics` table (from Suggestion 3) maintained by a worker for the dashboard.

**Design**:
- A `projections` worker recomputes `rm_season_analytics` (record, top performers, team stats, attendance) on match-complete and stat-write events; serves `GET /seasons/:id/analytics` as a single-row read.
- Idempotent recompute keyed on season; `projection_checkpoint` tracks last processed event.

**Testing**:
- `Integration: complete a match → projection updates wins/goals_for within the job`.
- `Integration: rebuild projection from scratch matches on-the-fly aggregation`.

---

## Phase 7: Payments & Financial Management

### Purpose
Collect registration fees and dues and track team finances via Stripe Connect. After this phase clubs can take payments online and reconcile per-team expenses, keeping PCI scope minimal.

### Tasks

#### 7.1 — Stripe Connect onboarding & registration payments

**What**: Connect club accounts and collect registration/dues with hosted checkout.

**Design**:
- Club onboarding: `POST /clubs/:id/stripe/connect` → Account Link; stores `stripe_account_id`.
- `POST /payments` creates a `payments` row (`status='pending'`) and a Stripe Checkout Session (destination charge to the connected account); returns the checkout URL. Idempotency-Key used per `standards.md`.
- Payment types per Suggestion 1 enum (registration, membership_dues, kit_order, …).

**Testing**:
- `Integration (Stripe test mode/mock): create payment → pending row + checkout URL; idempotency key reused → same session`.
- `Integration: payment for another club's user → 403`.

#### 7.2 — Webhook ingestion & reconciliation

**What**: Handle Stripe webhooks to settle payment state.

**Design**:
- `POST /webhooks/stripe` verifies signature (`STRIPE_WEBHOOK_SECRET`), enqueues a `stripe` job. Worker maps `checkout.session.completed` → `paid`, `charge.refunded` → `refunded`, etc.; sets `paid_at`, `stripe_payment_id`. All wrapped as CloudEvents internally and audited `actor_type='stripe_webhook'`.
- Idempotent on Stripe event id.

**Testing**:
- `Integration: valid signed event → payment marked paid; replayed event → no double-processing`.
- `Integration: invalid signature → 400, no job enqueued`.

#### 7.3 — Financial reporting

**What**: Per-team and per-club financial summaries.

**Design**:
- `GET /clubs/:id/finance?teamId=&from=&to=` → totals by type, outstanding vs collected, registration completion rate. Money always in integer `amount_cents` + `currency`.

**Testing**:
- `Integration (seeded): finance summary totals match seeded payments; outstanding excludes waived`.

---

## Phase 8: Compliance — GDPR / COPPA / SafeSport

### Purpose
Make privacy for minors first-class: parental-consent workflows, SafeSport clearance tracking, consent-gated access, and right-to-erasure tooling. This is the highest-risk area per `standards.md` and gates several earlier features. After this phase the platform can lawfully operate youth rosters.

### Tasks

#### 8.1 — Parental consent & minor lifecycle

**What**: Consent capture/verification and minor-account gating.

**Design**:
- On registering a user with `date_of_birth` making them a minor, set `is_minor=true` and require a `parent_user_id`; minor cannot be activated until `parental_consent_at` recorded.
- Endpoints: `POST /users/:id/consent` (parent grants/revokes; verified via email/parent auth), `GET /users/:id/consent-status`.
- Consent state changes recorded both on `users` and as audit events (`parental_consent_granted/revoked`).
- Guard `requireMinorConsent(userId)` used by roster activation (2.3), messaging (4.2), and any minor-data read.

**Testing**:
- `Unit: DOB 14 years ago (relative to currentDate 2026-05-30) → is_minor=true`.
- `Integration: activate minor without consent → 409; after consent → allowed`.
- `Integration: revoke consent → minor reads gated, audit row written`.

#### 8.2 — SafeSport clearance tracking

**What**: Track and enforce SafeSport clearance for adults working with minors.

**Design**:
- `POST /users/:id/safesport` records clearance + `safesport_cleared_at`; coaches/medical without active clearance are blocked from being added to teams containing minors (configurable per club).

**Testing**:
- `Integration: add uncleared coach to minor-containing team → 409 "safesport_required"`.
- `Integration: cleared coach → allowed`.

#### 8.3 — Right-to-erasure & data export

**What**: GDPR data export and erasure for a user.

**Design**:
- `POST /users/:id/gdpr/export` → enqueues a job assembling all of the user's PII (profile, stats, RSVPs, messages, payments metadata) into a downloadable JSON/CSV bundle; link emailed.
- `POST /users/:id/gdpr/erase` → anonymises PII in place (replace name/email/contact with tombstones, drop avatar/documents from S3) while preserving aggregate stat history keyed to an anonymised id, so team records stay intact (per Suggestion 1 key decision #1). Erasure itself audited with `involves_minor` set appropriately.

**Testing**:
- `Integration: export bundle includes user's stats and RSVPs; excludes other users' PII`.
- `Integration: erase user → name/email tombstoned, documents deleted from MinIO, player_stats rows still present with anonymised id`.
- `Integration: post-erasure audit row exists and is immutable`.

---

## Phase 9: Training, Injuries & Scouting

### Purpose
Round out the operations suite with training planning + load monitoring, the injury/availability tracker, and the scouting database with attribute ratings and similarity search. After this phase the platform matches enterprise feature breadth for mid-market clubs.

### Tasks

#### 9.1 — Training planner & load monitoring

**What**: Design practice sessions with a drill library, log attendance, and track training load.

**Design**:
- `drills` table (club-scoped library: name, category, description, default_duration, intensity). Practice events gain a `training_json` payload (drills, total_load_au, session_rpe, per-player attendance + load) — mirrors Suggestion 2's `training_json` shape, attached to the `events` row.
- Endpoints: `GET/POST /clubs/:id/drills`, `PUT /events/:id/training`, `GET /users/:id/load?window=` (acute:chronic workload ratio).
- Load = sum of per-session `load_au`; ACWR = 7-day acute / 28-day chronic average.

**Testing**:
- `Unit: ACWR computed correctly for a known load series; ratio >1.5 flagged high`.
- `Integration: log training attendance → reflected in user load endpoint`.

#### 9.2 — Injury & availability tracker

**What**: Record injuries, return-to-play protocols, and availability.

**Design**:
- Full CRUD over the Suggestion 1 `injuries` table. Endpoints: `POST /users/:id/injuries`, `GET /teams/:id/injuries?status=`, `PATCH /injuries/:id` (status transitions `active → recovering → cleared`, or `chronic`), clearance requires `medical_staff` role.
- An active/recovering injury marks the player's roster availability and surfaces on the roster view (joins per Suggestion 1's roster query). Medical notes are minor-sensitive → audited with `involves_minor`.

**Testing**:
- `Unit: status transition cleared requires cleared_by medical staff → else 403`.
- `Integration: active injury appears on team injury list and flags roster availability`.

#### 9.3 — Scouting database & similarity search

**What**: Prospect records with attribute ratings and pgvector similarity search.

**Design**:
- CRUD over `prospects` (Suggestion 1). On `prospect_evaluated`, compute an attribute embedding (normalise the rated attributes into a `vector(384)`; for richer cases combine with stat embeddings) stored in `prospects.attribute_embedding`.
- `GET /prospects/:id/similar` → pgvector cosine nearest neighbours among prospects and (optionally) current players, returning `{id, name, similarity, sharedAttributes}` (Suggestion 3 scouting board shape). `comparison_json` cached.
- `GET /clubs/:id/prospects?status=&sport=&minOverall=` filterable board.

**Testing**:
- `Unit: identical attribute vectors → similarity ~1.0; orthogonal → ~0`.
- `Integration (real pgvector): seed 3 prospects → similar returns nearest first`.
- `Integration: status transition watching→signed audited`.

---

## Phase 10: AI-Native Layer & MCP Server

### Purpose
Deliver the AI-native advantages that justify the project: NL stat queries, LLM recaps/parent emails, smart-scheduling suggestions, injury-risk warnings, video auto-tagging, and an MCP server exposing team data to assistants. After this phase the product's headline differentiators are live.

### Tasks

#### 10.1 — LLM provider abstraction & prompt templates

**What**: A provider-agnostic LLM client and the prompt templates that power text generation, persisting outputs to `ai_suggestions`.

**Design**:
- `ai/provider.ts`: `generate({system, user, schema?, model?})` over the Vercel AI SDK; structured outputs validated with zod. `LLM_PROVIDER` selects Anthropic/OpenAI/local.
- Every generation writes an `ai_suggestions` row (`suggestion_type`, `body`, `suggestion_data`, `llm_model`, `tokens_used`, `is_applied`, `is_dismissed`).
- Prompt template — **match recap** (system):
  > You are a sports reporter for an amateur club. Write a concise, factual {sport} match recap (120-180 words) from the structured data. Use only provided facts. Neutral, encouraging tone. Do not invent statistics. Refer to minors by first name only.
  user payload = `{teams, score, scorers, key_stats, top_performers}`.
- Prompt template — **parent email**: system constrains to logistics + encouragement, no medical/PII speculation, first names for minors.
- Endpoints: `POST /events/:id/ai/recap`, `POST /events/:id/ai/parent-email`, returning a draft `ai_suggestion`; `POST /ai/suggestions/:id/apply|dismiss`.

**Testing**:
- `Unit (mocked LLM): recap call → ai_suggestions row with tokens_used and model recorded`.
- `Unit: structured-output schema mismatch → retried then surfaced as 502`.
- `Integration: apply suggestion → is_applied=true; audit actor_type='ai'`.

#### 10.2 — Natural-language stat queries

**What**: "Show me Sam's pass completion in the last 5 games" answered over team data.

**Design**:
- A constrained NL→query pipeline: LLM maps the question to a typed `StatQuery` DTO (`{playerRef, metric, scope, window}`) — **never** raw SQL — which a deterministic resolver executes against `player_stats` via the Phase 6 aggregation service. Result + a one-line NL summary returned and stored as `ai_suggestions(type='stat_query_response')`.
- Endpoint: `POST /teams/:id/ai/query` body `{question}`.
- Guardrails: resolver enforces club scoping and minor-consent gating regardless of LLM output; unknown players/metrics → clarifying response, not a guess.

**Testing**:
- `Unit (mocked LLM): question → StatQuery DTO; resolver returns correct aggregate from seeded data`.
- `Unit: LLM emits a metric not in the template → resolver returns "metric unavailable", no crash`.
- `Integration: query referencing a minor without consent → gated`.

#### 10.3 — Smart scheduling & injury-risk suggestions

**What**: AI-assisted conflict resolution proposals and injury-risk early warnings.

**Design**:
- Smart scheduling: given detected conflicts (Phase 3) and historical club preferences (preferred days/venues mined from past events), the LLM proposes alternative slots; output is a structured list of candidate `{startAt, venue}` re-validated through `detectConflicts`. Stored as `suggestion_type='schedule_conflict'`.
- Injury risk: a rules+model hybrid — flag players whose ACWR (9.1) exceeds threshold and whose attendance/load trend is adverse; the LLM drafts the human-readable warning. Stored as `suggestion_type='injury_risk'`, surfaced on the season analytics dashboard.

**Testing**:
- `Unit: proposed alternative slots all pass detectConflicts (no hard conflicts re-introduced)`.
- `Unit: player with ACWR 1.8 + declining attendance → flagged; normal load → not flagged`.

#### 10.4 — Video upload & auto-tagging (vision sidecar)

**What**: Upload match video and auto-tag events (goals, key plays) using open vision models.

**Design**:
- `POST /teams/:id/videos` → presigned S3 upload; on confirm, enqueue a `video` job. The Node worker calls the **Python `vision-worker`** (FastAPI) which runs an open detection+embedding pipeline and returns candidate tags `[{timestamp_sec, label, confidence}]`.
- Tags written to `videos.tags_json` with `ai_generated=true`; coaches confirm/edit (human tags `ai_generated=false`). Highlight clips assembled from confirmed tags. Clean-room: rely on open models only, no patented auto-camera/sync mechanisms (per IP summary).
- Video embeddings (pgvector) feed scouting similarity (9.3).

**Testing**:
- `Integration (mocked vision-worker): video confirm → tags_json populated, is_ai_tagged=true`.
- `Integration: coach edits a tag → ai_generated flips to false on that tag`.
- `E2E (sample clip, optional/real): pipeline returns ≥1 tag with timestamp within clip bounds`.

#### 10.5 — MCP server

**What**: An MCP server exposing read/query/draft tools to LLM assistants.

**Design**:
- `apps/mcp` using `@modelcontextprotocol/typescript-sdk`. Tools (per `standards.md`): `get_roster(teamId)`, `query_stats(teamId, question)` (delegates to 10.2), `draft_match_recap(eventId)` (delegates to 10.1), `find_schedule_conflicts(teamId, proposedEvent)` (delegates to 3.2).
- Auth: the MCP server authenticates as a club-scoped service principal; every tool enforces the same RBAC, club scoping, and minor-consent gating as the HTTP API by reusing `@stm/core` services. All tool calls audited `actor_type='ai'`.

**Testing**:
- `Integration: MCP get_roster → same payload as HTTP roster endpoint for the same team`.
- `Integration: query_stats via MCP respects minor-consent gating`.
- `Unit: tool schemas validate against MCP spec`.

---

## Phase 11: Web PWA, Native Shell & Offline UX

### Purpose
Deliver the user-facing application: a Next.js PWA for coaches, players, and parents, wrapped in a Capacitor native shell, with the offline match-day experience wired to Phase 5's sync. After this phase the product is usable end-to-end by its real personas.

### Tasks

#### 11.1 — App shell, auth, and role-based navigation

**What**: Next.js App Router shell consuming `@stm/sdk`, with login (incl. passkeys/OIDC) and role-aware navigation.

**Design**:
- Routes: `/login`, `/dashboard`, `/teams/[id]` (roster, schedule, analytics tabs), `/events/[id]` (RSVP, lineup, live, stats), `/players/[id]`, `/scouting`, `/finance`, `/admin`. Role gates hide admin/finance from players/parents.
- shadcn/ui + Tailwind; WCAG 2.2 AA (keyboard nav, focus management, contrast, labelled controls).
- Token storage: refresh in httpOnly cookie (web) / secure storage (Capacitor); silent refresh.

**Testing**:
- `E2E (Playwright): login as coach → sees Finance; login as parent → Finance hidden`.
- `E2E: axe-core accessibility scan on dashboard → no critical violations`.

#### 11.2 — PWA service worker & offline match-day

**What**: Installable PWA with offline caching and the match-day outbox UI.

**Design**:
- Workbox service worker: app shell precache + runtime caching of roster/event data for the active team. Match-day screen reads cached roster, writes intents to the Dexie outbox (5.3), shows an "N pending changes" indicator, and flushes via `POST /sync/replay` on reconnect.
- Manifest enables install; live-score screen consumes the SSE stream when online.

**Testing**:
- `E2E (Playwright offline): go offline → record score + stats → outbox shows pending → go online → /sync/replay called → server state matches`.
- `E2E: PWA installable (manifest + SW registration assertions)`.

#### 11.3 — Capacitor native shell & push

**What**: iOS/Android wrappers with native push notifications.

**Design**:
- Capacitor wraps the Next.js export; native push tokens registered against the user for the notify dispatcher (4.3). Deep links open the relevant event/message.

**Testing**:
- `Build: Capacitor iOS + Android builds succeed in CI (no device run required)`.
- `Integration (mocked): native push token registration stored against user`.

---

## Phase 12: Hardening, Interop Export & Deployment

### Purpose
Production-readiness: security hardening, the open data-interchange export the project champions, observability, and packaged self-hosted + cloud deployment. After this phase the platform is shippable and self-hostable.

### Tasks

#### 12.1 — Security hardening (OWASP ASVS / API Top 10)

**What**: Apply OWASP ASVS 4.0.3 and API Security Top 10 controls.

**Design**:
- Rate limiting (Redis) per IP + per user; strict CORS; security headers (HSTS, CSP, X-Content-Type-Options); per-object authorization checks audited (BOLA prevention — API Top 10 #1); input validation already enforced via schemas; secrets only from env; dependency and SAST scans in CI.
- A scripted ASVS checklist mapped to tests.

**Testing**:
- `Integration: user A requests user B's stats (different club) → 403 (BOLA)`.
- `Integration: exceed rate limit → 429 RFC 9457`.
- `Security: CI runs npm audit + semgrep; high findings fail the build`.

#### 12.2 — Open interchange export (JSON Schema + SportsML-G2)

**What**: Publish the open export format the README/`standards.md` notes call for.

**Design**:
- `GET /teams/:id/export?format=json|sportsml` — a JSON-Schema-2020-12-documented bundle (roster, schedule, results, season stats) and a best-effort **SportsML-G2** XML rendering for rosters/results. Schema published under `infra/`.

**Testing**:
- `Unit: export validates against the published JSON Schema`.
- `Unit: SportsML-G2 output is well-formed XML and validates against the IPTC schema for roster/results`.

#### 12.3 — Observability & health

**What**: Tracing, metrics, structured logs, health/readiness probes.

**Design**:
- OpenTelemetry across API → queue → worker → vision-worker; pino logs with request/trace ids; `/healthz` (liveness) and `/readyz` (DB/Redis/S3 checks). Audit-log integrity check job.

**Testing**:
- `Integration: a request that enqueues an LLM job produces a single linked trace across API and worker`.
- `Integration: /readyz returns 503 when Postgres is down`.

#### 12.4 — Deployment packaging (cloud + self-hosted)

**What**: Production Docker images and compose/Helm for both deployment modes.

**Design**:
- Multi-stage `Dockerfile.api`/`.worker`/`.web` and the Python `vision-worker` image. `docker-compose.prod.yml` wires api, worker, vision-worker, web, postgres (pgvector), redis, minio with healthchecks and migration init job. Self-hosted profile defaults LLM to a local provider and storage to MinIO. A short ops runbook (env, migrations, backups, restore).

**Testing**:
- `Smoke: docker compose -f docker-compose.prod.yml up → /readyz green; seed; login; create event end-to-end`.
- `Build: all images build in CI; image vulnerability scan passes thresholds`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (schemas, DB, auth)        ─── required by everything
    │
Phase 2: Org model (clubs, users, teams, RBAC, audit)  ─── requires 1
    │
    ├── Phase 3: Scheduling & conflict detection ─── requires 2
    │       │
    │       ├── Phase 4: Comms, RSVP, notifications ── requires 3 (events) ┐ can parallel
    │       └── Phase 5: Match day, live scoring, offline ── requires 3    ┘ (4 ∥ 5)
    │                 │
    │                 └── Phase 6: Analytics & benchmarking ── requires 5 (stats)
    │
    ├── Phase 7: Payments ─────────────── requires 2; ∥ with 3/4/5
    │
    └── Phase 8: Compliance (GDPR/COPPA/SafeSport) ── requires 2; gates 2.3/4.2 (build early)
         │
Phase 9: Training, injuries, scouting ── requires 2 + 5 (events/stats); 9.3 needs pgvector (1)
    │
Phase 10: AI layer + MCP ── requires 6 (stats), 3 (conflicts), 9 (video→scouting)
    │
Phase 11: Web PWA + native shell + offline ── requires 5 (sync) + most API modules (2-10)
    │
Phase 12: Hardening, interop export, deployment ── requires all; final
```

Parallelism opportunities:
- **Phases 4 and 5** can be built concurrently once Phase 3 lands.
- **Phase 7 (Payments)** and **Phase 8 (Compliance)** depend only on Phase 2 and can proceed alongside Phases 3-5. Build Phase 8's consent guard early since it gates roster activation (2.3) and minor messaging (4.2).
- Within Phase 10, sub-tasks 10.1/10.2/10.3 (text/query/scheduling) can parallel 10.4 (video), which depends on the vision sidecar.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test` green; Testcontainers integration suites included).
3. `pnpm lint` and `pnpm format:check` pass with no errors.
4. `pnpm typecheck` (`tsc --noEmit`) passes across affected packages.
5. New or changed database tables have a committed drizzle migration; `db:migrate` up/down verified on a clean DB.
6. The phase's feature works end-to-end against the dev `docker-compose` stack (manual or Playwright walkthrough).
7. New config/env keys are added to `.env.example` and documented.
8. New API endpoints appear in the generated `infra/openapi.json` (`@fastify/swagger`) and the `@stm/sdk` client regenerates cleanly.
9. New realtime channels appear in `infra/asyncapi.yaml` where applicable.
10. All mutating operations write an `audit_log` entry with the correct `involves_minor` flag; minor-data paths verified against Phase 8 guards.
11. Docker images affected by the phase build successfully in CI.
12. WCAG 2.2 AA checks (axe-core) pass for any new user-facing screens (Phase 11 onward).
