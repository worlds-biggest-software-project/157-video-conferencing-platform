# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Video Conferencing Platform · Created: 2026-05-19

## Philosophy

This model follows classical normalized relational design (3NF), giving every domain concept its own table with explicit foreign key relationships. Every meeting, participant, recording, transcript segment, and AI artefact occupies a distinct table, enabling strong referential integrity and precise queries across any dimension.

Normalized relational models are the workhorse of enterprise SaaS platforms. Zoom's management API exposes discrete resource types (meetings, recordings, participants, transcripts) that map naturally to separate tables. LiveKit's conceptual model of Rooms, Participants, and Tracks also reflects this entity-per-concept philosophy. The approach aligns well with PostgreSQL's strengths: rich indexing, transactional integrity, and mature tooling.

This architecture is best when data integrity is paramount, when complex cross-entity reporting is needed (e.g. "which users attended the most meetings last quarter that produced action items?"), and when the domain is well-understood and unlikely to change shape radically after launch.

**Best for:** Teams that prioritise query flexibility, reporting, and long-term schema stability over rapid iteration on variable data structures.

**Trade-offs:**
- (+) Strong referential integrity; impossible to create orphaned records
- (+) Excellent for complex JOIN-based analytics and reporting
- (+) Well-understood by most backend engineers; rich ORM support
- (+) Clear migration path as requirements evolve
- (-) Higher table count increases schema complexity
- (-) Schema migrations required for new fields; slower iteration than JSONB approaches
- (-) Many-to-many junction tables add write overhead
- (-) Deeply nested queries (e.g. org -> workspace -> meeting -> recording -> transcript -> segment) require multi-table JOINs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebRTC (W3C/IETF RFC 8825) | Room/Participant/Track conceptual model informs the `meetings`, `meeting_participants`, and `media_tracks` tables |
| OAuth 2.0 (RFC 6749) | `oauth_connections` table stores third-party integration tokens per user |
| ISO/IEC 27001:2022 | Audit trail tables (`audit_logs`) satisfy A.8.3 access restriction logging requirements |
| GDPR Articles 6, 9, 32 | `consent_records` table tracks explicit opt-in for recording and AI features; `data_retention_policies` enforce configurable retention |
| WCAG 2.2 | `captions` and `transcript_segments` tables support accessible meeting content |
| ISO 3166-1 | `jurisdictions` reference table for data residency and compliance scoping |
| HIPAA §164.312 | `audit_logs` with immutable append-only design satisfies technical safeguard requirements |
| SIP/SDP (RFC 3261/4566) | `sip_gateway_sessions` table models legacy telephony bridge connections |

---

## Identity & Multi-Tenancy

```sql
-- ============================================================
-- ORGANISATIONS AND USERS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',          -- free, pro, business, enterprise
    max_participants_per_meeting INT NOT NULL DEFAULT 100,
    data_residency_region TEXT,                            -- ISO 3166-1 alpha-2 (e.g. 'US', 'EU', 'AU')
    settings        JSONB NOT NULL DEFAULT '{}',          -- org-level feature flags
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisations_slug ON organisations (slug);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,                                  -- NULL if SSO-only
    auth_provider   TEXT NOT NULL DEFAULT 'local',         -- local, google, microsoft, saml
    auth_provider_id TEXT,                                 -- external identity provider ID
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    locale          TEXT NOT NULL DEFAULT 'en',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_auth_provider ON users (auth_provider, auth_provider_id);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',         -- owner, admin, member, guest
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
CREATE INDEX idx_org_members_org ON organisation_members (organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members (user_id);

-- ============================================================
-- ROLES AND PERMISSIONS (RBAC)
-- ============================================================

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,                          -- e.g. 'meeting_host', 'recorder', 'viewer'
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,        -- system roles cannot be deleted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_roles_org_name ON roles (organisation_id, name);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,                   -- e.g. 'meeting.create', 'recording.download'
    description     TEXT NOT NULL
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    organisation_member_id UUID NOT NULL REFERENCES organisation_members(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (organisation_member_id, role_id)
);
```

## Meetings & Scheduling

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
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    password        TEXT,                                   -- meeting access password (hashed)
    waiting_room_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    recording_enabled    BOOLEAN NOT NULL DEFAULT TRUE,
    transcription_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    ai_summary_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    max_participants     INT,
    join_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_org ON meetings (organisation_id);
CREATE INDEX idx_meetings_host ON meetings (host_user_id);
CREATE INDEX idx_meetings_status ON meetings (status) WHERE status IN ('scheduled', 'in_progress');
CREATE INDEX idx_meetings_scheduled ON meetings (scheduled_start) WHERE status = 'scheduled';

CREATE TABLE meeting_recurrence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    rrule           TEXT NOT NULL,                          -- iCal RRULE format (RFC 5545)
    recurrence_end  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- MEETING PARTICIPANTS
-- ============================================================

CREATE TABLE meeting_participants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),             -- NULL for anonymous/external guests
    display_name    TEXT NOT NULL,
    email           TEXT,
    role            TEXT NOT NULL DEFAULT 'attendee',       -- host, co_host, attendee, presenter, interpreter
    join_time       TIMESTAMPTZ,
    leave_time      TIMESTAMPTZ,
    duration_seconds INT,
    ip_address      INET,
    user_agent      TEXT,
    connection_quality TEXT,                                -- good, fair, poor
    device_type     TEXT,                                   -- desktop, mobile, tablet, sip_phone
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_participants_meeting ON meeting_participants (meeting_id);
CREATE INDEX idx_participants_user ON meeting_participants (user_id);

-- ============================================================
-- CALENDAR INTEGRATIONS
-- ============================================================

CREATE TABLE calendar_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    provider        TEXT NOT NULL,                          -- google, outlook, ical
    external_event_id TEXT NOT NULL,
    sync_status     TEXT NOT NULL DEFAULT 'synced',         -- synced, pending, error
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_calendar_events_meeting ON calendar_events (meeting_id);
```

## Media, Recordings & Tracks

```sql
-- ============================================================
-- MEDIA TRACKS (WebRTC model: Room -> Participants -> Tracks)
-- ============================================================

CREATE TABLE media_tracks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    participant_id  UUID NOT NULL REFERENCES meeting_participants(id) ON DELETE CASCADE,
    track_type      TEXT NOT NULL,                          -- audio, video, screen_share, data
    codec           TEXT,                                   -- opus, vp8, vp9, h264, av1
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    muted           BOOLEAN NOT NULL DEFAULT FALSE,
    simulcast_layers INT DEFAULT 1,                        -- number of simulcast quality layers
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_media_tracks_meeting ON media_tracks (meeting_id);
CREATE INDEX idx_media_tracks_participant ON media_tracks (participant_id);

-- ============================================================
-- RECORDINGS
-- ============================================================

CREATE TABLE recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_type  TEXT NOT NULL,                          -- composite, speaker_view, gallery_view,
                                                           -- screen_share, audio_only
    storage_provider TEXT NOT NULL DEFAULT 'cloud',         -- cloud, local, s3
    storage_path    TEXT NOT NULL,
    file_size_bytes BIGINT,
    duration_seconds INT,
    mime_type       TEXT NOT NULL,                          -- video/mp4, audio/webm, etc.
    status          TEXT NOT NULL DEFAULT 'processing',     -- processing, ready, failed, deleted
    auto_delete_at  TIMESTAMPTZ,                           -- retention policy enforcement
    encryption_key_id TEXT,                                 -- reference to key management service
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recordings_meeting ON recordings (meeting_id);
CREATE INDEX idx_recordings_status ON recordings (status);
CREATE INDEX idx_recordings_auto_delete ON recordings (auto_delete_at) WHERE auto_delete_at IS NOT NULL;
```

## Transcription & AI Intelligence

```sql
-- ============================================================
-- TRANSCRIPTS
-- ============================================================

CREATE TABLE transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_id    UUID REFERENCES recordings(id) ON DELETE SET NULL,
    language        TEXT NOT NULL DEFAULT 'en',             -- ISO 639-1 language code
    status          TEXT NOT NULL DEFAULT 'processing',     -- processing, ready, failed
    engine          TEXT NOT NULL DEFAULT 'whisper',        -- whisper, deepgram, assemblyai, custom
    word_count      INT,
    confidence_avg  NUMERIC(4,3),                          -- average confidence score 0.000-1.000
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transcripts_meeting ON transcripts (meeting_id);

CREATE TABLE transcript_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    participant_id  UUID REFERENCES meeting_participants(id),  -- speaker diarisation link
    speaker_label   TEXT,                                      -- fallback label if no participant match
    segment_index   INT NOT NULL,                              -- ordering within transcript
    start_time_ms   INT NOT NULL,                              -- milliseconds from meeting start
    end_time_ms     INT NOT NULL,
    text            TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    language        TEXT,                                       -- per-segment language if multilingual
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transcript_segments_transcript ON transcript_segments (transcript_id);
CREATE INDEX idx_transcript_segments_participant ON transcript_segments (participant_id);
CREATE INDEX idx_transcript_segments_time ON transcript_segments (transcript_id, start_time_ms);

-- Full-text search on transcript content
CREATE INDEX idx_transcript_segments_fts ON transcript_segments
    USING gin(to_tsvector('english', text));

-- ============================================================
-- AI MEETING SUMMARIES
-- ============================================================

CREATE TABLE meeting_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    transcript_id   UUID REFERENCES transcripts(id),
    summary_type    TEXT NOT NULL DEFAULT 'executive',      -- executive, detailed, decision_log
    content         TEXT NOT NULL,
    model_id        TEXT NOT NULL,                          -- AI model used (e.g. 'claude-opus-4-6')
    model_version   TEXT,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meeting_summaries_meeting ON meeting_summaries (meeting_id);

-- ============================================================
-- ACTION ITEMS
-- ============================================================

CREATE TABLE action_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    summary_id      UUID REFERENCES meeting_summaries(id),
    assigned_to     UUID REFERENCES users(id),
    title           TEXT NOT NULL,
    description     TEXT,
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'open',           -- open, in_progress, completed, dismissed
    source_segment_id UUID REFERENCES transcript_segments(id),  -- link to transcript moment
    external_task_id TEXT,                                  -- synced task ID in Jira/Asana/etc.
    external_provider TEXT,                                 -- jira, asana, linear, todoist
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_action_items_meeting ON action_items (meeting_id);
CREATE INDEX idx_action_items_assigned ON action_items (assigned_to) WHERE status != 'completed';

-- ============================================================
-- CAPTIONS (live captions stored for accessibility)
-- ============================================================

CREATE TABLE captions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    participant_id  UUID REFERENCES meeting_participants(id),
    language        TEXT NOT NULL DEFAULT 'en',
    text            TEXT NOT NULL,
    start_time_ms   INT NOT NULL,
    end_time_ms     INT NOT NULL,
    is_final        BOOLEAN NOT NULL DEFAULT TRUE,          -- false for interim captions
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_captions_meeting ON captions (meeting_id, start_time_ms);
```

## Chat, Breakout Rooms & Collaboration

```sql
-- ============================================================
-- IN-MEETING CHAT
-- ============================================================

CREATE TABLE meeting_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES meeting_participants(id),
    recipient_id    UUID REFERENCES meeting_participants(id),  -- NULL = broadcast to all
    message_type    TEXT NOT NULL DEFAULT 'text',               -- text, file, reaction
    content         TEXT NOT NULL,
    file_url        TEXT,
    file_name       TEXT,
    file_size_bytes BIGINT,
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meeting_messages_meeting ON meeting_messages (meeting_id, sent_at);

-- ============================================================
-- BREAKOUT ROOMS
-- ============================================================

CREATE TABLE breakout_rooms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    room_index      INT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',        -- pending, active, closed
    opened_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_breakout_rooms_meeting ON breakout_rooms (meeting_id);

CREATE TABLE breakout_room_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    breakout_room_id UUID NOT NULL REFERENCES breakout_rooms(id) ON DELETE CASCADE,
    participant_id  UUID NOT NULL REFERENCES meeting_participants(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (breakout_room_id, participant_id)
);
```

## Integrations, Webhooks & Audit

```sql
-- ============================================================
-- OAUTH CONNECTIONS (third-party integrations)
-- ============================================================

CREATE TABLE oauth_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,                          -- google, microsoft, slack, jira, salesforce
    access_token    TEXT NOT NULL,                          -- encrypted at rest
    refresh_token   TEXT,                                   -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_oauth_connections_user_provider ON oauth_connections (user_id, provider);

-- ============================================================
-- WEBHOOKS
-- ============================================================

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret          TEXT NOT NULL,                          -- HMAC signing secret
    events          TEXT[] NOT NULL,                        -- e.g. {'meeting.started', 'recording.ready'}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_webhooks_org ON webhooks (organisation_id);

-- ============================================================
-- CONSENT RECORDS (GDPR / HIPAA compliance)
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    meeting_id      UUID REFERENCES meetings(id),
    consent_type    TEXT NOT NULL,                          -- recording, transcription, ai_summary, biometric_voice
    granted         BOOLEAN NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);
CREATE INDEX idx_consent_records_user ON consent_records (user_id);
CREATE INDEX idx_consent_records_meeting ON consent_records (meeting_id);

-- ============================================================
-- DATA RETENTION POLICIES
-- ============================================================

CREATE TABLE data_retention_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    resource_type   TEXT NOT NULL,                          -- recording, transcript, chat_message, meeting
    retention_days  INT NOT NULL,
    auto_delete     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, resource_type)
);

-- ============================================================
-- AUDIT LOGS (append-only)
-- ============================================================

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,                          -- meeting.created, recording.downloaded, user.invited
    resource_type   TEXT NOT NULL,                          -- meeting, recording, transcript, user
    resource_id     UUID NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    metadata        JSONB,                                 -- additional context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_logs_org ON audit_logs (organisation_id, created_at DESC);
CREATE INDEX idx_audit_logs_user ON audit_logs (user_id, created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs (resource_type, resource_id);

-- Partition audit logs by month for performance
-- CREATE TABLE audit_logs_2026_05 PARTITION OF audit_logs
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

## Meeting Analytics

```sql
-- ============================================================
-- MEETING ANALYTICS (pre-aggregated for dashboards)
-- ============================================================

CREATE TABLE meeting_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE UNIQUE,
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    total_participants INT NOT NULL DEFAULT 0,
    total_duration_seconds INT,
    avg_participant_duration_seconds INT,
    total_messages  INT NOT NULL DEFAULT 0,
    total_reactions INT NOT NULL DEFAULT 0,
    recording_count INT NOT NULL DEFAULT 0,
    transcript_word_count INT,
    action_items_count INT NOT NULL DEFAULT 0,
    sentiment_score NUMERIC(4,3),                          -- AI-derived overall meeting sentiment
    participation_equity_score NUMERIC(4,3),                -- how evenly distributed was speaking time
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meeting_analytics_org ON meeting_analytics (organisation_id, computed_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 6 | organisations, users, organisation_members, roles, permissions, role_permissions, user_roles |
| Meetings & Scheduling | 4 | meetings, meeting_recurrence, meeting_participants, calendar_events |
| Media & Recordings | 2 | media_tracks, recordings |
| Transcription & AI | 4 | transcripts, transcript_segments, meeting_summaries, action_items |
| Captions | 1 | captions |
| Chat & Collaboration | 3 | meeting_messages, breakout_rooms, breakout_room_assignments |
| Integrations | 2 | oauth_connections, webhooks |
| Compliance & Audit | 3 | consent_records, data_retention_policies, audit_logs |
| Analytics | 1 | meeting_analytics |
| **Total** | **~26** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployments, and prevents enumeration attacks on meeting or recording URLs.

2. **Separate `meeting_participants` from `users`** — a participant record captures per-meeting context (join time, role, device type) while the user record is the persistent identity. Anonymous/external guests have no user record.

3. **Transcript segments with speaker diarisation links** — each segment references a `meeting_participants` row, enabling queries like "show everything Alice said in this meeting" via a simple JOIN.

4. **Full-text search index on transcript content** — `to_tsvector` GIN index enables searching across all meeting transcripts within an organisation without a separate search engine at MVP scale.

5. **iCal RRULE for recurrence** — stores recurring meeting patterns in the standard RFC 5545 format, directly compatible with Google Calendar, Outlook, and iCal integrations.

6. **Append-only audit logs with partitioning** — audit_logs are never updated or deleted; monthly partitioning keeps query performance manageable as volume grows. Satisfies ISO 27001 and HIPAA audit requirements.

7. **Consent records are per-meeting** — GDPR Article 9 biometric consent (for speaker voiceprints) is tracked per meeting, not as a blanket org-level setting, reflecting 2026 EDPB guidance.

8. **Action items link back to transcript segments** — when an AI extracts an action item, the source segment is preserved, enabling users to "jump to the moment" in the recording where the action was discussed.

9. **Analytics table is pre-aggregated** — rather than computing participation metrics on every dashboard load, a background job populates `meeting_analytics` after each meeting ends, keeping read queries fast.

10. **Media tracks modelled separately from recordings** — real-time tracks (WebRTC publish/subscribe) are distinct from stored recordings. A meeting may have many tracks but zero recordings if recording was disabled.
