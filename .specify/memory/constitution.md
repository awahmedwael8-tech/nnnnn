# Zizo LMS Constitution

## Core Principles

### I. Code Quality & Type Safety

TypeScript strict mode MUST be enabled on both the React 18 frontend and the
Node.js + Express backend; `tsconfig.json` MUST set `"strict": true` with
`noImplicitAny`, `strictNullChecks`, and `noUncheckedIndexedAccess` on. The `any`
type is forbidden in application code; `unknown` plus a Zod parse or a type
guard MUST be used instead. ESLint and Prettier MUST run in CI and MUST block
merges on error-level findings. Functions and React components MUST be small
and single-purpose. Backend code MUST keep a clear separation between routes
(HTTP wiring), controllers (request/response shaping), services (business
logic), and data access (Prisma). Every request body, query, and route param
MUST be validated with a Zod schema at the controller boundary before any
service call. Dead code and commented-out code MUST be deleted, not shipped.

**Rationale**: Strict types catch defects before runtime, Zod turns untrusted
input into typed data at a single enforced boundary, and layered separation
keeps authorization, validation, and persistence from leaking into one another.

### II. Test-First & Comprehensive Coverage (NON-NEGOTIABLE)

Business-critical logic MUST be written test-first: a failing test is produced
and reviewed before the implementation. Authentication, role checks, payment
approve/decline, enrollment-based video access, and Admin↔Student chat
permissions MUST each have dedicated tests covering success, failure, and
authorization edges. Unit tests MUST use Vitest or Jest. Every API endpoint
MUST have at least one integration test that exercises the real route handler
against a real database (test schema managed by Prisma Migrate). Every
acceptance criterion in a spec MUST map to at least one named test. CI MUST
block merges when any test fails, when coverage drops below the agreed
threshold for the modules named above, or when a new endpoint ships without an
integration test.

**Rationale**: These are the paths where a silent regression lets a non-paying
user watch paid content, a student read another student's chat, or an admin
double-enroll a buyer. They cannot be left to ad-hoc testing.

### III. Security & Server-Side Access Control (NON-NEGOTIABLE)

Every protected endpoint MUST verify two things on the server: the caller's
role AND their ownership of, or enrollment in, the specific resource. The
frontend MAY hide UI, but it MUST NOT be the only gate. Videos, attachments,
and payment-receipt screenshots MUST live in private S3-compatible storage and
MUST be served only through short-lived pre-signed URLs (expiry ≤ 15 minutes);
the API server MUST NOT proxy video bytes. Passwords MUST be hashed with
bcrypt (cost ≥ 12). All string inputs MUST be validated and sanitized through
Zod before reaching services. Login and password-reset endpoints MUST be rate
limited (per IP and per account). Secrets MUST NOT be committed to the
repository; `.env` files MUST be gitignored and loaded from the host or a
secret manager. Socket.IO connections MUST authenticate on handshake and
re-check room membership on every emit.

**Rationale**: An LMS that sells access is only as trustworthy as its weakest
authorization check. Signed URLs plus server-side ownership checks make it
impossible to share a link to steal paid content.

### IV. Data Integrity & Schema Governance

Prisma Migrate is the ONLY mechanism permitted to change the database schema;
ad-hoc SQL, `prisma db push` against shared environments, and hand-edited
migration files are forbidden. Every migration MUST be committed with the
change that requires it. Unique constraints MUST be declared at the database
level for identity fields (user email, enrollment of a given user in a given
course, payment reference). Operations that span multiple writes MUST run
inside a Prisma `$transaction`; in particular, approving a payment and
creating its Enrollment MUST be atomic — either both land or neither does.
The domain model MUST keep `Payment` (a request, with a status) and
`Enrollment` (the access right) as separate entities; access checks MUST read
`Enrollment`, never `Payment.status`.

**Rationale**: Treating "paid" and "enrolled" as the same thing hides refunds,
reversals, and partial failures. Transactions and unique constraints are the
only durable way to prevent duplicate enrollments and lost grants under
concurrency.

### V. UX Consistency & Accessibility

Every user-facing feature MUST explicitly define and implement four states:
validation (inline, per-field), loading, error (actionable message, not a raw
stack), and success. Forms MUST validate inline before submission and MUST
surface server-side errors against the correct field. Destructive actions
(delete, decline payment, remove enrollment, remove chat message) MUST require
an explicit confirmation step naming the target. Layouts MUST remain usable
and non-overlapping from 360px width upward. Components MUST meet WCAG 2.1 AA
for color contrast, focus visibility, keyboard operability, and labeled
controls. Tailwind utility classes MUST be the styling mechanism; one-off CSS
files are permitted only when a utility expression is unreadable.

**Rationale**: An LMS used on phones in Egyptian 3G conditions needs explicit
loading and error states and a 360px baseline. Accessibility is a correctness
requirement, not a cosmetic one.

### VI. Performance Budgets

API endpoints MUST meet a p95 latency of 500 ms or less under expected load;
regressions above this threshold MUST block release. Page LCP MUST be 2 s or
less on the median student device profile. Video bytes MUST stream directly
from object storage to the client via signed URLs and MUST NOT pass through
the Express API. Redis MUST be used for session storage, rate-limit counters,
and short-lived caches of hot read paths (course listing, enrollment lookup).
N+1 Prisma queries MUST be eliminated before merge; list endpoints MUST
paginate.

**Rationale**: Proxied video kills throughput and bandwidth cost; unbounded
list endpoints and N+1 queries are the two most common ways an LMS falls over
when a course launches.

### VII. Scoped Delivery & Maintainability

REST endpoints MUST live under `/api/v1/`; breaking changes require a new
version prefix, not an in-place mutation. Features MUST be delivered in the
MoSCoW order from the PRD: all P0 (Must) items complete before any P1
(Should), and P2 (Could) only when capacity remains after P0 and P1. The
following are OUT OF SCOPE and MUST NOT be built in this version: automated
payment-gateway integration, native mobile app, certificates, affiliate
system, live streaming. Pull requests MUST be small and focused — one logical
change per PR — and MUST be reviewed by at least one other contributor before
merge. Non-obvious decisions (why a transaction wraps X, why a cache is 60s,
why a role check lives in Y) MUST be documented in the repo (ADR, code
comment, or spec note), not in Slack or memory.

**Rationale**: A single-instructor platform fails from scope creep more often
than from technical limits. The versioned API prefix and the explicit
out-of-scope list exist so that "while we're in there" never ships.

## Technology Stack & Platform Constraints

The following stack is fixed for this version and MUST NOT be substituted
without a constitution amendment:

- Frontend: React 18, TypeScript (strict), Tailwind CSS.
- Backend: Node.js (active LTS), Express, TypeScript (strict).
- Data: PostgreSQL via Prisma ORM; Prisma Migrate for schema changes.
- Cache & rate limiting: Redis.
- File storage: S3-compatible object storage, private buckets only, access via
  pre-signed URLs.
- Realtime: Socket.IO, authenticated on handshake, room membership re-checked
  on every emit.
- Validation: Zod at every HTTP and Socket.IO input boundary.

Environment configuration MUST be loaded from environment variables; the
repository MUST contain a committed `.env.example` listing every required key
with safe placeholder values and MUST NOT contain real secrets. Local
development MUST be reproducible via a single documented command (compose
file, Makefile target, or npm script).

## Development Workflow & Quality Gates

- Branching: feature branches off `main`; `main` is protected and only merged
  into via reviewed PRs.
- CI MUST run, on every PR, in this order and MUST block on failure:
  typecheck, lint, unit tests, integration tests, build. A merge is forbidden
  if any step fails.
- Every new or changed endpoint MUST ship with: a Zod schema for its inputs, a
  server-side authorization check, at least one integration test, and an entry
  in the API documentation.
- Every schema change MUST ship with its Prisma migration file in the same PR.
- Database transactions MUST be used for any operation that writes to more
  than one table in a single logical step; this is reviewed explicitly during
  code review of payment, enrollment, and chat features.
- Destructive production actions (data migrations against live data, index
  rebuilds, bulk updates) MUST be reviewed by at least two contributors and
  MUST be preceded by a reversible plan.
- Secrets found in a diff MUST block the merge; rotation is required before
  the PR is reopened.

## Governance

This constitution supersedes all other practices, style guides, and informal
agreements within the Zizo LMS repository. Where any other document or habit
conflicts with this constitution, the constitution wins and the other
document MUST be updated or discarded.

- Amendment procedure: Changes to this constitution MUST be proposed via a
  pull request that modifies `.specify/memory/constitution.md`, includes a
  Sync Impact Report, and is approved by the project owner.
- Versioning policy: This document is versioned with semantic versioning.
  MAJOR for backward-incompatible removal or redefinition of a principle or
  governance rule; MINOR for a new principle or materially expanded guidance;
  PATCH for wording, clarification, or non-semantic refinement.
- Compliance review: Every pull request MUST be reviewed against the
  applicable principles. Reviewers MUST reject or require changes to PRs that
  violate a NON-NEGOTIABLE principle. Any deviation from a non-negotiable
  principle MUST be documented, justified, and time-boxed in the PR
  description; silent deviations are treated as defects.
- Runtime guidance: Day-to-day engineering guidance (coding patterns, naming,
  review checklists) lives in `CLAUDE.md` and `.specify/memory/` adjacent
  documents and MUST remain consistent with this constitution.

**Version**: 1.0.0 | **Ratified**: 2026-10-28 | **Last Amended**: 2026-10-28
