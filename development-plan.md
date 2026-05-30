# Corporate Governance Platform — Phased Development Plan

> Project: 435-corporate-governance-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the primary schema: a normalized relational core (boards, meetings, votes, users, entities — taken from Suggestion 1) with JSONB columns for jurisdiction-variable and fast-evolving data (questionnaire definitions, annotation geometry, audit snapshots, AI metadata). This balances referential integrity for compliance-critical relationships against the schema flexibility a multi-jurisdiction governance domain demands. Suggestion 4's graph queries are deferred to a backlog phase (entity-hierarchy analytics) rather than introducing polyglot operational complexity into the MVP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js 22 LTS) | API-and-integration-heavy product (SSO, DocuSign, calendar, EDGAR, MCP). One language across API, web frontend, and shared types reduces duplication. Strong typing matters for compliance correctness. |
| Language (AI workers) | TypeScript via Vercel AI SDK | Keeps the stack uniform; AI SDK gives provider-agnostic LLM access (OpenAI, Anthropic, Azure OpenAI) so self-hosters are not locked to one vendor — a stated differentiator over Diligent's proprietary AI. |
| API framework | NestJS | Opinionated, modular DI architecture suits a 40+-table domain split into bounded modules (boards, meetings, documents, compliance). First-class OpenAPI 3.1 generation via `@nestjs/swagger` satisfies the standards requirement. Guards/interceptors give clean RBAC and audit hooks. |
| API style | REST + OpenAPI 3.1, JSON:API-influenced resource shapes | `standards.md` cites OpenAPI 3.1 and JSON:API v1.0 as industry norms (Diligent's APIs). REST is the lowest-friction surface for the open developer ecosystem this project targets. |
| Database | PostgreSQL 16 | Suggestion 3. Relational integrity for governance relationships + JSONB for variable data + native Row-Level Security for multi-tenant isolation + tsvector full-text search for documents/resolutions. |
| ORM / migrations | Prisma | Type-safe client generating TypeScript types shared with the API layer; first-class migration tooling; JSONB support; works cleanly with Postgres RLS via session variables. |
| Object storage | S3-compatible (MinIO self-hosted; AWS S3 / Azure Blob in cloud) | Board packs are large encrypted binaries that must not live in the DB. Pluggable provider abstraction supports self-hosted (data sovereignty) and cloud modes. |
| Encryption | AES-256-GCM envelope encryption; KMS-pluggable (local keyfile, AWS KMS, Azure Key Vault, HashiCorp Vault) | README promises end-to-end encryption with zero-knowledge key-management options. Per-document data keys wrapped by a KMS-held master key. |
| Task queue | BullMQ on Redis | Async workloads: AI minute drafting, document text extraction, calendar sync, EDGAR polling, notification fan-out, e-signature webhooks. Redis also serves rate-limiting and ephemeral cache. |
| Auth | Passport.js + `node-saml` (SAML 2.0) + `openid-client` (OIDC) + `@simplewebauthn/server` (WebAuthn/FIDO2) | `standards.md` mandates SAML 2.0, OIDC, SCIM 2.0, WebAuthn. JWT (RFC 7519) for API sessions; refresh-token rotation. |
| SCIM | `scimgateway`-style custom controller | RFC 7644 user provisioning/deprovisioning — compliance-critical for enterprise adoption. |
| Frontend | Next.js 16 (App Router) + React 19 + shadcn/ui + Tailwind | Mobile-responsive PWA with offline board-pack access (service worker + IndexedDB cache) satisfies the "mobile + offline" table-stakes feature without separate native apps in v1. SSR for fast director access. |
| PDF rendering / annotation | `pdf.js` (render) + custom annotation layer persisting to `document_annotations` | Native annotation is table-stakes; pdf.js is the standard open web PDF engine. |
| AI integration | Vercel AI SDK + an MCP server (`@modelcontextprotocol/sdk`) | `standards.md` highlights MCP as a differentiator — expose governance tools to any MCP-compatible assistant rather than locking in one LLM. |
| Containerisation | Docker + docker-compose (dev/self-host); Helm chart (k8s, backlog) | Self-hosted deployment is the core value proposition. |
| Testing | Vitest (unit), Supertest (API integration), Playwright (E2E), Testcontainers (real Postgres/Redis/MinIO) | Standard TS ecosystem; Testcontainers gives realistic integration tests for RLS and storage behaviour. |
| Code quality | ESLint + Prettier + `tsc --noEmit` + `eslint-plugin-security` | OWASP Top 10 alignment (`standards.md`) needs static security linting. |
| Monorepo tooling | pnpm workspaces + Turborepo | Shared types between `api`, `web`, and `mcp-server` packages. |
| Observability | OpenTelemetry + structured JSON logs (pino) | Audit and SOC 2 evidence; trace AI calls and integration latency. |

### Project Structure

```
corporate-governance-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # postgres, redis, minio, api, web, worker
├── Dockerfile.api
├── Dockerfile.web
├── Dockerfile.worker
├── .env.example
├── packages/
│   ├── shared/                       # Cross-package TS types, Zod schemas, constants
│   │   └── src/
│   │       ├── enums.ts              # role, status, classification enums
│   │       ├── dto/                  # shared request/response DTOs
│   │       └── permissions.ts        # permission catalogue
│   ├── db/                           # Prisma schema + migrations + seed
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   └── src/rls.ts                # RLS session-variable helpers
│   ├── api/                          # NestJS application
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/
│   │       │   ├── guards/           # AuthGuard, RbacGuard, RlsInterceptor
│   │       │   ├── interceptors/     # AuditInterceptor
│   │       │   ├── crypto/           # envelope encryption, KMS providers
│   │       │   └── audit/            # audit-log service
│   │       ├── modules/
│   │       │   ├── auth/             # SAML, OIDC, WebAuthn, JWT, SCIM
│   │       │   ├── organisations/    # entities, subsidiaries, relationships
│   │       │   ├── users/
│   │       │   ├── rbac/             # roles, permissions, scoped grants
│   │       │   ├── boards/           # boards, committees, memberships
│   │       │   ├── documents/        # repository, folders, versions, perms, annotations
│   │       │   ├── meetings/         # meetings, agendas, attendees, minutes
│   │       │   ├── voting/           # votes, vote-records, resolutions, action-items
│   │       │   ├── signatures/       # built-in + DocuSign
│   │       │   ├── compliance/       # insider trading, disclosures, questionnaires, filings
│   │       │   ├── effectiveness/    # skills, evaluations, board composition
│   │       │   ├── messaging/        # secure threads
│   │       │   ├── calendar/         # iCal, Outlook, Google sync
│   │       │   ├── ai/               # minute drafting, summarisation, COI detection
│   │       │   └── notifications/
│   │       └── jobs/                 # BullMQ processors
│   ├── web/                          # Next.js 16 frontend (PWA)
│   │   └── src/app/
│   └── mcp-server/                   # MCP server exposing governance tools
│       └── src/
├── infra/
│   ├── helm/                         # k8s chart (backlog)
│   └── otel/
└── tests/
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation, Multi-Tenancy & Data Model

### Purpose
Establish the monorepo, the PostgreSQL schema with the hybrid relational+JSONB model, multi-tenant isolation via Row-Level Security, the shared type/permission catalogue, and the container environment. After this phase the database can be migrated and seeded, RLS provably isolates tenants, and every later module has a typed foundation to build on.

### Tasks

#### 1.1 — Monorepo, tooling, and container environment

**What**: Scaffold the pnpm/Turborepo workspace with `shared`, `db`, `api`, `web`, `mcp-server` packages and a working `docker-compose` stack (Postgres 16, Redis 7, MinIO).

**Design**:
- `docker-compose.yml` services: `postgres` (16, with `POSTGRES_DB=cgp`), `redis`, `minio` (console + API ports), `api`, `web`, `worker`. Healthchecks gate `api`/`worker` startup on DB readiness.
- `.env.example` keys (load-bearing):
  - `DATABASE_URL=postgresql://cgp:cgp@postgres:5432/cgp`
  - `REDIS_URL=redis://redis:6379`
  - `S3_ENDPOINT`, `S3_BUCKET=board-packs`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_REGION`
  - `KMS_PROVIDER=local` (`local|aws|azure|vault`), `KMS_LOCAL_MASTER_KEY` (base64 32-byte)
  - `JWT_SECRET`, `JWT_ACCESS_TTL=900`, `JWT_REFRESH_TTL=2592000`
  - `AI_PROVIDER=anthropic` (`anthropic|openai|azure`), `AI_API_KEY`, `AI_MODEL`
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test`, `test:integration`.
- Root `tsconfig` with project references; `eslint` config includes `eslint-plugin-security`.

**Testing**:
- `Unit: env loader rejects missing DATABASE_URL → throws with key name in message`.
- `Integration: docker-compose up → postgres healthcheck passes within 60s`.
- `Smoke: pnpm -r typecheck exits 0 on empty scaffold`.
- `Smoke: api boots, GET /health → 200 {status:"ok", db:"up", redis:"up"}`.

#### 1.2 — Prisma schema (hybrid relational + JSONB core)

**What**: Define the full Prisma schema covering organisations/entities, users, RBAC, boards/committees, meetings/agendas/minutes, voting/resolutions/action-items, documents/folders/annotations, signatures, compliance (insider trading, disclosures, questionnaires, filings), effectiveness, messaging, notifications, calendar, and audit log.

**Design**:
- Adopt Suggestion 1's relational tables as Prisma models; apply Suggestion 3's JSONB choices for: `questionnaires.templateContent`, `questionnaireResponses.responses`, `documentAnnotations.positionData`, `auditLog.oldValues/newValues`, `userSsoIdentities.metadata`, and a generic `metadata Json?` on `documents`, `meetings`, and `resolutions` for jurisdiction-specific extension.
- Key enums live in `packages/shared/src/enums.ts` and are mirrored as Prisma enums: `OrganisationType`, `MemberRole`, `BoardRole`, `MeetingStatus`, `VoteType`, `VoteStatus`, `ResolutionType`, `DocumentClassification`, `PreclearanceStatus`, `InsiderType`, `AuditSeverity`.
- Representative model (Prisma):
  ```prisma
  model Meeting {
    id              String   @id @default(uuid()) @db.Uuid
    organisationId  String   @db.Uuid
    boardId         String?  @db.Uuid
    committeeId     String?  @db.Uuid
    title           String   @db.VarChar(500)
    meetingType     MeetingType   @default(regular)
    status          MeetingStatus @default(draft)
    scheduledStart  DateTime? @db.Timestamptz
    scheduledEnd    DateTime? @db.Timestamptz
    quorumMet       Boolean?
    metadata        Json?
    createdBy       String   @db.Uuid
    createdAt       DateTime @default(now()) @db.Timestamptz
    updatedAt       DateTime @updatedAt @db.Timestamptz
    deletedAt       DateTime? @db.Timestamptz
    organisation    Organisation @relation(fields: [organisationId], references: [id])
    agendaItems     AgendaItem[]
    attendees       MeetingAttendee[]
    @@index([organisationId])
    @@index([scheduledStart])
  }
  ```
- Every tenant-scoped table carries `organisationId`. Soft-delete (`deletedAt`) on all governance records — physical deletion is forbidden (compliance retention).
- `auditLog` uses `BigInt` autoincrement PK and is created as a monthly **range-partitioned** table via a raw SQL migration appended after `prisma migrate` (Prisma cannot express partitioning).

**Testing**:
- `Unit: generated Prisma client compiles; types for Meeting include metadata: JsonValue | null`.
- `Integration (Testcontainers Postgres): migrate deploy → all 40+ tables + enums present (query information_schema)`.
- `Integration: insert meeting without organisationId → DB rejects (NOT NULL)`.
- `Integration: self-referential entity_relationship parent==child → CHECK violation`.
- `Integration: audit_log is partitioned (pg_partitioned_table contains audit_log)`.

#### 1.3 — Row-Level Security & tenant context

**What**: Enable Postgres RLS on all tenant-scoped tables and wire a request-scoped tenant context that sets `SET LOCAL app.current_org` per transaction.

**Design**:
- Raw SQL migration: `ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;` plus policy `USING (organisation_id = current_setting('app.current_org')::uuid)` on each tenant table. A `bypass_rls` role is used only by migrations and the SCIM/admin provisioning path.
- `packages/db/src/rls.ts`: `withTenant(prisma, orgId, fn)` opens a transaction, runs `SET LOCAL app.current_org = $orgId` and `SET LOCAL app.current_user = $userId`, executes `fn`, commits.
- NestJS `RlsInterceptor` reads the authenticated org from the JWT and wraps the request's Prisma calls via an `AsyncLocalStorage` tenant store.

**Testing**:
- `Integration: seed two orgs A,B; query meetings under tenant A → returns only A's rows`.
- `Integration: attempt cross-tenant read by passing B's id with A's session → 0 rows (policy denies)`.
- `Integration: missing app.current_org setting → query returns 0 rows (fail-closed, not fail-open)`.
- `Unit: withTenant rolls back transaction on thrown error`.

#### 1.4 — Permission catalogue & seed data

**What**: Define the canonical permission list and seed system roles, base permissions, and a demo organisation.

**Design**:
- `packages/shared/src/permissions.ts`: `PERMISSIONS` as `{ resourceType, action }[]` covering `board|meeting|document|entity|trading|disclosure|questionnaire|signature|messaging *|admin` × `read|write|delete|approve|admin`.
- `seed.ts` inserts: system roles (`owner`, `admin`, `company_secretary`, `director`, `officer`, `observer`, `auditor`) with `isSystem=true`; full `permissions` catalogue; `role_permissions` mappings (e.g. `director` gets `meeting:read`, `document:read`, `document:annotate`, vote-cast; `company_secretary` gets write across meetings/documents/minutes; `auditor` gets read-only + `audit:read`).
- Demo org with one board, three users (chair, secretary, director).

**Testing**:
- `Integration: seed → 7 system roles exist; director role lacks document:delete`.
- `Unit: permission catalogue has no duplicate (resourceType,action) pairs`.

### Definition of Done
Migrations apply cleanly; RLS isolation proven by tests; seed produces a working demo tenant; typecheck/lint/`pnpm test:integration` green; `docker-compose up` yields a healthy stack.

---

## Phase 2: Identity, Authentication & RBAC

### Purpose
Deliver enterprise-grade authentication (SAML 2.0, OIDC, WebAuthn MFA, JWT API sessions), SCIM provisioning, and the scoped RBAC enforcement layer. Without this nothing else can be securely exposed; it is foundational and ships early because every competitor treats SSO/MFA as table-stakes and `standards.md` mandates these protocols.

### Tasks

#### 2.1 — Local auth, JWT sessions & MFA (TOTP/WebAuthn)

**What**: Password login (argon2id), JWT access/refresh with rotation, TOTP enrolment, and WebAuthn (FIDO2) registration/assertion.

**Design**:
- `POST /auth/login {email,password}` → `200 {accessToken, refreshToken}` or `401`. If `mfaEnabled`, returns `202 {mfaToken, methods:["totp","webauthn"]}` requiring a second step.
- `POST /auth/mfa/verify {mfaToken, method, code|assertion}` → tokens on success.
- Access JWT claims: `sub` (userId), `org` (organisationId), `roles` (scoped role ids), `iat`, `exp` (RFC 7519). Refresh tokens stored hashed in Redis with rotation + reuse detection (revoke family on reuse).
- WebAuthn via `@simplewebauthn/server`: `POST /auth/webauthn/register/options`, `/register/verify`, `/authenticate/options`, `/authenticate/verify`. Credentials persisted to a `webauthn_credentials` table.
- Argon2id params: memory 19 MiB, iterations 2, parallelism 1 (OWASP guidance).

**Testing**:
- `Unit: argon2 verify correct/incorrect password`.
- `Integration: login with MFA enabled → 202, no access token issued`.
- `Integration: refresh-token reuse → entire token family revoked, 401`.
- `Integration (mocked authenticator): WebAuthn register then authenticate → success`.
- `Security: 6 failed logins within 15 min → 429 rate-limited (OWASP A07)`.

#### 2.2 — SAML 2.0 & OIDC SSO

**What**: SP-initiated SAML 2.0 and OIDC login, mapping IdP subjects to `user_sso_identities` and provisioning org membership.

**Design**:
- SAML via `node-saml`: `GET /auth/saml/:orgSlug/login` → redirect to IdP; `POST /auth/saml/:orgSlug/acs` consumes the assertion, validates signature against the org's configured IdP cert, maps `NameID`→`external_id`, just-in-time provisions a `user` + `organisation_member` if absent, issues JWTs.
- OIDC via `openid-client`: authorization-code + PKCE; `/auth/oidc/:orgSlug/login`, `/auth/oidc/:orgSlug/callback`.
- Per-org IdP config stored in a `sso_connections` table (entityId, ssoUrl, x509 cert, attribute mapping JSONB). Supports Okta, Azure AD, Ping, Google Workspace.

**Testing**:
- `Integration (mocked IdP via samlify test harness): valid signed assertion → JIT user created, JWT issued`.
- `Integration: tampered assertion signature → 401, no user created`.
- `Integration: OIDC callback with mismatched state/PKCE → 401`.

#### 2.3 — SCIM 2.0 provisioning

**What**: SCIM endpoints for automated user provisioning/deprovisioning (RFC 7644).

**Design**:
- `/scim/v2/Users` (GET list w/ filter, POST create, GET/PUT/PATCH/DELETE by id) and `/scim/v2/Groups`. Bearer-token auth per org (token in `sso_connections`).
- DELETE/`active:false` sets `users.status='deactivated'` and ends `organisation_members.end_date` (soft, never physical — retention). Maps SCIM `userName`→email, `name.givenName/familyName`.

**Testing**:
- `Integration: POST /scim/v2/Users → 201, user provisioned with status 'invited'`.
- `Integration: PATCH active:false → org membership ended, user retained, audit entry written`.
- `Integration: SCIM call without bearer token → 401`.

#### 2.4 — Scoped RBAC guard

**What**: `RbacGuard` enforcing `(resourceType, action)` permissions with optional board/committee/entity scope.

**Design**:
- Route decorator `@RequirePermission('document','write', { scope: 'board' })`. Guard resolves the user's `user_roles` (org-wide + scoped to the route's board/committee param), unions granted permissions, denies with `403` if absent.
- Scope resolution: a board-scoped `director` role grants permission only when the requested resource belongs to that board.

**Testing**:
- `Unit: user with org-wide admin → passes any permission check`.
- `Unit: board-scoped director requesting another board's doc → 403`.
- `Integration: auditor calling DELETE /documents/:id → 403`.

### Definition of Done
All three login flows issue valid JWTs; MFA enforced; SCIM provisions/deprovisions; RbacGuard covers every mutating route; OpenAPI spec lists auth endpoints; tests green.

---

## Phase 3: Secure Document Repository (Core Value — Board Packs)

### Purpose
Deliver the encrypted, versioned, permissioned document repository — the heart of a board portal. This is the MVP's primary value: secure board-pack creation, distribution, and access. It depends on Phases 1–2 and unblocks meetings (pre-reads), signatures, and AI summarisation.

### Tasks

#### 3.1 — Envelope encryption & pluggable KMS

**What**: AES-256-GCM envelope encryption with a KMS-abstracted master key.

**Design**:
- `KmsProvider` interface: `wrapDataKey(dek: Buffer): Promise<WrappedKey>`, `unwrapDataKey(w: WrappedKey): Promise<Buffer>`. Implementations: `LocalKmsProvider` (master key from env, AES-KW), `AwsKmsProvider`, `AzureKeyVaultProvider`, `VaultProvider`.
- Per-document: generate random 256-bit DEK → encrypt bytes with AES-256-GCM (store IV + auth tag) → wrap DEK via KMS → persist `encryption_key_id`=wrapped DEK reference. Zero-knowledge mode: KMS lives outside the app trust boundary so operators never see plaintext keys.

**Testing**:
- `Unit: encrypt then decrypt round-trips bytes for each provider (Local mocked others)`.
- `Unit: tampered ciphertext/auth tag → GCM auth failure throws`.
- `Unit: wrong wrapped key → unwrap fails, decryption refused`.

#### 3.2 — Document upload, storage & versioning

**What**: Multipart upload → encrypt → store in S3-compatible object storage → persist metadata with SHA-256 checksum and version chaining.

**Design**:
- `POST /documents` (multipart: file + JSON metadata) → stream to encryptor → `PutObject` to `S3_BUCKET` at `org/{orgId}/{uuid}` → insert `documents` row (`checksum_sha256`, `file_size_bytes`, `mime_type`, `classification`, `document_type`, `version=1`, `is_latest=true`).
- New version: `POST /documents/:id/versions` sets prior `is_latest=false`, links `parent_document_id`, increments `version`.
- `GET /documents/:id/content` → authorise (RBAC + `document_permissions`) → fetch + decrypt + stream; every download writes an audit entry.
- `StorageProvider` interface mirrors KMS pluggability (`s3`, `azure_blob`, `local`).

**Testing**:
- `Integration (Testcontainers MinIO): upload PDF → object stored encrypted, metadata persisted, checksum matches`.
- `Integration: download → decrypted bytes equal original; audit 'download' row written`.
- `Integration: upload v2 → v1.is_latest=false, version chain intact`.
- `Integration: stored object bytes are NOT the plaintext (encryption verified)`.

#### 3.3 — Folders & granular permissions

**What**: Hierarchical folders (materialised path) and per-document/per-folder permissions for users and roles.

**Design**:
- `document_folders` with `folder_path` materialised path (`/board/2026/q1/`), optional `board_id`/`committee_id` scoping.
- `document_permissions` granting `read|annotate|download|edit|admin` to a `user_id` or `role_id`, with optional `expires_at`. Effective permission = max(folder-inherited, document-explicit, RBAC role grant).
- `GET /folders/:id/tree`, `POST /folders`, `POST /documents/:id/permissions`.

**Testing**:
- `Integration: grant director read on folder → director lists docs in folder; cannot see sibling restricted folder`.
- `Integration: expired permission (expires_at past) → access denied`.
- `Unit: effective-permission resolver picks the highest level across sources`.

#### 3.4 — Audit interceptor

**What**: Global `AuditInterceptor` recording every create/update/delete/read-of-sensitive-resource with before/after JSONB snapshots, actor, IP, user-agent.

**Design**:
- Interceptor captures `action`, `resourceType`, `resourceId`, `oldValues`/`newValues` (JSONB diff), `ip_address`, `user_agent`, `severity`. Writes to partitioned `audit_log` asynchronously via BullMQ to avoid blocking the request, with a synchronous fallback for `security`-severity events.
- Append-only: no UPDATE/DELETE grants on `audit_log` for the app role (ISO 27001 / SOC 2 tamper-evidence).

**Testing**:
- `Integration: update a document classification → audit row has old & new classification`.
- `Integration: app role attempts UPDATE audit_log → permission denied`.
- `Integration: login event recorded with ip_address`.

### Definition of Done
Documents encrypted at rest (verified), versioned, permissioned; folder tree works; every action audited immutably; OpenAPI updated; integration tests against real MinIO/Postgres green.

---

## Phase 4: Meeting Management

### Purpose
Add the full meeting lifecycle — agenda builder with templates, attendees and RSVP, pre-read distribution (linking Phase 3 documents), and a minutes editor with versioning. This is the second pillar of the MVP and prerequisite for voting (Phase 5) and AI minute-drafting (Phase 7).

### Tasks

#### 4.1 — Meetings & attendees

**What**: CRUD for meetings (board or committee scoped) with status lifecycle and attendee/RSVP tracking.

**Design**:
- State machine: `draft → scheduled → in_progress → (adjourned →) completed`; `cancelled` reachable from `draft|scheduled`. Transitions validated server-side; illegal transition → `409`.
- `POST /meetings`, `PATCH /meetings/:id/status`, `POST /meetings/:id/attendees`, `PATCH /meetings/:id/attendees/:userId/rsvp {status}`.
- Quorum: on transition to `in_progress`, compute `quorum_met` from confirmed/attended attendees vs `boards.quorum_requirement` (count or percentage).

**Testing**:
- `Unit: completed → in_progress transition rejected (409)`.
- `Integration: 3 of 5 attend, quorum_requirement 3 (count) → quorum_met true`.
- `Integration: RSVP decline → attendance_status 'declined', notification queued`.

#### 4.2 — Agenda builder with templates & nested items

**What**: Ordered, nestable agenda items with types, presenters, time allocations, consent-agenda flag, and reusable templates.

**Design**:
- `agenda_items` with `parent_item_id`, `sort_order`, `item_number` ('1', '1.1'), `item_type` (`procedural|information|discussion|decision|vote|presentation|consent`), `allocated_minutes`, `is_consent_item`, `is_confidential`.
- `meeting_templates` (JSONB `structure`) → `POST /meetings/:id/agenda/apply-template/:templateId` materialises items.
- `PATCH /meetings/:id/agenda/reorder` accepts an ordered id array; server recomputes `sort_order` and `item_number`.

**Testing**:
- `Unit: item numbering — nested child of item 2 → '2.1'`.
- `Integration: apply template → agenda items created in order`.
- `Integration: reorder → sort_order and item_number recomputed`.

#### 4.3 — Pre-read distribution

**What**: Attach Phase-3 documents to agenda items as pre-reads and expose an assembled board pack.

**Design**:
- `agenda_attachments` (`attachment_type` `pre_read|reference|presentation|handout|appendix`). `POST /meetings/:id/agenda/:itemId/attachments {documentId, type}`.
- `GET /meetings/:id/board-pack` returns ordered agenda + attachment metadata; attaching a document auto-grants `read` permission to confirmed attendees (writes `document_permissions`).

**Testing**:
- `Integration: attach pre-read → confirmed attendees gain read permission on the document`.
- `Integration: board-pack endpoint returns items in agenda order with attachment links`.

#### 4.4 — Minutes editor & versioning

**What**: Versioned minutes with draft→review→approved→signed lifecycle.

**Design**:
- `meeting_minutes` (`version`, `status` `draft|ai_generated|review|approved|signed`, `content` markdown, `ai_generated` flag). Approval (`PATCH /minutes/:id/approve`) requires `meeting:approve` permission; sets `approved_by/approved_at`. Signing links a `signed_document_id` (Phase 6).
- New edits after approval create a new version rather than mutating.

**Testing**:
- `Integration: approve minutes without permission → 403`.
- `Integration: edit approved minutes → new version created, prior preserved`.

### Definition of Done
Meeting lifecycle enforced; agenda templates and reordering work; pre-reads grant access correctly; minutes versioned and approvable; OpenAPI updated; tests green.

---

## Phase 5: Voting, Resolutions & Action Items

### Purpose
Capture formal board decisions: motions/votes with configurable thresholds, individual vote records with recusal, formal resolutions (including written/circular resolutions), and action-item tracking through to completion. Completes the core decision-record loop of the MVP.

### Tasks

#### 5.1 — Votes & vote records

**What**: Open/close motions tied to agenda items, record per-member votes, auto-tally against the threshold.

**Design**:
- `votes` (`vote_type` `simple_majority|supermajority|unanimous|plurality|show_of_hands|roll_call|consent`, `required_threshold`, `status`). `POST /meetings/:id/votes`, `PATCH /votes/:id/open|close`, `POST /votes/:id/cast {vote_cast}` (`for|against|abstain|recuse`).
- On close: tally `votes_for/against/abstain`; recusals excluded from base; compare to threshold (e.g. supermajority 66.67%) → set `passed|failed`. One record per user (`uq_vote_record`); recused members flagged for COI cross-reference (Phase 8).

**Testing**:
- `Unit: supermajority 66.67%, 7 for / 3 against → passed`.
- `Unit: unanimous with 1 against → failed`.
- `Integration: recused vote excluded from tally base`.
- `Integration: second cast by same user → 409 (idempotent/unique)`.

#### 5.2 — Resolutions

**What**: Formal resolutions, including circular/written resolutions decided outside a meeting, with org-unique numbering and supersession.

**Design**:
- `resolutions` (`resolution_number` org-unique e.g. `RES-2026-001`, `resolution_type` `ordinary|special|written|circular|consent`, `status`, `supersedes_id`). Auto-number on creation per org per year.
- Circular resolution: created with `meeting_id NULL`, asynchronous e-sign approvals (Phase 6) drive it to `passed`.

**Testing**:
- `Unit: resolution numbering increments per org per year`.
- `Integration: passing a resolution that supersedes another → prior status 'superseded'`.

#### 5.3 — Action items

**What**: Assignable, prioritised, due-dated action items extracted from meetings/resolutions, tracked to completion.

**Design**:
- `action_items` (`assigned_to`, `priority`, `status` `open|in_progress|completed|deferred|cancelled`, `due_date`). `GET /action-items?assignee=&status=&overdue=true`. Overdue detection via a scheduled job sets reminders/notifications.

**Testing**:
- `Unit: due_date < today & status open → reported overdue`.
- `Integration: complete action item → completed_at set, notification to assigner`.

### Definition of Done
Vote tallying correct across all thresholds; resolutions numbered and superseded correctly; action items tracked and overdue-detected; tests green; OpenAPI updated.

---

## Phase 6: E-Signature & Calendar Integration

### Purpose
Connect governance records to the outside world: e-signature (built-in + DocuSign) for minutes/resolutions/circular resolutions, and calendar sync (iCalendar/Outlook/Google) for meetings. These can be developed in parallel after Phases 4–5.

### Tasks

#### 6.1 — Built-in e-signature

**What**: In-app signature requests with ordered signers, drawn/typed signature capture, and a tamper-evident signed PDF.

**Design**:
- `signature_requests` + `signature_signers` (`sign_order`, `status`). `POST /signatures {documentId, signers[]}`. Sequential ordering: signer N notified only after N-1 signs. On final signature, stamp a certificate page (signer, timestamp, IP, SHA-256 of source doc) and store as a new encrypted `documents` row; link back to `meeting_minutes.signed_document_id` or `resolutions.signed_document_id`.
- Each signature records `ip_address`, `user_agent`, hashed signature data.

**Testing**:
- `Integration: 2-signer ordered request → signer 2 cannot sign before signer 1`.
- `Integration: completed request → signed PDF stored, source checksum embedded`.

#### 6.2 — DocuSign integration

**What**: Route signature requests to DocuSign with webhook status sync.

**Design**:
- `provider='docusign'`: create envelope via DocuSign eSignature API (OAuth 2.0 / JWT grant), store `external_envelope_id`. `POST /webhooks/docusign` (Connect) updates signer/request status; verify HMAC signature on the webhook.

**Testing**:
- `Integration (mocked DocuSign): create request → envelope id stored`.
- `Integration: webhook 'completed' with valid HMAC → request completed`.
- `Integration: webhook with bad HMAC → 401, status unchanged`.

#### 6.3 — Calendar sync (iCalendar / Outlook / Google)

**What**: Emit RFC 5545 invites and two-way sync meetings to Outlook (Microsoft Graph) and Google Calendar.

**Design**:
- `GET /meetings/:id/ical` → RFC 5545 `VEVENT` (UID = `calendar_events.ical_uid`, organiser, attendees, location/virtual URL).
- OAuth connect per user (Microsoft Graph / Google Calendar). `calendar_events` tracks `external_event_id`, `sync_status`. Meeting reschedule enqueues a sync job updating the external event.

**Testing**:
- `Unit: ical export → valid VEVENT, UID stable across re-exports`.
- `Integration (mocked Graph/Google): create meeting → external event created, sync_status 'synced'`.
- `Integration: reschedule → external event updated`.

### Definition of Done
Built-in signing produces tamper-evident PDFs; DocuSign round-trips via verified webhooks; iCal valid and Outlook/Google sync works (mocked); tests green; OpenAPI updated.

---

## Phase 7: AI-Native Layer & MCP Server

### Purpose
Deliver the AI-native differentiators: minute drafting from agenda/notes/transcripts, board-pack summarisation, action-item extraction, and an MCP server exposing governance tools to any compatible assistant — a stated advantage over vendor-locked AI. Depends on Phases 3–5 (documents, meetings, votes).

### Tasks

#### 7.1 — Provider-agnostic AI service & document text extraction

**What**: An AI gateway (Vercel AI SDK) abstracting Anthropic/OpenAI/Azure OpenAI, plus a worker that extracts text from uploaded documents for downstream AI use.

**Design**:
- `AiService.complete({system, prompt, schema?})` returns text or, with a Zod `schema`, validated structured output (`generateObject`). Provider chosen by `AI_PROVIDER`. All AI calls are audited (`severity='info'`, model + token counts in `metadata`).
- Text-extraction BullMQ job: PDF→text (pdf.js), DOCX→text (`mammoth`), persisted to a `document_text` table (with tsvector index) for summarisation and search.

**Testing**:
- `Unit (mocked provider): structured output validated against Zod schema; invalid output → retried once then errors`.
- `Integration: upload PDF → text extracted and indexed`.

#### 7.2 — AI minute drafting & action-item extraction

**What**: Generate a draft minutes document from agenda + attendee list + typed notes / transcript, and extract action items.

**Design**:
- `POST /meetings/:id/minutes/ai-draft {transcript?, notes?}` → assembles context (agenda items, attendees, decisions, vote results) → LLM → creates `meeting_minutes` row `status='ai_generated'`, `ai_generated=true`.
- System prompt template (stored in `packages/api/src/modules/ai/prompts/minutes.ts`):
  > "You are a corporate company secretary drafting formal board meeting minutes. Use the supplied agenda, attendee list, recorded votes, and notes. Output structured minutes: Attendance, Quorum, Agenda items with discussion summary and decisions, Resolutions passed (verbatim), Action items (owner + due date). Be factual; never invent decisions not present in the input. Mark anything uncertain as [TO CONFIRM]."
- Action-item extraction uses `generateObject` with schema `{ title, assigneeEmail?, dueDate?, priority }[]`; matched assignees create `action_items`.

**Testing**:
- `Integration (mocked LLM): draft from fixture meeting → ai_generated minutes row created`.
- `Unit: extraction maps assigneeEmail to user; unknown email → action_item with null assignee + flag`.
- `Guard: AI draft never auto-approves (status stays ai_generated until human approval)`.

#### 7.3 — Board-pack summarisation

**What**: Summarise a board pack or a single document, surfacing key decisions and risks.

**Design**:
- `POST /documents/:id/summarise` and `POST /meetings/:id/board-pack/summarise`. Chunked map-reduce summarisation over extracted text; result cached on `documents.metadata.aiSummary` with source-text checksum so it invalidates when the document changes.

**Testing**:
- `Integration (mocked LLM): summarise → summary cached; re-request returns cached unless checksum changed`.

#### 7.4 — MCP server

**What**: An MCP server exposing read/act governance tools to MCP-compatible assistants.

**Design**:
- `packages/mcp-server` using `@modelcontextprotocol/sdk`. Tools: `list_meetings`, `get_board_pack`, `get_document_text`, `query_resolutions`, `create_agenda_item`, `create_action_item`, `get_entity_hierarchy`. Each tool authenticates via a scoped API token and enforces the same RBAC + RLS as the REST API (calls the API internally, never the DB directly).

**Testing**:
- `Integration: MCP list_meetings with a director token → only authorised meetings returned`.
- `Integration: MCP create_action_item without write permission → tool error (403 propagated)`.

### Definition of Done
AI service provider-swappable; minutes/summaries/action-items generated (mocked LLM in CI); MCP server enforces RBAC/RLS; all AI actions audited; tests green.

---

## Phase 8: Compliance — Insider Trading, Disclosures, Questionnaires & Entities

### Purpose
Add the differentiating compliance modules largely absent from mid-market competitors: insider-trading pre-clearance with trading-window enforcement, conflict-of-interest disclosures and the interest register, D&O questionnaires, the entity/subsidiary registry with LEI, and filing-deadline tracking. Depends on Phases 1–2; integrates with voting recusals (Phase 5) and AI COI detection (Phase 7).

### Tasks

#### 8.1 — Entity registry & subsidiary hierarchy

**What**: Manage legal entities, parent/child relationships with ownership percentages, and LEI lookup.

**Design**:
- Reuse `organisations` + `entity_relationships` (Suggestion 1). `GET /entities/:id/hierarchy` returns the ownership tree via a recursive CTE (Suggestion 3's recursive-query approach; full graph analytics deferred to backlog Suggestion 4).
- `POST /entities/:id/lei/verify` calls the GLEIF API (ISO 17442) to validate/enrich an LEI; no auth required for public lookup.

**Testing**:
- `Integration: 3-level subsidiary chain → hierarchy endpoint returns full tree`.
- `Integration (mocked GLEIF): valid LEI → entity enriched; invalid → 422`.

#### 8.2 — Trading windows & insider designations

**What**: Define blackout/open windows and designate Section 16 insiders.

**Design**:
- `trading_windows` (`window_type`, `blackout_start/end`, `applies_to_all`) and `insider_designations` (`insider_type`, `sec_filing_status`). A `isInBlackout(userId, date)` service returns whether trading is permitted for a given insider on a date.

**Testing**:
- `Unit: date inside blackout window → isInBlackout true`.
- `Unit: designated insider outside all windows → false`.

#### 8.3 — Pre-clearance workflow

**What**: Insiders request trade pre-clearance; compliance reviews/approves with auto-blackout checks and time-boxed approvals.

**Design**:
- `preclearance_requests` state machine: `pending → approved|denied`, approved → `executed` or `expired` (typically 2–5 business days). On submit, the service auto-checks `isInBlackout`; if in blackout, the request is flagged and defaults to deny-recommended. `POST /preclearance`, `PATCH /preclearance/:id/review {decision, notes}`.
- Approval sets `approval_expires_at`; a job auto-expires lapsed approvals.

**Testing**:
- `Integration: request during blackout → flagged in_blackout, review prefilled deny-recommended`.
- `Integration: approved request past approval_expires_at → auto-expired by job`.
- `Integration: every decision writes an audit entry (clearance auditability)`.

#### 8.4 — Disclosures, interest register & D&O questionnaires

**What**: Director interest disclosures, the interest register, and JSONB-defined D&O questionnaires with responses.

**Design**:
- `interest_disclosures` (`disclosure_type`, `is_material`, effective dating). `GET /interest-register?orgId=` aggregates active disclosures.
- `questionnaires.template_content` (JSONB question definitions) + `questionnaire_responses.responses` (JSONB answers) — Suggestion 3's variable-schema fit. `POST /questionnaires`, `POST /questionnaires/:id/responses`.

**Testing**:
- `Integration: submit D&O response → stored as JSONB, status submitted`.
- `Unit: interest register returns only active (non-expired) disclosures`.

#### 8.5 — Filing-deadline tracker

**What**: Track multi-jurisdiction regulatory filing deadlines with recurrence and reminders.

**Design**:
- `filing_deadlines` (`jurisdiction`, `filing_type`, `regulatory_body`, `due_date`, `recurrence`, `status`). A daily job advances recurring deadlines and emits reminder notifications at configurable lead times; overdue deadlines flagged.

**Testing**:
- `Unit: annual recurrence → next due_date is +1 year on filing`.
- `Integration: deadline within reminder window → notification queued`.

### Definition of Done
Entity hierarchy + LEI verification work; blackout enforcement correct; pre-clearance fully audited with auto-expiry; disclosures/questionnaires stored as JSONB; filing deadlines tracked and reminded; tests green.

---

## Phase 9: Web Frontend, Offline & Secure Messaging

### Purpose
Deliver the director-facing experience: a mobile-responsive Next.js PWA with offline board-pack access, PDF annotation, dashboards, and secure messaging that replaces email. Depends on Phases 2–8 APIs.

### Tasks

#### 9.1 — App shell, auth & dashboards

**What**: Next.js App Router shell with SSO/MFA login, role-aware navigation, and director/secretary dashboards.

**Design**:
- shadcn/ui + Tailwind. Auth via the Phase-2 endpoints; access token in memory, refresh via httpOnly cookie. Director dashboard: upcoming meetings, board packs, action items, votes open. Secretary dashboard: meeting builder, minutes review, compliance alerts.

**Testing**:
- `E2E (Playwright): login → director sees only their boards' meetings`.
- `E2E: auditor sees read-only UI (no edit/delete controls rendered)`.

#### 9.2 — PDF viewer, annotation & offline cache

**What**: pdf.js viewer with annotation persistence and offline access to board packs.

**Design**:
- Annotation layer persists to `document_annotations` (`position_data` JSONB); private vs shared annotations. Service worker + IndexedDB cache board-pack documents (decrypted client-side only in memory; cached ciphertext + ephemeral key per session) for offline read; sync annotations on reconnect.

**Testing**:
- `E2E: create highlight → persisted and reloads at same coordinates`.
- `E2E (offline): go offline → previously opened board pack still readable`.

#### 9.3 — Secure messaging

**What**: Encrypted threaded messaging scoped to boards/committees, replacing email.

**Design**:
- `message_threads` / `message_participants` / `messages` (encrypted `body`). `GET /threads`, `POST /threads`, `POST /threads/:id/messages`. Read receipts via `last_read_at`. Notifications on new messages.

**Testing**:
- `Integration: non-participant cannot read thread → 403`.
- `E2E: send message → recipient sees unread badge, marks read updates last_read_at`.

### Definition of Done
PWA installable and responsive; SSO/MFA login works end-to-end; PDF annotation persists; offline board-pack read works; secure messaging functional; Playwright E2E suite green.

---

## Phase 10: Notifications, Board Effectiveness & Hardening

### Purpose
Complete the platform with the notification engine, board-effectiveness analytics (skills tracking, composition gap analysis, evaluations), and production hardening (OWASP, observability, OpenAPI publication, GDPR workflows). Depends on all prior phases.

### Tasks

#### 10.1 — Notification engine

**What**: Multi-channel notifications (in-app, email, push) driven by domain events.

**Design**:
- `notifications` rows + BullMQ fan-out. Channels: in-app (polled/SSE), email (SMTP/SES), web push. Event triggers: meeting invite, document shared, vote open, action due, pre-clearance status, filing reminder. Per-user preferences.

**Testing**:
- `Integration: vote opened → in-app notification to all board members`.
- `Unit: user with email muted → only in-app channel used`.

#### 10.2 — Board effectiveness & composition analytics

**What**: Skills taxonomy, director skills mapping, composition gap analysis, and evaluations.

**Design**:
- `skills_taxonomy` + `director_skills` + `board_evaluations`. `GET /boards/:id/composition` returns coverage per critical skill and gaps (skills marked `is_critical` with no `expert`-level director). Optional AI recommendation (Phase 7) for composition gaps.

**Testing**:
- `Unit: critical skill with no expert director → reported as a gap`.
- `Integration: evaluation lifecycle draft→completed with overall_rating`.

#### 10.3 — Production hardening & compliance

**What**: OWASP Top 10 pass, OpenAPI 3.1 publication, GDPR right-to-erasure (pseudonymisation, not physical delete), observability, rate limiting, and CI security gates.

**Design**:
- Helmet, CSRF, input validation (Zod/class-validator on every DTO), per-route rate limits. `GET /openapi.json` served from `@nestjs/swagger`. GDPR erasure pseudonymises PII while retaining audit/governance records (legal-hold aware). OpenTelemetry traces; pino structured logs. CI runs `eslint-plugin-security` + dependency audit and fails on high-severity findings.

**Testing**:
- `Security: OWASP injection probes on search endpoints → parameterised, no injection`.
- `Integration: GDPR erasure → user PII pseudonymised, audit_log retained, governance records intact`.
- `Smoke: GET /openapi.json validates against OpenAPI 3.1 schema`.

### Definition of Done
Notifications delivered across channels; composition analytics correct; OWASP/security gates pass in CI; OpenAPI 3.1 published and valid; GDPR erasure preserves compliance records; full suite (unit + integration + E2E) green; Docker images build.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Multi-Tenancy & Data Model   ─── required by everything
    │
Phase 2: Identity, Auth & RBAC                     ─── requires P1
    │
Phase 3: Secure Document Repository                ─── requires P2
    │
Phase 4: Meeting Management                        ─── requires P3
    │
Phase 5: Voting, Resolutions & Action Items        ─── requires P4
    │
    ├── Phase 6: E-Signature & Calendar            ─── requires P4,P5 · parallel with P7
    ├── Phase 7: AI Layer & MCP Server             ─── requires P3,P4,P5 · parallel with P6
    └── Phase 8: Compliance Modules                ─── requires P2 · parallel with P6,P7
         │
Phase 9: Web Frontend, Offline & Messaging         ─── requires P2–P8 APIs
    │
Phase 10: Notifications, Effectiveness & Hardening ─── requires all prior
```

**Parallelism opportunities**
- Phases 6, 7, and 8 can be developed concurrently once Phase 5 is complete (Phase 8 only strictly needs Phase 2 but integrates with Phase 5 recusals and Phase 7 COI detection).
- Within Phase 9, messaging (9.3) is independent of the PDF/offline work (9.2) and can be parallelised.
- Frontend (Phase 9) work can begin against mocked APIs as soon as each backend module's OpenAPI contract is published.

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; new code has meaningful coverage of happy-path and edge cases.
3. `pnpm lint` (incl. `eslint-plugin-security`) passes with no errors.
4. `pnpm typecheck` (`tsc --noEmit`) passes across all packages.
5. `docker-compose up` brings the stack up healthy; affected Docker images build.
6. The phase's feature works end-to-end against real Postgres/Redis/MinIO (Testcontainers) and, where user-facing, via a Playwright E2E test.
7. New configuration options are documented in `.env.example`.
8. New/changed API endpoints appear in the generated OpenAPI 3.1 spec (`GET /openapi.json`).
9. Prisma migrations are created, reviewed, and apply cleanly forward; RLS policies cover any new tenant-scoped tables.
10. Every state-changing operation introduced is covered by the audit interceptor (verified by a test asserting an `audit_log` entry).
11. Relevant standards are honoured where the phase touches them (SAML 2.0 / OIDC / SCIM 2.0 / WebAuthn in P2; AES-256 + ISO 27001 audit immutability in P3; RFC 5545 iCal in P6; ISO 17442 LEI in P8; OpenAPI 3.1 + OWASP Top 10 + GDPR in P10).
```
