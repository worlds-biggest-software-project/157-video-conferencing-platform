# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Video Conferencing Platform · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core entities (organisations, users, meetings, recordings) with JSONB columns for variable, extensible, and jurisdiction-specific data. The stable, well-understood parts of the domain get proper columns with types and constraints; the parts that vary by deployment, customer, or feature tier live in typed JSONB fields.

Video conferencing platforms serve wildly different contexts: a healthcare telehealth session requires HIPAA consent tracking and patient identifiers; an education classroom needs attendance rosters and grade-level metadata; a sales call needs CRM opportunity linking. Rather than adding nullable columns for every vertical, a JSONB `metadata` or `settings` field absorbs this variation while the relational core remains clean and queryable.

This pattern is proven at scale by platforms like Stripe (relational core with `metadata` JSONB on every resource), Zoom (whose API returns both fixed fields and variable `settings` objects), and Slack (where channel properties mix fixed schema with extensible metadata). PostgreSQL's JSONB support — including GIN indexing, containment operators, and jsonpath queries — makes this approach practical without sacrificing query performance.

**Best for:** Rapid MVP development, multi-vertical deployments (healthcare, education, enterprise), and teams that want relational safety for core operations with JSONB flexibility for feature experimentation.

**Trade-offs:**
- (+) Fewer tables than fully normalised — faster to build and iterate
- (+) New per-vertical or per-tenant fields can be added without schema migrations
- (+) JSONB GIN indexes enable efficient queries on variable fields
- (+) Natural fit for REST API responses that mix fixed and dynamic fields
- (+) Easy to prototype features in JSONB and later promote to columns if they stabilise
- (-) JSONB fields lack column-level type enforcement; validation must happen in application code or CHECK constraints
- (-) Complex JSONB queries are less readable than column-based WHERE clauses
- (-) Reporting tools and BI connectors may struggle with nested JSONB structures
- (-) Risk of "JSONB dump" anti-pattern if discipline is not maintained about what belongs in columns vs. JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebRTC (W3C/IETF RFC 8825) | Core meeting/participant/track model in relational columns; SDP negotiation details in JSONB |
| OAuth 2.0 (RFC 6749) | Integration tokens in relational columns; provider-specific scopes and metadata in JSONB |
| ISO/IEC 27001:2022 | Audit logs table with JSONB `details` for flexible event context |
| GDPR Articles 6, 9, 32 | `consent_details` JSONB captures jurisdiction-specific consent requirements |
| HIPAA §164.312 | Healthcare-specific meeting metadata stored in `meeting.metadata` JSONB without polluting the core schema |
| ISO 3166-1 | `jurisdiction` column on organisations; jurisdiction-specific rules in JSONB |
| RFC 5545 (iCal) | Recurrence rules in `meetings.recurrence` JSONB field |
| EU AI Act (2024/2847) | AI feature transparency metadata in `ai_artefacts.metadata` JSONB |

---

## Core Identity & Tenancy

```sql
-- ============================================================
-- ORGANISATIONS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    jurisdiction    TEXT NOT NULL DEFAULT 'US',             -- ISO 3166-1 alpha-2
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "max_participants": 100,
    --   "recording_default": true,
    --   "transcription_default": true,
    --   "ai_features_enabled": true,
    --   "data_residency": "eu-west-1",
    --   "allowed_auth_providers": ["google", "saml"],
    --   "branding": {
    --     "logo_url": "https://...",
    --     "primary_color": "#1a73e8",
    --     "custom_domain": "meet.acme.com"
    --   },
    --   "compliance": {
    --     "hipaa_enabled": true,
    --     "baa_signed_at": "2026-03-15",
    --     "retention_days": 365
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisations_slug ON organisations (slug);
CREATE INDEX idx_organisations_plan ON organisations (plan);
-- Query orgs with specific settings
CREATE INDEX idx_organisations_settings ON organisations USING gin(settings);

-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_provider_id TEXT,
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "timezone": "America/New_York",
    --   "locale": "en-US",
    --   "notifications": {
    --     "email_summary": true,
    --     "action_item_reminders": true,
    --     "recording_ready": true
    --   },
    --   "meeting_defaults": {
    --     "camera_on": false,
    --     "mic_on": true,
    --     "virtual_background": "blur"
    --   }
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_auth ON users (auth_provider, auth_provider_id);

-- ============================================================
-- ORGANISATION MEMBERSHIPS (with RBAC in JSONB)
-- ============================================================

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',         -- owner, admin, member, guest
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example (additive overrides beyond role defaults):
    -- ["meeting.create", "recording.download", "transcript.export", "billing.view"]
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
CREATE INDEX idx_org_members_org ON organisation_members (organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members (user_id);
CREATE INDEX idx_org_members_permissions ON organisation_members USING gin(permissions);
```

## Meetings & Participants

```sql
-- ============================================================
-- MEETINGS
-- ============================================================

CREATE TABLE meetings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    host_user_id    UUID NOT NULL REFERENCES users(id),
    title           TEXT NOT NULL,
    description     TEXT,
    meeting_type    TEXT NOT NULL DEFAULT 'instant',        -- instant, scheduled, recurring, webinar
    status          TEXT NOT NULL DEFAULT 'scheduled',      -- scheduled, in_progress, ended, cancelled
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    join_url        TEXT,
    password_hash   TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "waiting_room": true,
    --   "recording_enabled": true,
    --   "transcription_enabled": true,
    --   "ai_summary_enabled": true,
    --   "max_participants": 50,
    --   "e2ee_enabled": false,
    --   "breakout_rooms": {
    --     "auto_assign": false,
    --     "allow_self_select": true
    --   }
    -- }
    recurrence      JSONB,
    -- recurrence example (RFC 5545 compatible):
    -- {
    --   "rrule": "FREQ=WEEKLY;BYDAY=TU,TH;UNTIL=20260630T000000Z",
    --   "exceptions": ["2026-06-02"],
    --   "timezone": "America/New_York"
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example (vertical-specific):
    -- Healthcare: {"patient_id": "P-12345", "encounter_type": "telehealth", "hipaa_consent": true}
    -- Education:  {"course_id": "CS101", "section": "A", "lecture_number": 14}
    -- Sales:      {"crm_opportunity_id": "OPP-789", "deal_stage": "negotiation"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_org ON meetings (organisation_id);
CREATE INDEX idx_meetings_host ON meetings (host_user_id);
CREATE INDEX idx_meetings_status ON meetings (status) WHERE status IN ('scheduled', 'in_progress');
CREATE INDEX idx_meetings_scheduled ON meetings (scheduled_start) WHERE status = 'scheduled';
CREATE INDEX idx_meetings_settings ON meetings USING gin(settings);
CREATE INDEX idx_meetings_metadata ON meetings USING gin(metadata);

-- ============================================================
-- MEETING PARTICIPANTS
-- ============================================================

CREATE TABLE meeting_participants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    display_name    TEXT NOT NULL,
    email           TEXT,
    role            TEXT NOT NULL DEFAULT 'attendee',       -- host, co_host, attendee, presenter, interpreter
    join_time       TIMESTAMPTZ,
    leave_time      TIMESTAMPTZ,
    duration_seconds INT,
    connection_info JSONB NOT NULL DEFAULT '{}',
    -- connection_info example:
    -- {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0 ...",
    --   "device_type": "desktop",
    --   "browser": "chrome",
    --   "os": "macos",
    --   "connection_quality": "good",
    --   "bandwidth_kbps": 2500,
    --   "codec_audio": "opus",
    --   "codec_video": "vp9",
    --   "joined_via": "browser"       -- browser, mobile_app, sip, pstn
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_participants_meeting ON meeting_participants (meeting_id);
CREATE INDEX idx_participants_user ON meeting_participants (user_id);
```

## Recordings & Media

```sql
-- ============================================================
-- RECORDINGS
-- ============================================================

CREATE TABLE recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_type  TEXT NOT NULL,                          -- composite, speaker_view, gallery_view,
                                                           -- screen_share, audio_only
    storage_provider TEXT NOT NULL DEFAULT 's3',
    storage_path    TEXT NOT NULL,
    file_size_bytes BIGINT,
    duration_seconds INT,
    mime_type       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'processing',     -- processing, ready, failed, deleted
    processing_info JSONB NOT NULL DEFAULT '{}',
    -- processing_info example:
    -- {
    --   "encoder": "ffmpeg",
    --   "resolution": "1920x1080",
    --   "bitrate_kbps": 4000,
    --   "fps": 30,
    --   "chapters": [
    --     {"title": "Introduction", "start_ms": 0, "end_ms": 180000},
    --     {"title": "Q2 Review", "start_ms": 180000, "end_ms": 900000}
    --   ],
    --   "thumbnail_url": "https://..."
    -- }
    retention_policy JSONB,
    -- retention_policy example:
    -- {"auto_delete_at": "2027-05-20", "reason": "org_policy", "policy_id": "..."}
    encryption_key_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recordings_meeting ON recordings (meeting_id);
CREATE INDEX idx_recordings_status ON recordings (status);
```

## Transcription & AI Artefacts

```sql
-- ============================================================
-- TRANSCRIPTS
-- ============================================================

CREATE TABLE transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_id    UUID REFERENCES recordings(id) ON DELETE SET NULL,
    language        TEXT NOT NULL DEFAULT 'en',
    status          TEXT NOT NULL DEFAULT 'processing',
    engine          TEXT NOT NULL DEFAULT 'whisper',
    engine_config   JSONB NOT NULL DEFAULT '{}',
    -- engine_config example:
    -- {
    --   "model": "whisper-large-v3",
    --   "language_detection": true,
    --   "diarization_enabled": true,
    --   "diarization_model": "pyannote-3.1",
    --   "vocabulary_boost": ["WebRTC", "SFU", "SRTP"]
    -- }
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example:
    -- {"word_count": 4521, "segment_count": 187, "confidence_avg": 0.943,
    --  "speaker_count": 5, "languages_detected": ["en", "es"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transcripts_meeting ON transcripts (meeting_id);

-- ============================================================
-- TRANSCRIPT SEGMENTS
-- ============================================================

CREATE TABLE transcript_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    participant_id  UUID REFERENCES meeting_participants(id),
    speaker_label   TEXT,
    segment_index   INT NOT NULL,
    start_time_ms   INT NOT NULL,
    end_time_ms     INT NOT NULL,
    text            TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    language        TEXT,
    word_timings    JSONB,
    -- word_timings example (for precise highlighting during playback):
    -- [
    --   {"word": "Let's", "start": 1823400, "end": 1823600, "confidence": 0.98},
    --   {"word": "review", "start": 1823600, "end": 1823900, "confidence": 0.96},
    --   {"word": "the", "start": 1823900, "end": 1824000, "confidence": 0.99},
    --   {"word": "budget", "start": 1824000, "end": 1824400, "confidence": 0.97}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segments_transcript ON transcript_segments (transcript_id, segment_index);
CREATE INDEX idx_segments_participant ON transcript_segments (participant_id);
CREATE INDEX idx_segments_fts ON transcript_segments USING gin(to_tsvector('english', text));

-- ============================================================
-- AI ARTEFACTS (unified table for all AI-generated content)
-- ============================================================

CREATE TABLE ai_artefacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    artefact_type   TEXT NOT NULL,                          -- summary, action_item, decision, topic, sentiment
    status          TEXT NOT NULL DEFAULT 'ready',          -- processing, ready, failed, dismissed
    content         TEXT NOT NULL,                          -- primary text content
    assigned_to     UUID REFERENCES users(id),             -- for action_items
    due_date        DATE,                                   -- for action_items
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata varies by artefact_type:
    --
    -- summary:
    -- {
    --   "summary_type": "executive",
    --   "model_id": "claude-opus-4-6",
    --   "model_version": "2026-05-01",
    --   "key_decisions": ["Approved Q3 budget", "Hired 2 engineers"],
    --   "topics": ["budget", "hiring", "product roadmap"]
    -- }
    --
    -- action_item:
    -- {
    --   "source_segment_id": "...",
    --   "source_time_ms": 1823400,
    --   "source_text": "Alice, can you send the updated budget by Friday?",
    --   "confidence": 0.94,
    --   "external_sync": {
    --     "provider": "jira",
    --     "task_id": "PROJ-456",
    --     "synced_at": "2026-05-20T15:30:00Z"
    --   }
    -- }
    --
    -- sentiment:
    -- {
    --   "overall_score": 0.72,
    --   "engagement_score": 0.81,
    --   "participation_equity": 0.65,
    --   "per_speaker": [
    --     {"user_id": "...", "speaking_time_pct": 0.35, "sentiment": 0.80},
    --     {"user_id": "...", "speaking_time_pct": 0.28, "sentiment": 0.65}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_artefacts_meeting ON ai_artefacts (meeting_id);
CREATE INDEX idx_ai_artefacts_type ON ai_artefacts (artefact_type);
CREATE INDEX idx_ai_artefacts_assigned ON ai_artefacts (assigned_to) WHERE artefact_type = 'action_item' AND status = 'ready';
CREATE INDEX idx_ai_artefacts_metadata ON ai_artefacts USING gin(metadata);
```

## Chat, Integrations & Compliance

```sql
-- ============================================================
-- IN-MEETING CHAT
-- ============================================================

CREATE TABLE meeting_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES meeting_participants(id),
    recipient_id    UUID REFERENCES meeting_participants(id),
    message_type    TEXT NOT NULL DEFAULT 'text',
    content         TEXT NOT NULL,
    attachments     JSONB,
    -- attachments example:
    -- [
    --   {"name": "report.pdf", "url": "https://...", "size_bytes": 245000, "mime_type": "application/pdf"},
    --   {"name": "screenshot.png", "url": "https://...", "size_bytes": 89000, "mime_type": "image/png"}
    -- ]
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_meeting ON meeting_messages (meeting_id, sent_at);

-- ============================================================
-- BREAKOUT ROOMS
-- ============================================================

CREATE TABLE breakout_rooms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    room_index      INT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    participant_ids UUID[] NOT NULL DEFAULT '{}',           -- array of meeting_participant IDs
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {"auto_close_after_minutes": 15, "allow_return_to_main": true}
    opened_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_breakout_rooms_meeting ON breakout_rooms (meeting_id);

-- ============================================================
-- INTEGRATIONS (unified table for all third-party connections)
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,  -- user-level or org-level
    provider        TEXT NOT NULL,                          -- google, microsoft, slack, jira, salesforce, zoom_phone
    connection_type TEXT NOT NULL DEFAULT 'oauth',          -- oauth, api_key, webhook
    credentials     JSONB NOT NULL,                         -- encrypted; structure varies by provider
    -- oauth credentials example:
    -- {"access_token": "...", "refresh_token": "...", "expires_at": "...", "scopes": [...]}
    -- webhook credentials example:
    -- {"webhook_url": "https://...", "secret": "...", "events": ["meeting.ended", "recording.ready"]}
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example (Jira integration):
    -- {"project_key": "PROJ", "issue_type": "Task", "auto_sync_action_items": true}
    status          TEXT NOT NULL DEFAULT 'active',         -- active, expired, revoked, error
    last_synced_at  TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_org ON integrations (organisation_id);
CREATE INDEX idx_integrations_user ON integrations (user_id);
CREATE UNIQUE INDEX idx_integrations_unique ON integrations (
    COALESCE(organisation_id, '00000000-0000-0000-0000-000000000000'),
    COALESCE(user_id, '00000000-0000-0000-0000-000000000000'),
    provider
);

-- ============================================================
-- AUDIT LOGS
-- ============================================================

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    actor_id        UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0 ...",
    --   "changes": {"status": {"from": "scheduled", "to": "in_progress"}},
    --   "consent_type": "recording",
    --   "granted": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_time ON audit_logs (organisation_id, created_at DESC);
CREATE INDEX idx_audit_actor ON audit_logs (actor_id, created_at DESC) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_resource ON audit_logs (resource_type, resource_id);
CREATE INDEX idx_audit_details ON audit_logs USING gin(details);

-- ============================================================
-- CONSENT RECORDS
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    meeting_id      UUID REFERENCES meetings(id),
    consent_type    TEXT NOT NULL,                          -- recording, transcription, ai_summary, biometric_voice
    granted         BOOLEAN NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "ip_address": "203.0.113.42",
    --   "jurisdiction": "DE",
    --   "legal_basis": "explicit_consent",
    --   "gdpr_article": "9(2)(a)",
    --   "disclosure_shown": true,
    --   "disclosure_version": "2026-05-01"
    -- }
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);
CREATE INDEX idx_consent_user ON consent_records (user_id);
CREATE INDEX idx_consent_meeting ON consent_records (meeting_id);
```

## Example Queries

```sql
-- Find all healthcare meetings for an organisation
SELECT id, title, scheduled_start, metadata
FROM meetings
WHERE organisation_id = '<org_id>'
  AND metadata @> '{"encounter_type": "telehealth"}'
ORDER BY scheduled_start DESC;

-- Find users who prefer camera off by default
SELECT id, display_name, preferences->'meeting_defaults'->>'virtual_background' AS bg
FROM users
WHERE preferences @> '{"meeting_defaults": {"camera_on": false}}';

-- Find all action items assigned to a user, with external sync status
SELECT ai.content AS title,
       ai.due_date,
       ai.metadata->'external_sync'->>'provider' AS synced_to,
       ai.metadata->'external_sync'->>'task_id' AS external_id,
       m.title AS meeting_title
FROM ai_artefacts ai
JOIN meetings m ON m.id = ai.meeting_id
WHERE ai.artefact_type = 'action_item'
  AND ai.assigned_to = '<user_id>'
  AND ai.status = 'ready'
ORDER BY ai.due_date NULLS LAST;

-- Full-text search across transcript segments within an org
SELECT m.title AS meeting_title,
       ts.text,
       ts.start_time_ms,
       ts.speaker_label
FROM transcript_segments ts
JOIN transcripts t ON t.id = ts.transcript_id
JOIN meetings m ON m.id = t.meeting_id
WHERE m.organisation_id = '<org_id>'
  AND to_tsvector('english', ts.text) @@ plainto_tsquery('english', 'budget approval')
ORDER BY m.created_at DESC, ts.start_time_ms;

-- Organisations with HIPAA compliance enabled
SELECT id, name, settings->'compliance' AS compliance_settings
FROM organisations
WHERE settings @> '{"compliance": {"hipaa_enabled": true}}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Tenancy | 3 | organisations, users, organisation_members (RBAC in JSONB) |
| Meetings & Participants | 2 | meetings, meeting_participants |
| Media & Recordings | 1 | recordings |
| Transcription | 2 | transcripts, transcript_segments |
| AI Artefacts | 1 | ai_artefacts (unified: summaries, action items, decisions, sentiment) |
| Chat & Collaboration | 2 | meeting_messages, breakout_rooms |
| Integrations | 1 | integrations (unified: all providers and connection types) |
| Compliance & Audit | 2 | audit_logs, consent_records |
| **Total** | **~14** | Compared to ~26 in the fully normalised model |

---

## Key Design Decisions

1. **Unified `ai_artefacts` table** — instead of separate tables for summaries, action items, decisions, and sentiment analysis, a single table with `artefact_type` discriminator and type-specific JSONB `metadata`. This dramatically reduces table count and makes adding new AI features (e.g. topic extraction, coaching feedback) a zero-migration operation.

2. **Unified `integrations` table** — rather than separate tables for OAuth connections, webhooks, and API keys, one table handles all integration types. The `credentials` and `config` JSONB fields absorb provider-specific structure.

3. **RBAC permissions as JSONB array** — instead of a four-table roles/permissions/role_permissions/user_roles structure, permissions are stored as a JSONB array on `organisation_members`. The GIN index enables efficient containment queries (`permissions @> '["recording.download"]'`). For most deployments, the role + additive permissions pattern is sufficient.

4. **Meeting `metadata` for vertical-specific fields** — healthcare, education, and sales verticals each need different fields on a meeting. Rather than nullable columns or separate vertical tables, JSONB `metadata` absorbs this variation. GIN indexing makes vertical-specific queries efficient.

5. **`settings` JSONB on organisations and meetings** — feature flags, limits, and configuration that vary by plan tier or customer preference live in JSONB rather than as boolean columns. New settings can be added without migration.

6. **Word-level timings in JSONB** — `transcript_segments.word_timings` stores per-word start/end times for precise playback highlighting. This data is too granular and variable for separate rows but perfectly suited to JSONB.

7. **Recording chapters in JSONB** — AI-generated chapter markers (like Zoom's Smart Chapters) are stored in `recordings.processing_info` JSONB rather than a separate chapters table. The list is small and always read together with the recording.

8. **Breakout room assignments as UUID array** — instead of a junction table, `breakout_rooms.participant_ids` uses a PostgreSQL UUID array. This is simpler for the common operations (assign all, read all) and the array is always small.

9. **Attachment data in JSONB** — chat message attachments are stored as a JSONB array on `meeting_messages` rather than a separate attachments table. Each message has at most a few attachments, making a full table unnecessary.

10. **Consent `details` captures jurisdiction-specific compliance** — different jurisdictions require different consent metadata (GDPR Article 9 for EU biometric data, HIPAA for US healthcare). The JSONB `details` field absorbs this variation while the core consent record (type, granted, timestamp) remains relational.
