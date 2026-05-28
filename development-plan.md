# Video Conferencing Platform — Phased Development Plan

> Project: 157-video-conferencing-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | WebRTC ecosystem is JavaScript-native; LiveKit server SDK, client SDK, and most WebRTC tooling are TypeScript-first; single language across server and client reduces context switching |
| Media server (SFU) | LiveKit (Apache 2.0, self-hosted) | Open-source SFU with built-in AI Agents framework (STT/LLM/TTS pipeline); simulcast/SVC support; proven at scale; avoids vendor lock-in vs. Daily/Agora; aligns with standards.md recommendation |
| API framework | Fastify 5 | Fastest Node.js HTTP framework; native OpenAPI 3.1 generation via @fastify/swagger; WebSocket support via @fastify/websocket; schema-based validation |
| Database | PostgreSQL 16 | JSONB + GIN indexing for hybrid relational/JSONB model (Data Model Suggestion 3); full-text search via tsvector for transcript search; partitioning for audit logs; mature ORM support |
| ORM / query builder | Drizzle ORM | TypeScript-native schema definitions; generates migration SQL; lightweight runtime; supports PostgreSQL JSONB and array types natively |
| Cache / real-time | Redis 7 (Valkey) | Pub/sub for real-time presence and signalling relay; caching for session tokens and room state; BullMQ job queue backend |
| Task queue | BullMQ | Node.js-native job queue on Redis; supports delayed jobs, rate limiting, retries; used for recording processing, transcription, AI summary generation |
| Frontend framework | Next.js 15 (App Router) | React 19 server components for dashboard; client components for meeting UI; built-in API routes if needed; Vercel-deployable or self-hosted |
| UI component library | shadcn/ui + Tailwind CSS 4 | Accessible components (WCAG 2.2 aligned); copy-paste components reduce bundle; Radix primitives for keyboard navigation |
| WebRTC client SDK | livekit-client (TypeScript) | Official LiveKit browser SDK; handles SDP negotiation, track management, reconnection; typed API |
| Transcription engine | Whisper (via LiveKit Agents) + Deepgram fallback | Whisper for self-hosted accuracy; Deepgram for real-time streaming with lower latency; speaker diarisation via pyannote |
| AI / LLM | Claude API (Anthropic SDK) | Meeting summaries, action item extraction, coaching; prompt caching for cost efficiency on repeated transcript patterns |
| Object storage | S3-compatible (MinIO for self-hosted, AWS S3 for cloud) | Recordings and attachments; pre-signed URLs for secure access; lifecycle policies for retention |
| Authentication | NextAuth.js v5 (Auth.js) | OAuth 2.0 providers (Google, Microsoft, SAML); JWT session tokens; credential-based login for local auth |
| Containerisation | Docker + Docker Compose | Multi-service deployment (API, LiveKit, PostgreSQL, Redis, MinIO); single docker-compose.yml for local dev |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, ESM-native); Playwright for E2E browser testing of meeting UI |
| Code quality | Biome (lint + format) + TypeScript strict mode | Single tool for linting and formatting; faster than ESLint + Prettier; strict TypeScript catches type errors at build time |
| Package manager | pnpm 9 | Workspace support for monorepo; strict dependency resolution; disk-efficient |
| Monorepo structure | pnpm workspaces | Shared types between server and client; separate packages for API, web app, SDK, and workers |

---

### Project Structure

```
video-conferencing-platform/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile.api
├── Dockerfile.web
├── Dockerfile.worker
├── .env.example
├── drizzle.config.ts
├── packages/
│   ├── shared/                          # Shared types, constants, validation schemas
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── meeting.ts           # Meeting, participant, room types
│   │       │   ├── recording.ts         # Recording, transcript types
│   │       │   ├── ai.ts               # AI artefact types (summaries, actions)
│   │       │   ├── user.ts             # User, organisation, auth types
│   │       │   └── events.ts           # WebSocket event payloads
│   │       ├── schemas/                 # Zod validation schemas (shared client/server)
│   │       ├── constants/               # Status enums, error codes, limits
│   │       └── index.ts
│   └── livekit-plugins/                 # Custom LiveKit agent plugins
│       ├── package.json
│       └── src/
│           ├── transcription-agent.ts
│           ├── summary-agent.ts
│           └── coaching-agent.ts
├── apps/
│   ├── api/                             # Fastify REST API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts                # Fastify app entry point
│   │       ├── config.ts               # Environment config with defaults
│   │       ├── db/
│   │       │   ├── schema.ts           # Drizzle schema definitions
│   │       │   ├── migrations/          # SQL migration files
│   │       │   └── seed.ts             # Dev seed data
│   │       ├── routes/
│   │       │   ├── auth.ts             # Login, register, OAuth callbacks
│   │       │   ├── organisations.ts     # Org CRUD, membership management
│   │       │   ├── meetings.ts         # Meeting CRUD, scheduling, join
│   │       │   ├── participants.ts     # Participant management
│   │       │   ├── recordings.ts       # Recording access, download
│   │       │   ├── transcripts.ts      # Transcript access, search
│   │       │   ├── ai-artefacts.ts     # Summaries, action items
│   │       │   ├── integrations.ts     # OAuth connections, webhooks
│   │       │   └── admin.ts            # Org settings, billing, audit
│   │       ├── services/
│   │       │   ├── livekit.ts          # LiveKit room/token management
│   │       │   ├── recording.ts        # Recording lifecycle
│   │       │   ├── transcription.ts    # Transcription pipeline
│   │       │   ├── ai-summary.ts       # LLM summary generation
│   │       │   ├── action-items.ts     # Action item extraction
│   │       │   ├── calendar.ts         # Calendar sync (Google, Outlook)
│   │       │   ├── consent.ts          # GDPR/HIPAA consent management
│   │       │   └── notifications.ts    # Email, in-app notifications
│   │       ├── middleware/
│   │       │   ├── auth.ts             # JWT verification, session loading
│   │       │   ├── org-context.ts      # Multi-tenant org scoping
│   │       │   ├── rate-limit.ts       # Per-user and per-org rate limiting
│   │       │   └── audit.ts           # Audit log middleware
│   │       ├── websocket/
│   │       │   ├── handler.ts          # WebSocket connection manager
│   │       │   └── events.ts          # Real-time event broadcasting
│   │       └── plugins/
│   │           ├── swagger.ts          # OpenAPI spec generation
│   │           └── cors.ts
│   ├── web/                             # Next.js frontend
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── next.config.ts
│   │   └── src/
│   │       ├── app/
│   │       │   ├── layout.tsx
│   │       │   ├── page.tsx             # Landing / login
│   │       │   ├── (auth)/
│   │       │   │   ├── login/page.tsx
│   │       │   │   └── register/page.tsx
│   │       │   ├── (dashboard)/
│   │       │   │   ├── layout.tsx       # Sidebar, org switcher
│   │       │   │   ├── meetings/page.tsx
│   │       │   │   ├── meetings/[id]/page.tsx
│   │       │   │   ├── recordings/page.tsx
│   │       │   │   ├── recordings/[id]/page.tsx
│   │       │   │   ├── transcripts/page.tsx
│   │       │   │   ├── action-items/page.tsx
│   │       │   │   └── settings/page.tsx
│   │       │   └── meet/
│   │       │       └── [roomId]/page.tsx  # Meeting room (video UI)
│   │       ├── components/
│   │       │   ├── meeting/
│   │       │   │   ├── video-grid.tsx
│   │       │   │   ├── participant-tile.tsx
│   │       │   │   ├── controls-bar.tsx
│   │       │   │   ├── chat-panel.tsx
│   │       │   │   ├── participants-panel.tsx
│   │       │   │   ├── screen-share.tsx
│   │       │   │   ├── breakout-rooms.tsx
│   │       │   │   └── captions-overlay.tsx
│   │       │   ├── dashboard/
│   │       │   │   ├── meeting-list.tsx
│   │       │   │   ├── recording-player.tsx
│   │       │   │   ├── transcript-viewer.tsx
│   │       │   │   └── action-item-list.tsx
│   │       │   └── ui/                  # shadcn/ui components
│   │       ├── hooks/
│   │       │   ├── use-meeting.ts
│   │       │   ├── use-livekit.ts
│   │       │   └── use-realtime.ts
│   │       └── lib/
│   │           ├── api-client.ts        # Typed fetch wrapper
│   │           └── auth.ts              # NextAuth config
│   └── worker/                          # BullMQ job processor
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts                 # Worker entry point
│           ├── jobs/
│           │   ├── process-recording.ts
│           │   ├── generate-transcript.ts
│           │   ├── generate-summary.ts
│           │   ├── extract-action-items.ts
│           │   ├── sync-calendar.ts
│           │   ├── enforce-retention.ts
│           │   └── send-notification.ts
│           └── processors/
│               ├── ffmpeg.ts            # Recording post-processing
│               └── whisper.ts           # Whisper model runner
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
├── config/
│   ├── livekit.yaml                     # LiveKit server config
│   └── redis.conf
└── docs/
    └── api/                             # Auto-generated OpenAPI spec output
```

---

## Phase 1: Foundation and Project Scaffolding

### Purpose

Establish the monorepo structure, database schema, configuration system, and development tooling. After this phase, developers can run the full stack locally with Docker Compose, connect to PostgreSQL, and execute the first database migrations. No user-facing features yet, but the foundation supports all subsequent phases.

### Tasks

#### 1.1 — Monorepo Initialisation and Tooling

**What**: Create the pnpm workspace with shared TypeScript configuration, Biome linting, and Docker Compose for local services.

**Design**:

Root `package.json`:
```json
{
  "name": "video-conferencing-platform",
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "db:migrate": "pnpm --filter api db:migrate",
    "db:seed": "pnpm --filter api db:seed"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.0.0",
    "turbo": "^2.4.0",
    "typescript": "^5.7.0"
  }
}
```

`tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUncheckedIndexedAccess": true
  }
}
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: videoconf
      POSTGRES_USER: videoconf
      POSTGRES_PASSWORD: localdev
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data

  livekit:
    image: livekit/livekit-server:latest
    ports: ["7880:7880", "7881:7881", "7882:7882/udp"]
    command: --config /etc/livekit.yaml --dev
    volumes:
      - ./config/livekit.yaml:/etc/livekit.yaml

volumes:
  pgdata:
  minio-data:
```

`.env.example`:
```bash
# Database
DATABASE_URL=postgresql://videoconf:localdev@localhost:5432/videoconf

# Redis
REDIS_URL=redis://localhost:6379

# LiveKit
LIVEKIT_URL=ws://localhost:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=devsecretdevsecretdevsecret1234

# S3-compatible storage
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=recordings
S3_REGION=us-east-1

# Auth
AUTH_SECRET=dev-auth-secret-at-least-32-characters
NEXTAUTH_URL=http://localhost:3000

# AI
ANTHROPIC_API_KEY=sk-ant-...
```

**Testing**:
- `Unit: pnpm install completes without errors across all workspaces`
- `Unit: pnpm build compiles all packages without TypeScript errors`
- `Unit: biome check passes on all source files`
- `Integration: docker-compose up starts PostgreSQL, Redis, MinIO, and LiveKit without errors`
- `Integration: PostgreSQL accepts connections on port 5432 with configured credentials`
- `Integration: Redis PING returns PONG on port 6379`
- `Integration: MinIO health check passes on port 9000`

#### 1.2 — Database Schema and Migrations

**What**: Implement the Drizzle ORM schema based on Data Model Suggestion 3 (Hybrid Relational + JSONB) and generate initial migrations.

**Design**:

`apps/api/src/db/schema.ts` (core tables, following Data Model Suggestion 3):
```typescript
import { pgTable, uuid, text, timestamp, boolean, integer, jsonb, inet, numeric, date, index, uniqueIndex } from 'drizzle-orm/pg-core';

// ── Organisations ──────────────────────────────────
export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: text('plan').notNull().default('free'),           // free, pro, business, enterprise
  jurisdiction: text('jurisdiction').notNull().default('US'),
  settings: jsonb('settings').notNull().default('{}'),    // max_participants, branding, compliance
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_organisations_slug').on(table.slug),
]);

// ── Users ──────────────────────────────────────────
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  displayName: text('display_name').notNull(),
  avatarUrl: text('avatar_url'),
  passwordHash: text('password_hash'),
  authProvider: text('auth_provider').notNull().default('local'),
  authProviderId: text('auth_provider_id'),
  emailVerified: boolean('email_verified').notNull().default(false),
  preferences: jsonb('preferences').notNull().default('{}'),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_users_email').on(table.email),
  index('idx_users_auth').on(table.authProvider, table.authProviderId),
]);

// ── Organisation Members ───────────────────────────
export const organisationMembers = pgTable('organisation_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: text('role').notNull().default('member'),          // owner, admin, member, guest
  permissions: jsonb('permissions').notNull().default('[]'),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_org_members_unique').on(table.organisationId, table.userId),
  index('idx_org_members_org').on(table.organisationId),
  index('idx_org_members_user').on(table.userId),
]);

// ── Meetings ───────────────────────────────────────
export const meetings = pgTable('meetings', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  hostUserId: uuid('host_user_id').notNull().references(() => users.id),
  title: text('title').notNull(),
  description: text('description'),
  meetingType: text('meeting_type').notNull().default('instant'),
  status: text('status').notNull().default('scheduled'),
  scheduledStart: timestamp('scheduled_start', { withTimezone: true }),
  scheduledEnd: timestamp('scheduled_end', { withTimezone: true }),
  actualStart: timestamp('actual_start', { withTimezone: true }),
  actualEnd: timestamp('actual_end', { withTimezone: true }),
  joinUrl: text('join_url'),
  passwordHash: text('password_hash'),
  settings: jsonb('settings').notNull().default('{}'),
  recurrence: jsonb('recurrence'),
  metadata: jsonb('metadata').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_meetings_org').on(table.organisationId),
  index('idx_meetings_host').on(table.hostUserId),
]);

// ── Meeting Participants ───────────────────────────
export const meetingParticipants = pgTable('meeting_participants', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').references(() => users.id),
  displayName: text('display_name').notNull(),
  email: text('email'),
  role: text('role').notNull().default('attendee'),
  joinTime: timestamp('join_time', { withTimezone: true }),
  leaveTime: timestamp('leave_time', { withTimezone: true }),
  durationSeconds: integer('duration_seconds'),
  connectionInfo: jsonb('connection_info').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_participants_meeting').on(table.meetingId),
  index('idx_participants_user').on(table.userId),
]);

// ── Recordings ─────────────────────────────────────
export const recordings = pgTable('recordings', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  recordingType: text('recording_type').notNull(),
  storageProvider: text('storage_provider').notNull().default('s3'),
  storagePath: text('storage_path').notNull(),
  fileSizeBytes: integer('file_size_bytes'),
  durationSeconds: integer('duration_seconds'),
  mimeType: text('mime_type').notNull(),
  status: text('status').notNull().default('processing'),
  processingInfo: jsonb('processing_info').notNull().default('{}'),
  retentionPolicy: jsonb('retention_policy'),
  encryptionKeyId: text('encryption_key_id'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_recordings_meeting').on(table.meetingId),
  index('idx_recordings_status').on(table.status),
]);

// ── Transcripts ────────────────────────────────────
export const transcripts = pgTable('transcripts', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  recordingId: uuid('recording_id').references(() => recordings.id, { onDelete: 'set null' }),
  language: text('language').notNull().default('en'),
  status: text('status').notNull().default('processing'),
  engine: text('engine').notNull().default('whisper'),
  engineConfig: jsonb('engine_config').notNull().default('{}'),
  stats: jsonb('stats').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_transcripts_meeting').on(table.meetingId),
]);

// ── Transcript Segments ────────────────────────────
export const transcriptSegments = pgTable('transcript_segments', {
  id: uuid('id').primaryKey().defaultRandom(),
  transcriptId: uuid('transcript_id').notNull().references(() => transcripts.id, { onDelete: 'cascade' }),
  participantId: uuid('participant_id').references(() => meetingParticipants.id),
  speakerLabel: text('speaker_label'),
  segmentIndex: integer('segment_index').notNull(),
  startTimeMs: integer('start_time_ms').notNull(),
  endTimeMs: integer('end_time_ms').notNull(),
  text: text('text').notNull(),
  confidence: numeric('confidence', { precision: 4, scale: 3 }),
  language: text('language'),
  wordTimings: jsonb('word_timings'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_segments_transcript').on(table.transcriptId, table.segmentIndex),
  index('idx_segments_participant').on(table.participantId),
]);

// ── AI Artefacts ───────────────────────────────────
export const aiArtefacts = pgTable('ai_artefacts', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  artefactType: text('artefact_type').notNull(),          // summary, action_item, decision, topic, sentiment
  status: text('status').notNull().default('ready'),
  content: text('content').notNull(),
  assignedTo: uuid('assigned_to').references(() => users.id),
  dueDate: date('due_date'),
  metadata: jsonb('metadata').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_ai_artefacts_meeting').on(table.meetingId),
  index('idx_ai_artefacts_type').on(table.artefactType),
]);

// ── Meeting Messages ───────────────────────────────
export const meetingMessages = pgTable('meeting_messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  senderId: uuid('sender_id').notNull().references(() => meetingParticipants.id),
  recipientId: uuid('recipient_id').references(() => meetingParticipants.id),
  messageType: text('message_type').notNull().default('text'),
  content: text('content').notNull(),
  attachments: jsonb('attachments'),
  sentAt: timestamp('sent_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_messages_meeting').on(table.meetingId, table.sentAt),
]);

// ── Breakout Rooms ─────────────────────────────────
export const breakoutRooms = pgTable('breakout_rooms', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  roomIndex: integer('room_index').notNull(),
  status: text('status').notNull().default('pending'),
  participantIds: text('participant_ids').array().notNull().default([]),
  settings: jsonb('settings').notNull().default('{}'),
  openedAt: timestamp('opened_at', { withTimezone: true }),
  closedAt: timestamp('closed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_breakout_rooms_meeting').on(table.meetingId),
]);

// ── Integrations ───────────────────────────────────
export const integrations = pgTable('integrations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').references(() => organisations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').references(() => users.id, { onDelete: 'cascade' }),
  provider: text('provider').notNull(),
  connectionType: text('connection_type').notNull().default('oauth'),
  credentials: jsonb('credentials').notNull(),
  config: jsonb('config').notNull().default('{}'),
  status: text('status').notNull().default('active'),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  errorMessage: text('error_message'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_integrations_org').on(table.organisationId),
  index('idx_integrations_user').on(table.userId),
]);

// ── Audit Logs ─────────────────────────────────────
export const auditLogs = pgTable('audit_logs', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  actorId: uuid('actor_id'),
  action: text('action').notNull(),
  resourceType: text('resource_type').notNull(),
  resourceId: uuid('resource_id').notNull(),
  details: jsonb('details').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_audit_org_time').on(table.organisationId, table.createdAt),
  index('idx_audit_resource').on(table.resourceType, table.resourceId),
]);

// ── Consent Records ────────────────────────────────
export const consentRecords = pgTable('consent_records', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id),
  meetingId: uuid('meeting_id').references(() => meetings.id),
  consentType: text('consent_type').notNull(),
  granted: boolean('granted').notNull(),
  details: jsonb('details').notNull().default('{}'),
  grantedAt: timestamp('granted_at', { withTimezone: true }).notNull().defaultNow(),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
}, (table) => [
  index('idx_consent_user').on(table.userId),
  index('idx_consent_meeting').on(table.meetingId),
]);
```

**Testing**:
- `Unit: Drizzle schema compiles without TypeScript errors`
- `Unit: All table names match the Data Model Suggestion 3 specification`
- `Integration: drizzle-kit generate produces migration SQL matching expected DDL`
- `Integration: drizzle-kit migrate runs against PostgreSQL without errors`
- `Integration: Seed script inserts test organisation, user, and membership`
- `Integration: Foreign key constraints reject orphan meeting_participants`
- `Integration: JSONB default values are correctly set for settings, preferences, metadata columns`

#### 1.3 — Shared Types and Validation Schemas

**What**: Define TypeScript types and Zod validation schemas shared between API server, web client, and worker.

**Design**:

`packages/shared/src/types/meeting.ts`:
```typescript
export type MeetingType = 'instant' | 'scheduled' | 'recurring' | 'webinar';
export type MeetingStatus = 'scheduled' | 'in_progress' | 'ended' | 'cancelled';
export type ParticipantRole = 'host' | 'co_host' | 'attendee' | 'presenter' | 'interpreter';
export type ConnectionQuality = 'good' | 'fair' | 'poor';
export type DeviceType = 'desktop' | 'mobile' | 'tablet' | 'sip_phone';

export interface MeetingSettings {
  waitingRoom: boolean;
  recordingEnabled: boolean;
  transcriptionEnabled: boolean;
  aiSummaryEnabled: boolean;
  maxParticipants: number;
  e2eeEnabled: boolean;
  breakoutRooms?: {
    autoAssign: boolean;
    allowSelfSelect: boolean;
  };
}

export interface Meeting {
  id: string;
  organisationId: string;
  hostUserId: string;
  title: string;
  description?: string;
  meetingType: MeetingType;
  status: MeetingStatus;
  scheduledStart?: string;       // ISO 8601
  scheduledEnd?: string;
  actualStart?: string;
  actualEnd?: string;
  joinUrl?: string;
  settings: MeetingSettings;
  recurrence?: MeetingRecurrence;
  metadata: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}

export interface MeetingRecurrence {
  rrule: string;                  // RFC 5545 RRULE
  exceptions?: string[];          // ISO date strings
  timezone: string;
}

export interface MeetingParticipant {
  id: string;
  meetingId: string;
  userId?: string;
  displayName: string;
  email?: string;
  role: ParticipantRole;
  joinTime?: string;
  leaveTime?: string;
  durationSeconds?: number;
  connectionInfo: ConnectionInfo;
}

export interface ConnectionInfo {
  ipAddress?: string;
  userAgent?: string;
  deviceType?: DeviceType;
  browser?: string;
  os?: string;
  connectionQuality?: ConnectionQuality;
  bandwidthKbps?: number;
  codecAudio?: string;
  codecVideo?: string;
  joinedVia?: 'browser' | 'mobile_app' | 'sip' | 'pstn';
}
```

`packages/shared/src/types/ai.ts`:
```typescript
export type ArtefactType = 'summary' | 'action_item' | 'decision' | 'topic' | 'sentiment';
export type ArtefactStatus = 'processing' | 'ready' | 'failed' | 'dismissed';

export interface AiArtefact {
  id: string;
  meetingId: string;
  artefactType: ArtefactType;
  status: ArtefactStatus;
  content: string;
  assignedTo?: string;
  dueDate?: string;
  metadata: SummaryMetadata | ActionItemMetadata | SentimentMetadata;
  createdAt: string;
  updatedAt: string;
}

export interface SummaryMetadata {
  summaryType: 'executive' | 'detailed' | 'decision_log';
  modelId: string;
  modelVersion?: string;
  keyDecisions?: string[];
  topics?: string[];
}

export interface ActionItemMetadata {
  sourceSegmentId?: string;
  sourceTimeMs?: number;
  sourceText?: string;
  confidence: number;
  externalSync?: {
    provider: string;              // jira, asana, linear
    taskId: string;
    syncedAt: string;
  };
}

export interface SentimentMetadata {
  overallScore: number;
  engagementScore: number;
  participationEquity: number;
  perSpeaker: Array<{
    userId: string;
    speakingTimePct: number;
    sentiment: number;
  }>;
}
```

`packages/shared/src/schemas/meeting.ts` (Zod validation):
```typescript
import { z } from 'zod';

export const createMeetingSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().max(2000).optional(),
  meetingType: z.enum(['instant', 'scheduled', 'recurring', 'webinar']).default('instant'),
  scheduledStart: z.string().datetime().optional(),
  scheduledEnd: z.string().datetime().optional(),
  settings: z.object({
    waitingRoom: z.boolean().default(false),
    recordingEnabled: z.boolean().default(true),
    transcriptionEnabled: z.boolean().default(true),
    aiSummaryEnabled: z.boolean().default(true),
    maxParticipants: z.number().int().min(2).max(500).default(100),
    e2eeEnabled: z.boolean().default(false),
  }).default({}),
  recurrence: z.object({
    rrule: z.string(),
    exceptions: z.array(z.string()).optional(),
    timezone: z.string().default('UTC'),
  }).optional(),
  metadata: z.record(z.unknown()).default({}),
  password: z.string().min(4).max(50).optional(),
});

export type CreateMeetingInput = z.infer<typeof createMeetingSchema>;
```

**Testing**:
- `Unit: MeetingSettings interface accepts valid meeting configuration`
- `Unit: createMeetingSchema validates a correct meeting creation payload`
- `Unit: createMeetingSchema rejects title longer than 200 characters`
- `Unit: createMeetingSchema rejects maxParticipants > 500`
- `Unit: createMeetingSchema applies default values for omitted optional fields`
- `Unit: ArtefactType union accepts all five artefact types and rejects unknown strings`
- `Unit: z.string().datetime() rejects non-ISO-8601 date strings`

#### 1.4 — API Server Skeleton and Configuration

**What**: Create the Fastify server with configuration loading, health checks, CORS, and OpenAPI spec generation.

**Design**:

`apps/api/src/config.ts`:
```typescript
import { z } from 'zod';

const configSchema = z.object({
  PORT: z.coerce.number().default(4000),
  HOST: z.string().default('0.0.0.0'),
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  LIVEKIT_URL: z.string(),
  LIVEKIT_API_KEY: z.string(),
  LIVEKIT_API_SECRET: z.string().min(24),
  S3_ENDPOINT: z.string().url(),
  S3_ACCESS_KEY: z.string(),
  S3_SECRET_KEY: z.string(),
  S3_BUCKET: z.string().default('recordings'),
  S3_REGION: z.string().default('us-east-1'),
  AUTH_SECRET: z.string().min(32),
  ANTHROPIC_API_KEY: z.string().optional(),
  CORS_ORIGINS: z.string().default('http://localhost:3000'),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  return configSchema.parse(process.env);
}
```

`apps/api/src/server.ts`:
```typescript
import Fastify from 'fastify';
import cors from '@fastify/cors';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import { loadConfig } from './config.js';

export async function buildServer() {
  const config = loadConfig();
  const app = Fastify({ logger: { level: config.LOG_LEVEL } });

  await app.register(cors, { origin: config.CORS_ORIGINS.split(',') });

  await app.register(swagger, {
    openapi: {
      info: {
        title: 'Video Conferencing Platform API',
        version: '0.1.0',
        description: 'WebRTC-based video conferencing with AI meeting intelligence',
      },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
    },
  });

  await app.register(swaggerUi, { routePrefix: '/docs' });

  app.get('/health', { schema: { response: { 200: { type: 'object', properties: { status: { type: 'string' }, timestamp: { type: 'string' } } } } } }, async () => ({
    status: 'ok',
    timestamp: new Date().toISOString(),
  }));

  return app;
}
```

**Testing**:
- `Unit: loadConfig() parses valid environment variables into Config`
- `Unit: loadConfig() throws ZodError when DATABASE_URL is missing`
- `Unit: loadConfig() applies default PORT=4000 when PORT is not set`
- `Integration: GET /health returns { status: "ok" } with 200 status`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: OpenAPI spec at /docs/json contains correct title and version`
- `Integration: CORS headers are set for configured origins`
- `Integration: Request from non-allowed origin is rejected`

---

## Phase 2: Authentication and Organisation Management

### Purpose

Implement user registration, login (local credentials + OAuth), organisation creation, and membership management. After this phase, users can create accounts, form organisations, invite members, and authenticate to all API endpoints. This is prerequisite for all subsequent features which are scoped to an authenticated user within an organisation.

### Tasks

#### 2.1 — User Registration and Local Authentication

**What**: Implement password-based registration and login with JWT session tokens.

**Design**:

`apps/api/src/routes/auth.ts`:
```typescript
// POST /api/auth/register
interface RegisterRequest {
  email: string;
  displayName: string;
  password: string;                // min 8 chars, max 128
}
interface RegisterResponse {
  user: { id: string; email: string; displayName: string };
  token: string;                   // JWT access token
}

// POST /api/auth/login
interface LoginRequest {
  email: string;
  password: string;
}
interface LoginResponse {
  user: { id: string; email: string; displayName: string; avatarUrl?: string };
  token: string;
}

// POST /api/auth/refresh
// Authorization: Bearer <token>
interface RefreshResponse {
  token: string;
}
```

Password hashing: argon2id with default cost parameters. JWT payload:
```typescript
interface JwtPayload {
  sub: string;           // user ID
  email: string;
  iat: number;
  exp: number;           // 15 minutes
}
```

Refresh token: stored in httpOnly cookie, 7-day expiry, rotated on each use.

**Testing**:
- `Unit: password hashing with argon2id produces non-deterministic output`
- `Unit: password verification returns true for correct password, false for incorrect`
- `Unit: JWT token contains correct sub, email, and exp claims`
- `Unit: JWT with expired timestamp is rejected by middleware`
- `Integration: POST /api/auth/register creates user in database and returns 201 with token`
- `Integration: POST /api/auth/register with duplicate email returns 409`
- `Integration: POST /api/auth/register with password < 8 chars returns 400`
- `Integration: POST /api/auth/login with valid credentials returns 200 with token`
- `Integration: POST /api/auth/login with wrong password returns 401`
- `Integration: POST /api/auth/login with non-existent email returns 401 (same error, no enumeration)`
- `Integration: POST /api/auth/refresh with valid refresh cookie returns new access token`
- `Integration: POST /api/auth/refresh with expired refresh cookie returns 401`

#### 2.2 — OAuth 2.0 Provider Integration

**What**: Implement Google and Microsoft OAuth 2.0 login flows using Auth.js patterns.

**Design**:

```typescript
// GET /api/auth/oauth/google       → redirects to Google consent screen
// GET /api/auth/oauth/google/callback?code=...&state=...  → exchanges code, creates/links user
// GET /api/auth/oauth/microsoft    → redirects to Azure AD consent screen
// GET /api/auth/oauth/microsoft/callback?code=...&state=...

interface OAuthState {
  provider: 'google' | 'microsoft';
  nonce: string;          // CSRF protection
  returnUrl?: string;     // where to redirect after login
}

// On successful OAuth callback:
// 1. Exchange authorization code for access_token + refresh_token
// 2. Fetch user profile from provider
// 3. If user with this email exists: link auth_provider_id, log in
// 4. If no user exists: create new user, auto-verify email, log in
// 5. Store OAuth tokens in integrations table for calendar access
// 6. Redirect to returnUrl with session cookie set
```

**Testing**:
- `Unit: OAuth state serialisation includes nonce and provider`
- `Unit: OAuth state deserialisation rejects tampered or expired state`
- `Integration (mocked): Google callback with valid code creates user and returns session`
- `Integration (mocked): Google callback for existing email links provider and logs in`
- `Integration (mocked): Microsoft callback with valid code creates user with microsoft auth_provider`
- `Integration (mocked): OAuth callback with invalid code returns 400`
- `Integration (mocked): OAuth callback with mismatched state nonce returns 403`

#### 2.3 — Organisation CRUD and Membership Management

**What**: Implement organisation creation, member invitation, role assignment, and org-scoped middleware.

**Design**:

```typescript
// POST /api/organisations
interface CreateOrgRequest {
  name: string;
  slug: string;           // URL-safe identifier, unique
}
// Creator becomes owner automatically

// GET /api/organisations                → list user's organisations
// GET /api/organisations/:orgId         → org details (member only)
// PATCH /api/organisations/:orgId       → update name, settings (admin+)
// DELETE /api/organisations/:orgId      → delete org (owner only)

// POST /api/organisations/:orgId/members/invite
interface InviteMemberRequest {
  email: string;
  role: 'admin' | 'member' | 'guest';
}

// GET /api/organisations/:orgId/members → list members
// PATCH /api/organisations/:orgId/members/:memberId → change role
// DELETE /api/organisations/:orgId/members/:memberId → remove member

// Middleware: org-context
// All routes under /api/organisations/:orgId/* extract orgId from URL,
// verify the authenticated user is a member, and attach org + membership to request.
```

Org-scoped middleware:
```typescript
interface OrgContext {
  organisation: Organisation;
  membership: OrganisationMember;
  isOwner: boolean;
  isAdmin: boolean;
}

// Attached to request via Fastify decorator:
// request.org: OrgContext
```

**Testing**:
- `Unit: slug validation accepts lowercase alphanumeric with hyphens`
- `Unit: slug validation rejects spaces, uppercase, and special characters`
- `Integration: POST /api/organisations creates org with authenticated user as owner`
- `Integration: POST /api/organisations with duplicate slug returns 409`
- `Integration: GET /api/organisations returns only orgs where user is a member`
- `Integration: POST invite with valid email creates pending membership`
- `Integration: POST invite as member (non-admin) returns 403`
- `Integration: PATCH member role as admin succeeds`
- `Integration: DELETE member as owner succeeds`
- `Integration: DELETE self as last owner returns 400`
- `Integration: Org-context middleware returns 403 for non-member accessing org routes`
- `Integration: Org-context middleware populates request.org for valid member`

#### 2.4 — Auth Middleware and Audit Logging

**What**: Implement authentication middleware for all protected routes and automatic audit log creation.

**Design**:

```typescript
// Auth middleware: runs on all routes except /health, /api/auth/*, /docs
// 1. Extract JWT from Authorization: Bearer <token> header
// 2. Verify signature and expiration
// 3. Load user from database (or cache)
// 4. Attach user to request: request.user

// Audit middleware: runs on all mutating routes (POST, PATCH, PUT, DELETE)
// Writes to audit_logs table:
interface AuditLogEntry {
  organisationId: string;
  actorId: string;
  action: string;          // e.g. 'organisation.created', 'member.invited'
  resourceType: string;    // e.g. 'organisation', 'meeting', 'user'
  resourceId: string;
  details: {
    ipAddress: string;
    userAgent: string;
    changes?: Record<string, { from: unknown; to: unknown }>;
  };
}
```

**Testing**:
- `Unit: Auth middleware extracts valid JWT and attaches user to request`
- `Unit: Auth middleware returns 401 for missing Authorization header`
- `Unit: Auth middleware returns 401 for malformed Bearer token`
- `Integration: POST to protected route without token returns 401`
- `Integration: POST to protected route with valid token succeeds and creates audit log entry`
- `Integration: Audit log entry contains correct action, resource_type, resource_id, and actor_id`
- `Integration: Audit log details include IP address and user agent from request`
- `Integration: GET /health is accessible without authentication`

---

## Phase 3: Meeting Lifecycle and LiveKit Integration

### Purpose

Implement the core meeting lifecycle: creating, scheduling, joining, and ending meetings via the LiveKit SFU. After this phase, users can create meetings that generate LiveKit rooms, obtain join tokens, and connect to a real WebRTC session. This is the heart of the platform.

### Tasks

#### 3.1 — LiveKit Service Layer

**What**: Create a service that manages LiveKit rooms and generates participant access tokens.

**Design**:

```typescript
// apps/api/src/services/livekit.ts
import { RoomServiceClient, AccessToken, VideoGrant } from 'livekit-server-sdk';

export class LiveKitService {
  private roomService: RoomServiceClient;

  constructor(
    private livekitUrl: string,
    private apiKey: string,
    private apiSecret: string,
  ) {
    this.roomService = new RoomServiceClient(livekitUrl, apiKey, apiSecret);
  }

  async createRoom(meetingId: string, maxParticipants: number): Promise<Room> {
    return this.roomService.createRoom({
      name: meetingId,
      maxParticipants,
      emptyTimeout: 300,           // close room 5 min after last participant leaves
      metadata: JSON.stringify({ meetingId }),
    });
  }

  generateToken(meetingId: string, participantId: string, displayName: string, grants: Partial<VideoGrant>): string {
    const token = new AccessToken(this.apiKey, this.apiSecret, {
      identity: participantId,
      name: displayName,
      ttl: '6h',
      metadata: JSON.stringify({ participantId }),
    });
    token.addGrant({
      room: meetingId,
      roomJoin: true,
      canPublish: grants.canPublish ?? true,
      canSubscribe: grants.canSubscribe ?? true,
      canPublishData: grants.canPublishData ?? true,
      ...grants,
    });
    return token.toJwt();
  }

  async listParticipants(meetingId: string): Promise<ParticipantInfo[]> {
    return this.roomService.listParticipants(meetingId);
  }

  async removeParticipant(meetingId: string, participantId: string): Promise<void> {
    await this.roomService.removeParticipant(meetingId, participantId);
  }

  async deleteRoom(meetingId: string): Promise<void> {
    await this.roomService.deleteRoom(meetingId);
  }
}
```

**Testing**:
- `Unit (mocked): createRoom calls RoomServiceClient.createRoom with correct name and maxParticipants`
- `Unit: generateToken produces a valid JWT with room grant for the specified meeting`
- `Unit: generateToken sets canPublish=false when grants.canPublish is false`
- `Unit: generateToken TTL defaults to 6 hours`
- `Integration (real LiveKit): createRoom creates a room visible in LiveKit admin`
- `Integration (real LiveKit): generateToken allows a test client to connect to the room`
- `Integration (real LiveKit): deleteRoom removes the room from LiveKit`

#### 3.2 — Meeting CRUD API

**What**: Implement meeting creation, listing, updating, and deletion endpoints with LiveKit room provisioning.

**Design**:

```typescript
// POST /api/organisations/:orgId/meetings
// Creates meeting record + LiveKit room. Returns meeting with joinUrl.
// Body: CreateMeetingInput (from Zod schema in Phase 1)
// Response 201: Meeting

// GET /api/organisations/:orgId/meetings
// Query params: status?, page?, limit?, from?, to?
// Response 200: { meetings: Meeting[], total: number, page: number }

// GET /api/organisations/:orgId/meetings/:meetingId
// Response 200: Meeting (with participant count, recording status)

// PATCH /api/organisations/:orgId/meetings/:meetingId
// Body: Partial<CreateMeetingInput>
// Only host or admin can update. Cannot change meetingType after creation.
// Response 200: Meeting

// DELETE /api/organisations/:orgId/meetings/:meetingId
// Host or admin. If in_progress, ends meeting first (calls LiveKit deleteRoom).
// Response 204

// POST /api/organisations/:orgId/meetings/:meetingId/join
// Creates meeting_participant record and returns LiveKit access token.
interface JoinMeetingResponse {
  participantId: string;
  livekitToken: string;
  livekitUrl: string;
  meeting: Meeting;
}

// POST /api/organisations/:orgId/meetings/:meetingId/end
// Host only. Sets status='ended', actual_end=now(), deletes LiveKit room.
// Response 200: Meeting
```

Meeting status state machine:
```
scheduled → in_progress → ended
    │                       ↑
    └─── cancelled          │
                            │
instant ──→ in_progress ────┘
```

**Testing**:
- `Unit: Meeting status transitions follow the state machine (scheduled→in_progress, in_progress→ended)`
- `Unit: Meeting status rejects invalid transitions (ended→in_progress, cancelled→in_progress)`
- `Integration: POST /meetings creates meeting record in database and LiveKit room`
- `Integration: POST /meetings with type=instant sets status to in_progress immediately`
- `Integration: POST /meetings with type=scheduled sets status to scheduled`
- `Integration: GET /meetings returns paginated list filtered by org`
- `Integration: GET /meetings?status=scheduled returns only scheduled meetings`
- `Integration: GET /meetings/:id returns meeting with participant count`
- `Integration: POST /meetings/:id/join creates participant record and returns LiveKit token`
- `Integration: POST /meetings/:id/join as non-member of org returns 403`
- `Integration: POST /meetings/:id/join when meeting is ended returns 400`
- `Integration: POST /meetings/:id/end sets status=ended, actual_end timestamp, and deletes LiveKit room`
- `Integration: DELETE /meetings/:id while in_progress ends meeting first`
- `Integration: PATCH /meetings/:id as non-host returns 403`

#### 3.3 — LiveKit Webhook Handler

**What**: Process LiveKit server webhooks to update meeting and participant state in real time.

**Design**:

```typescript
// POST /api/webhooks/livekit
// LiveKit sends webhooks for room and participant events.
// Verify webhook signature using LIVEKIT_API_KEY + LIVEKIT_API_SECRET.

// Events handled:
// - room_started:       Update meeting status to 'in_progress', set actual_start
// - room_finished:      Update meeting status to 'ended', set actual_end
// - participant_joined: Update meeting_participant join_time, connection_info
// - participant_left:   Update meeting_participant leave_time, duration_seconds
// - track_published:    Log media track info to meeting_participants.connection_info
// - track_unpublished:  Update connection_info
// - recording_started:  Create recording record with status='processing'
// - recording_ended:    Update recording with storage_path and duration

interface LiveKitWebhookPayload {
  event: string;
  room?: { name: string; sid: string; numParticipants: number };
  participant?: { identity: string; name: string; state: string; joinedAt: number };
  track?: { type: string; source: string; sid: string };
}
```

**Testing**:
- `Unit: Webhook signature verification accepts valid LiveKit signature`
- `Unit: Webhook signature verification rejects tampered payload`
- `Integration: room_started webhook updates meeting status to in_progress`
- `Integration: participant_joined webhook sets join_time on meeting_participant`
- `Integration: participant_left webhook sets leave_time and computes duration_seconds`
- `Integration: room_finished webhook sets meeting status to ended`
- `Integration: Webhook with invalid signature returns 401`
- `Integration: Webhook for non-existent meeting returns 404 (logged, not retried)`

#### 3.4 — Waiting Room and Access Controls

**What**: Implement meeting password verification, waiting room admission flow, and host controls.

**Design**:

```typescript
// When meeting.settings.waitingRoom is true:
// POST /meetings/:id/join places participant in 'waiting' state
// Host sees waiting participants via WebSocket event
// POST /meetings/:id/participants/:pid/admit → grants LiveKit token
// POST /meetings/:id/participants/:pid/reject → removes from waiting list

// When meeting has a password:
// POST /meetings/:id/join requires { password: string } in body
// Server compares argon2id hash

// Host controls (WebSocket commands via LiveKit data channel):
// - mute_participant(participantId, trackType)
// - remove_participant(participantId)
// - lock_meeting()         → no new joins
// - unlock_meeting()
```

Participant state machine:
```
waiting → admitted → joined → left
    │                    ↑
    └── rejected         │
                         │
(no waiting room) ───────┘
```

**Testing**:
- `Integration: Join meeting with waitingRoom=true returns participant in waiting state (no LiveKit token)`
- `Integration: Host POST /admit grants LiveKit token and updates participant state to admitted`
- `Integration: Host POST /reject removes participant from waiting list`
- `Integration: Non-host attempting /admit returns 403`
- `Integration: Join meeting with password and correct password succeeds`
- `Integration: Join meeting with password and wrong password returns 401`
- `Integration: Lock meeting prevents new joins with 403`
- `Integration: Unlock meeting re-enables joins`

---

## Phase 4: Meeting UI and Real-Time Communication

### Purpose

Build the browser-based meeting experience: video grid, participant controls, screen sharing, in-meeting chat, and captions overlay. After this phase, users can join meetings from the web app, see and hear other participants, share their screen, and exchange chat messages.

### Tasks

#### 4.1 — Meeting Room Page and LiveKit Connection

**What**: Create the Next.js meeting room page that connects to LiveKit and renders the video grid.

**Design**:

```typescript
// apps/web/src/app/meet/[roomId]/page.tsx
// 1. Fetch meeting details and LiveKit token from API (POST /meetings/:id/join)
// 2. Connect to LiveKit room using livekit-client SDK
// 3. Render VideoGrid component with remote and local tracks

// apps/web/src/hooks/use-livekit.ts
interface UseLiveKitReturn {
  room: Room | null;
  participants: RemoteParticipant[];
  localParticipant: LocalParticipant | null;
  connectionState: ConnectionState;    // 'disconnected' | 'connecting' | 'connected' | 'reconnecting'
  connect: (url: string, token: string) => Promise<void>;
  disconnect: () => void;
  error: Error | null;
}

// apps/web/src/components/meeting/video-grid.tsx
interface VideoGridProps {
  participants: Participant[];      // includes local + remote
  activeSpeaker?: string;          // participant identity
  layout: 'grid' | 'speaker' | 'sidebar';
  pinnedParticipant?: string;
}
// Grid layout: CSS Grid with automatic columns (1-4 cols based on participant count)
// Speaker layout: active speaker large, others in strip
// Sidebar layout: screen share large, participants in sidebar

// apps/web/src/components/meeting/participant-tile.tsx
interface ParticipantTileProps {
  participant: Participant;
  isActiveSpeaker: boolean;
  isPinned: boolean;
  onPin: () => void;
  videoTrack?: Track;
  audioTrack?: Track;
}
// Shows: video feed, display name, mute indicators, connection quality badge
// Fallback: avatar with initials when video is off
```

**Testing**:
- `Unit: VideoGrid renders correct number of ParticipantTile components`
- `Unit: VideoGrid switches to speaker layout when layout prop is 'speaker'`
- `Unit: ParticipantTile shows avatar fallback when videoTrack is undefined`
- `Unit: ParticipantTile highlights active speaker with border indicator`
- `E2E (Playwright): Navigate to /meet/test-room, verify connection state reaches 'connected'`
- `E2E (Playwright): Two browser tabs join same room, each sees the other's video tile`
- `E2E (Playwright): Muting microphone shows mute indicator on own tile`

#### 4.2 — Meeting Controls Bar

**What**: Implement the bottom controls bar with microphone, camera, screen share, chat, participants, and leave controls.

**Design**:

```typescript
// apps/web/src/components/meeting/controls-bar.tsx
interface ControlsBarProps {
  localParticipant: LocalParticipant;
  isChatOpen: boolean;
  isParticipantsOpen: boolean;
  isCaptionsEnabled: boolean;
  isRecording: boolean;
  isScreenSharing: boolean;
  onToggleChat: () => void;
  onToggleParticipants: () => void;
  onToggleCaptions: () => void;
  onToggleRecording: () => void;
  onLeave: () => void;
  onEndMeeting: () => void;           // host only
  meetingRole: ParticipantRole;
}

// Controls:
// - Microphone toggle (with device selector dropdown)
// - Camera toggle (with device selector dropdown)
// - Screen share toggle
// - Chat toggle (with unread badge)
// - Participants toggle (with count badge)
// - Captions toggle
// - Record toggle (host only)
// - More menu: layout switch, virtual background, settings
// - Leave button (red, always visible)
// - End meeting button (host only, appears in more menu or beside leave)
```

**Testing**:
- `Unit: ControlsBar renders microphone, camera, screen share, chat, participants, leave buttons`
- `Unit: ControlsBar shows End Meeting button only when meetingRole is host`
- `Unit: Microphone button shows muted icon when local audio track is muted`
- `Unit: Chat button shows unread badge when isChatOpen is false and unread count > 0`
- `E2E: Click microphone button toggles local audio track mute state`
- `E2E: Click camera button toggles local video track mute state`
- `E2E: Click screen share button starts screen sharing (shows screen track)`
- `E2E: Click leave button disconnects from room and redirects to dashboard`

#### 4.3 — In-Meeting Chat

**What**: Implement real-time chat within the meeting using LiveKit data channels.

**Design**:

```typescript
// Chat messages sent via LiveKit DataChannel (reliable mode)
// Payload format:
interface ChatMessage {
  id: string;                      // UUID
  senderId: string;                // participant identity
  senderName: string;
  content: string;
  timestamp: string;               // ISO 8601
  recipientId?: string;            // undefined = broadcast to all
  messageType: 'text' | 'file' | 'reaction';
  attachments?: Array<{
    name: string;
    url: string;
    sizeBytes: number;
    mimeType: string;
  }>;
}

// apps/web/src/components/meeting/chat-panel.tsx
interface ChatPanelProps {
  messages: ChatMessage[];
  localParticipantId: string;
  participants: Participant[];
  onSend: (content: string, recipientId?: string) => void;
  isOpen: boolean;
}
// Features:
// - Message input with send button and Enter-to-send
// - Message list with sender name, timestamp, content
// - Private message selector (dropdown of participants)
// - File attachment via drag-and-drop or button (upload to S3, send URL)
// - Emoji reactions via picker
// - Auto-scroll to newest message
// - Unread message count when panel is closed

// Messages are also persisted to meeting_messages table via API call
// when meeting ends (batch write from host's client)
```

**Testing**:
- `Unit: ChatPanel renders message list with correct sender names and timestamps`
- `Unit: ChatPanel auto-scrolls to bottom when new message arrives`
- `Unit: ChatPanel shows private message indicator for messages with recipientId`
- `Unit: ChatMessage serialisation/deserialisation preserves all fields`
- `E2E: Send chat message in one tab, verify it appears in second tab`
- `E2E: Send private message to specific participant, verify only recipient sees it`
- `E2E: Chat panel shows unread badge when closed and new message arrives`

#### 4.4 — Screen Sharing

**What**: Implement screen sharing with annotation support.

**Design**:

```typescript
// Screen sharing uses LiveKit's built-in screen capture API:
// localParticipant.setScreenShareEnabled(true)
// This publishes a new video track with source='screen_share'

// apps/web/src/components/meeting/screen-share.tsx
interface ScreenShareProps {
  screenTrack: Track | null;
  sharerName: string;
  isLocal: boolean;
  onStop: () => void;               // only for local screen share
}
// When someone shares their screen:
// 1. Layout switches to 'sidebar' mode automatically
// 2. Screen share fills the main area
// 3. Participant tiles move to a side strip
// 4. "Stop sharing" button overlays the local screen share

// Annotation layer (canvas overlay on screen share):
// - Drawing tools: pen, highlighter, arrow, text
// - Colours: red, blue, green, yellow, white, black
// - Clear all, undo
// - Annotations sent via DataChannel to all participants
// - Annotations overlay on top of the screen share track
```

**Testing**:
- `Unit: ScreenShare component renders video element with screen track`
- `Unit: ScreenShare shows stop button only when isLocal is true`
- `Unit: Layout auto-switches to sidebar mode when screen share track is present`
- `E2E: Click screen share button, select screen, verify screen track appears for other participants`
- `E2E: Stop screen share, verify layout returns to grid mode`
- `E2E: Draw annotation on shared screen, verify annotation appears for other participants`

#### 4.5 — Virtual Backgrounds

**What**: Implement client-side virtual background and blur effects using MediaPipe or TensorFlow.js.

**Design**:

```typescript
// Virtual background processing pipeline:
// 1. Capture camera frame
// 2. Run segmentation model (MediaPipe Selfie Segmentation)
// 3. Separate foreground (person) from background
// 4. Replace background with: blur, image, or solid colour
// 5. Publish processed frame as video track

// apps/web/src/hooks/use-virtual-background.ts
interface UseVirtualBackgroundReturn {
  isSupported: boolean;               // false if WebGL not available
  isEnabled: boolean;
  backgroundType: 'none' | 'blur' | 'image' | 'color';
  setBackgroundType: (type: BackgroundType) => void;
  setBackgroundImage: (url: string) => void;
  setBlurAmount: (amount: number) => void;  // 1-20
  processedStream: MediaStream | null;
}
```

**Testing**:
- `Unit: useVirtualBackground returns isSupported=false when WebGL is unavailable`
- `Unit: setBackgroundType('blur') activates blur processing pipeline`
- `Unit: setBackgroundType('none') returns unprocessed camera stream`
- `E2E: Enable blur background, verify processed stream is published`
- `E2E: Switch to image background, verify background image replaces real background`

---

## Phase 5: Recording and Transcription

### Purpose

Implement cloud recording of meetings, post-meeting transcription with speaker diarisation, and a searchable transcript viewer. After this phase, meetings can be recorded to S3-compatible storage, automatically transcribed after ending, and transcripts are full-text searchable across the organisation.

### Tasks

#### 5.1 — Cloud Recording via LiveKit Egress

**What**: Implement recording start/stop using LiveKit's Egress API, with post-processing and S3 storage.

**Design**:

```typescript
// apps/api/src/services/recording.ts
export class RecordingService {
  async startRecording(meetingId: string, recordingType: 'composite' | 'audio_only'): Promise<string> {
    // 1. Call LiveKit EgressService.startRoomCompositeEgress()
    // 2. Configure output: S3 upload with path recordings/{orgId}/{meetingId}/{recordingId}.mp4
    // 3. Create recordings table entry with status='processing'
    // 4. Return recording ID
  }

  async stopRecording(egressId: string): Promise<void> {
    // 1. Call LiveKit EgressService.stopEgress(egressId)
    // 2. Update recording status to 'processing' (post-processing pending)
  }

  async onRecordingComplete(egressId: string, storagePath: string): Promise<void> {
    // Called by LiveKit webhook when egress completes
    // 1. Update recording status='ready', storage_path, file_size, duration
    // 2. Enqueue transcription job if meeting.settings.transcriptionEnabled
    // 3. Generate thumbnail
    // 4. Send notification to meeting host
  }
}

// Worker job: process-recording
// 1. Download recording from S3
// 2. Run ffprobe to extract metadata (duration, resolution, bitrate)
// 3. Generate thumbnail at 10% mark using ffmpeg
// 4. Upload thumbnail to S3
// 5. Update recording.processing_info with metadata
```

API endpoints:
```typescript
// POST /api/organisations/:orgId/meetings/:meetingId/recordings/start
// Body: { recordingType: 'composite' | 'audio_only' }
// Response 201: { recordingId: string, status: 'processing' }

// POST /api/organisations/:orgId/meetings/:meetingId/recordings/stop
// Response 200: { status: 'processing' }

// GET /api/organisations/:orgId/recordings
// Query: meetingId?, status?, page?, limit?
// Response 200: { recordings: Recording[], total: number }

// GET /api/organisations/:orgId/recordings/:recordingId
// Response 200: Recording (includes pre-signed download URL)

// GET /api/organisations/:orgId/recordings/:recordingId/download
// Response 302: Redirect to pre-signed S3 URL (15-minute expiry)

// DELETE /api/organisations/:orgId/recordings/:recordingId
// Response 204: Deletes S3 object and database record
```

**Testing**:
- `Unit (mocked): startRecording calls LiveKit EgressService with correct S3 config`
- `Unit (mocked): stopRecording calls EgressService.stopEgress with correct egressId`
- `Unit: onRecordingComplete updates recording status to ready`
- `Unit: onRecordingComplete enqueues transcription job when transcription is enabled`
- `Integration: POST /recordings/start during active meeting returns 201 with recording ID`
- `Integration: POST /recordings/start when meeting is not in_progress returns 400`
- `Integration: GET /recordings/:id returns recording with pre-signed download URL`
- `Integration: GET /recordings/:id/download returns 302 with valid S3 pre-signed URL`
- `Integration: DELETE /recordings/:id removes S3 object and database record`
- `Worker: process-recording job extracts metadata from test MP4 file`
- `Worker: process-recording job generates thumbnail image`

#### 5.2 — Transcription Pipeline

**What**: Build the async transcription pipeline using Whisper (via worker) with speaker diarisation and segment storage.

**Design**:

```typescript
// Worker job: generate-transcript
// Input: { meetingId: string, recordingId: string }
//
// Pipeline:
// 1. Download recording from S3
// 2. Extract audio track (ffmpeg -i input.mp4 -vn -acodec pcm_s16le output.wav)
// 3. Run Whisper large-v3 model for transcription
//    - Output: segments with timestamps and text
// 4. Run pyannote-audio speaker diarisation
//    - Output: speaker segments with timestamps
// 5. Align Whisper segments with diarisation segments
//    - Match by overlapping time ranges
//    - Assign speaker labels to transcript segments
// 6. Match speaker labels to meeting_participants
//    - Use voice embeddings or temporal heuristics (first speaker = host)
// 7. Write transcript record with status='ready'
// 8. Write transcript_segments with speaker assignments
// 9. Update transcript.stats with word_count, segment_count, confidence_avg
// 10. Enqueue AI summary generation if meeting.settings.aiSummaryEnabled

// Transcription engine interface (supports swapping Whisper for Deepgram):
interface TranscriptionEngine {
  transcribe(audioPath: string, options: TranscribeOptions): Promise<TranscriptionResult>;
}

interface TranscribeOptions {
  language?: string;                    // ISO 639-1, or undefined for auto-detect
  diarizationEnabled: boolean;
  vocabularyBoost?: string[];           // domain terms to boost recognition
}

interface TranscriptionResult {
  segments: TranscriptSegment[];
  language: string;
  wordCount: number;
  confidenceAvg: number;
  speakerCount: number;
}

interface TranscriptSegment {
  speakerLabel: string;                 // 'SPEAKER_00', 'SPEAKER_01', etc.
  segmentIndex: number;
  startTimeMs: number;
  endTimeMs: number;
  text: string;
  confidence: number;
  wordTimings?: Array<{
    word: string;
    start: number;
    end: number;
    confidence: number;
  }>;
}
```

**Testing**:
- `Unit: Audio extraction from MP4 produces valid WAV file`
- `Unit: Whisper engine returns segments with timestamps and confidence scores`
- `Unit: Speaker diarisation alignment assigns speaker labels to transcript segments`
- `Unit: TranscriptionResult serialisation matches transcript_segments schema`
- `Integration: generate-transcript job processes test recording end-to-end`
- `Integration: Transcript segments are written to database with correct foreign keys`
- `Integration: Transcript stats (word_count, confidence_avg) match computed values`
- `Fixture: Test with 3-speaker audio fixture, verify diarisation produces 3 speaker labels`
- `Fixture: Test with English+Spanish mixed audio, verify language detection per segment`

#### 5.3 — Transcript API and Full-Text Search

**What**: Expose transcript data via REST API with full-text search across all organisation transcripts.

**Design**:

```typescript
// GET /api/organisations/:orgId/transcripts
// Query: meetingId?, status?, language?, page?, limit?
// Response 200: { transcripts: Transcript[], total: number }

// GET /api/organisations/:orgId/transcripts/:transcriptId
// Response 200: Transcript with segments included
// Segments ordered by segment_index

// GET /api/organisations/:orgId/transcripts/:transcriptId/segments
// Query: speakerLabel?, fromMs?, toMs?, page?, limit?
// Response 200: { segments: TranscriptSegment[], total: number }

// GET /api/organisations/:orgId/transcripts/search
// Query: q (search query), meetingId? (optional filter), page?, limit?
// Uses PostgreSQL full-text search with ts_rank for relevance scoring
interface TranscriptSearchResult {
  meetingId: string;
  meetingTitle: string;
  transcriptId: string;
  segmentId: string;
  text: string;
  speakerLabel: string;
  startTimeMs: number;
  relevanceScore: number;
  highlight: string;                 // text with <mark> tags around matches
}

// SQL for search:
// SELECT m.id AS meeting_id, m.title AS meeting_title,
//        ts.id AS segment_id, ts.text,
//        ts.speaker_label, ts.start_time_ms,
//        ts_rank(to_tsvector('english', ts.text), plainto_tsquery('english', $query)) AS score,
//        ts_headline('english', ts.text, plainto_tsquery('english', $query)) AS highlight
// FROM transcript_segments ts
// JOIN transcripts t ON t.id = ts.transcript_id
// JOIN meetings m ON m.id = t.meeting_id
// WHERE m.organisation_id = $orgId
//   AND to_tsvector('english', ts.text) @@ plainto_tsquery('english', $query)
// ORDER BY score DESC
// LIMIT $limit OFFSET $offset;
```

**Testing**:
- `Integration: GET /transcripts returns transcripts for meetings in the org`
- `Integration: GET /transcripts/:id includes segments ordered by segment_index`
- `Integration: GET /transcripts/search?q=budget returns segments containing 'budget'`
- `Integration: Search results include highlight with <mark> tags`
- `Integration: Search is scoped to organisation (cannot search other org's transcripts)`
- `Integration: GET /transcripts/search?q=nonexistent returns empty results array`
- `Integration: Search relevance ranking places exact phrase matches higher`
- `Fixture: Seed 3 transcripts with known content, verify search returns correct matches`

#### 5.4 — Live Captions

**What**: Stream real-time captions during meetings using LiveKit's speech-to-text integration.

**Design**:

```typescript
// Live captions use LiveKit's transcription agent:
// A LiveKit Agent joins the room and publishes transcription events
// via DataChannel to all participants.

// Caption event payload (sent via LiveKit DataChannel):
interface CaptionEvent {
  participantId: string;
  speakerName: string;
  text: string;
  isFinal: boolean;                   // false for interim/partial results
  startTimeMs: number;
  endTimeMs: number;
  language: string;
}

// apps/web/src/components/meeting/captions-overlay.tsx
interface CaptionsOverlayProps {
  captions: CaptionEvent[];
  isEnabled: boolean;
  fontSize: 'small' | 'medium' | 'large';   // WCAG 2.2: user-configurable size
  position: 'bottom' | 'top';
}
// Displays rolling captions at bottom of video area
// Shows speaker name + text
// Interim captions shown in lighter colour, replaced when final arrives
// Maximum 3 lines visible at once
// Auto-dismiss after 5 seconds
```

**Testing**:
- `Unit: CaptionsOverlay renders caption text with speaker name`
- `Unit: CaptionsOverlay shows interim captions in lighter colour`
- `Unit: CaptionsOverlay replaces interim caption when final arrives`
- `Unit: CaptionsOverlay limits display to 3 lines`
- `Unit: CaptionsOverlay respects fontSize prop for WCAG compliance`
- `E2E: Enable captions, speak into microphone, verify caption text appears`
- `E2E: Disable captions, verify captions stop appearing`

---

## Phase 6: AI Meeting Intelligence

### Purpose

Implement AI-powered meeting summaries, action item extraction, and decision logging. After this phase, every meeting automatically generates an executive summary, extracts action items with assigned owners, and logs key decisions. This is the platform's primary differentiator.

### Tasks

#### 6.1 — AI Summary Generation

**What**: Build the post-meeting summary generation pipeline using Claude API.

**Design**:

```typescript
// Worker job: generate-summary
// Input: { meetingId: string, transcriptId: string }
//
// Pipeline:
// 1. Load full transcript text from transcript_segments
// 2. Load meeting metadata (title, description, participant list)
// 3. Call Claude API with system prompt and transcript
// 4. Parse structured response
// 5. Write ai_artefacts records (type='summary')

// Prompt template:
const SUMMARY_SYSTEM_PROMPT = `You are a meeting analyst. Given a meeting transcript with speaker labels, produce a structured summary.

Output JSON with this exact structure:
{
  "executive_summary": "2-3 sentence overview of the meeting",
  "key_topics": ["topic1", "topic2", ...],
  "key_decisions": [
    {"decision": "what was decided", "context": "brief context", "decided_by": "speaker name"}
  ],
  "discussion_highlights": [
    {"topic": "topic discussed", "summary": "brief summary", "speakers": ["name1", "name2"]}
  ],
  "follow_up_items": ["item1", "item2", ...],
  "meeting_sentiment": "positive" | "neutral" | "negative",
  "participation_notes": "brief observation about participation patterns"
}`;

// Claude API call with prompt caching:
import Anthropic from '@anthropic-ai/sdk';

async function generateSummary(transcript: string, meetingContext: MeetingContext): Promise<SummaryResponse> {
  const client = new Anthropic();
  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: [
      {
        type: 'text',
        text: SUMMARY_SYSTEM_PROMPT,
        cache_control: { type: 'ephemeral' },   // cache system prompt
      },
    ],
    messages: [
      {
        role: 'user',
        content: `Meeting: ${meetingContext.title}\nParticipants: ${meetingContext.participants.join(', ')}\n\nTranscript:\n${transcript}`,
      },
    ],
  });
  return JSON.parse(response.content[0].text) as SummaryResponse;
}
```

API endpoints:
```typescript
// GET /api/organisations/:orgId/meetings/:meetingId/summaries
// Response 200: AiArtefact[] (where artefact_type='summary')

// POST /api/organisations/:orgId/meetings/:meetingId/summaries/regenerate
// Re-runs summary generation (e.g. after transcript correction)
// Response 202: { status: 'processing' }
```

**Testing**:
- `Unit: Summary system prompt produces valid JSON when given a sample transcript`
- `Unit (mocked Claude): generateSummary returns structured SummaryResponse`
- `Unit (mocked Claude): generateSummary handles transcript > 100k tokens by chunking`
- `Unit: Summary response is correctly written as ai_artefacts record with type=summary`
- `Integration: generate-summary worker job processes test transcript and creates summary`
- `Integration: GET /meetings/:id/summaries returns generated summary`
- `Integration: POST /regenerate enqueues new summary job and returns 202`
- `Fixture: Process known 10-minute meeting transcript, verify summary contains expected topics`

#### 6.2 — Action Item Extraction

**What**: Extract action items from transcripts with assignee matching and due date detection.

**Design**:

```typescript
// Worker job: extract-action-items
// Input: { meetingId: string, transcriptId: string }
//
// Pipeline:
// 1. Load transcript segments
// 2. Call Claude API with action item extraction prompt
// 3. Match assignee names to meeting_participants → users
// 4. Write ai_artefacts records (type='action_item')
// 5. Send notifications to assigned users

const ACTION_ITEM_PROMPT = `You are a meeting analyst extracting action items from a transcript.

For each action item, output:
{
  "title": "brief action item title",
  "description": "fuller description if available",
  "assigned_to": "person name as mentioned in transcript",
  "due_date": "YYYY-MM-DD if mentioned, null if not",
  "source_text": "exact quote from transcript that contains this action",
  "source_segment_index": <segment number>,
  "confidence": 0.0 to 1.0
}

Only extract items where someone commits to doing something or is asked to do something.
Do not invent action items that are not clearly stated or implied.
Return a JSON array of action items.`;

// Assignee matching:
// 1. Extract "assigned_to" name from Claude response
// 2. Fuzzy match against meeting_participants.display_name (Levenshtein distance < 3)
// 3. If match found, link to user_id
// 4. If no match, store as unassigned with the raw name in metadata
```

API endpoints:
```typescript
// GET /api/organisations/:orgId/meetings/:meetingId/action-items
// Response 200: AiArtefact[] (where artefact_type='action_item')

// GET /api/organisations/:orgId/action-items
// Query: assignedTo?, status?, dueDate?, page?, limit?
// Returns action items across all meetings for the org

// PATCH /api/organisations/:orgId/action-items/:itemId
// Body: { status?: 'completed' | 'dismissed', assignedTo?: string, dueDate?: string }
// Response 200: AiArtefact

// POST /api/organisations/:orgId/action-items/:itemId/sync
// Body: { provider: 'jira' | 'asana' | 'linear', projectKey?: string }
// Creates external task and stores link in metadata.external_sync
// Response 200: AiArtefact (with updated metadata)
```

**Testing**:
- `Unit (mocked Claude): Action item extraction returns structured array from sample transcript`
- `Unit: Assignee fuzzy matching correctly matches "Alice" to participant "Alice Chen"`
- `Unit: Assignee fuzzy matching returns null for unrecognizable names`
- `Unit: Due date parsing handles "by Friday", "next week", "end of month" relative to meeting date`
- `Integration: extract-action-items job creates ai_artefacts records with type=action_item`
- `Integration: GET /meetings/:id/action-items returns extracted items with assignees`
- `Integration: PATCH /action-items/:id updates status to completed`
- `Integration: GET /action-items?assignedTo=userId returns items assigned to that user`
- `Fixture: Process transcript with 3 clear action items, verify all 3 are extracted`
- `Fixture: Process transcript with no action items, verify empty result`

#### 6.3 — AI Noise Suppression

**What**: Implement client-side AI noise suppression using RNNoise or Krisp-style model.

**Design**:

```typescript
// Client-side noise suppression using RNNoise WebAssembly module
// Processes audio frames in real-time before publishing to LiveKit

// apps/web/src/hooks/use-noise-suppression.ts
interface UseNoiseSuppressionReturn {
  isEnabled: boolean;
  isSupported: boolean;          // false if WASM not available
  toggle: () => void;
  suppressionLevel: 'low' | 'medium' | 'high';
  setSuppressionLevel: (level: SuppressionLevel) => void;
  processedStream: MediaStream | null;
}

// Audio processing pipeline:
// 1. Get raw microphone MediaStream
// 2. Create AudioContext and connect to RNNoise WASM processor
// 3. RNNoise processes 480-sample frames (10ms at 48kHz)
// 4. Output clean audio stream
// 5. Publish processed stream to LiveKit instead of raw stream
```

**Testing**:
- `Unit: useNoiseSuppression returns isSupported=false when WASM is unavailable`
- `Unit: RNNoise WASM module loads and initialises successfully`
- `Unit: Noise suppression toggle switches between raw and processed streams`
- `E2E: Enable noise suppression, verify processed audio stream is published to LiveKit`
- `Fixture: Process audio file with background noise, verify SNR improvement`

---

## Phase 7: Scheduling and Calendar Integration

### Purpose

Implement meeting scheduling with calendar integration for Google Calendar and Microsoft Outlook. After this phase, users can schedule future meetings, set up recurring meetings with RRULE patterns, and have meetings automatically appear in their calendar with join links.

### Tasks

#### 7.1 — Meeting Scheduling

**What**: Implement scheduled and recurring meeting creation with timezone-aware scheduling.

**Design**:

```typescript
// Scheduled meetings extend the instant meeting flow:
// - scheduledStart and scheduledEnd are required
// - Meeting remains in 'scheduled' status until host starts it
// - Reminder notifications sent before meeting

// Recurring meetings use RFC 5545 RRULE:
// - meetings.recurrence JSONB stores RRULE, exceptions, timezone
// - Each occurrence is a virtual instance (not a separate row)
// - When a recurring meeting is started, a concrete meeting row is created

// apps/api/src/services/calendar.ts
interface CalendarService {
  expandRecurrence(rrule: string, timezone: string, from: Date, to: Date): Date[];
  getNextOccurrence(rrule: string, timezone: string, after: Date): Date | null;
}

// Notification scheduling:
// - BullMQ delayed job: send reminder 15 minutes before scheduled_start
// - BullMQ delayed job: send reminder 5 minutes before scheduled_start
// - Notifications via email and in-app push
```

**Testing**:
- `Unit: expandRecurrence("FREQ=WEEKLY;BYDAY=TU,TH") produces correct dates for a given range`
- `Unit: expandRecurrence with exceptions skips excluded dates`
- `Unit: getNextOccurrence returns the next occurrence after a given timestamp`
- `Unit: Timezone conversion handles DST transitions correctly`
- `Integration: POST /meetings with type=scheduled creates meeting with scheduled status`
- `Integration: Recurring meeting listing returns virtual instances for the requested date range`
- `Integration: Starting a recurring instance creates a concrete meeting row`
- `Integration: Reminder jobs are enqueued at correct times relative to scheduled_start`

#### 7.2 — Google Calendar Integration

**What**: Sync meetings bidirectionally with Google Calendar via Google Calendar API.

**Design**:

```typescript
// OAuth flow:
// 1. User connects Google account via OAuth 2.0 (scopes: calendar.events)
// 2. Tokens stored in integrations table (provider='google')

// Sync operations:
// - On meeting creation: create Google Calendar event with video conferencing link
// - On meeting update: update Google Calendar event
// - On meeting deletion: delete Google Calendar event
// - On Google Calendar change (webhook): update local meeting

// apps/api/src/services/calendar-google.ts
export class GoogleCalendarService {
  async createEvent(meeting: Meeting, accessToken: string): Promise<string> {
    // POST https://www.googleapis.com/calendar/v3/calendars/primary/events
    // Body includes conferenceData with platform's join URL
    // Returns external_event_id
  }

  async updateEvent(externalEventId: string, meeting: Meeting, accessToken: string): Promise<void> {
    // PUT https://www.googleapis.com/calendar/v3/calendars/primary/events/{id}
  }

  async deleteEvent(externalEventId: string, accessToken: string): Promise<void> {
    // DELETE https://www.googleapis.com/calendar/v3/calendars/primary/events/{id}
  }

  async setupWebhook(accessToken: string, channelId: string): Promise<void> {
    // POST https://www.googleapis.com/calendar/v3/calendars/primary/events/watch
    // Receives push notifications for calendar changes
  }
}
```

**Testing**:
- `Unit (mocked): createEvent sends correct payload to Google Calendar API`
- `Unit (mocked): createEvent includes conferenceData with join URL`
- `Unit (mocked): updateEvent sends PATCH with changed fields only`
- `Integration (mocked): Creating a meeting with Google Calendar connected creates calendar event`
- `Integration (mocked): Deleting a meeting removes the Google Calendar event`
- `Integration (mocked): Google Calendar webhook notification updates local meeting time`
- `Integration: OAuth flow stores Google tokens in integrations table`

#### 7.3 — Microsoft Outlook Calendar Integration

**What**: Sync meetings with Microsoft Outlook/365 Calendar via Microsoft Graph API.

**Design**:

```typescript
// OAuth flow:
// 1. User connects Microsoft account via Azure AD OAuth 2.0 (scopes: Calendars.ReadWrite)
// 2. Tokens stored in integrations table (provider='microsoft')

// apps/api/src/services/calendar-microsoft.ts
export class MicrosoftCalendarService {
  async createEvent(meeting: Meeting, accessToken: string): Promise<string> {
    // POST https://graph.microsoft.com/v1.0/me/events
    // Body includes onlineMeeting with platform's join URL
    // Returns external_event_id
  }

  async updateEvent(externalEventId: string, meeting: Meeting, accessToken: string): Promise<void> {
    // PATCH https://graph.microsoft.com/v1.0/me/events/{id}
  }

  async deleteEvent(externalEventId: string, accessToken: string): Promise<void> {
    // DELETE https://graph.microsoft.com/v1.0/me/events/{id}
  }

  async setupSubscription(accessToken: string): Promise<void> {
    // POST https://graph.microsoft.com/v1.0/subscriptions
    // Receives change notifications for calendar events
  }
}
```

**Testing**:
- `Unit (mocked): createEvent sends correct payload to Microsoft Graph API`
- `Unit (mocked): Microsoft event body includes correct datetime format with timezone`
- `Integration (mocked): Creating a meeting with Outlook connected creates Outlook event`
- `Integration (mocked): Outlook change notification updates local meeting`
- `Integration: Azure AD OAuth flow stores Microsoft tokens in integrations table`

---

## Phase 8: Dashboard and Meeting Management UI

### Purpose

Build the web dashboard for managing meetings, viewing recordings, browsing transcripts, and tracking action items. After this phase, users have a complete web interface for all non-meeting activities.

### Tasks

#### 8.1 — Dashboard Layout and Navigation

**What**: Create the authenticated dashboard layout with sidebar navigation, org switcher, and user menu.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
// Sidebar navigation items:
// - Meetings (calendar icon) → /meetings
// - Recordings (film icon) → /recordings
// - Transcripts (file-text icon) → /transcripts
// - Action Items (check-square icon) → /action-items
// - Settings (gear icon) → /settings
//
// Top bar:
// - Org switcher dropdown (shows current org name, switch between orgs)
// - Search bar (transcript search across org)
// - "New Meeting" button (primary CTA)
// - User avatar dropdown (profile, preferences, logout)

// Org switcher:
interface OrgSwitcherProps {
  currentOrg: Organisation;
  orgs: Organisation[];
  onSwitch: (orgId: string) => void;
  onCreateOrg: () => void;
}
```

**Testing**:
- `Unit: Dashboard layout renders sidebar with all navigation items`
- `Unit: Org switcher displays current organisation name`
- `Unit: Org switcher dropdown lists all user's organisations`
- `E2E: Navigate to /meetings, verify sidebar highlights Meetings item`
- `E2E: Click org switcher, select different org, verify dashboard reloads with new org context`
- `E2E: Click "New Meeting", verify meeting creation dialog appears`

#### 8.2 — Meetings List and Detail Pages

**What**: Build the meetings page with list view, calendar view, and individual meeting detail page.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/meetings/page.tsx
// Views:
// - List view: table of upcoming and recent meetings
// - Calendar view: month/week/day calendar showing scheduled meetings
// Filters: status (upcoming, past, cancelled), date range
// Actions: New Meeting, Join (for active meetings)

// apps/web/src/app/(dashboard)/meetings/[id]/page.tsx
// Sections:
// - Meeting details (title, time, host, participants)
// - Recording player (if recording exists)
// - Transcript viewer (if transcript exists)
// - AI Summary (if generated)
// - Action Items (if extracted)
// - Chat history (persisted messages)

// Meeting creation dialog:
interface CreateMeetingDialogProps {
  onSubmit: (data: CreateMeetingInput) => Promise<void>;
  isOpen: boolean;
  onClose: () => void;
}
// Fields: title, description, type (instant/scheduled), date/time picker,
// duration, recurring toggle (with RRULE builder), settings (recording, transcription, AI)
```

**Testing**:
- `Unit: Meeting list renders correct number of meeting cards`
- `Unit: Meeting list shows "Join" button for in_progress meetings`
- `Unit: Meeting detail page shows recording player when recording is available`
- `Unit: Meeting creation dialog validates required fields`
- `E2E: Create instant meeting from dialog, verify redirect to meeting room`
- `E2E: Create scheduled meeting, verify it appears in meetings list`
- `E2E: Click on past meeting, verify transcript and summary are displayed`

#### 8.3 — Recording Player and Transcript Viewer

**What**: Build a recording playback page with synchronised transcript highlighting and jump-to-timestamp.

**Design**:

```typescript
// apps/web/src/components/dashboard/recording-player.tsx
interface RecordingPlayerProps {
  recordingUrl: string;
  transcript?: TranscriptWithSegments;
  chapters?: Array<{ title: string; startMs: number; endMs: number }>;
}
// Features:
// - Video player with standard controls (play, pause, seek, volume, fullscreen)
// - Chapter markers on seek bar
// - Transcript panel synchronised with playback position
// - Active segment highlighted as playback progresses
// - Click transcript segment to jump to that timestamp
// - Word-level highlighting during playback (using wordTimings)
// - Search within transcript with result highlighting
// - Speaker filter (show segments from selected speakers only)
// - Download transcript as text/SRT/VTT

// apps/web/src/components/dashboard/transcript-viewer.tsx
interface TranscriptViewerProps {
  segments: TranscriptSegment[];
  currentTimeMs?: number;                // current playback position
  onSeek?: (timeMs: number) => void;     // callback when user clicks a segment
  searchQuery?: string;
  speakerFilter?: string[];
}
```

**Testing**:
- `Unit: RecordingPlayer renders video element with correct source URL`
- `Unit: TranscriptViewer highlights segment matching currentTimeMs`
- `Unit: TranscriptViewer filters segments by selected speakers`
- `Unit: Click on transcript segment calls onSeek with correct timestamp`
- `Unit: Chapter markers appear at correct positions on seek bar`
- `E2E: Play recording, verify transcript auto-scrolls to current segment`
- `E2E: Click transcript segment, verify video seeks to that time`
- `E2E: Search transcript for keyword, verify matching segments are highlighted`

#### 8.4 — Action Items Dashboard

**What**: Build an action items page showing all action items across meetings with status management.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/action-items/page.tsx
// Views:
// - My Action Items: items assigned to current user
// - All Action Items: items across the org (admin view)
// Filters: status (open, in_progress, completed, dismissed), assignee, due date
// Sort: due date, meeting date, status
// Actions: mark complete, dismiss, change assignee, set due date, sync to external tool

// apps/web/src/components/dashboard/action-item-list.tsx
interface ActionItemListProps {
  items: AiArtefact[];
  onStatusChange: (itemId: string, status: ArtefactStatus) => void;
  onAssigneeChange: (itemId: string, userId: string) => void;
  onSync: (itemId: string, provider: string) => void;
}
// Each item shows:
// - Title and description
// - Source meeting (link to meeting detail page)
// - Assigned to (avatar + name)
// - Due date (with overdue highlighting)
// - Status badge
// - "Jump to moment" link (opens recording at source timestamp)
// - Sync status (Jira/Asana icon if synced)
```

**Testing**:
- `Unit: ActionItemList renders items with correct status badges`
- `Unit: ActionItemList highlights overdue items (due date in the past)`
- `Unit: "Jump to moment" link navigates to recording at correct timestamp`
- `Unit: Status dropdown shows correct options`
- `E2E: Mark action item as completed, verify status updates`
- `E2E: Filter by assignee, verify only matching items shown`
- `E2E: Click "Jump to moment", verify recording opens at source timestamp`

---

## Phase 9: Breakout Rooms and Collaboration

### Purpose

Implement breakout rooms for small-group discussions within a meeting, and a collaborative whiteboard. After this phase, hosts can split participants into breakout rooms and participants can collaborate visually on a shared whiteboard.

### Tasks

#### 9.1 — Breakout Room Management

**What**: Implement breakout room creation, participant assignment, and room lifecycle management via LiveKit sub-rooms.

**Design**:

```typescript
// Breakout rooms are implemented as separate LiveKit rooms with a naming convention:
// Main room: meeting-{meetingId}
// Breakout:  meeting-{meetingId}-breakout-{roomIndex}

// API endpoints:
// POST /api/organisations/:orgId/meetings/:meetingId/breakout-rooms
interface CreateBreakoutRoomsRequest {
  count: number;                       // 2-20 rooms
  assignmentMode: 'manual' | 'random' | 'self_select';
  autoCloseAfterMinutes?: number;
  names?: string[];                    // optional room names
}

// POST /api/organisations/:orgId/meetings/:meetingId/breakout-rooms/open
// Moves assigned participants to breakout rooms

// POST /api/organisations/:orgId/meetings/:meetingId/breakout-rooms/close
// Returns all participants to main room

// POST /api/organisations/:orgId/meetings/:meetingId/breakout-rooms/:roomId/assign
// Body: { participantIds: string[] }

// Participant flow:
// 1. Host creates breakout rooms and assigns participants
// 2. Host opens breakout rooms
// 3. Each participant receives a new LiveKit token for their breakout room
// 4. Client disconnects from main room and connects to breakout room
// 5. Host broadcasts message to all rooms (via main room data channel)
// 6. Host closes breakout rooms
// 7. Participants receive token for main room, reconnect
```

**Testing**:
- `Unit: Random assignment distributes N participants across K rooms evenly`
- `Unit: CreateBreakoutRoomsRequest validates count is between 2 and 20`
- `Integration: POST /breakout-rooms creates correct number of LiveKit sub-rooms`
- `Integration: POST /breakout-rooms/open generates new tokens for each participant`
- `Integration: POST /breakout-rooms/close generates return tokens for main room`
- `Integration: Host broadcast message reaches all breakout rooms`
- `E2E: Create breakout rooms, assign participants, open rooms, verify participants move to separate rooms`
- `E2E: Close breakout rooms, verify all participants return to main room`

#### 9.2 — Collaborative Whiteboard

**What**: Implement a shared whiteboard within meetings using a canvas-based drawing tool.

**Design**:

```typescript
// Whiteboard is a shared canvas synchronised via LiveKit DataChannel
// Drawing operations are broadcast as incremental commands

interface WhiteboardCommand {
  id: string;
  type: 'draw' | 'shape' | 'text' | 'erase' | 'clear' | 'undo';
  authorId: string;
  timestamp: number;
  data: DrawData | ShapeData | TextData | EraseData;
}

interface DrawData {
  tool: 'pen' | 'highlighter';
  colour: string;                     // hex
  width: number;                      // px
  points: Array<{ x: number; y: number }>;  // normalised 0-1 coordinates
}

interface ShapeData {
  shape: 'rectangle' | 'circle' | 'arrow' | 'line';
  colour: string;
  width: number;
  startX: number; startY: number;
  endX: number; endY: number;
}

// apps/web/src/components/meeting/whiteboard.tsx
interface WhiteboardProps {
  commands: WhiteboardCommand[];
  onCommand: (cmd: WhiteboardCommand) => void;
  isReadOnly: boolean;
}
// Tools: pen, highlighter, shapes, text, eraser, colour picker, undo, clear
// Canvas is rendered using HTML5 Canvas API
// All coordinates normalised (0-1) for resolution independence
```

**Testing**:
- `Unit: Whiteboard renders canvas element at correct dimensions`
- `Unit: Draw command with normalised coordinates renders at correct pixel position`
- `Unit: Undo command removes last drawn element`
- `Unit: Clear command resets canvas`
- `E2E: Draw on whiteboard in one tab, verify drawing appears in second tab`
- `E2E: Undo in one tab, verify element removed in both tabs`

---

## Phase 10: Integrations and Webhooks

### Purpose

Build the integration framework for external tools (Slack, Jira, Salesforce) and the webhook system for custom integrations. After this phase, action items can be synced to project management tools, meeting notifications can go to Slack, and external systems can subscribe to platform events.

### Tasks

#### 10.1 — Webhook System

**What**: Implement outbound webhooks that notify external systems of platform events.

**Design**:

```typescript
// Webhook events:
// - meeting.started, meeting.ended
// - recording.ready
// - transcript.ready
// - summary.ready
// - action_item.created, action_item.completed

// apps/api/src/services/webhook.ts
export class WebhookService {
  async dispatch(orgId: string, event: string, payload: Record<string, unknown>): Promise<void> {
    // 1. Find all active webhooks for this org subscribed to this event
    // 2. For each webhook, enqueue a delivery job
  }
}

// Worker job: deliver-webhook
// 1. POST to webhook URL with JSON payload
// 2. Include X-Signature header: HMAC-SHA256(payload, secret)
// 3. Retry with exponential backoff (3 attempts: 1s, 30s, 300s)
// 4. Log delivery status (success, failed, retrying)

// API endpoints:
// POST /api/organisations/:orgId/webhooks
interface CreateWebhookRequest {
  url: string;                        // HTTPS required in production
  events: string[];                   // e.g. ['meeting.ended', 'recording.ready']
}
// Response 201: Webhook (includes auto-generated secret)

// GET /api/organisations/:orgId/webhooks
// PATCH /api/organisations/:orgId/webhooks/:id
// DELETE /api/organisations/:orgId/webhooks/:id
// POST /api/organisations/:orgId/webhooks/:id/test
// Sends a test event to verify the webhook endpoint
```

**Testing**:
- `Unit: HMAC-SHA256 signature is computed correctly for a given payload and secret`
- `Unit: Webhook dispatch finds correct webhooks subscribed to the event`
- `Integration: Creating a webhook stores URL, events, and generated secret`
- `Integration: Meeting ended triggers webhook delivery to all subscribed endpoints`
- `Integration (mocked): Webhook delivery sends POST with correct signature header`
- `Integration (mocked): Webhook delivery retries on 500 response`
- `Integration (mocked): Webhook delivery marks as failed after 3 retries`
- `Integration: POST /webhooks/:id/test sends test payload and returns delivery result`

#### 10.2 — Jira Integration

**What**: Implement bidirectional sync between action items and Jira issues.

**Design**:

```typescript
// OAuth flow for Jira Cloud:
// 1. User connects Atlassian account (OAuth 2.0, 3LO)
// 2. Tokens stored in integrations table (provider='jira')

// apps/api/src/services/integrations/jira.ts
export class JiraIntegrationService {
  async createIssue(actionItem: AiArtefact, config: JiraConfig): Promise<string> {
    // POST https://api.atlassian.com/ex/jira/{cloudId}/rest/api/3/issue
    // Returns issue key (e.g. 'PROJ-456')
  }

  async updateIssueStatus(externalTaskId: string, status: string, config: JiraConfig): Promise<void> {
    // POST https://api.atlassian.com/ex/jira/{cloudId}/rest/api/3/issue/{key}/transitions
  }

  async syncFromJira(externalTaskId: string, config: JiraConfig): Promise<{ status: string }> {
    // GET issue from Jira, update local action item status
  }
}

interface JiraConfig {
  cloudId: string;
  projectKey: string;
  issueType: string;                  // 'Task', 'Story', 'Bug'
  accessToken: string;
}
```

**Testing**:
- `Unit (mocked): createIssue sends correct payload to Jira API`
- `Unit (mocked): updateIssueStatus sends correct transition`
- `Integration (mocked): POST /action-items/:id/sync with provider=jira creates Jira issue`
- `Integration (mocked): Completing action item updates Jira issue status`
- `Integration: Jira OAuth flow stores tokens in integrations table`

#### 10.3 — Slack Notification Integration

**What**: Send meeting notifications (start, end, summary, action items) to Slack channels.

**Design**:

```typescript
// Slack Bot integration:
// 1. Install Slack app via OAuth (Bot Token scopes: chat:write, channels:read)
// 2. Configure channel mapping per org

// Notifications sent to Slack:
// - Meeting started: "Meeting '{title}' has started. Join: {joinUrl}"
// - Meeting ended: "Meeting '{title}' has ended. {participantCount} participants, {duration}."
// - Summary ready: Rich message with executive summary, key decisions, action items
// - Action item assigned: DM to assignee with action item details

// apps/api/src/services/integrations/slack.ts
export class SlackIntegrationService {
  async sendMeetingSummary(summary: AiArtefact, channel: string, botToken: string): Promise<void> {
    // POST https://slack.com/api/chat.postMessage
    // Uses Block Kit for rich formatting
  }

  async sendActionItemDM(actionItem: AiArtefact, slackUserId: string, botToken: string): Promise<void> {
    // POST https://slack.com/api/chat.postMessage
    // Channel = user's DM channel
  }
}
```

**Testing**:
- `Unit (mocked): sendMeetingSummary sends Block Kit formatted message to Slack API`
- `Unit (mocked): sendActionItemDM sends DM to correct Slack user`
- `Integration (mocked): Meeting end triggers Slack notification to configured channel`
- `Integration (mocked): Summary ready triggers rich Slack message with decisions and action items`
- `Integration: Slack OAuth flow stores bot token in integrations table`

---

## Phase 11: Compliance, Security, and Administration

### Purpose

Implement GDPR consent management, data retention policies, SSO/2FA authentication, and admin controls. After this phase, the platform meets enterprise compliance requirements for GDPR, HIPAA, and SOC 2 readiness.

### Tasks

#### 11.1 — GDPR Consent Management

**What**: Implement per-meeting, per-feature consent tracking as required by GDPR Articles 6, 9, and 32, and EU AI Act transparency requirements.

**Design**:

```typescript
// Consent is required before activating:
// - Recording (GDPR Article 6 lawful processing)
// - Transcription (GDPR Article 6)
// - AI summary generation (EU AI Act transparency)
// - Speaker voice identification (GDPR Article 9 biometric data)

// Consent flow:
// 1. When participant joins a meeting with recording/transcription/AI enabled,
//    a consent dialog is shown before publishing media tracks
// 2. Participant must explicitly opt-in to each feature
// 3. Consent is recorded in consent_records table with full context
// 4. Participant can revoke consent at any time during the meeting
// 5. If consent is revoked, participant's audio/video is excluded from recording

// API endpoints:
// POST /api/organisations/:orgId/meetings/:meetingId/consent
interface GrantConsentRequest {
  consentType: 'recording' | 'transcription' | 'ai_summary' | 'biometric_voice';
  granted: boolean;
}
// Response 200: ConsentRecord

// GET /api/organisations/:orgId/meetings/:meetingId/consent
// Returns consent status for all participants

// DELETE /api/users/me/consent/:consentId
// Revokes a specific consent record
```

**Testing**:
- `Integration: Joining meeting with recording enabled prompts consent dialog`
- `Integration: Granting consent creates consent_record with IP, user agent, and timestamp`
- `Integration: Revoking consent updates revoked_at timestamp`
- `Integration: Participant without recording consent is excluded from recording track`
- `Integration: Consent details include jurisdiction-specific fields (GDPR article for EU users)`
- `Unit: Consent dialog shows correct feature descriptions for each consent type`

#### 11.2 — Data Retention and Right to Erasure

**What**: Implement configurable data retention policies and GDPR right-to-erasure (right to be forgotten).

**Design**:

```typescript
// Data retention policies (per org, per resource type):
// - Recordings: default 90 days, configurable 30-365 days
// - Transcripts: default 90 days
// - Chat messages: default 90 days
// - Audit logs: minimum 365 days (compliance requirement)

// Worker job: enforce-retention
// Runs daily. For each org:
// 1. Query data_retention_policies
// 2. Find recordings/transcripts/messages older than retention period
// 3. Delete S3 objects for recordings
// 4. Delete database records
// 5. Log deletions in audit_logs

// Right to erasure (GDPR Article 17):
// POST /api/users/me/erasure-request
// 1. Anonymise user record (hash email, remove display_name, clear avatar)
// 2. Remove user from all organisation_members
// 3. Anonymise meeting_participants records (replace display_name with 'Deleted User')
// 4. Delete consent_records
// 5. Delete integration tokens
// 6. Retain anonymised audit logs (legal requirement)
// 7. Send confirmation email
```

**Testing**:
- `Integration: Retention policy deletes recordings older than configured days`
- `Integration: Retention policy preserves recordings newer than configured days`
- `Integration: Audit logs are never deleted by retention (minimum 365 days)`
- `Integration: Erasure request anonymises user data across all tables`
- `Integration: Erasure request removes OAuth tokens and integration data`
- `Integration: Anonymised participants show "Deleted User" in meeting history`
- `Integration: Erasure request creates audit log entry before anonymising`

#### 11.3 — SSO and Two-Factor Authentication

**What**: Implement SAML 2.0 SSO for enterprise customers and TOTP-based 2FA for all users.

**Design**:

```typescript
// SAML 2.0 SSO:
// - Org admin configures SAML identity provider (IdP metadata URL, entity ID)
// - SP-initiated flow: user visits /login → redirect to IdP → callback → session
// - IdP-initiated flow: user clicks app in IdP portal → callback → session
// - JIT (Just-In-Time) user provisioning on first SAML login

// TOTP 2FA:
// - User enables 2FA in settings
// - Server generates TOTP secret (base32 encoded)
// - User scans QR code with authenticator app
// - User verifies with a valid TOTP code
// - On subsequent logins, TOTP code required after password

// API endpoints:
// POST /api/users/me/2fa/enable → { secret: string, qrCodeUrl: string }
// POST /api/users/me/2fa/verify → { code: string } → { backupCodes: string[] }
// POST /api/users/me/2fa/disable → { code: string }
// POST /api/auth/login → if 2FA enabled, returns { requires2FA: true, tempToken: string }
// POST /api/auth/2fa/verify → { tempToken: string, code: string } → { token: string }
```

**Testing**:
- `Unit: TOTP secret generation produces valid base32 string`
- `Unit: TOTP code verification accepts valid code for current time window`
- `Unit: TOTP code verification rejects code from previous window (after drift allowance)`
- `Integration: Enable 2FA flow generates secret and QR code URL`
- `Integration: Login with 2FA enabled returns requires2FA=true`
- `Integration: 2FA verify with correct code returns session token`
- `Integration: 2FA verify with incorrect code returns 401`
- `Integration (mocked SAML): SAML callback creates user with auth_provider=saml`
- `Integration (mocked SAML): JIT provisioning creates org membership on first SAML login`

#### 11.4 — Admin Dashboard

**What**: Build organisation admin pages for member management, billing, audit logs, and compliance settings.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/settings/page.tsx
// Tabs:
// - General: org name, slug, branding
// - Members: invite, remove, change roles
// - Security: SSO config, 2FA enforcement, allowed auth providers
// - Compliance: data retention policies, HIPAA mode, data residency
// - Integrations: connected apps (Slack, Jira, calendars)
// - Billing: plan, usage, invoices (future)
// - Audit Log: searchable, filterable audit log viewer

// Audit log viewer:
interface AuditLogViewerProps {
  logs: AuditLogEntry[];
  filters: {
    actorId?: string;
    action?: string;
    resourceType?: string;
    dateFrom?: string;
    dateTo?: string;
  };
  onFilterChange: (filters: Filters) => void;
  page: number;
  totalPages: number;
}
```

**Testing**:
- `Unit: Settings page renders all tabs`
- `Unit: Members tab shows invite form and member list`
- `Unit: Audit log viewer renders log entries with correct action and timestamp`
- `E2E: Admin invites new member, verify invite appears in members list`
- `E2E: Admin changes data retention to 30 days, verify setting persists`
- `E2E: Audit log filters by action type, verify filtered results`
- `E2E: Non-admin accessing settings sees only profile tab (no admin tabs)`

---

## Phase 12: Advanced AI Features and Polish

### Purpose

Implement the advanced AI differentiators: real-time meeting coaching, sentiment analysis, and a searchable meeting knowledge base. Also add mobile responsiveness and performance optimisation. This phase delivers the features that distinguish the platform from incumbents.

### Tasks

#### 12.1 — Real-Time Meeting Coach

**What**: Build an AI coaching agent that provides live prompts during meetings for active listening, question quality, and agenda adherence.

**Design**:

```typescript
// The meeting coach runs as a LiveKit Agent (Python, via LiveKit Agents framework)
// It receives real-time transcript segments and provides coaching feedback

// packages/livekit-plugins/src/coaching-agent.ts (TypeScript orchestration)
// The actual agent runs in Python using LiveKit's agents framework

// Coaching categories:
// 1. Participation balance: "You haven't spoken in 5 minutes. Consider sharing your perspective."
// 2. Question quality: "Try asking an open-ended question to deepen the discussion."
// 3. Agenda adherence: "The conversation has drifted from the agenda topic '{topic}'."
// 4. Active listening: "Consider summarising what {speaker} just said before responding."
// 5. Time management: "You're at 75% of the scheduled time with 2 agenda items remaining."

// Coaching messages delivered via private DataChannel to the requesting participant only
interface CoachingMessage {
  category: 'participation' | 'question_quality' | 'agenda' | 'listening' | 'time';
  message: string;
  priority: 'low' | 'medium' | 'high';
  timestamp: string;
}

// Coach is opt-in per participant (not visible to others)
// Coaching messages appear as subtle toast notifications in the meeting UI
```

**Testing**:
- `Unit (mocked LLM): Coach detects 5-minute silence from participant and generates prompt`
- `Unit (mocked LLM): Coach detects topic drift from meeting agenda`
- `Unit (mocked LLM): Coach suggests open-ended question after series of closed questions`
- `Unit: Coaching message is delivered only to requesting participant (not broadcast)`
- `Integration: Enable coaching in meeting, speak for 2 minutes, verify coaching message received`
- `E2E: Enable coaching, verify coaching toast appears in UI`

#### 12.2 — Sentiment and Engagement Analysis

**What**: Analyse meeting transcripts for sentiment, engagement patterns, and participation equity.

**Design**:

```typescript
// Post-meeting analysis, runs after transcript is complete
// Worker job: analyse-sentiment
// Input: { meetingId: string, transcriptId: string }

// Analysis dimensions:
// 1. Overall meeting sentiment (positive/neutral/negative, 0-1 score)
// 2. Per-speaker sentiment over time (detect mood shifts)
// 3. Participation equity (Gini coefficient of speaking time)
// 4. Engagement indicators (question count, response rate, topic diversity)
// 5. Meeting effectiveness score (decisions made / time spent)

// Results stored as ai_artefacts with artefact_type='sentiment'
interface SentimentAnalysis {
  overallScore: number;               // 0-1
  overallLabel: 'positive' | 'neutral' | 'negative';
  engagementScore: number;            // 0-1
  participationEquity: number;        // 0-1 (1 = perfectly equal)
  effectivenessScore: number;         // 0-1
  perSpeaker: Array<{
    userId: string;
    displayName: string;
    speakingTimePct: number;
    sentimentScore: number;
    questionCount: number;
    sentimentOverTime: Array<{ timeMs: number; score: number }>;
  }>;
  topicSentiment: Array<{
    topic: string;
    sentiment: number;
    startTimeMs: number;
    endTimeMs: number;
  }>;
}
```

**Testing**:
- `Unit (mocked LLM): Sentiment analysis returns valid scores for sample transcript`
- `Unit: Participation equity calculation returns 1.0 for equal speaking time`
- `Unit: Participation equity returns < 0.5 when one speaker dominates`
- `Integration: analyse-sentiment job creates ai_artefacts record with type=sentiment`
- `Integration: Meeting detail page displays sentiment visualisation`
- `Fixture: Analyse transcript with clear positive/negative segments, verify per-topic sentiment`

#### 12.3 — Searchable Meeting Knowledge Base

**What**: Build a cross-meeting search interface that enables users to find information across all past meetings.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/transcripts/page.tsx
// Global search across all meetings in the organisation
// Features:
// - Full-text search with PostgreSQL tsvector
// - Filters: date range, participants, meeting title
// - Results show: meeting title, speaker, transcript excerpt with highlights, timestamp
// - Click result → opens recording at that timestamp
// - "Ask AI" mode: natural language question answered from transcript corpus

// "Ask AI" endpoint:
// POST /api/organisations/:orgId/knowledge-base/ask
interface AskKnowledgeBaseRequest {
  question: string;                    // "What did we decide about the Q3 budget?"
  dateFrom?: string;
  dateTo?: string;
  meetingIds?: string[];
}
interface AskKnowledgeBaseResponse {
  answer: string;
  sources: Array<{
    meetingId: string;
    meetingTitle: string;
    segmentId: string;
    text: string;
    timestamp: number;
    speaker: string;
  }>;
  confidence: number;
}

// Implementation:
// 1. Full-text search for relevant segments matching the question
// 2. Retrieve top 20 segments by relevance
// 3. Send segments + question to Claude API
// 4. Claude generates answer with source citations
// 5. Return answer with linked sources
```

**Testing**:
- `Unit: Knowledge base search returns results ranked by relevance`
- `Unit (mocked Claude): Ask AI generates answer with source citations from provided segments`
- `Integration: POST /knowledge-base/ask returns answer referencing specific meetings`
- `Integration: Search results are scoped to the user's organisation`
- `E2E: Type question in knowledge base search, verify AI answer appears with source links`
- `E2E: Click source link, verify recording opens at cited timestamp`

#### 12.4 — Mobile Responsiveness and Performance

**What**: Optimise the meeting UI for mobile browsers and improve performance for low-bandwidth connections.

**Design**:

```typescript
// Mobile meeting UI adaptations:
// - Single-participant focus view (swipe to switch)
// - Bottom sheet for controls (instead of horizontal bar)
// - Collapsible chat/participants panels
// - Touch-friendly button sizes (min 44x44px per WCAG)
// - Landscape mode for video, portrait for controls

// Performance optimisations:
// - Adaptive video quality based on bandwidth detection
//   - LiveKit simulcast: subscribe to low/medium/high layer based on viewport size
//   - < 500kbps: audio only with avatars
//   - 500kbps-2Mbps: low quality video (360p)
//   - > 2Mbps: high quality video (720p+)
// - Lazy load dashboard pages
// - Image optimisation for avatars and thumbnails
// - Service worker for offline dashboard access
// - Web Worker for transcript search (avoid blocking main thread)

// Bandwidth detection:
interface BandwidthMonitor {
  currentBandwidthKbps: number;
  connectionQuality: ConnectionQuality;
  recommendedVideoQuality: 'off' | 'low' | 'medium' | 'high';
  onQualityChange: (quality: ConnectionQuality) => void;
}
```

**Testing**:
- `Unit: Mobile layout renders single-participant view at viewport < 768px`
- `Unit: BandwidthMonitor recommends 'low' quality at 800kbps`
- `Unit: BandwidthMonitor recommends 'off' (audio only) at 300kbps`
- `E2E (mobile viewport): Meeting room renders correctly at 375x667 viewport`
- `E2E (mobile viewport): Swipe gesture switches between participant tiles`
- `E2E: Simulate slow network, verify video quality degrades gracefully`
- `Performance: Dashboard page loads in < 2 seconds on simulated 3G connection`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Auth & Org Management         ─── requires Phase 1
    │
Phase 3: Meeting Lifecycle & LiveKit   ─── requires Phase 2
    │
    ├── Phase 4: Meeting UI & Real-Time ─── requires Phase 3
    │       │
    │       └── Phase 5: Recording & Transcription ─── requires Phase 4
    │               │
    │               ├── Phase 6: AI Meeting Intelligence ─── requires Phase 5
    │               │
    │               └── Phase 8: Dashboard & Management UI ─── requires Phase 5
    │                       │
    │                       └── Phase 12: Advanced AI & Polish ─── requires Phase 6 + Phase 8
    │
    ├── Phase 7: Scheduling & Calendar  ─── requires Phase 3 (can parallel Phase 4-5)
    │
    ├── Phase 9: Breakout Rooms         ─── requires Phase 4 (can parallel Phase 5-6)
    │
    ├── Phase 10: Integrations & Webhooks ─── requires Phase 6 (can parallel Phase 8)
    │
    └── Phase 11: Compliance & Security ─── requires Phase 2 (can parallel Phase 4+)
```

### Parallelism Opportunities

- **Phase 7** (Calendar) can be developed concurrently with Phases 4-5 once Phase 3 is complete
- **Phase 9** (Breakout Rooms) can be developed concurrently with Phases 5-6 once Phase 4 is complete
- **Phase 10** (Integrations) can be developed concurrently with Phase 8 once Phase 6 is complete
- **Phase 11** (Compliance) can be started as early as Phase 3, developed concurrently with most other phases

---

## Definition of Done (per phase)

1. All tasks implemented as specified in the Design sections.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass against local Docker Compose services.
4. Biome linting passes with zero errors (`pnpm lint`).
5. TypeScript strict mode compilation passes with zero errors (`pnpm build`).
6. Docker build succeeds for all modified services (api, web, worker).
7. Database migrations run cleanly on a fresh database.
8. New API endpoints appear in auto-generated OpenAPI spec at `/docs`.
9. New environment variables documented in `.env.example`.
10. No secrets or credentials committed to source control.
11. Feature works end-to-end in a local development environment.
12. Accessibility: new UI components pass axe-core automated checks (WCAG 2.2 Level AA).
