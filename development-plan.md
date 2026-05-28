# AI Tutoring System -- Development Plan

> Project: AI Tutoring System (Candidate #114)
> Created: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Infrastructure](#phase-1-foundation--core-infrastructure)
5. [Phase 2: Knowledge Domain & Curriculum Engine](#phase-2-knowledge-domain--curriculum-engine)
6. [Phase 3: Socratic Dialogue Engine](#phase-3-socratic-dialogue-engine)
7. [Phase 4: Student Knowledge Model & Adaptive Sequencing](#phase-4-student-knowledge-model--adaptive-sequencing)
8. [Phase 5: Assessment & Hint Scaffolding](#phase-5-assessment--hint-scaffolding)
9. [Phase 6: Teacher Dashboard & Alerts](#phase-6-teacher-dashboard--alerts)
10. [Phase 7: LMS Integration (LTI 1.3 & OneRoster)](#phase-7-lms-integration-lti-13--oneroster)
11. [Phase 8: Misconception Detection & Remediation](#phase-8-misconception-detection--remediation)
12. [Phase 9: Multilingual Support](#phase-9-multilingual-support)
13. [Phase 10: Learning Analytics & xAPI](#phase-10-learning-analytics--xapi)
14. [Phase 11: Accessibility, Compliance & Hardening](#phase-11-accessibility-compliance--hardening)
15. [Phase 12: Gamification, Collaboration & Advanced Features](#phase-12-gamification-collaboration--advanced-features)
16. [Definition of Done (Global)](#definition-of-done-global)

---

## Technology Decisions

### Data Model: Hybrid Relational + JSONB (Suggestion 3, enhanced with select Graph elements from Suggestion 4)

**Rationale:** The Hybrid Relational + JSONB model (Suggestion 3) is chosen as the primary data model for these reasons:

1. **Lowest table count (15 core tables)** reduces migration burden and accelerates MVP delivery.
2. **JSONB flexibility** handles multi-subject, multi-jurisdiction variation (math items vs. essay prompts, FERPA vs. GDPR consent fields) without per-variant schema migrations.
3. **Relational core for enrollment, identity, and sessions** preserves foreign-key integrity where it matters most (OneRoster/Ed-Fi alignment).
4. **JSONB assessment items** accommodate QTI 3.0's diverse interaction types (choice, text_entry, extended_text, image_input) within a single table.

**Graph enhancement:** The `ltree` extension from Suggestion 4 is adopted specifically for curriculum standard hierarchies (CCSS > Math > Grade 8 > EE > A.1). The full graph_node/graph_edge layer from Suggestion 4 is deferred to Phase 8 (Misconception Detection) where the misconception-KC-remediation subgraph adds genuine value for adaptive reasoning. This avoids front-loading graph complexity before it is needed.

**Rejected alternatives:**
- *Suggestion 1 (Fully Normalized):* 28+ tables with high JOIN cost for common reads. Appropriate for enterprise reporting but over-engineered for MVP.
- *Suggestion 2 (Event-Sourced CQRS):* Excellent audit trail and temporal queries, but projection worker infrastructure adds operational complexity incompatible with a small early team. The crypto-shredding GDPR pattern is borrowed for Phase 11.
- *Suggestion 4 (Graph-Relational Hybrid):* Full graph layer deferred; `ltree` for hierarchies and targeted graph tables for misconception networks adopted in later phases.

### Backend Framework & Language

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | **TypeScript (Node.js)** | Largest open-source EdTech contributor base; shared language with frontend; strong LLM SDK ecosystem (Anthropic, OpenAI, LangChain.js) |
| Framework | **NestJS** | Modular architecture maps to domain boundaries (tutoring, knowledge, enrollment, integration); built-in dependency injection; OpenAPI generation; WebSocket support for real-time sessions |
| ORM | **Drizzle ORM** | Type-safe SQL with explicit query control; native JSONB support; lightweight migration system; avoids Prisma's query-engine overhead |
| Database | **PostgreSQL 16+** | JSONB with GIN indexes; `ltree` extension; Row-Level Security for multi-tenancy; materialized views for dashboards; partitioning for xAPI statement growth |
| Cache | **Redis 7** | Session state, BKT mastery cache, rate limiting, real-time pub/sub for teacher dashboard updates |
| Search | **Meilisearch** | Assessment item and knowledge component search for content authoring; simpler ops than Elasticsearch for early stage |

### Frontend

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Framework | **Next.js 15 (App Router)** | SSR for initial page load (accessibility, SEO); React Server Components for teacher dashboard data loading; API routes for BFF pattern |
| UI Library | **Radix UI + Tailwind CSS** | Accessibility-first primitives (WCAG 2.2 compliance built-in); composable; no design-system lock-in |
| Math Rendering | **KaTeX** | Fastest LaTeX rendering; SSR-compatible; lighter than MathJax |
| Real-time | **Socket.IO** | WebSocket transport for live tutoring session turns and teacher dashboard updates; fallback polling for restrictive school networks |
| State | **Zustand** | Lightweight; works with React Server Components; minimal boilerplate for session state |

### AI / LLM

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Primary LLM | **Claude API (Anthropic)** | Strong Socratic dialogue capability; prompt caching reduces per-session cost; tool use for structured pedagogical actions; long context for session history |
| Fallback LLM | **Open-source (LLaMA 3.1 70B fine-tuned)** | Self-hosted option for institutions requiring data residency; fine-tuned on tutoring dialogue for domain specificity |
| LLM Orchestration | **Vercel AI SDK** | Streaming responses for real-time tutor turns; provider-agnostic (swap Claude/OpenAI/local); structured output parsing |
| Embeddings | **Voyage AI** | High-quality embeddings for assessment item similarity and misconception matching; education-domain fine-tuning available |
| Vector Store | **pgvector (PostgreSQL extension)** | Embeddings stored alongside relational data; avoids a separate vector database; cosine similarity search for item retrieval |

### Infrastructure & DevOps

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Container Runtime | **Docker + Docker Compose (dev), Kubernetes (prod)** | Standard containerization; Helm charts for institutional self-hosted deployments |
| CI/CD | **GitHub Actions** | Free for open-source; matrix testing across Node versions; deployment to multiple targets |
| Hosting (SaaS) | **Vercel (frontend) + Railway / Fly.io (backend)** | Edge-optimized frontend; auto-scaling backend; simple PostgreSQL provisioning |
| Hosting (Self-hosted) | **Helm chart for K8s** | Institutional customers deploy on their own infrastructure for FERPA compliance |
| Monitoring | **OpenTelemetry + Grafana** | Vendor-neutral tracing; LLM latency and token cost tracking; learning analytics pipeline |
| Secrets | **Doppler or Infisical** | Environment-based secret management; rotation support for LTI private keys |

### Testing

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Unit / Integration | **Vitest** | Fast; native TypeScript; compatible with NestJS testing modules |
| E2E (API) | **Supertest + Vitest** | HTTP-level API testing with assertion library |
| E2E (Browser) | **Playwright** | Cross-browser testing; accessibility assertions built-in; visual regression |
| Load | **k6** | Scriptable in JavaScript; simulates concurrent tutoring sessions |
| LLM Testing | **Custom harness + Promptfoo** | Evaluate Socratic dialogue quality; measure hint relevance; detect answer leakage |

---

## Project Structure

```
ai-tutoring-system/
├── apps/
│   ├── web/                          # Next.js 15 frontend
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages
│   │   │   │   ├── (public)/         # Landing, login
│   │   │   │   ├── (student)/        # Student tutoring UI
│   │   │   │   ├── (teacher)/        # Teacher dashboard
│   │   │   │   ├── (admin)/          # Institution admin
│   │   │   │   └── api/              # BFF API routes
│   │   │   ├── components/
│   │   │   │   ├── tutoring/         # Chat interface, hint display, step-by-step
│   │   │   │   ├── dashboard/        # Teacher views, mastery grids, alerts
│   │   │   │   ├── assessment/       # Item rendering, response capture
│   │   │   │   └── shared/           # Layout, navigation, accessibility
│   │   │   ├── hooks/
│   │   │   ├── lib/
│   │   │   └── styles/
│   │   ├── public/
│   │   └── tests/
│   │       ├── e2e/                  # Playwright tests
│   │       └── components/           # Component tests
│   │
│   └── api/                          # NestJS backend
│       ├── src/
│       │   ├── modules/
│       │   │   ├── auth/             # Authentication, RBAC, consent
│       │   │   ├── institution/      # Multi-tenancy, institution config
│       │   │   ├── enrollment/       # Courses, sections, OneRoster sync
│       │   │   ├── knowledge/        # KCs, curriculum standards, prerequisites
│       │   │   ├── assessment/       # Items, hints, QTI import/export
│       │   │   ├── tutoring/         # Session management, conversation turns
│       │   │   ├── dialogue/         # LLM orchestration, Socratic engine
│       │   │   ├── student-model/    # BKT, mastery tracking, knowledge state
│       │   │   ├── misconception/    # Detection, classification, remediation
│       │   │   ├── teacher/          # Dashboard data, alerts, class analytics
│       │   │   ├── lti/              # LTI 1.3 launch, grade passback
│       │   │   ├── xapi/             # Statement generation, LRS endpoints
│       │   │   ├── analytics/        # Reporting, aggregations, exports
│       │   │   └── media/            # Image upload, OCR processing
│       │   ├── common/
│       │   │   ├── database/         # Drizzle schema, migrations
│       │   │   ├── guards/           # Auth guards, RLS helpers
│       │   │   ├── interceptors/     # Audit logging, request tracing
│       │   │   ├── filters/          # Error handling
│       │   │   └── pipes/            # Validation (JSON Schema for JSONB)
│       │   ├── config/
│       │   └── main.ts
│       ├── drizzle/
│       │   ├── schema/               # Table definitions
│       │   └── migrations/           # Migration files
│       └── tests/
│           ├── unit/
│           ├── integration/
│           └── llm/                  # Promptfoo evaluation suites
│
├── packages/
│   ├── shared-types/                 # Shared TypeScript types
│   ├── qti-parser/                   # QTI 3.0 import/export library
│   ├── bkt-engine/                   # Bayesian Knowledge Tracing implementation
│   ├── xapi-client/                  # xAPI statement builder and LRS client
│   └── lti-toolkit/                  # LTI 1.3 launch and service utilities
│
├── infrastructure/
│   ├── docker/
│   │   ├── docker-compose.yml        # Local development stack
│   │   ├── Dockerfile.api
│   │   └── Dockerfile.web
│   ├── helm/                         # Kubernetes Helm chart for self-hosted
│   ├── terraform/                    # Cloud infrastructure (optional SaaS)
│   └── scripts/
│       ├── seed-knowledge-graph.ts   # Seed KC prerequisites and standards
│       └── seed-assessment-items.ts  # Seed sample items for testing
│
├── docs/
│   ├── architecture/                 # ADRs, system diagrams
│   ├── api/                          # Generated OpenAPI docs
│   └── deployment/                   # Self-hosted deployment guide
│
├── turbo.json                        # Turborepo config
├── package.json
├── tsconfig.json
└── .github/
    └── workflows/
        ├── ci.yml                    # Lint, test, build
        ├── deploy-staging.yml
        └── deploy-production.yml
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Infrastructure
    │
    ├──> Phase 2: Knowledge Domain & Curriculum Engine
    │       │
    │       ├──> Phase 3: Socratic Dialogue Engine
    │       │       │
    │       │       └──> Phase 5: Assessment & Hint Scaffolding
    │       │               │
    │       │               ├──> Phase 8: Misconception Detection & Remediation
    │       │               │
    │       │               └──> Phase 10: Learning Analytics & xAPI
    │       │
    │       └──> Phase 4: Student Knowledge Model & Adaptive Sequencing
    │               │
    │               ├──> Phase 6: Teacher Dashboard & Alerts
    │               │
    │               └──> Phase 8: Misconception Detection & Remediation
    │
    ├──> Phase 7: LMS Integration (LTI 1.3 & OneRoster)  [can start after Phase 1]
    │
    ├──> Phase 9: Multilingual Support  [can start after Phase 3]
    │
    ├──> Phase 11: Accessibility, Compliance & Hardening  [can start after Phase 6]
    │
    └──> Phase 12: Gamification, Collaboration & Advanced Features  [requires Phases 5, 6, 8]
```

**Critical Path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 5 -> Phase 8

**Parallel Tracks:**
- Track A (Core Tutoring): Phases 1 -> 2 -> 3 -> 5 -> 8
- Track B (Student Model): Phases 2 -> 4 -> 6
- Track C (Integration): Phases 1 -> 7
- Track D (Polish): Phases 9, 11, 12

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the monorepo, database, authentication, multi-tenancy, and deployment pipeline so that all subsequent phases build on solid infrastructure.

**Duration:** 3-4 weeks

### Task 1.1: Monorepo Scaffolding

**What:** Initialize the Turborepo monorepo with `apps/web` (Next.js 15), `apps/api` (NestJS), and `packages/shared-types`. Configure TypeScript, ESLint, Prettier, and Husky pre-commit hooks. Set up path aliases and shared tsconfig.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "db:migrate": { "cache": false }
  }
}

// packages/shared-types/src/index.ts
export interface Institution {
  id: string;
  name: string;
  slug: string;
  institutionType: 'district' | 'school' | 'university' | 'department' | 'corporate';
  jurisdictionCountry: string;  // ISO 3166-1 alpha-2
  privacyFramework: 'ferpa' | 'gdpr' | 'coppa' | 'ccpa' | 'custom';
  config: InstitutionConfig;
}

export interface InstitutionConfig {
  allowedSessionModes: SessionMode[];
  maxHintLevels: number;
  coppaAgeThreshold: number;
  dataRetentionDays: Record<string, number>;
  llmConfig: { model: string; maxTokens: number };
  enabledSubjects: string[];
}

export type SessionMode = 'socratic' | 'worked_example' | 'practice' | 'assessment';

export type UserRole = 'student' | 'teacher' | 'parent' | 'admin' | 'institution_admin';
```

**Testing:**
- `test-1.1.1`: Run `turbo build` from repo root -- all three packages compile without errors.
- `test-1.1.2`: Run `turbo lint` -- zero lint errors across all workspaces.
- `test-1.1.3`: Verify `shared-types` is importable from both `apps/web` and `apps/api` with correct type resolution.
- `test-1.1.4`: Verify Husky pre-commit hook runs lint and type-check before allowing commit.

---

### Task 1.2: Database Schema & Migrations

**What:** Set up PostgreSQL 16 with Drizzle ORM. Implement the core Hybrid Relational + JSONB schema: `institution`, `user_account`, `consent_record`, `course`, `section`, `enrollment`, and `audit_log` tables. Enable `ltree` and `pgvector` extensions. Create seed script for development data.

**Design:**

```typescript
// apps/api/drizzle/schema/institution.ts
import { pgTable, uuid, text, timestamp, jsonb, char, varchar, index } from 'drizzle-orm/pg-core';

export const institution = pgTable('institution', {
  id: uuid('id').primaryKey().defaultRandom(),
  parentInstitutionId: uuid('parent_institution_id').references(() => institution.id),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  institutionType: text('institution_type').notNull(),
  jurisdictionCountry: char('jurisdiction_country', { length: 2 }).notNull(),
  jurisdictionSubdivision: varchar('jurisdiction_subdivision', { length: 6 }),
  privacyFramework: text('privacy_framework').notNull().default('ferpa'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  slugIdx: index('idx_institution_slug').on(table.slug),
  parentIdx: index('idx_institution_parent').on(table.parentInstitutionId),
}));

// apps/api/drizzle/schema/user-account.ts
export const userAccount = pgTable('user_account', {
  id: uuid('id').primaryKey().defaultRandom(),
  institutionId: uuid('institution_id').notNull().references(() => institution.id),
  email: text('email'),
  displayName: text('display_name').notNull(),
  roles: text('roles').array().notNull().default(sql`'{"student"}'`),
  locale: text('locale').notNull().default('en'),
  isMinor: boolean('is_minor').notNull().default(false),
  profile: jsonb('profile').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  institutionIdx: index('idx_user_institution').on(table.institutionId),
}));

// apps/api/drizzle/schema/audit-log.ts
export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => userAccount.id),
  institutionId: uuid('institution_id').references(() => institution.id),
  action: text('action').notNull(),
  entityType: text('entity_type').notNull(),
  entityId: uuid('entity_id').notNull(),
  changes: jsonb('changes'),
  context: jsonb('context').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  userIdx: index('idx_audit_user').on(table.userId, table.createdAt),
  entityIdx: index('idx_audit_entity').on(table.entityType, table.entityId),
}));
```

**Testing:**
- `test-1.2.1`: Run `drizzle-kit push` against a clean PostgreSQL instance -- all tables created with correct constraints.
- `test-1.2.2`: Run `drizzle-kit generate` and verify migration SQL is deterministic and re-runnable.
- `test-1.2.3`: Insert an institution with JSONB config, retrieve it, verify config round-trips correctly (including nested objects).
- `test-1.2.4`: Attempt to insert a `user_account` with a non-existent `institution_id` -- expect FK violation error.
- `test-1.2.5`: Verify `ltree` extension is enabled: `SELECT 'a.b.c'::ltree @> 'a.b'::ltree` returns true.
- `test-1.2.6`: Verify `pgvector` extension is enabled: `CREATE TABLE test_vec (embedding vector(1536))` succeeds.
- `test-1.2.7`: Run seed script and verify development data is queryable.

---

### Task 1.3: Authentication & RBAC

**What:** Implement authentication using NextAuth.js (frontend) with JWT sessions. Build NestJS auth guards that validate JWTs and enforce role-based access. Implement COPPA age-gating and parental consent workflow. Build the `consent_record` lifecycle.

**Design:**

```typescript
// apps/api/src/modules/auth/guards/roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { UserRole } from '@repo/shared-types';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}

// apps/api/src/modules/auth/services/consent.service.ts
export class ConsentService {
  async requestParentalConsent(minorUserId: string, parentEmail: string): Promise<ConsentRecord> {
    // 1. Create consent_record with status 'pending'
    // 2. Send verification email to parent with signed token
    // 3. Return pending record
  }

  async grantConsent(token: string, ipAddress: string): Promise<ConsentRecord> {
    // 1. Validate signed token (expiry, tampering)
    // 2. Update consent_record status to 'granted'
    // 3. Write audit_log entry
    // 4. Return updated record
  }

  async revokeConsent(consentId: string, revokedBy: string): Promise<ConsentRecord> {
    // 1. Insert new consent_record with status 'revoked' (immutable log)
    // 2. Disable minor's account access
    // 3. Write audit_log entry
  }

  async checkConsent(userId: string): Promise<boolean> {
    // 1. Check if user is_minor
    // 2. If minor, verify active 'granted' consent_record exists
    // 3. Return boolean
  }
}
```

**Testing:**
- `test-1.3.1`: Authenticate as a student -- receive JWT with correct roles claim; access student-only endpoint succeeds.
- `test-1.3.2`: Authenticate as a student -- access teacher-only endpoint returns 403 Forbidden.
- `test-1.3.3`: Create a minor user (is_minor=true) without consent -- access to tutoring endpoints blocked with 403 and message "Parental consent required."
- `test-1.3.4`: Complete parental consent flow (request -> email token -> grant) -- minor user can now access tutoring endpoints.
- `test-1.3.5`: Revoke consent -- minor user access is immediately blocked; audit_log entry created.
- `test-1.3.6`: Verify consent_record table is append-only: granting then revoking creates two records, not an update.
- `test-1.3.7`: JWT with expired token returns 401 Unauthorized.
- `test-1.3.8`: Institution admin can create users within their institution but not in other institutions.

---

### Task 1.4: Multi-Tenancy & Row-Level Security

**What:** Implement tenant isolation using PostgreSQL Row-Level Security (RLS). Create a NestJS middleware that sets `app.current_user_id` and `app.current_institution_id` on each database connection. Verify that queries cannot leak data across tenants.

**Design:**

```typescript
// apps/api/src/common/database/rls.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { DrizzleService } from './drizzle.service';

@Injectable()
export class RlsMiddleware implements NestMiddleware {
  constructor(private drizzle: DrizzleService) {}

  async use(req: any, res: any, next: () => void) {
    const userId = req.user?.id;
    const institutionId = req.user?.institutionId;

    if (userId && institutionId) {
      await this.drizzle.execute(sql`
        SET LOCAL app.current_user_id = ${userId};
        SET LOCAL app.current_institution_id = ${institutionId};
      `);
    }
    next();
  }
}
```

```sql
-- RLS policies
ALTER TABLE user_account ENABLE ROW LEVEL SECURITY;

CREATE POLICY institution_isolation ON user_account
    FOR ALL
    USING (institution_id = current_setting('app.current_institution_id')::UUID);

ALTER TABLE tutoring_session ENABLE ROW LEVEL SECURITY;

CREATE POLICY student_own_sessions ON tutoring_session
    FOR SELECT
    USING (student_id = current_setting('app.current_user_id')::UUID);

CREATE POLICY teacher_section_sessions ON tutoring_session
    FOR SELECT
    USING (section_id IN (
        SELECT section_id FROM enrollment
        WHERE user_id = current_setting('app.current_user_id')::UUID
        AND role = 'teacher'
    ));
```

**Testing:**
- `test-1.4.1`: With RLS enabled, query `user_account` as User A from Institution 1 -- returns zero users from Institution 2.
- `test-1.4.2`: With RLS enabled, query `tutoring_session` as Student X -- returns only Student X's sessions.
- `test-1.4.3`: With RLS enabled, query `tutoring_session` as Teacher Y -- returns sessions for all students in Teacher Y's sections but not other sections.
- `test-1.4.4`: Admin bypass: institution_admin can query all users within their institution.
- `test-1.4.5`: Attempt to set `app.current_institution_id` to a different institution's UUID via SQL injection in a request header -- middleware sanitizes input; RLS prevents cross-tenant access.

---

### Task 1.5: Docker Development Environment & CI Pipeline

**What:** Create `docker-compose.yml` for local development (PostgreSQL 16, Redis 7, Meilisearch, API, Web). Set up GitHub Actions CI workflow with lint, type-check, unit tests, integration tests (against Dockerized PostgreSQL), and build verification.

**Design:**

```yaml
# infrastructure/docker/docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: ai_tutor
      POSTGRES_USER: tutor
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-extensions.sql:/docker-entrypoint-initdb.d/01-extensions.sql

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  meilisearch:
    image: getmeili/meilisearch:v1.12
    ports: ["7700:7700"]
    environment:
      MEILI_ENV: development

  api:
    build: { context: ../.., dockerfile: infrastructure/docker/Dockerfile.api }
    depends_on: [postgres, redis]
    ports: ["3001:3001"]
    environment:
      DATABASE_URL: postgresql://tutor:dev_password@postgres:5432/ai_tutor
      REDIS_URL: redis://redis:6379

  web:
    build: { context: ../.., dockerfile: infrastructure/docker/Dockerfile.web }
    depends_on: [api]
    ports: ["3000:3000"]
```

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env: { POSTGRES_DB: test_db, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo build
```

**Testing:**
- `test-1.5.1`: Run `docker compose up` -- all services start and health checks pass within 60 seconds.
- `test-1.5.2`: API container connects to PostgreSQL and Redis on startup without errors.
- `test-1.5.3`: Web container connects to API and renders the landing page at `http://localhost:3000`.
- `test-1.5.4`: CI pipeline passes on a clean PR with all checks green.
- `test-1.5.5`: CI pipeline fails correctly when a lint error is introduced (assert non-zero exit).

---

### Phase 1 Definition of Done

- [ ] Monorepo builds from clean checkout with `npm ci && npx turbo build`
- [ ] PostgreSQL schema deployed with all Phase 1 tables (institution, user_account, consent_record, course, section, enrollment, audit_log)
- [ ] Authentication flow works end-to-end (login -> JWT -> protected endpoint)
- [ ] RBAC enforced: student, teacher, admin roles with correct access boundaries
- [ ] RLS prevents cross-tenant data access (verified by integration tests)
- [ ] COPPA consent flow works end-to-end for minor users
- [ ] Docker Compose environment starts in under 2 minutes
- [ ] CI pipeline runs lint, typecheck, and tests on every PR
- [ ] Seed script populates development database with 2 institutions, 5 users, 3 courses

---

## Phase 2: Knowledge Domain & Curriculum Engine

**Goal:** Build the knowledge component system, curriculum standard hierarchy, and prerequisite graph that underpin all tutoring, assessment, and adaptive sequencing.

**Duration:** 2-3 weeks

### Task 2.1: Knowledge Component CRUD & Prerequisite Graph

**What:** Implement the `knowledge_component` table and API. Build prerequisite relationship management with cycle detection. Enable `ltree` for curriculum standard hierarchies. Create a knowledge graph visualization endpoint for content authoring.

**Design:**

```typescript
// apps/api/src/modules/knowledge/services/knowledge-component.service.ts
export class KnowledgeComponentService {
  async create(dto: CreateKCDto): Promise<KnowledgeComponent> {
    // Validate prerequisites exist and no cycles would be introduced
    // Insert KC with JSONB metadata including standards_alignment, misconceptions
    // Return created KC
  }

  async addPrerequisite(kcId: string, prerequisiteId: string, strength: 'required' | 'recommended') {
    // 1. Verify both KCs exist
    // 2. Check for cycles via BFS/DFS on prerequisite graph
    // 3. Insert into metadata.prerequisites array
    // 4. Update prerequisite_count
  }

  async getPrerequisiteChain(kcId: string): Promise<KCWithDepth[]> {
    // Recursive query to get transitive closure of prerequisites
    // Returns ordered list with depth for visualization
  }

  async getReadyToLearn(studentId: string): Promise<KnowledgeComponent[]> {
    // 1. Get student's mastered KCs from student_knowledge
    // 2. Find KCs where ALL required prerequisites are mastered
    // 3. Filter out already-mastered KCs
    // 4. Order by difficulty level ascending
  }
}

// Cycle detection
async detectCycle(kcId: string, newPrereqId: string): Promise<boolean> {
  // BFS from newPrereqId following prerequisites
  // If we reach kcId, adding this edge would create a cycle
  const visited = new Set<string>();
  const queue = [newPrereqId];

  while (queue.length > 0) {
    const current = queue.shift()!;
    if (current === kcId) return true; // Cycle detected
    if (visited.has(current)) continue;
    visited.add(current);

    const prereqs = await this.getDirectPrerequisites(current);
    queue.push(...prereqs.map(p => p.id));
  }
  return false;
}
```

**Testing:**
- `test-2.1.1`: Create a KC with metadata including standards_alignment and misconceptions -- verify JSONB round-trips correctly.
- `test-2.1.2`: Add prerequisite A -> B -> C. Query prerequisite chain for C -- returns [B (depth 1), A (depth 2)].
- `test-2.1.3`: Attempt to add prerequisite C -> A (creating a cycle A -> B -> C -> A) -- returns 409 Conflict with "Cycle detected" message.
- `test-2.1.4`: Query `getReadyToLearn` for a student who has mastered A and B but not C -- returns C (and any other KCs whose prerequisites are all mastered).
- `test-2.1.5`: KC with no prerequisites is always ready to learn for any student.
- `test-2.1.6`: JSONB containment query: find all KCs aligned to standard "CCSS.MATH.CONTENT.8.EE.A.1" via `metadata @> '{"standards_alignment": [{"code": "CCSS.MATH.CONTENT.8.EE.A.1"}]}'` returns correct results.

---

### Task 2.2: Curriculum Standard Hierarchy

**What:** Implement curriculum standard management using `ltree` for hierarchical queries. Build import functionality for Common Core State Standards (CCSS) and NGSS. Create API endpoints for browsing standard trees and aligning KCs to standards.

**Design:**

```typescript
// apps/api/drizzle/schema/curriculum-standard.ts
export const curriculumStandard = pgTable('curriculum_standard', {
  id: uuid('id').primaryKey().defaultRandom(),
  framework: text('framework').notNull(),      // 'CCSS', 'NGSS', 'state_TX'
  code: text('code').notNull(),                 // 'CCSS.MATH.CONTENT.8.EE.A.1'
  title: text('title').notNull(),
  description: text('description'),
  subject: text('subject').notNull(),
  gradeBand: text('grade_band'),
  hierarchyPath: text('hierarchy_path'),         // ltree: 'ccss.math.grade8.ee.a1'
  parentStandardId: uuid('parent_standard_id').references(() => curriculumStandard.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// API: Get all standards under a hierarchy path
// GET /api/standards?path=ccss.math.grade8&framework=CCSS
// Uses: SELECT * FROM curriculum_standard WHERE hierarchy_path <@ 'ccss.math.grade8'::ltree
```

**Testing:**
- `test-2.2.1`: Import CCSS math standards -- verify hierarchical structure: root -> Math -> Grade 8 -> EE -> A.1.
- `test-2.2.2`: Query descendants of `ccss.math.grade8` using `ltree <@` operator -- returns all 8th-grade math standards.
- `test-2.2.3`: Query ancestors of `ccss.math.grade8.ee.a1` -- returns path from root to leaf.
- `test-2.2.4`: Align a KC to a standard -- verify bidirectional query works (KC -> standards, standard -> KCs).
- `test-2.2.5`: Import NGSS standards alongside CCSS -- verify framework filtering isolates correctly.

---

### Task 2.3: Assessment Item Authoring & QTI Import

**What:** Build the `assessment_item` table with JSONB content supporting multiple item types (choice, text_entry, extended_text, image_input). Implement QTI 3.0 import/export in the `packages/qti-parser` package. Create API for CRUD operations with JSON Schema validation on the JSONB content column.

**Design:**

```typescript
// packages/qti-parser/src/index.ts
export class QTIParser {
  parseItemXML(xml: string): AssessmentItemDTO {
    // Parse QTI 3.0 XML into internal JSONB format
    // Map QTI interaction types to internal item_type
    // Extract response declarations, scoring rules, hints
  }

  exportItemToQTI(item: AssessmentItem): string {
    // Convert internal JSONB format to QTI 3.0 XML
    // Map internal fields to QTI response-declaration
  }
}

// JSON Schema validation for assessment item content
const choiceItemSchema = {
  type: 'object',
  required: ['stem', 'choices'],
  properties: {
    stem: { type: 'string' },
    stem_format: { enum: ['text', 'latex', 'markdown'] },
    choices: {
      type: 'array',
      items: {
        type: 'object',
        required: ['id', 'value', 'correct'],
        properties: {
          id: { type: 'string' },
          value: { type: 'string' },
          correct: { type: 'boolean' },
          misconception: { type: 'string' },
        }
      },
      minItems: 2,
    },
    hints: { type: 'array' },
    knowledge_components: { type: 'array', items: { type: 'string', format: 'uuid' } },
  }
};
```

**Testing:**
- `test-2.3.1`: Create a multiple-choice math item with JSONB content -- verify all fields persist and are queryable.
- `test-2.3.2`: Create a free-text essay item with rubric in JSONB -- verify different content structure is accepted for `extended_text` type.
- `test-2.3.3`: Submit invalid JSONB content (missing required `stem` field) -- returns 422 Validation Error with JSON Schema violation details.
- `test-2.3.4`: Import a QTI 3.0 XML file with 10 choice items -- verify all 10 are created with correct internal structure.
- `test-2.3.5`: Export an item to QTI 3.0 XML, re-import it -- verify round-trip preserves all fields.
- `test-2.3.6`: Query items by KC using JSONB containment: `content @> '{"knowledge_components": ["uuid"]}'` -- returns correct items.

---

### Phase 2 Definition of Done

- [ ] Knowledge components can be created, updated, and queried with JSONB metadata
- [ ] Prerequisite graph supports add/remove with cycle detection
- [ ] `getReadyToLearn` returns correct KC recommendations given a student's mastery state
- [ ] Curriculum standards import works for CCSS; `ltree` hierarchy queries return correct results
- [ ] Assessment items support at least 3 item types (choice, text_entry, extended_text) with JSON Schema validation
- [ ] QTI 3.0 import/export round-trips correctly for choice items
- [ ] All endpoints are documented in generated OpenAPI spec
- [ ] Integration tests cover all CRUD operations and edge cases

---

## Phase 3: Socratic Dialogue Engine

**Goal:** Build the core LLM-powered Socratic tutoring experience: session management, conversation turn handling, streaming responses, and the pedagogical prompt engineering that guides without giving away answers.

**Duration:** 3-4 weeks

### Task 3.1: Tutoring Session Lifecycle

**What:** Implement the `tutoring_session` and `conversation_turn` tables and APIs. Build session creation (selecting mode, subject, target KCs), turn management (student message -> tutor response), and session ending (compute metrics, update session record).

**Design:**

```typescript
// apps/api/src/modules/tutoring/services/session.service.ts
export class SessionService {
  async startSession(dto: StartSessionDto): Promise<TutoringSession> {
    // 1. Verify student has consent (if minor)
    // 2. Load institution config for allowed modes and LLM settings
    // 3. Create tutoring_session record
    // 4. Initialize session state in Redis (for real-time updates)
    // 5. Return session with WebSocket connection details
  }

  async addStudentTurn(sessionId: string, content: string, format: string): Promise<ConversationTurn> {
    // 1. Validate session is active
    // 2. Insert student conversation_turn
    // 3. Trigger Socratic dialogue engine (Task 3.2)
    // 4. Stream tutor response back via WebSocket
    // 5. Insert tutor conversation_turn when complete
    // 6. Update session metrics JSONB
  }

  async endSession(sessionId: string): Promise<SessionSummary> {
    // 1. Compute final metrics (total_turns, items_attempted, items_correct, duration)
    // 2. Update tutoring_session record
    // 3. Clean up Redis session state
    // 4. Trigger async knowledge state update (Phase 4)
    // 5. Return session summary
  }
}

// WebSocket gateway for real-time session communication
// apps/api/src/modules/tutoring/gateways/session.gateway.ts
@WebSocketGateway({ namespace: '/tutoring' })
export class SessionGateway {
  @SubscribeMessage('student_message')
  async handleStudentMessage(client: Socket, payload: StudentMessageDto) {
    // 1. Authenticate via JWT in handshake
    // 2. Delegate to SessionService.addStudentTurn
    // 3. Stream tutor response tokens to client as they arrive
    client.emit('tutor_token', { token: '...', done: false });
    // ... final message:
    client.emit('tutor_token', { token: '', done: true, fullResponse: '...' });
  }
}
```

**Testing:**
- `test-3.1.1`: Start a Socratic session -- returns session ID, mode, and WebSocket URL.
- `test-3.1.2`: Send a student message -- receives streamed tutor response tokens via WebSocket.
- `test-3.1.3`: End a session -- session metrics JSONB is populated with correct counts.
- `test-3.1.4`: Attempt to add a turn to an ended session -- returns 409 "Session already ended."
- `test-3.1.5`: Minor without consent attempts to start session -- returns 403.
- `test-3.1.6`: Session respects institution's `allowedSessionModes` -- attempting a disallowed mode returns 400.
- `test-3.1.7`: Concurrent sessions from same student -- configurable limit enforced (default: 1 active session).

---

### Task 3.2: Socratic Dialogue Prompt Engineering

**What:** Design and implement the system prompt architecture for Socratic tutoring. Build the prompt construction pipeline that assembles: system instructions (pedagogical rules), domain context (KC being tutored, prerequisite state), session history, and student response. Configure the Vercel AI SDK for streaming with Claude API.

**Design:**

```typescript
// apps/api/src/modules/dialogue/services/socratic-engine.service.ts
export class SocraticEngineService {
  constructor(
    private llmProvider: LLMProviderService,
    private knowledgeService: KnowledgeComponentService,
  ) {}

  async generateResponse(session: TutoringSession, studentMessage: string): Promise<AsyncIterable<string>> {
    const systemPrompt = this.buildSystemPrompt(session);
    const conversationHistory = await this.getSessionHistory(session.id);
    const domainContext = await this.buildDomainContext(session);

    return this.llmProvider.streamChat({
      model: session.llmModel,
      system: systemPrompt,
      messages: [
        { role: 'system', content: domainContext },
        ...conversationHistory,
        { role: 'user', content: studentMessage },
      ],
      temperature: 0.3,  // Low temperature for pedagogical consistency
      maxTokens: 512,
    });
  }

  private buildSystemPrompt(session: TutoringSession): string {
    return `You are a Socratic tutor. Your role is to guide the student to understanding through questioning, not by providing direct answers.

RULES:
1. NEVER give the student the direct answer to a problem.
2. When the student gives a wrong answer, ask a guiding question that leads them toward the correct reasoning.
3. When the student gives a correct answer, confirm it and ask them to explain WHY it is correct.
4. Use at most ${session.maxHintLevels} levels of increasingly specific hints before showing a worked step.
5. Match your language complexity to grade level: ${session.gradeLevel}.
6. Respond in ${session.language}.
7. If you detect a specific misconception, address it with a targeted counter-example.
8. Keep responses concise (2-4 sentences per turn).
9. Use LaTeX notation for mathematical expressions, wrapped in $ delimiters.

SESSION CONTEXT:
- Mode: ${session.sessionMode}
- Subject: ${session.subject}
- Student grade level: ${session.gradeLevel}
- Current topic: ${session.currentKC?.displayName}`;
  }

  private async buildDomainContext(session: TutoringSession): string {
    const kc = await this.knowledgeService.getById(session.currentKCId);
    const misconceptions = kc.metadata.misconceptions || [];
    const prerequisites = await this.knowledgeService.getDirectPrerequisites(kc.id);

    return `DOMAIN KNOWLEDGE:
Topic: ${kc.displayName}
Description: ${kc.metadata.description || 'N/A'}
Common misconceptions to watch for:
${misconceptions.map(m => `- ${m.name}: ${m.description} (remediation: ${m.remediation})`).join('\n')}

Prerequisite topics (student should already know these):
${prerequisites.map(p => `- ${p.displayName}`).join('\n')}`;
  }
}

// apps/api/src/modules/dialogue/services/llm-provider.service.ts
import { createAnthropic } from '@ai-sdk/anthropic';
import { streamText } from 'ai';

export class LLMProviderService {
  private anthropic = createAnthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

  async streamChat(options: ChatOptions): Promise<AsyncIterable<string>> {
    const result = streamText({
      model: this.anthropic('claude-sonnet-4-20250514'),
      system: options.system,
      messages: options.messages,
      temperature: options.temperature,
      maxTokens: options.maxTokens,
    });
    return result.textStream;
  }
}
```

**Testing:**
- `test-3.2.1`: Given a math problem where student answers incorrectly, verify tutor response contains a guiding question (not the answer). Use LLM evaluation rubric: "Does the response contain a question mark?" AND "Does the response NOT contain the correct answer?"
- `test-3.2.2`: Given a correct student answer, verify tutor confirms and asks for explanation.
- `test-3.2.3`: System prompt includes correct KC context (display name, misconceptions, prerequisites).
- `test-3.2.4`: Verify streaming works: first token arrives within 2 seconds; all tokens arrive; final message is coherent.
- `test-3.2.5`: Session in Spanish (`language: 'es'`) -- tutor responds in Spanish.
- `test-3.2.6`: **Promptfoo evaluation suite:** Run 50 test cases across 5 math topics. Measure: (a) answer leakage rate < 5%, (b) question-posing rate > 80%, (c) misconception detection when student exhibits known misconception > 60%.
- `test-3.2.7`: Token count per response stays within maxTokens limit.

---

### Task 3.3: Student Tutoring Interface

**What:** Build the Next.js student tutoring page with chat interface, LaTeX rendering (KaTeX), session controls (start, pause, end), and real-time streaming display. Implement mobile-responsive layout.

**Design:**

```typescript
// apps/web/src/app/(student)/tutor/[sessionId]/page.tsx
// Server component that loads session metadata, renders client chat component

// apps/web/src/components/tutoring/ChatInterface.tsx
'use client';
import { useSocket } from '@/hooks/useSocket';
import { TutorMessage } from './TutorMessage';
import { StudentInput } from './StudentInput';
import { KaTeXRenderer } from './KaTeXRenderer';

export function ChatInterface({ sessionId }: { sessionId: string }) {
  const { messages, sendMessage, isStreaming } = useSocket(sessionId);

  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg) => (
          msg.role === 'tutor'
            ? <TutorMessage key={msg.id} content={msg.content} isStreaming={msg.isStreaming} />
            : <StudentMessage key={msg.id} content={msg.content} />
        ))}
      </div>
      <StudentInput
        onSend={sendMessage}
        disabled={isStreaming}
        placeholder="Type your answer or ask a question..."
      />
    </div>
  );
}

// apps/web/src/components/tutoring/TutorMessage.tsx
// Renders tutor message with KaTeX for LaTeX expressions
// Detects $...$ and $$...$$ delimiters and renders via KaTeX
export function TutorMessage({ content, isStreaming }: TutorMessageProps) {
  const parts = parseLatex(content); // Split into text and LaTeX segments
  return (
    <div className="bg-blue-50 rounded-lg p-3 max-w-[80%]" role="log" aria-live="polite">
      {parts.map((part, i) =>
        part.type === 'latex'
          ? <KaTeXRenderer key={i} expression={part.value} />
          : <span key={i}>{part.value}</span>
      )}
      {isStreaming && <span className="animate-pulse">|</span>}
    </div>
  );
}
```

**Testing:**
- `test-3.3.1`: Student starts a session from the UI -- chat interface loads with welcome message from tutor.
- `test-3.3.2`: Student types a message and presses Enter -- message appears in chat; tutor response streams in character-by-character.
- `test-3.3.3`: LaTeX expression `$3x + 5 = 20$` in tutor message renders as formatted math (not raw LaTeX).
- `test-3.3.4`: Student input is disabled while tutor response is streaming; re-enabled when streaming completes.
- `test-3.3.5`: Mobile viewport (375px wide) -- chat interface is usable with no horizontal scroll.
- `test-3.3.6`: Screen reader announces new tutor messages via `aria-live="polite"`.
- `test-3.3.7`: End session button -- confirmation dialog appears; session ends; summary is displayed.

---

### Phase 3 Definition of Done

- [ ] Students can start a Socratic tutoring session, exchange messages, and end the session
- [ ] Tutor responses stream in real-time via WebSocket
- [ ] Socratic engine withholds direct answers and poses guiding questions (verified by Promptfoo suite: answer leakage < 5%)
- [ ] LaTeX renders correctly in tutor messages
- [ ] Session metrics (turns, duration, items) are computed and stored on session end
- [ ] Conversation history persists and is reloadable
- [ ] Mobile-responsive chat interface passes visual review on 375px viewport

---

## Phase 4: Student Knowledge Model & Adaptive Sequencing

**Goal:** Implement Bayesian Knowledge Tracing (BKT) to maintain per-student, per-KC mastery estimates. Build adaptive sequencing that selects the next KC or item based on the student's current knowledge state.

**Duration:** 2-3 weeks

### Task 4.1: BKT Engine Implementation

**What:** Build the `packages/bkt-engine` library implementing Bayesian Knowledge Tracing. The engine takes a student's current mastery state and an observation (correct/incorrect) and returns updated BKT parameters. Implement both standard BKT and individualized BKT (per-student slip/guess parameters).

**Design:**

```typescript
// packages/bkt-engine/src/index.ts
export interface BKTParams {
  pMastery: number;  // P(L_n): probability student has learned the KC
  pTransit: number;  // P(T): probability of learning on each opportunity
  pSlip: number;     // P(S): probability of incorrect answer despite mastery
  pGuess: number;    // P(G): probability of correct answer despite non-mastery
}

export interface BKTObservation {
  isCorrect: boolean;
  timestamp: Date;
  hintCount: number;       // Penalise mastery estimate if hints were used
  timeSpentMs: number;     // Very fast correct answers may be guesses
}

export function updateBKT(prior: BKTParams, observation: BKTObservation): BKTParams {
  const { pMastery, pTransit, pSlip, pGuess } = prior;
  const { isCorrect } = observation;

  // Step 1: Compute posterior P(L_n | observation)
  let posteriorMastery: number;
  if (isCorrect) {
    // P(L_n | correct) = P(correct | L_n) * P(L_n) / P(correct)
    const pCorrectGivenMastered = 1 - pSlip;
    const pCorrectGivenNotMastered = pGuess;
    const pCorrect = pCorrectGivenMastered * pMastery + pCorrectGivenNotMastered * (1 - pMastery);
    posteriorMastery = (pCorrectGivenMastered * pMastery) / pCorrect;
  } else {
    // P(L_n | incorrect) = P(incorrect | L_n) * P(L_n) / P(incorrect)
    const pIncorrectGivenMastered = pSlip;
    const pIncorrectGivenNotMastered = 1 - pGuess;
    const pIncorrect = pIncorrectGivenMastered * pMastery + pIncorrectGivenNotMastered * (1 - pMastery);
    posteriorMastery = (pIncorrectGivenMastered * pMastery) / pIncorrect;
  }

  // Step 2: Apply learning transition
  // P(L_{n+1}) = P(L_n | obs) + (1 - P(L_n | obs)) * P(T)
  const newPMastery = posteriorMastery + (1 - posteriorMastery) * pTransit;

  // Penalty for hint usage: reduce mastery gain
  const hintPenalty = Math.min(observation.hintCount * 0.05, 0.15);
  const adjustedMastery = Math.max(0, newPMastery - hintPenalty);

  return {
    pMastery: Math.min(1, Math.max(0, adjustedMastery)),
    pTransit: pTransit,
    pSlip: pSlip,
    pGuess: pGuess,
  };
}

export function getMasteryStatus(pMastery: number): MasteryStatus {
  if (pMastery >= 0.85) return 'mastered';
  if (pMastery >= 0.3) return 'learning';
  if (pMastery > 0) return 'not_started';  // attempted but low
  return 'not_started';
}
```

**Testing:**
- `test-4.1.1`: Initial P(mastery) = 0.3; correct answer without hints -- P(mastery) increases.
- `test-4.1.2`: Initial P(mastery) = 0.3; incorrect answer -- P(mastery) decreases.
- `test-4.1.3`: After 5 consecutive correct answers, P(mastery) exceeds 0.85 (mastered threshold).
- `test-4.1.4`: Correct answer with 3 hints -- mastery increase is less than correct answer with 0 hints.
- `test-4.1.5`: P(mastery) never exceeds 1.0 or drops below 0.0.
- `test-4.1.6`: Known BKT reference values: given P(L0)=0.3, P(T)=0.09, P(S)=0.1, P(G)=0.2, one correct observation -> expected P(L1) matches published BKT formula result to 4 decimal places.
- `test-4.1.7`: getMasteryStatus correctly categorizes: 0.0 -> not_started, 0.5 -> learning, 0.9 -> mastered.

---

### Task 4.2: Student Knowledge State Persistence

**What:** Implement the `student_knowledge` table with JSONB `model_params`. Build the service that updates mastery after each student attempt, stores mastery history in the JSONB, and caches current mastery in Redis for fast access during sessions.

**Design:**

```typescript
// apps/api/src/modules/student-model/services/knowledge-state.service.ts
export class KnowledgeStateService {
  async updateMastery(studentId: string, kcId: string, observation: BKTObservation): Promise<StudentKnowledge> {
    // 1. Load current state from DB (or create if first attempt)
    const state = await this.getOrCreate(studentId, kcId);

    // 2. Run BKT update
    const currentParams: BKTParams = {
      pMastery: state.pMastery,
      pTransit: state.modelParams.bkt.p_transit,
      pSlip: state.modelParams.bkt.p_slip,
      pGuess: state.modelParams.bkt.p_guess,
    };
    const updated = updateBKT(currentParams, observation);

    // 3. Update JSONB model_params with new values and append to mastery_history
    const newModelParams = {
      ...state.modelParams,
      bkt: { p_transit: updated.pTransit, p_slip: updated.pSlip, p_guess: updated.pGuess },
      total_attempts: state.modelParams.total_attempts + 1,
      correct_attempts: state.modelParams.correct_attempts + (observation.isCorrect ? 1 : 0),
      mastery_history: [
        ...state.modelParams.mastery_history,
        { date: new Date().toISOString(), p_mastery: updated.pMastery },
      ],
    };

    // 4. Persist to DB
    const result = await this.db.update(studentKnowledge)
      .set({
        pMastery: updated.pMastery,
        masteryStatus: getMasteryStatus(updated.pMastery),
        modelParams: newModelParams,
        lastAttemptAt: new Date(),
        updatedAt: new Date(),
      })
      .where(and(
        eq(studentKnowledge.studentId, studentId),
        eq(studentKnowledge.knowledgeComponentId, kcId),
      ))
      .returning();

    // 5. Update Redis cache
    await this.redis.hset(`mastery:${studentId}`, kcId, JSON.stringify({
      pMastery: updated.pMastery,
      status: getMasteryStatus(updated.pMastery),
    }));

    return result[0];
  }

  async getStudentProfile(studentId: string): Promise<MasteryProfile> {
    // Return all KCs with mastery state for a student
    // Use Redis cache first, fall back to DB
  }
}
```

**Testing:**
- `test-4.2.1`: First attempt on a KC creates `student_knowledge` record with default BKT params.
- `test-4.2.2`: Multiple attempts update `model_params.mastery_history` array with timestamped entries.
- `test-4.2.3`: Redis cache reflects latest mastery after update; subsequent reads hit cache, not DB.
- `test-4.2.4`: getStudentProfile returns mastery for all practiced KCs; unpracticed KCs are absent.
- `test-4.2.5`: Concurrent updates to the same student-KC pair are serialized (no lost updates).

---

### Task 4.3: Adaptive Item Selection

**What:** Build the adaptive sequencing service that selects the next assessment item for a student based on: (1) target KC, (2) item difficulty relative to student mastery, (3) item discrimination, and (4) recently-seen item avoidance.

**Design:**

```typescript
// apps/api/src/modules/student-model/services/adaptive-sequencer.service.ts
export class AdaptiveSequencerService {
  async selectNextItem(studentId: string, targetKCId: string): Promise<AssessmentItem> {
    const mastery = await this.knowledgeState.getMastery(studentId, targetKCId);

    // Target difficulty band: items slightly above current mastery (zone of proximal development)
    const targetDifficulty = Math.min(mastery.pMastery + 0.15, 0.95);
    const difficultyRange = { min: targetDifficulty - 0.15, max: targetDifficulty + 0.15 };

    // Get candidate items for this KC within difficulty range
    const candidates = await this.assessmentService.findByKCAndDifficulty(
      targetKCId, difficultyRange.min, difficultyRange.max
    );

    // Filter out recently-seen items (last 5 sessions)
    const recentItemIds = await this.getRecentItemIds(studentId, 5);
    const filtered = candidates.filter(item => !recentItemIds.includes(item.id));

    // If no items available after filtering, widen difficulty range
    if (filtered.length === 0) {
      return this.selectFallbackItem(studentId, targetKCId);
    }

    // Select item with highest discrimination (most informative for mastery estimation)
    filtered.sort((a, b) => (b.discrimination ?? 0) - (a.discrimination ?? 0));
    return filtered[0];
  }
}
```

**Testing:**
- `test-4.3.1`: Student with P(mastery)=0.4 receives items with difficulty around 0.55 (ZPD targeting).
- `test-4.3.2`: Student who just answered item X does not receive item X again in the next 5 sessions.
- `test-4.3.3`: When no items match the difficulty range, fallback returns any item for the KC (not an error).
- `test-4.3.4`: Items with higher discrimination are preferred over lower discrimination items at the same difficulty.
- `test-4.3.5`: Student with P(mastery)=0.9 (mastered) -- sequencer recommends moving to next KC in prerequisite chain.

---

### Phase 4 Definition of Done

- [ ] BKT engine produces correct mastery updates matching published reference values
- [ ] Student knowledge state persists across sessions with full mastery history
- [ ] Redis cache accelerates mastery lookups during active sessions
- [ ] Adaptive sequencer selects items in the zone of proximal development
- [ ] Recently-seen items are avoided
- [ ] Mastery status transitions (not_started -> learning -> mastered) work correctly

---

## Phase 5: Assessment & Hint Scaffolding

**Goal:** Build the complete assessment experience: item presentation, response capture, hint scaffolding (Socratic question -> worked step -> direct hint), and step-by-step worked solution mode.

**Duration:** 2-3 weeks

### Task 5.1: Item Rendering & Response Capture

**What:** Build React components that render assessment items from JSONB content. Support multiple item types: multiple-choice, text entry, extended text, and image input. Capture student responses and submit to the API.

**Design:**

```typescript
// apps/web/src/components/assessment/ItemRenderer.tsx
export function ItemRenderer({ item, onSubmit }: ItemRendererProps) {
  switch (item.itemType) {
    case 'choice':
      return <ChoiceItem item={item} onSubmit={onSubmit} />;
    case 'text_entry':
      return <TextEntryItem item={item} onSubmit={onSubmit} />;
    case 'extended_text':
      return <ExtendedTextItem item={item} onSubmit={onSubmit} />;
    case 'image_input':
      return <ImageInputItem item={item} onSubmit={onSubmit} />;
    default:
      return <UnsupportedItem />;
  }
}

// apps/web/src/components/assessment/ChoiceItem.tsx
export function ChoiceItem({ item, onSubmit }: ChoiceItemProps) {
  const [selected, setSelected] = useState<string | null>(null);
  const content = item.content as ChoiceItemContent;

  return (
    <div role="group" aria-labelledby="item-stem">
      <div id="item-stem" className="text-lg mb-4">
        <KaTeXRenderer content={content.stem} format={content.stem_format} />
      </div>
      <RadioGroup value={selected} onValueChange={setSelected}>
        {content.choices.map(choice => (
          <RadioGroupItem key={choice.id} value={choice.id} aria-label={choice.value}>
            <KaTeXRenderer content={choice.value} />
          </RadioGroupItem>
        ))}
      </RadioGroup>
      <Button onClick={() => onSubmit({ choiceId: selected })} disabled={!selected}>
        Submit Answer
      </Button>
    </div>
  );
}
```

**Testing:**
- `test-5.1.1`: Choice item renders all options; selecting one and submitting sends correct response payload.
- `test-5.1.2`: Text entry item accepts free-text input; LaTeX input is preserved in response.
- `test-5.1.3`: Extended text item renders rubric criteria; word count is displayed.
- `test-5.1.4`: Image input item accepts camera/file upload; preview is shown before submission.
- `test-5.1.5`: Keyboard navigation through choice items works (Tab, Space/Enter to select, Enter to submit).
- `test-5.1.6`: Screen reader reads item stem and choice values correctly.

---

### Task 5.2: Hint Scaffolding System

**What:** Implement the multi-level hint system. When a student answers incorrectly, offer escalating hints: Level 1 (Socratic question) -> Level 2 (worked step) -> Level 3 (direct hint / full worked solution). Track hint usage in session metrics and apply BKT hint penalty.

**Design:**

```typescript
// apps/api/src/modules/assessment/services/hint.service.ts
export class HintService {
  async getNextHint(sessionId: string, itemId: string, currentLevel: number): Promise<Hint> {
    const item = await this.assessmentService.getById(itemId);
    const hints = item.content.hints || [];

    if (currentLevel >= hints.length) {
      // All predefined hints exhausted: generate one via LLM
      return this.generateDynamicHint(item, currentLevel);
    }

    const hint = hints[currentLevel];

    // Record hint request in session metrics
    await this.sessionService.recordHintRequest(sessionId, itemId, currentLevel + 1);

    return {
      level: currentLevel + 1,
      type: hint.type,
      content: hint.text,
      isLast: currentLevel + 1 >= hints.length,
      revealsAnswer: hint.type === 'direct',
    };
  }

  private async generateDynamicHint(item: AssessmentItem, level: number): Promise<Hint> {
    // Use LLM to generate a contextual hint without revealing the answer
    const prompt = `Generate a hint for the following problem that guides the student
    without giving away the answer. This is hint level ${level + 1}, so be more specific
    than a general Socratic question but still don't reveal the full solution.
    Problem: ${item.content.stem}`;

    const hintContent = await this.llm.generate(prompt);
    return { level: level + 1, type: 'ai_generated', content: hintContent, isLast: false, revealsAnswer: false };
  }
}

// apps/web/src/components/assessment/HintPanel.tsx
export function HintPanel({ hints, onRequestHint, maxLevel }: HintPanelProps) {
  return (
    <div className="border-l-4 border-amber-400 bg-amber-50 p-4">
      <h3 className="font-semibold mb-2">Hints</h3>
      {hints.map((hint, i) => (
        <div key={i} className="mb-2">
          <Badge variant={hint.type === 'socratic' ? 'outline' : 'default'}>
            Level {hint.level}: {hint.type}
          </Badge>
          <p className="mt-1"><KaTeXRenderer content={hint.content} /></p>
        </div>
      ))}
      {hints.length < maxLevel && (
        <Button variant="outline" size="sm" onClick={onRequestHint}>
          Get Another Hint ({maxLevel - hints.length} remaining)
        </Button>
      )}
    </div>
  );
}
```

**Testing:**
- `test-5.2.1`: First hint request returns Level 1 Socratic question (does not contain the answer).
- `test-5.2.2`: Second hint request returns Level 2 worked step (shows partial solution).
- `test-5.2.3`: Third hint request returns Level 3 direct hint (may reveal the answer path).
- `test-5.2.4`: Hint request after all predefined hints are exhausted triggers LLM-generated hint.
- `test-5.2.5`: Session metrics JSONB reflects correct `hints_requested` count after hint requests.
- `test-5.2.6`: BKT update after a correct answer with 2 hints penalizes mastery gain compared to 0 hints.
- `test-5.2.7`: Institution config `maxHintLevels: 2` limits hints to 2 levels even if item has 3 defined.

---

### Task 5.3: Step-by-Step Worked Solution Mode

**What:** Implement the "worked_example" session mode where students can reveal solution steps one at a time. Each step has an explanation. Students can attempt to predict the next step before revealing it.

**Design:**

```typescript
// apps/api/src/modules/assessment/services/worked-solution.service.ts
export class WorkedSolutionService {
  async getWorkedSolution(itemId: string): Promise<WorkedSolution> {
    const item = await this.assessmentService.getById(itemId);

    // For items with predefined solution steps, return them
    if (item.content.solution_steps) {
      return {
        totalSteps: item.content.solution_steps.length,
        steps: item.content.solution_steps.map((step, i) => ({
          number: i + 1,
          description: step.description,
          content: step.content,
          explanation: step.explanation,
        })),
      };
    }

    // For items without predefined steps, generate via LLM
    return this.generateWorkedSolution(item);
  }
}

// apps/web/src/components/assessment/WorkedSolution.tsx
export function WorkedSolution({ solution }: WorkedSolutionProps) {
  const [revealedSteps, setRevealedSteps] = useState(0);

  return (
    <div className="space-y-4">
      {solution.steps.slice(0, revealedSteps).map((step) => (
        <div key={step.number} className="border rounded-lg p-4">
          <div className="font-semibold">Step {step.number}: {step.description}</div>
          <KaTeXRenderer content={step.content} />
          <p className="text-gray-600 mt-2">{step.explanation}</p>
        </div>
      ))}
      {revealedSteps < solution.totalSteps && (
        <Button onClick={() => setRevealedSteps(prev => prev + 1)}>
          Reveal Step {revealedSteps + 1} of {solution.totalSteps}
        </Button>
      )}
    </div>
  );
}
```

**Testing:**
- `test-5.3.1`: Worked solution for an item with 4 steps initially shows 0 steps.
- `test-5.3.2`: Clicking "Reveal Step 1" shows only Step 1; Step 2 is hidden.
- `test-5.3.3`: All steps revealed sequentially -- final state shows complete solution.
- `test-5.3.4`: Items without predefined steps get LLM-generated worked solution with coherent step progression.
- `test-5.3.5`: LaTeX in solution steps renders correctly via KaTeX.

---

### Phase 5 Definition of Done

- [ ] All 4 item types (choice, text_entry, extended_text, image_input) render and capture responses correctly
- [ ] Hint scaffolding escalates through levels (Socratic -> worked step -> direct)
- [ ] LLM-generated hints are available when predefined hints are exhausted
- [ ] Hint usage is tracked and penalizes BKT mastery gain
- [ ] Step-by-step worked solution mode reveals steps one at a time
- [ ] All assessment interactions are accessible (keyboard, screen reader)

---

## Phase 6: Teacher Dashboard & Alerts

**Goal:** Build the teacher-facing dashboard showing class-wide mastery, stuck-student alerts, session activity, and actionable re-teaching recommendations.

**Duration:** 2-3 weeks

### Task 6.1: Class Mastery Dashboard

**What:** Build the teacher dashboard showing a mastery grid: KCs on one axis, students on the other, with color-coded mastery status. Include section-level aggregates (% mastered, % learning, % struggling). Use materialized views for performance.

**Design:**

```typescript
// apps/api/src/modules/teacher/services/dashboard.service.ts
export class DashboardService {
  async getClassMastery(sectionId: string): Promise<ClassMasteryGrid> {
    // Query student_knowledge for all students enrolled in section
    // Group by KC, compute aggregates
    const students = await this.enrollmentService.getStudentsInSection(sectionId);
    const studentIds = students.map(s => s.id);

    const masteryData = await this.db
      .select()
      .from(studentKnowledge)
      .where(inArray(studentKnowledge.studentId, studentIds));

    // Build grid: rows = students, columns = KCs
    return this.buildMasteryGrid(students, masteryData);
  }

  async getClassSummary(sectionId: string): Promise<ClassSummary> {
    // Aggregate mastery across all students and KCs
    // Return: total students, % mastered per KC, most common misconceptions
  }
}

// apps/web/src/components/dashboard/MasteryGrid.tsx
// Color-coded grid: green (mastered > 0.85), yellow (learning 0.3-0.85), red (struggling < 0.3)
export function MasteryGrid({ data }: MasteryGridProps) {
  return (
    <table role="grid" aria-label="Class mastery overview">
      <thead>
        <tr>
          <th>Student</th>
          {data.knowledgeComponents.map(kc => <th key={kc.id}>{kc.displayName}</th>)}
        </tr>
      </thead>
      <tbody>
        {data.students.map(student => (
          <tr key={student.id}>
            <td>{student.displayName}</td>
            {data.knowledgeComponents.map(kc => {
              const mastery = data.getMastery(student.id, kc.id);
              return (
                <td key={kc.id} className={masteryColorClass(mastery?.pMastery)}>
                  {mastery ? `${Math.round(mastery.pMastery * 100)}%` : '--'}
                </td>
              );
            })}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Testing:**
- `test-6.1.1`: Dashboard loads for a section with 25 students and 10 KCs -- renders grid within 2 seconds.
- `test-6.1.2`: Color coding is correct: green for P(mastery) >= 0.85, yellow for 0.3-0.85, red for < 0.3.
- `test-6.1.3`: Students not enrolled in the section do not appear in the grid.
- `test-6.1.4`: Teacher can only view dashboard for sections they are enrolled in as `role: 'teacher'`.
- `test-6.1.5`: Grid is accessible with ARIA table roles; screen reader announces cell values.

---

### Task 6.2: Stuck-Student Alerts

**What:** Build an alert system that detects struggling students and notifies teachers. Alert triggers: (1) student has 3+ incorrect attempts on same item, (2) student's mastery is declining over last 5 attempts, (3) student has been inactive for >10 minutes during an active session. Alerts appear in real-time via WebSocket.

**Design:**

```typescript
// apps/api/src/modules/teacher/services/alert.service.ts
export class AlertService {
  async evaluateAlerts(studentId: string, sessionId: string, kcId: string): Promise<void> {
    // Called after each student attempt

    // Alert 1: Stuck on same item
    const recentAttempts = await this.getRecentAttemptsOnItem(sessionId, kcId);
    if (recentAttempts.filter(a => !a.isCorrect).length >= 3) {
      await this.createAlert({
        alertType: 'student_stuck',
        severity: 'warning',
        title: `${student.displayName} is stuck on ${kc.displayName}`,
        details: { consecutiveIncorrect: 3, itemId: recentAttempts[0].assessmentItemId },
      });
    }

    // Alert 2: Declining mastery
    const masteryHistory = await this.getMasteryTrend(studentId, kcId, 5);
    if (this.isDeclinining(masteryHistory)) {
      await this.createAlert({
        alertType: 'reteach_needed',
        severity: 'critical',
        title: `${student.displayName}'s mastery of ${kc.displayName} is declining`,
        details: { trend: masteryHistory },
      });
    }
  }

  private async createAlert(dto: CreateAlertDto): Promise<void> {
    // 1. Insert teacher_alert record
    // 2. Push real-time notification to teacher via WebSocket
    const teachers = await this.enrollmentService.getTeachersForSection(dto.sectionId);
    for (const teacher of teachers) {
      await this.db.insert(teacherAlert).values({ ...dto, teacherId: teacher.id });
      this.wsGateway.sendToUser(teacher.id, 'alert', dto);
    }
  }
}
```

**Testing:**
- `test-6.2.1`: Student gets 3 consecutive wrong answers on same item -- teacher receives 'student_stuck' alert.
- `test-6.2.2`: Student's mastery declines over 5 attempts (0.6 -> 0.5 -> 0.45 -> 0.4 -> 0.35) -- teacher receives 'reteach_needed' alert.
- `test-6.2.3`: Alert appears in real-time on teacher's dashboard via WebSocket (no page refresh needed).
- `test-6.2.4`: Marking an alert as read updates `is_read` and `read_at` fields.
- `test-6.2.5`: Teacher only receives alerts for students in their sections, not other teachers' sections.
- `test-6.2.6`: Duplicate alerts for same student/KC are suppressed within a 30-minute window.

---

### Task 6.3: Session Activity View

**What:** Build a real-time view of active tutoring sessions in a section. Teachers see which students are currently tutoring, their current KC, turn count, and time elapsed. Enables teachers to monitor engagement during class.

**Design:**

```typescript
// apps/web/src/components/dashboard/ActiveSessions.tsx
export function ActiveSessions({ sectionId }: { sectionId: string }) {
  const { sessions } = useRealtimeSessions(sectionId);  // WebSocket subscription

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {sessions.map(session => (
        <Card key={session.id}>
          <CardHeader>
            <Avatar>{session.student.initials}</Avatar>
            <span>{session.student.displayName}</span>
            <Badge variant={session.isActive ? 'success' : 'secondary'}>
              {session.isActive ? 'Active' : 'Idle'}
            </Badge>
          </CardHeader>
          <CardContent>
            <p>Topic: {session.currentKC}</p>
            <p>Turns: {session.totalTurns}</p>
            <p>Duration: {formatDuration(session.elapsed)}</p>
            <p>Hints used: {session.hintsRequested}</p>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

**Testing:**
- `test-6.3.1`: When a student starts a session, the teacher's active sessions view updates within 2 seconds.
- `test-6.3.2`: Session card shows correct KC, turn count, and duration.
- `test-6.3.3`: When a student ends a session, the card disappears from the active view.
- `test-6.3.4`: Idle detection: student with no activity for 10+ minutes shows "Idle" badge.

---

### Phase 6 Definition of Done

- [ ] Teacher dashboard shows class mastery grid with correct color coding
- [ ] Stuck-student alerts trigger on 3+ incorrect attempts or declining mastery
- [ ] Alerts appear in real-time via WebSocket without page refresh
- [ ] Active session view shows live student activity during class
- [ ] All dashboard data respects RLS (teachers see only their sections)
- [ ] Dashboard loads within 2 seconds for classes up to 35 students

---

## Phase 7: LMS Integration (LTI 1.3 & OneRoster)

**Goal:** Enable institutional deployment by implementing LTI 1.3 (embed tutoring in Canvas/Moodle/Blackboard) and OneRoster (import rosters from SIS).

**Duration:** 3-4 weeks

### Task 7.1: LTI 1.3 Tool Registration & Launch

**What:** Implement the LTI 1.3 launch flow in the `packages/lti-toolkit` library. Handle OpenID Connect login initiation, JWT validation, role mapping, and deep linking. Build the NestJS module for LTI configuration and launch processing.

**Design:**

```typescript
// packages/lti-toolkit/src/launch.ts
export class LTILaunchHandler {
  async handleOIDCLogin(params: OIDCLoginParams): Promise<string> {
    // 1. Validate login_hint and target_link_uri
    // 2. Generate state and nonce
    // 3. Return redirect URL to platform auth endpoint
  }

  async handleLaunch(idToken: string, state: string): Promise<LTILaunchContext> {
    // 1. Validate state matches stored nonce
    // 2. Decode and validate JWT id_token using platform JWKS
    // 3. Extract claims: user identity, roles, context (course/section)
    // 4. Map LTI roles to internal roles
    // 5. Create or update user_account
    // 6. Return launch context with session details
  }
}

// Role mapping
const LTI_ROLE_MAP: Record<string, UserRole> = {
  'http://purl.imsglobal.org/vocab/lis/v2/membership#Instructor': 'teacher',
  'http://purl.imsglobal.org/vocab/lis/v2/membership#Learner': 'student',
  'http://purl.imsglobal.org/vocab/lis/v2/institution/person#Administrator': 'institution_admin',
};
```

**Testing:**
- `test-7.1.1`: LTI 1.3 launch from Canvas -- student is authenticated and redirected to tutoring session.
- `test-7.1.2`: LTI 1.3 launch from Moodle -- teacher is authenticated and redirected to dashboard.
- `test-7.1.3`: Invalid JWT signature -- launch is rejected with 401.
- `test-7.1.4`: LTI role 'Instructor' maps to internal 'teacher' role.
- `test-7.1.5`: First-time LTI user -- user_account is auto-created with LTI-provided name and email.
- `test-7.1.6`: Returning LTI user -- existing user_account is found and session is created.

---

### Task 7.2: Grade Passback (LTI Assignment and Grade Services)

**What:** Implement LTI 1.3 Assignment and Grade Services (AGS) for posting tutoring session scores back to the LMS gradebook. Build the score calculation from session mastery data.

**Design:**

```typescript
// packages/lti-toolkit/src/ags.ts
export class AGSClient {
  async postScore(lineItemUrl: string, score: LTIScore): Promise<void> {
    // 1. Get OAuth 2.0 Client Credentials token from platform
    // 2. POST score to lineitem URL with Bearer token
    // Score payload per LTI AGS spec:
    // { scoreOf, userId, scoreGiven, scoreMaximum, activityProgress, gradingProgress, timestamp }
  }
}

// apps/api/src/modules/lti/services/grade-sync.service.ts
export class GradeSyncService {
  async syncSessionGrade(sessionId: string): Promise<void> {
    const session = await this.sessionService.getById(sessionId);
    if (!session.ltiLineItemUrl) return;  // Not an LTI-launched session

    // Calculate score from mastery progress during session
    const score = this.calculateSessionScore(session);

    await this.agsClient.postScore(session.ltiLineItemUrl, {
      userId: session.ltiUserId,
      scoreGiven: score,
      scoreMaximum: 100,
      activityProgress: session.endedAt ? 'Completed' : 'InProgress',
      gradingProgress: 'FullyGraded',
      timestamp: new Date().toISOString(),
    });
  }
}
```

**Testing:**
- `test-7.2.1`: Student completes an LTI-launched session -- score is posted to LMS gradebook.
- `test-7.2.2`: Score reflects mastery progress: student who mastered 3 of 5 target KCs gets 60%.
- `test-7.2.3`: In-progress session posts `activityProgress: 'InProgress'` to LMS.
- `test-7.2.4`: OAuth token refresh works when token expires mid-session.
- `test-7.2.5`: Non-LTI sessions do not attempt grade passback (no errors).

---

### Task 7.3: OneRoster Roster Import

**What:** Implement OneRoster 1.1 REST API client to import class rosters (organizations, courses, classes, users, enrollments) from Student Information Systems. Build sync logic that creates/updates/deactivates users and enrollments.

**Design:**

```typescript
// apps/api/src/modules/enrollment/services/oneroster-sync.service.ts
export class OneRosterSyncService {
  async syncRoster(institutionId: string, config: OneRosterConfig): Promise<SyncResult> {
    // 1. Fetch organizations, classes, users, enrollments from OneRoster API
    // 2. Map OneRoster entities to internal models
    // 3. Upsert users (match on external_id/sourcedId)
    // 4. Upsert sections (match on external_id)
    // 5. Sync enrollments (add new, deactivate removed)
    // 6. Return sync result with counts
  }
}
```

**Testing:**
- `test-7.3.1`: OneRoster sync imports 100 students, 5 teachers, 8 sections -- all created correctly.
- `test-7.3.2`: Re-running sync with updated roster -- new students are added, removed students are deactivated (not deleted).
- `test-7.3.3`: OneRoster API returns error -- sync fails gracefully with error report, no partial state corruption.
- `test-7.3.4`: Student with `external_id` matching existing user is updated, not duplicated.

---

### Phase 7 Definition of Done

- [ ] LTI 1.3 launch works with Canvas (verified against Canvas Free for Teachers)
- [ ] LTI 1.3 launch works with Moodle (verified against Moodle sandbox)
- [ ] Grade passback posts scores to LMS gradebook after session completion
- [ ] OneRoster roster import creates/updates users and enrollments
- [ ] LTI configuration stored in `lti_registration` JSONB
- [ ] Documentation: LTI setup guide for Canvas and Moodle administrators

---

## Phase 8: Misconception Detection & Remediation

**Goal:** Build the misconception detection pipeline that identifies specific conceptual errors from student responses and delivers targeted remediation. Introduce the graph layer (from data model Suggestion 4) for misconception-KC-remediation networks.

**Duration:** 3-4 weeks

### Task 8.1: Misconception Classification Pipeline

**What:** Build a pipeline that analyzes student wrong answers to classify the specific misconception. Use a combination of: (1) rule-based matching for known misconceptions (e.g., distractor analysis in choice items), (2) LLM-based classification for free-text responses. Store misconception detections in student knowledge JSONB.

**Design:**

```typescript
// apps/api/src/modules/misconception/services/detector.service.ts
export class MisconceptionDetectorService {
  async detect(attempt: StudentAttempt, item: AssessmentItem): Promise<DetectedMisconception | null> {
    if (attempt.isCorrect) return null;

    // Strategy 1: Distractor-based detection (for choice items)
    if (item.itemType === 'choice') {
      const selectedChoice = item.content.choices.find(c => c.id === attempt.studentResponse.choiceId);
      if (selectedChoice?.misconception) {
        return this.lookupMisconception(selectedChoice.misconception, item);
      }
    }

    // Strategy 2: LLM-based detection (for text and free-form responses)
    const kcs = await this.knowledgeService.getKCsForItem(item.id);
    const knownMisconceptions = kcs.flatMap(kc => kc.metadata.misconceptions || []);

    const detection = await this.llm.classifyMisconception({
      problem: item.content.stem,
      correctAnswer: item.content.correct_response,
      studentAnswer: attempt.studentResponse,
      knownMisconceptions: knownMisconceptions,
    });

    return detection;
  }
}

// LLM prompt for misconception classification
const MISCONCEPTION_PROMPT = `You are an expert mathematics educator analyzing a student's incorrect answer.

PROBLEM: {problem}
CORRECT ANSWER: {correctAnswer}
STUDENT'S ANSWER: {studentAnswer}

KNOWN MISCONCEPTIONS for this topic:
{misconceptions}

TASK: Identify which misconception (if any) best explains the student's error.
Return a JSON object with:
- misconception_code: the code from the known misconceptions list, or "UNKNOWN" if none match
- confidence: 0.0-1.0 confidence in the classification
- evidence: one sentence explaining why this misconception explains the student's error
- counter_example: a targeted example that would help the student see their error`;
```

**Testing:**
- `test-8.1.1`: Student selects distractor with `misconception: "DISTRIBUTE_NEGATIVE"` -- detector returns that misconception with high confidence.
- `test-8.1.2`: Student types free-text answer "x = 7" when correct answer is "x = -7" -- LLM classifies as sign error misconception.
- `test-8.1.3`: Student's answer doesn't match any known misconception -- detector returns `UNKNOWN` with appropriate evidence.
- `test-8.1.4`: Correct answer -- detector returns null (no misconception).
- `test-8.1.5`: Misconception detection is recorded in `student_knowledge.model_params.misconceptions_exhibited`.
- `test-8.1.6`: Detection latency is under 3 seconds (LLM call is the bottleneck).

---

### Task 8.2: Graph Layer for Misconception Networks

**What:** Introduce the targeted graph tables (`graph_node`, `graph_edge`) from data model Suggestion 4 for modeling misconception-KC-remediation relationships. Build the graph traversal queries that power adaptive remediation selection.

**Design:**

```sql
-- Add graph tables (subset of Suggestion 4)
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type TEXT NOT NULL,   -- 'misconception', 'remediation_strategy', 'knowledge_component'
    external_id UUID,          -- links to relational entity
    label TEXT NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_type TEXT NOT NULL,   -- 'targets', 'remediates', 'indicates'
    source_node_id UUID NOT NULL REFERENCES graph_node(id),
    target_node_id UUID NOT NULL REFERENCES graph_node(id),
    weight NUMERIC(5,4) DEFAULT 1.0,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// apps/api/src/modules/misconception/services/remediation.service.ts
export class RemediationService {
  async selectRemediation(misconceptionId: string): Promise<RemediationStrategy> {
    // Graph traversal: misconception -> remediates -> strategy
    // Select highest-weighted remediation strategy
    const strategies = await this.db.execute(sql`
      SELECT gn.label, gn.properties, ge.weight
      FROM graph_edge ge
      JOIN graph_node gn ON gn.id = ge.source_node_id
      WHERE ge.target_node_id = (
        SELECT id FROM graph_node WHERE external_id = ${misconceptionId}
      )
      AND ge.edge_type = 'remediates'
      ORDER BY ge.weight DESC
      LIMIT 1
    `);

    if (strategies.length === 0) {
      // No predefined remediation -- generate via LLM
      return this.generateRemediation(misconceptionId);
    }

    return strategies[0];
  }
}
```

**Testing:**
- `test-8.2.1`: Graph query returns remediation strategies for a known misconception.
- `test-8.2.2`: When multiple strategies exist, highest-weight strategy is returned first.
- `test-8.2.3`: Unknown misconception with no graph edges -- LLM-generated remediation is returned.
- `test-8.2.4`: Seeding 50 misconceptions with remediation strategies -- all graph relationships are queryable.

---

### Task 8.3: Remediation Delivery in Socratic Dialogue

**What:** Integrate misconception detection and remediation into the Socratic dialogue flow. When the tutor detects a misconception, it delivers a targeted counter-example or follow-up question addressing the specific error.

**Design:**

```typescript
// Enhanced SocraticEngineService from Phase 3
async generateResponse(session: TutoringSession, studentMessage: string): Promise<AsyncIterable<string>> {
  const lastAttempt = await this.getLastAttempt(session.id);

  if (lastAttempt && !lastAttempt.isCorrect && lastAttempt.detectedMisconception) {
    // Misconception-aware response
    const remediation = await this.remediationService.selectRemediation(lastAttempt.detectedMisconception.id);
    const misconceptionContext = `
DETECTED MISCONCEPTION: ${lastAttempt.detectedMisconception.name}
Evidence: ${lastAttempt.detectedMisconception.evidence}
REMEDIATION STRATEGY: ${remediation.content}
Deliver a counter-example or targeted question that addresses this specific misconception.
Do NOT mention the word "misconception" to the student.`;

    // Inject misconception context into the prompt
    return this.llmProvider.streamChat({
      ...baseOptions,
      messages: [
        ...conversationHistory,
        { role: 'system', content: misconceptionContext },
        { role: 'user', content: studentMessage },
      ],
    });
  }

  // Standard Socratic response (no misconception detected)
  return this.standardSocraticResponse(session, studentMessage);
}
```

**Testing:**
- `test-8.3.1`: Student exhibits "distribute negative sign" misconception -- tutor's next response addresses sign distribution specifically (not generic "try again").
- `test-8.3.2`: Counter-example in tutor response is relevant to the specific misconception (evaluated by Promptfoo rubric).
- `test-8.3.3`: Tutor does NOT use the word "misconception" in its response to the student.
- `test-8.3.4`: After successful remediation (student answers correctly), tutor confirms understanding and moves on.
- `test-8.3.5`: Misconception-targeted responses are measurably different from generic incorrect-answer responses (Promptfoo A/B comparison).

---

### Phase 8 Definition of Done

- [ ] Misconception detection pipeline classifies errors for both choice and free-text items
- [ ] Graph layer stores misconception-KC-remediation relationships
- [ ] Socratic dialogue engine delivers misconception-targeted responses
- [ ] Student knowledge JSONB tracks exhibited misconceptions with timestamps and remediation status
- [ ] Detection accuracy > 60% on known misconceptions (measured by Promptfoo suite)
- [ ] Counter-example generation produces relevant, age-appropriate responses

---

## Phase 9: Multilingual Support

**Goal:** Enable high-quality tutoring in multiple languages without degrading pedagogical quality. Support at least 5 languages for initial launch.

**Duration:** 2-3 weeks

### Task 9.1: UI Internationalization (i18n)

**What:** Implement Next.js internationalization using `next-intl`. Translate all UI strings (navigation, buttons, labels, error messages) into Spanish, French, Portuguese, and Mandarin Chinese. Use BCP 47 language tags.

**Design:**

```typescript
// apps/web/src/i18n/messages/en.json
{
  "tutoring": {
    "startSession": "Start Tutoring Session",
    "endSession": "End Session",
    "hint": "Get a Hint",
    "submitAnswer": "Submit Answer",
    "correctFeedback": "Correct! Great job!",
    "incorrectFeedback": "Not quite. Let me help you think through this."
  },
  "dashboard": {
    "classMastery": "Class Mastery Overview",
    "activeStudents": "Active Students",
    "alerts": "Alerts"
  }
}
```

**Testing:**
- `test-9.1.1`: Setting locale to `es` -- all UI strings render in Spanish.
- `test-9.1.2`: All 5 language files have identical key structure (no missing translations).
- `test-9.1.3`: Language selector in user profile changes UI language without page reload.
- `test-9.1.4`: RTL layout does not apply for any of the 5 initial languages (none are RTL).

---

### Task 9.2: Multilingual Socratic Dialogue

**What:** Configure the Socratic dialogue engine to generate tutoring responses in the student's preferred language. Add language-specific prompt engineering to maintain pedagogical quality across languages.

**Design:**

```typescript
// Language-specific system prompt additions
const LANGUAGE_INSTRUCTIONS: Record<string, string> = {
  es: `Responde SIEMPRE en español. Usa terminología matemática estándar en español.
       Mantén un tono cálido y alentador apropiado para estudiantes hispanohablantes.`,
  fr: `Répondez TOUJOURS en français. Utilisez la terminologie mathématique standard en français.
       Maintenez un ton chaleureux et encourageant.`,
  'pt-BR': `Responda SEMPRE em português brasileiro. Use terminologia matemática padrão em português.`,
  'zh-CN': `请始终用简体中文回答。使用标准数学术语。保持温暖和鼓励的语气。`,
};
```

**Testing:**
- `test-9.2.1`: Session with `language: 'es'` -- tutor responds entirely in Spanish.
- `test-9.2.2`: Mathematical terminology in Spanish is correct (e.g., "ecuacion lineal" not "linear equation").
- `test-9.2.3`: Socratic questioning quality in non-English languages is comparable (Promptfoo evaluation: answer leakage rate < 10% for each language).
- `test-9.2.4`: Mixed-language input (student writes in English during Spanish session) -- tutor continues in Spanish.

---

### Phase 9 Definition of Done

- [ ] UI available in 5 languages: English, Spanish, French, Portuguese, Mandarin Chinese
- [ ] Socratic dialogue generates pedagogically sound responses in all 5 languages
- [ ] Assessment items can be tagged with language; items are filterable by language
- [ ] Language preference persists in user profile

---

## Phase 10: Learning Analytics & xAPI

**Goal:** Implement xAPI statement generation for all learning interactions, build an internal Learning Record Store (LRS), and create analytics dashboards for institutional reporting.

**Duration:** 2-3 weeks

### Task 10.1: xAPI Statement Generation

**What:** Build the `packages/xapi-client` library that converts tutoring interactions into xAPI statements. Generate statements for: session started/ended, item attempted, hint requested, mastery updated. Store in the `xapi_statement` table.

**Design:**

```typescript
// packages/xapi-client/src/builder.ts
export class XAPIStatementBuilder {
  buildAttemptStatement(attempt: StudentAttempt, student: UserAccount): XAPIStatement {
    return {
      id: randomUUID(),
      actor: {
        objectType: 'Agent',
        mbox: `mailto:${student.email}`,
        name: student.displayName,
      },
      verb: {
        id: 'http://adlnet.gov/expapi/verbs/answered',
        display: { 'en-US': 'answered' },
      },
      object: {
        objectType: 'Activity',
        id: `${BASE_IRI}/assessment-item/${attempt.assessmentItemId}`,
        definition: {
          type: 'http://adlnet.gov/expapi/activities/assessment',
          name: { 'en-US': 'Assessment Item' },
        },
      },
      result: {
        success: attempt.isCorrect,
        score: { scaled: attempt.partialCredit ?? (attempt.isCorrect ? 1.0 : 0.0) },
        duration: formatISO8601Duration(attempt.timeSpentSeconds),
      },
      timestamp: attempt.createdAt.toISOString(),
    };
  }
}
```

**Testing:**
- `test-10.1.1`: Student attempts an item -- xAPI statement is generated with correct verb, actor, object, and result.
- `test-10.1.2`: Generated statement validates against xAPI JSON Schema.
- `test-10.1.3`: Statement stored in `xapi_statement` table with indexed `actor_id`, `verb_id`, `object_id`.
- `test-10.1.4`: Bulk export of xAPI statements for a student returns valid xAPI JSON array.
- `test-10.1.5`: Voiding a statement marks it as `voided: true` and creates a voiding statement.

---

### Task 10.2: Analytics Dashboard

**What:** Build institutional analytics views: student progress over time, class comparison, mastery trends, session engagement metrics. Create export functionality for CSV and PDF reports.

**Testing:**
- `test-10.2.1`: Student progress chart shows mastery trend across 30 days.
- `test-10.2.2`: Class comparison view ranks sections by average mastery.
- `test-10.2.3`: CSV export contains all xAPI statements for a date range.
- `test-10.2.4`: Analytics queries complete within 5 seconds for 1 year of data.

---

### Phase 10 Definition of Done

- [ ] All tutoring interactions generate valid xAPI statements
- [ ] xAPI statements are queryable by actor, verb, object, and date range
- [ ] Analytics dashboard shows student progress, class comparison, and engagement metrics
- [ ] CSV export works for institutional reporting
- [ ] xAPI endpoint is compatible with external LRS consumers

---

## Phase 11: Accessibility, Compliance & Hardening

**Goal:** Achieve WCAG 2.2 AA compliance, implement FERPA/COPPA/GDPR data handling, conduct security hardening, and prepare for institutional procurement review.

**Duration:** 3-4 weeks

### Task 11.1: WCAG 2.2 AA Accessibility Audit & Remediation

**What:** Conduct a full accessibility audit of all pages using axe-core automated testing and manual screen reader testing (NVDA on Windows, VoiceOver on macOS). Remediate all Level A and AA violations.

**Testing:**
- `test-11.1.1`: axe-core scan of every page returns zero Level A or AA violations.
- `test-11.1.2`: Complete tutoring session using only keyboard -- all interactions functional.
- `test-11.1.3`: Complete tutoring session using VoiceOver -- all content is announced, streaming responses are announced incrementally.
- `test-11.1.4`: Colour contrast ratio meets 4.5:1 for normal text, 3:1 for large text on all mastery color codes.
- `test-11.1.5`: Focus management: when tutor response completes streaming, focus moves to student input field.
- `test-11.1.6`: All images have alt text; all form inputs have associated labels.

---

### Task 11.2: FERPA / COPPA / GDPR Data Handling

**What:** Implement data retention policies, right-to-erasure (GDPR Article 17), data export (GDPR Article 20), and data processing agreements infrastructure. Build the crypto-shredding pattern from data model Suggestion 2 for GDPR erasure.

**Design:**

```typescript
// apps/api/src/modules/auth/services/data-privacy.service.ts
export class DataPrivacyService {
  async exportUserData(userId: string): Promise<DataExport> {
    // GDPR Article 20: Right to data portability
    // Export all user data in machine-readable format
    return {
      userAccount: await this.getUserAccount(userId),
      knowledgeState: await this.getKnowledgeState(userId),
      sessions: await this.getSessions(userId),
      xapiStatements: await this.getXAPIStatements(userId),
      consentRecords: await this.getConsentRecords(userId),
    };
  }

  async deleteUserData(userId: string, reason: string): Promise<DeletionResult> {
    // GDPR Article 17: Right to erasure
    // 1. Delete read models / mutable state
    // 2. Anonymize xAPI statements (replace actor with anonymous hash)
    // 3. Insert audit_log entry for the deletion
    // 4. Return confirmation
  }

  async applyRetentionPolicy(institutionId: string): Promise<RetentionResult> {
    // Apply institution's data retention policy
    // Delete/anonymize data older than retention period
  }
}
```

**Testing:**
- `test-11.2.1`: Data export for a user returns complete JSON with all their data.
- `test-11.2.2`: After data deletion, user's sessions, knowledge state, and PII are removed; xAPI statements are anonymized.
- `test-11.2.3`: Audit log records the deletion event (who requested, when, what was deleted).
- `test-11.2.4`: Retention policy with 730 days -- data older than 2 years is purged on schedule.
- `test-11.2.5`: Deleted user cannot be re-identified from remaining anonymized data.

---

### Task 11.3: Security Hardening

**What:** Implement rate limiting, input sanitization, CSRF protection, Content Security Policy, and penetration testing remediation. Conduct dependency vulnerability scan.

**Testing:**
- `test-11.3.1`: Rate limiter blocks more than 100 requests/minute per user.
- `test-11.3.2`: XSS payload in student message is sanitized before storage and display.
- `test-11.3.3`: SQL injection attempt via JSONB query parameters fails.
- `test-11.3.4`: CSP header blocks inline scripts.
- `test-11.3.5`: `npm audit` reports zero critical or high severity vulnerabilities.
- `test-11.3.6`: LTI private keys are encrypted at rest and not exposed in API responses.

---

### Phase 11 Definition of Done

- [ ] Zero WCAG 2.2 Level A or AA violations (axe-core + manual audit)
- [ ] GDPR data export and erasure work end-to-end
- [ ] FERPA audit trail covers all data access and modification
- [ ] COPPA consent flow verified for under-13 users
- [ ] Security hardening checklist complete (rate limiting, CSP, input sanitization)
- [ ] Zero critical/high npm audit vulnerabilities
- [ ] Data retention policies execute on schedule

---

## Phase 12: Gamification, Collaboration & Advanced Features

**Goal:** Add engagement mechanics (streaks, mastery milestones), multiplayer collaborative problem-solving, camera-based problem input, and competency-mapped credentials.

**Duration:** 3-4 weeks

### Task 12.1: Gamification Layer

**What:** Implement streaks (consecutive days of practice), mastery milestones (badges for mastering KC groups), and a non-competitive progress system. Design gamification to support intrinsic motivation without encouraging answer-seeking shortcuts (unlike Duolingo's streak-over-learning concern from features.md).

**Design:**

```typescript
// apps/api/src/modules/gamification/services/streak.service.ts
export class StreakService {
  async recordActivity(studentId: string): Promise<StreakUpdate> {
    // 1. Check if student already has activity today
    // 2. If first activity today, increment streak
    // 3. If streak was broken (yesterday had no activity), reset to 1
    // 4. Check for streak milestones (7, 30, 100 days)
    // 5. Award badges for milestones
  }
}

// Mastery milestone badges
export const MILESTONES = [
  { id: 'first_mastery', condition: 'First KC mastered', icon: 'star' },
  { id: 'five_mastered', condition: '5 KCs mastered', icon: 'trophy' },
  { id: 'subject_complete', condition: 'All KCs in a subject mastered', icon: 'crown' },
  { id: 'misconception_overcome', condition: 'Remediated a misconception successfully', icon: 'lightning' },
  { id: 'week_streak', condition: '7-day practice streak', icon: 'fire' },
];
```

**Testing:**
- `test-12.1.1`: Student practices today -- streak increments from 0 to 1.
- `test-12.1.2`: Student practices 7 consecutive days -- receives "week_streak" badge.
- `test-12.1.3`: Student skips a day -- streak resets to 0.
- `test-12.1.4`: Mastering 5th KC triggers "five_mastered" badge notification.
- `test-12.1.5`: Badge display does not reveal answers or provide shortcuts.

---

### Task 12.2: Camera-Based Problem Input

**What:** Implement image upload and OCR processing for mobile problem input. Students photograph a math problem and the system extracts the mathematical content for tutoring.

**Design:**

```typescript
// apps/api/src/modules/media/services/ocr.service.ts
export class OCRService {
  async processImage(imageBuffer: Buffer): Promise<OCRResult> {
    // 1. Upload to cloud vision API or local Tesseract
    // 2. Extract text with confidence scores
    // 3. Parse mathematical expressions from text
    // 4. Return structured result
  }
}
```

**Testing:**
- `test-12.2.1`: Photograph of printed equation "3x + 5 = 20" -- OCR extracts correct text.
- `test-12.2.2`: Handwritten equation -- OCR extracts with > 80% accuracy.
- `test-12.2.3`: OCR result is used to start a tutoring session on the detected problem.
- `test-12.2.4`: Invalid image (non-math content) -- graceful fallback with message "Could not detect a math problem."

---

### Task 12.3: Multiplayer Collaborative Problem-Solving

**What:** Build a collaborative mode where 2-4 students solve problems together with AI tutor facilitation. Implement shared workspace, turn-taking, and collaborative hints.

**Testing:**
- `test-12.3.1`: Two students join a collaborative session -- both see shared problem and each other's responses.
- `test-12.3.2`: AI tutor facilitates by asking different students to contribute different steps.
- `test-12.3.3`: Session metrics track individual contributions within the collaborative session.
- `test-12.3.4`: Disconnection and reconnection -- student re-joins and sees conversation history.

---

### Phase 12 Definition of Done

- [ ] Streak system tracks consecutive practice days with milestone badges
- [ ] Camera input processes printed and handwritten math with > 80% accuracy
- [ ] Collaborative sessions support 2-4 students with shared workspace
- [ ] Gamification does not incentivize answer-seeking over learning
- [ ] All new features pass accessibility audit

---

## Definition of Done (Global)

Every phase must meet these criteria before being marked complete:

### Code Quality
- [ ] All new code has TypeScript strict-mode compliance (no `any` types except in explicitly justified cases)
- [ ] All API endpoints have OpenAPI documentation generated by NestJS decorators
- [ ] All public functions have JSDoc comments
- [ ] No ESLint warnings or errors
- [ ] Code review approved by at least one other contributor

### Testing
- [ ] Unit test coverage >= 80% for new code (measured by Vitest coverage)
- [ ] Integration tests cover all API endpoints (happy path + error cases)
- [ ] E2E tests cover critical user journeys (Playwright)
- [ ] LLM evaluation suites pass quality thresholds (Promptfoo, where applicable)
- [ ] No regression in existing tests

### Security & Compliance
- [ ] No secrets committed to repository (scanned by CI)
- [ ] FERPA-sensitive data access is audit-logged
- [ ] RLS prevents cross-tenant data access (verified by integration tests)
- [ ] Dependencies scanned for known vulnerabilities (`npm audit`)

### Performance
- [ ] API response times < 200ms for non-LLM endpoints (p95)
- [ ] LLM first-token latency < 2 seconds (p95)
- [ ] Teacher dashboard loads < 2 seconds for 35-student class
- [ ] Database queries use indexes (no sequential scans on tables > 1000 rows)

### Accessibility
- [ ] New UI components pass axe-core automated accessibility scan
- [ ] Keyboard navigation works for all interactive elements
- [ ] Color is not the only means of conveying information

### Documentation
- [ ] Architecture Decision Records (ADRs) for significant technical decisions
- [ ] API changelog updated for breaking changes
- [ ] Deployment guide updated if infrastructure changes
