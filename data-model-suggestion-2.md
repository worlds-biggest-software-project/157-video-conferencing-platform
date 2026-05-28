# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Video Conferencing Platform · Created: 2026-05-19

## Philosophy

This model treats an immutable event store as the single source of truth. Every state change in the system — a meeting being created, a participant joining, a recording starting, an AI summary being generated — is captured as an ordered, immutable event. Current state is derived by replaying events or querying materialised read models (CQRS pattern).

Event sourcing is a natural fit for video conferencing because the domain is inherently event-driven: participants join and leave, tracks are published and unpublished, chat messages arrive, recordings start and stop. LiveKit's SFU architecture already operates this way internally — room state is the result of a sequence of signalling events. This model simply persists that event stream to the database.

The audit-first design directly satisfies GDPR, HIPAA, and ISO 27001 compliance requirements without bolting on a separate audit system. Every query of the form "what was true at time T?" is answerable by replaying events up to T. This is particularly valuable for dispute resolution ("who was in the meeting when the decision was made?") and for AI analytics that need to reason about temporal patterns.

**Best for:** Platforms where full audit trails, temporal queries, and regulatory compliance are primary requirements — especially healthcare (HIPAA), finance, and government deployments.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — no separate audit system needed
- (+) Temporal queries are trivial ("what was the participant list at 14:32?")
- (+) Event replay enables rebuilding any read model without data loss
- (+) Natural fit for real-time WebSocket event streaming to clients
- (+) AI analytics can process the event stream for pattern detection
- (-) Higher storage volume — events are never deleted (only compacted)
- (-) Read model lag — materialised views may be slightly behind the event store
- (-) More complex to implement than direct CRUD; requires event handlers and projections
- (-) Debugging requires understanding event replay, not just current state
- (-) Schema evolution of event payloads requires careful versioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebRTC (W3C/IETF RFC 8825) | WebRTC signalling events (join, leave, track_published) map directly to domain events in the event store |
| ISO/IEC 27001:2022 | The event store itself IS the audit log — A.8.3 and A.8.24 are satisfied by design |
| GDPR Articles 6, 9, 32 | Consent events are first-class entries in the event store; GDPR right-to-erasure handled via crypto-shredding (destroy encryption key, events become unreadable) |
| HIPAA §164.312 | Immutable event store satisfies audit control and integrity control requirements |
| EU AI Act (2024/2847) | AI feature usage events (transcription_started, summary_generated) provide transparency audit trail |
| OCSF (Open Cybersecurity Schema Framework) | Event schema structure influenced by OCSF's activity_id / category_uid / type_uid pattern |
| OAuth 2.0 (RFC 6749) | Token grant/revoke events tracked in the event store |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================

-- The event store is the ONLY write model. All other tables are
-- materialised read models rebuilt from events.

CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                         -- aggregate root ID (meeting_id, user_id, org_id)
    stream_type     TEXT NOT NULL,                          -- meeting, user, organisation, recording, transcript
    event_type      TEXT NOT NULL,                          -- e.g. 'meeting.created', 'participant.joined'
    event_version   INT NOT NULL,                           -- schema version for this event type
    sequence_number BIGINT NOT NULL,                        -- ordering within the stream
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        UUID,                                   -- user who caused this event (NULL for system events)
    actor_ip        INET,
    organisation_id UUID NOT NULL,                          -- tenant scoping
    payload         JSONB NOT NULL,                         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',            -- correlation IDs, causation IDs, user agent
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ordering within a stream
CREATE UNIQUE INDEX idx_events_stream_seq ON events (stream_id, sequence_number);
-- Query all events for an aggregate
CREATE INDEX idx_events_stream ON events (stream_id, occurred_at);
-- Query events by type across all streams (for projections)
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
-- Tenant-scoped event queries
CREATE INDEX idx_events_org ON events (organisation_id, occurred_at DESC);
-- Actor activity queries (audit)
CREATE INDEX idx_events_actor ON events (actor_id, occurred_at DESC) WHERE actor_id IS NOT NULL;

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- ============================================================
-- EVENT TYPE REGISTRY
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    category        TEXT NOT NULL,                          -- meeting, participant, recording, transcript, ai, system
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                         -- JSON Schema for payload validation
    version         INT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event type registrations:
-- INSERT INTO event_type_registry VALUES
--   ('meeting.created', 'meeting', 'A new meeting was created', '{"type":"object",...}', 1, false, now()),
--   ('meeting.started', 'meeting', 'Meeting went live', '...', 1, false, now()),
--   ('meeting.ended', 'meeting', 'Meeting concluded', '...', 1, false, now()),
--   ('participant.joined', 'participant', 'A participant entered the meeting', '...', 1, false, now()),
--   ('participant.left', 'participant', 'A participant left the meeting', '...', 1, false, now()),
--   ('track.published', 'media', 'A media track was published', '...', 1, false, now()),
--   ('track.unpublished', 'media', 'A media track was removed', '...', 1, false, now()),
--   ('recording.started', 'recording', 'Recording began', '...', 1, false, now()),
--   ('recording.completed', 'recording', 'Recording finished and is ready', '...', 1, false, now()),
--   ('transcript.completed', 'transcript', 'Transcription processing finished', '...', 1, false, now()),
--   ('ai.summary_generated', 'ai', 'AI meeting summary was produced', '...', 1, false, now()),
--   ('ai.action_item_extracted', 'ai', 'AI extracted an action item', '...', 1, false, now()),
--   ('consent.granted', 'compliance', 'User granted consent for a feature', '...', 1, false, now()),
--   ('consent.revoked', 'compliance', 'User revoked consent', '...', 1, false, now()),
--   ('message.sent', 'chat', 'In-meeting chat message sent', '...', 1, false, now());
```

### Example Event Payloads

```sql
-- meeting.created event payload:
-- {
--   "title": "Q2 Planning",
--   "meeting_type": "scheduled",
--   "scheduled_start": "2026-05-20T14:00:00Z",
--   "scheduled_end": "2026-05-20T15:00:00Z",
--   "settings": {
--     "waiting_room": true,
--     "recording_enabled": true,
--     "transcription_enabled": true,
--     "max_participants": 50
--   }
-- }

-- participant.joined event payload:
-- {
--   "user_id": "550e8400-e29b-41d4-a716-446655440000",
--   "display_name": "Alice Chen",
--   "role": "attendee",
--   "device_type": "desktop",
--   "user_agent": "Mozilla/5.0 ...",
--   "connection_quality": "good"
-- }

-- ai.action_item_extracted event payload:
-- {
--   "title": "Send updated budget to finance team",
--   "assigned_to_user_id": "550e8400-e29b-41d4-a716-446655440000",
--   "due_date": "2026-05-25",
--   "source_transcript_segment": {
--     "start_time_ms": 1823400,
--     "end_time_ms": 1830200,
--     "text": "Alice, can you send the updated budget to finance by Friday?"
--   },
--   "confidence": 0.94,
--   "model_id": "claude-opus-4-6"
-- }
```

## Materialised Read Models (CQRS)

```sql
-- ============================================================
-- READ MODEL: ORGANISATIONS
-- ============================================================

CREATE TABLE rm_organisations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    max_participants_per_meeting INT NOT NULL DEFAULT 100,
    data_residency_region TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,                       -- last processed event sequence
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- READ MODEL: USERS
-- ============================================================

CREATE TABLE rm_users (
    id              UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    locale          TEXT NOT NULL DEFAULT 'en',
    last_login_at   TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_users_email ON rm_users (email);

-- ============================================================
-- READ MODEL: MEETINGS (current state)
-- ============================================================

CREATE TABLE rm_meetings (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    host_user_id    UUID NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    meeting_type    TEXT NOT NULL,
    status          TEXT NOT NULL,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    settings        JSONB NOT NULL DEFAULT '{}',
    participant_count INT NOT NULL DEFAULT 0,
    join_url        TEXT,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_meetings_org ON rm_meetings (organisation_id);
CREATE INDEX idx_rm_meetings_status ON rm_meetings (status) WHERE status IN ('scheduled', 'in_progress');
CREATE INDEX idx_rm_meetings_host ON rm_meetings (host_user_id);

-- ============================================================
-- READ MODEL: ACTIVE PARTICIPANTS (current meeting state)
-- ============================================================

CREATE TABLE rm_meeting_participants (
    id              UUID PRIMARY KEY,
    meeting_id      UUID NOT NULL,
    user_id         UUID,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    join_time       TIMESTAMPTZ,
    leave_time      TIMESTAMPTZ,
    device_type     TEXT,
    connection_quality TEXT,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_participants_meeting ON rm_meeting_participants (meeting_id);
CREATE INDEX idx_rm_participants_active ON rm_meeting_participants (meeting_id) WHERE is_active = TRUE;

-- ============================================================
-- READ MODEL: RECORDINGS
-- ============================================================

CREATE TABLE rm_recordings (
    id              UUID PRIMARY KEY,
    meeting_id      UUID NOT NULL,
    recording_type  TEXT NOT NULL,
    storage_path    TEXT NOT NULL,
    file_size_bytes BIGINT,
    duration_seconds INT,
    mime_type       TEXT NOT NULL,
    status          TEXT NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_recordings_meeting ON rm_recordings (meeting_id);

-- ============================================================
-- READ MODEL: TRANSCRIPTS WITH SEGMENTS
-- ============================================================

CREATE TABLE rm_transcripts (
    id              UUID PRIMARY KEY,
    meeting_id      UUID NOT NULL,
    language        TEXT NOT NULL,
    status          TEXT NOT NULL,
    engine          TEXT NOT NULL,
    word_count      INT,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_transcripts_meeting ON rm_transcripts (meeting_id);

CREATE TABLE rm_transcript_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL,
    speaker_user_id UUID,
    speaker_label   TEXT,
    segment_index   INT NOT NULL,
    start_time_ms   INT NOT NULL,
    end_time_ms     INT NOT NULL,
    text            TEXT NOT NULL,
    confidence      NUMERIC(4,3)
);
CREATE INDEX idx_rm_segments_transcript ON rm_transcript_segments (transcript_id, segment_index);
CREATE INDEX idx_rm_segments_fts ON rm_transcript_segments
    USING gin(to_tsvector('english', text));

-- ============================================================
-- READ MODEL: ACTION ITEMS
-- ============================================================

CREATE TABLE rm_action_items (
    id              UUID PRIMARY KEY,
    meeting_id      UUID NOT NULL,
    assigned_to     UUID,
    title           TEXT NOT NULL,
    description     TEXT,
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'open',
    source_text     TEXT,                                  -- transcript excerpt
    source_time_ms  INT,                                   -- time in recording
    external_task_id TEXT,
    external_provider TEXT,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_action_items_meeting ON rm_action_items (meeting_id);
CREATE INDEX idx_rm_action_items_assigned ON rm_action_items (assigned_to) WHERE status = 'open';

-- ============================================================
-- READ MODEL: MEETING SUMMARIES
-- ============================================================

CREATE TABLE rm_meeting_summaries (
    id              UUID PRIMARY KEY,
    meeting_id      UUID NOT NULL,
    summary_type    TEXT NOT NULL,
    content         TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    generated_at    TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_summaries_meeting ON rm_meeting_summaries (meeting_id);
```

## Projection & Snapshot Infrastructure

```sql
-- ============================================================
-- PROJECTION CHECKPOINTS
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,                       -- e.g. 'rm_meetings', 'rm_recordings'
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'running',        -- running, paused, failed
    error_message   TEXT
);

-- ============================================================
-- AGGREGATE SNAPSHOTS (optimisation: avoid full replay)
-- ============================================================

CREATE TABLE aggregate_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version BIGINT NOT NULL,                       -- event sequence at time of snapshot
    state           JSONB NOT NULL,                         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Keep only latest snapshot per stream
CREATE INDEX idx_snapshots_latest ON aggregate_snapshots (stream_id, snapshot_version DESC);

-- ============================================================
-- CRYPTO SHREDDING (GDPR right-to-erasure)
-- ============================================================

CREATE TABLE encryption_keys (
    subject_id      UUID PRIMARY KEY,                       -- user_id whose data is encrypted
    subject_type    TEXT NOT NULL DEFAULT 'user',
    encryption_key  BYTEA NOT NULL,                         -- AES-256 key, encrypted with master key
    key_version     INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,          -- FALSE = shredded (key destroyed)
    shredded_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- When a user exercises GDPR right-to-erasure:
-- UPDATE encryption_keys SET encryption_key = NULL, is_active = FALSE, shredded_at = now()
-- WHERE subject_id = '<user_id>';
-- All events containing that user's PII become unreadable.
```

## Example Queries

```sql
-- Who was in a meeting at a specific time?
SELECT payload->>'display_name' AS participant,
       payload->>'role' AS role,
       occurred_at
FROM events
WHERE stream_id = '<meeting_id>'
  AND event_type = 'participant.joined'
  AND occurred_at <= '2026-05-20 14:32:00+00'
  AND NOT EXISTS (
      SELECT 1 FROM events e2
      WHERE e2.stream_id = events.stream_id
        AND e2.event_type = 'participant.left'
        AND e2.payload->>'user_id' = events.payload->>'user_id'
        AND e2.occurred_at <= '2026-05-20 14:32:00+00'
        AND e2.occurred_at > events.occurred_at
  );

-- Full timeline of a meeting (all events in order)
SELECT event_type, occurred_at, actor_id, payload
FROM events
WHERE stream_id = '<meeting_id>'
ORDER BY sequence_number;

-- Rebuild meeting state from events (application-layer pseudocode)
-- state = {}
-- for event in events.where(stream_id=meeting_id).order_by(sequence_number):
--     state = apply(state, event)
-- return state
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write model) | 2 | events, event_type_registry |
| Read Model: Identity | 2 | rm_organisations, rm_users |
| Read Model: Meetings | 2 | rm_meetings, rm_meeting_participants |
| Read Model: Media | 1 | rm_recordings |
| Read Model: AI/Transcription | 4 | rm_transcripts, rm_transcript_segments, rm_action_items, rm_meeting_summaries |
| Infrastructure | 3 | projection_checkpoints, aggregate_snapshots, encryption_keys |
| **Total** | **~14** | Plus the event store which replaces ~12 tables from a CRUD model |

---

## Key Design Decisions

1. **Single `events` table as source of truth** — all state changes flow through one append-only table. This eliminates the need for a separate audit_logs table and guarantees complete traceability.

2. **Event type registry with JSON Schema** — every event type has a registered schema, enabling payload validation at write time and safe schema evolution via versioning.

3. **Read models are disposable and rebuildable** — every `rm_*` table can be dropped and reconstructed by replaying events from the store. This makes schema changes to read models risk-free.

4. **`last_event_seq` on every read model row** — tracks which event last updated this row, enabling idempotent replay and debugging stale data.

5. **Crypto-shredding for GDPR erasure** — rather than deleting events (which would break the immutable log), personal data in event payloads is encrypted with a per-user key. Destroying the key renders the data unreadable while preserving the event sequence for non-PII analytics.

6. **Aggregate snapshots to avoid full replay** — for meetings with hundreds of events, snapshots capture the aggregate state at periodic intervals so rebuilds only need to replay events since the last snapshot.

7. **Monthly partitioning on the events table** — at scale, the events table will grow very large. Partitioning by month keeps queries fast and enables efficient archival of old partitions to cold storage.

8. **Event payloads carry full context** — each event payload includes all data needed to understand the state change without loading external tables. This makes event replay self-contained and enables offline processing.

9. **Projection checkpoints for reliable processing** — each read model projection tracks its position in the event stream, enabling crash recovery without missing or double-processing events.

10. **Natural WebSocket event streaming** — the same events written to the store can be simultaneously broadcast to connected clients via WebSocket, ensuring the UI stays in sync with the event stream without a separate notification system.
