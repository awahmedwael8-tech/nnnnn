# Implementation Plan: 001-zizo-lms

**Branch**: `001-zizo-lms` | **Date**: 2026-10-30 | **Spec**: `specs/001-zizo-lms/spec.md`

**Input**: Feature specification from `specs/001-zizo-lms/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Zizo LMS is a self-owned, single-instructor course platform where an Admin sells and delivers video courses to Students[cite: 1]. The platform handles manual payment workflows (e.g., InstaPay screenshots), robust Admin review processes for approvals and revocations, secure video/attachment delivery via short-lived signed URLs, one-level comment threads, and real-time private chat[cite: 1]. The system heavily emphasizes robust access control, strict type safety, and database transactional integrity to ensure paid content remains securely gated[cite: 2].

## Technical Context

**Language/Version**: TypeScript in strict mode for both client and server[cite: 2].

**Primary Dependencies**: React 18, Tailwind CSS, Node.js (active LTS), Express, Prisma, Socket.IO, Zod, Redis (for sessions, rate-limit counters, and caching)[cite: 2].

**Storage**: PostgreSQL via Prisma; private S3-compatible storage for videos, attachments, and payment screenshots[cite: 2].

**Testing**: Vitest or Jest for unit testing; integration tests utilizing a real database via Prisma Migrate[cite: 2].

**Target Platform**: Web application (responsive from 360px width upward)[cite: 1, 2].

**Project Type**: Web application (Frontend SPA + Node.js REST API + WebSockets).

**Performance Goals**: API p95 latency ≤ 500 ms; LCP ≤ 2 s on median student device profile; videos stream directly from S3 via signed URLs (bypassing Express)[cite: 2].

**Constraints**: REST API strictly scoped under `/api/v1/`[cite: 2]. Approving a payment and creating its Enrollment MUST happen in a single database `$transaction`[cite: 2]. Passwords must be hashed via bcrypt (cost ≥ 12)[cite: 2]. Signed URLs strictly limited to ≤ 15 minutes expiry[cite: 2].

**Data Model Entities**: `User`, `RefreshToken`, `Course`, `Section`, `Video`, `VideoAttachment`, `Payment` (fields: `declaredAmount`, `priceSnapshot`, `screenshotHash`, `status` [PENDING/APPROVED/DECLINED], `declineReason`, `reviewedBy`, `reviewedAt`), `Enrollment` (unique on `userId` and `courseId`), `Comment` (one-level replies, cascade delete of replies), `Conversation` (one per Student), `Message` (immutable, no edit or delete), `Notification`, `VideoProgress`, `Setting`.

**API Contracts Outline (P1/P0 Priority Coverage)**:
*   `POST /api/v1/courses` - Admin creates a course catalog[cite: 1].
*   `POST /api/v1/courses/:courseId/sections` - Admin adds a section[cite: 1].
*   `POST /api/v1/sections/:sectionId/videos` - Admin adds a video[cite: 1].
*   `POST /api/v1/payments` - Student buys a course (submits screenshot/amount)[cite: 1].
*   `GET /api/v1/videos/:videoId/watch` - Enrolled Student requests signed URL for video playback[cite: 1, 2].
*   `POST /api/v1/payments/:paymentId/approve` - Admin reviews and approves payment[cite: 1].
*   `POST /api/v1/payments/:paymentId/decline` - Admin declines payment[cite: 1].
*   `POST /api/v1/enrollments/:enrollmentId/revoke` - Admin revokes enrollment[cite: 1].

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

*   **Code Quality & Type Safety**: Enabled. Strict TypeScript everywhere. Zod boundaries enforced for all API and Socket.IO payloads[cite: 2].
*   **Test-First & Coverage**: Enforced. Payment approval, video access, and chat permissions will have dedicated tests written prior to implementation[cite: 2].
*   **Security**: S3 signed URLs used for all protected assets. Socket.IO verifies authentication on handshake and room emit[cite: 2]. Revoking enrollment blocks new signed URLs but does not terminate active sockets.
*   **Data Integrity**: Prisma Migrate exclusively used. `Payment` and `Enrollment` are strictly separate entities. `$transaction` used for payment approval + enrollment creation[cite: 2].
*   **Performance**: Redis caching utilized. File transfers bypass API entirely[cite: 2].
*   **Scoped Delivery**: Endpoints versioned under `/api/v1/`. Scope strictly limited to requested features[cite: 2].

## Project Structure

### Documentation (this feature)

```text
specs/001-zizo-lms/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── config/
│   ├── controllers/
│   ├── middleware/
│   ├── models/
│   ├── routes/
│   │   └── v1/
│   ├── services/
│   └── websockets/
└── tests/
    ├── contract/
    ├── integration/
    └── unit/

frontend/
├── src/
│   ├── assets/
│   ├── components/
│   ├── hooks/
│   ├── pages/
│   ├── services/
│   ├── store/
│   └── utils/
└── tests/
```

**Structure Decision**: Option 2 (Web Application). Separate `frontend/` and `backend/` directories without a monorepo configuration, explicitly adhering to the prompt's structural constraint while maintaining clear frontend/backend separation[cite: 3].

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| No Monorepo | Explicit user prompt requirement | A unified yarn workspace was rejected because the prompt strictly prohibits adding a monorepo setup. |