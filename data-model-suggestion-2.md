# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: AI Tutoring System · Created: 2026-05-19

## Philosophy

This model treats every meaningful action in the tutoring system as an immutable event stored in an append-only event store. The event store is the single source of truth; all queryable state (student dashboards, teacher views, knowledge mastery, analytics) is derived by projecting events into materialised read models optimised for specific query patterns. This is the CQRS (Command Query Responsibility Segregation) pattern applied to education.

The approach is inspired by the xAPI specification itself (which is fundamentally an event log of Actor-Verb-Object statements), by FERPA/COPPA audit requirements (which demand a complete record of every action taken on student data), and by the research literature on learning analytics (which depends on rich, timestamped interaction sequences to train knowledge tracing models). Event sourcing makes the audit trail the architecture, not an afterthought.

Every student interaction -- starting a session, answering a question, requesting a hint, the tutor detecting a misconception, a teacher viewing a dashboard -- is captured as a typed event with a timestamp, actor, and payload. Read models are rebuilt from events, meaning the system can retroactively compute new analytics (e.g., "how many students exhibited misconception X before we added a remediation strategy?") without schema changes.

**Best for:** Platforms requiring complete audit trails, FERPA/COPPA compliance by design, rich learning analytics, and the ability to retroactively compute new metrics from historical interaction data.

**Trade-offs:**
- (+) Complete, immutable audit trail -- every action is recorded and never modified
- (+) Temporal queries are natural: "what was the student's mastery on date X?" is a replay to that point
- (+) New read models can be built retroactively from existing events without migrations
- (+) Clean alignment with xAPI (events map directly to xAPI statements)
- (+) Excellent for training ML/AI models on interaction sequences
- (-) Higher storage requirements (events accumulate indefinitely)
- (-) Eventual consistency between writes and reads; dashboards may lag by seconds
- (-) More complex infrastructure: event store + projection workers + read model databases
- (-) Debugging requires understanding event replay; harder to "just look at the database"
- (-) GDPR right-to-erasure conflicts with immutability (requires crypto-shredding or tombstone events)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| xAPI (Experience API) | Events map 1:1 to xAPI statements; the event store IS the Learning Record Store |
| IMS Caliper 1.2 | Caliper events (AssessmentEvent, FeedbackEvent) are emitted as typed events in the store |
| FERPA / COPPA | Immutable event log provides audit trail by design; no separate audit table needed |
| GDPR | Crypto-shredding pattern: PII encrypted with per-user keys; key deletion = erasure without mutating events |
| IEEE P2247 (AIS) | Learner Model derived from event projections; Domain Model and Pedagogical Model as configuration |
| Ed-Fi Data Standard v5 | Read models for enrollment and assessment results projected into Ed-Fi compatible structures |
| IMS LTI 1.3 | LTI launch events captured in the event store; registration config in a small reference table |

---

## Event Store (Core)

```sql
-- The single source of truth: an append-only event log
-- Partitioned by month for query performance and archival
CREATE TABLE event_store (
    event_id UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,                      -- aggregate root ID (e.g., session ID, student ID)
    stream_type TEXT NOT NULL,                    -- 'tutoring_session', 'student_knowledge', 'enrollment', 'system'
    event_type TEXT NOT NULL,                     -- e.g., 'SessionStarted', 'StudentAnswered', 'HintRequested'
    event_version INT NOT NULL,                   -- sequence within stream for ordering
    actor_id UUID NOT NULL,                       -- who/what caused this event
    actor_type TEXT NOT NULL,                     -- 'student', 'tutor_ai', 'teacher', 'system'
    institution_id UUID NOT NULL,                 -- tenant partition key
    payload JSONB NOT NULL,                       -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',         -- correlation IDs, causation, LLM model, etc.
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    stored_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    encryption_key_id UUID,                       -- for crypto-shredding (GDPR erasure)
    PRIMARY KEY (event_id, occurred_at)
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions (example)
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automation

-- Indexes for common access patterns
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_actor ON event_store(actor_id, occurred_at);
CREATE INDEX idx_event_institution ON event_store(institution_id, occurred_at);
```

## Event Type Catalogue

```sql
-- Registry of all known event types with their JSON schemas
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    category TEXT NOT NULL,                       -- 'session', 'attempt', 'knowledge', 'enrollment', 'admin', 'lti'
    description TEXT NOT NULL,
    payload_schema JSONB NOT NULL,                -- JSON Schema for validation
    xapi_verb_iri TEXT,                           -- mapping to xAPI verb
    caliper_event_type TEXT,                      -- mapping to Caliper event type
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- SessionStarted        { session_mode, language, llm_model, section_id? }
-- SessionEnded          { duration_seconds, total_turns }
-- StudentAnswered       { item_id, response, is_correct, partial_credit, time_spent_ms }
-- HintRequested         { item_id, hint_level, hint_type }
-- HintDelivered         { item_id, hint_level, hint_content }
-- MisconceptionDetected { item_id, misconception_id, confidence, evidence }
-- RemediationDelivered  { misconception_id, strategy, content }
-- MasteryUpdated        { kc_id, old_p_mastery, new_p_mastery, old_status, new_status }
-- TutorMessageSent      { content, token_count, latency_ms }
-- StudentMessageSent    { content, response_latency_ms }
-- TeacherViewedDashboard { section_id, view_type }
-- AlertGenerated        { alert_type, severity, student_id?, kc_id? }
-- LTILaunchReceived     { deployment_id, roles, context_id, resource_link_id }
-- EnrollmentCreated     { user_id, section_id, role }
-- ConsentGranted        { minor_id, parent_id, consent_type, method }
-- ConsentRevoked        { minor_id, consent_type }
-- DataDeletionRequested { user_id, reason, requested_by }
```

## Example Event Payloads

```json
-- SessionStarted
{
  "event_type": "SessionStarted",
  "stream_type": "tutoring_session",
  "payload": {
    "session_mode": "socratic",
    "language": "en",
    "llm_model": "llama-3.1-70b-ft-tutor-v2",
    "section_id": "a1b2c3d4-...",
    "initial_kc_targets": ["uuid-1", "uuid-2"]
  }
}

-- StudentAnswered
{
  "event_type": "StudentAnswered",
  "stream_type": "tutoring_session",
  "payload": {
    "item_id": "uuid-item-1",
    "student_response": {"value": "x = 7"},
    "is_correct": false,
    "partial_credit": 0.0,
    "time_spent_ms": 34200,
    "attempt_number": 1,
    "knowledge_components": ["uuid-kc-1"]
  }
}

-- MisconceptionDetected
{
  "event_type": "MisconceptionDetected",
  "stream_type": "tutoring_session",
  "payload": {
    "item_id": "uuid-item-1",
    "misconception_id": "uuid-misc-1",
    "misconception_code": "DISTRIBUTE_NEGATIVE_SIGN",
    "confidence": 0.87,
    "evidence": "Student wrote x = 7 instead of x = -7, suggesting failure to distribute the negative sign"
  }
}

-- MasteryUpdated
{
  "event_type": "MasteryUpdated",
  "stream_type": "student_knowledge",
  "payload": {
    "knowledge_component_id": "uuid-kc-1",
    "old_p_mastery": 0.42,
    "new_p_mastery": 0.38,
    "old_status": "learning",
    "new_status": "learning",
    "triggered_by_attempt": "uuid-attempt-1",
    "bkt_parameters": {
      "p_transit": 0.09,
      "p_slip": 0.10,
      "p_guess": 0.20
    }
  }
}
```

## Reference Tables (Non-Event State)

```sql
-- Small reference tables that don't change via events
-- These are configuration, not mutable state

CREATE TABLE institution (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    institution_type TEXT NOT NULL,
    jurisdiction_country CHAR(2) NOT NULL,
    privacy_framework TEXT NOT NULL DEFAULT 'ferpa',
    encryption_master_key_id UUID,                -- for crypto-shredding
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE knowledge_component (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject TEXT NOT NULL,
    name TEXT NOT NULL,
    display_name TEXT NOT NULL,
    difficulty_level SMALLINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE misconception (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    remediation_strategy TEXT,
    counter_example_template TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE curriculum_standard (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework TEXT NOT NULL,
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    parent_standard_id UUID REFERENCES curriculum_standard(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assessment_item (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    item_type TEXT NOT NULL,
    content_body TEXT NOT NULL,
    correct_response JSONB NOT NULL,
    max_score NUMERIC(6,2) NOT NULL DEFAULT 1.0,
    language CHAR(5) NOT NULL DEFAULT 'en',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lti_registration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    client_id TEXT NOT NULL,
    issuer TEXT NOT NULL,
    auth_endpoint TEXT NOT NULL,
    token_endpoint TEXT NOT NULL,
    jwks_url TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Crypto-shredding key store (for GDPR erasure)
CREATE TABLE encryption_key (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    institution_id UUID NOT NULL REFERENCES institution(id),
    encrypted_key BYTEA NOT NULL,                 -- AES key encrypted with master key
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'destroyed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    destroyed_at TIMESTAMPTZ
);

CREATE UNIQUE INDEX idx_enc_key_user ON encryption_key(user_id) WHERE status = 'active';
```

## Projected Read Models

These tables are derived entirely from events. They can be dropped and rebuilt from the event store at any time.

```sql
-- READ MODEL: Current student knowledge state
-- Projected from: MasteryUpdated events
CREATE TABLE rm_student_knowledge (
    student_id UUID NOT NULL,
    knowledge_component_id UUID NOT NULL,
    p_mastery NUMERIC(5,4) NOT NULL,
    mastery_status TEXT NOT NULL,
    total_attempts INT NOT NULL DEFAULT 0,
    correct_attempts INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (student_id, knowledge_component_id)
);

-- READ MODEL: Active session state
-- Projected from: SessionStarted, StudentAnswered, TutorMessageSent, SessionEnded events
CREATE TABLE rm_active_session (
    session_id UUID PRIMARY KEY,
    student_id UUID NOT NULL,
    section_id UUID,
    session_mode TEXT NOT NULL,
    language TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    total_turns INT NOT NULL DEFAULT 0,
    items_attempted INT NOT NULL DEFAULT 0,
    items_correct INT NOT NULL DEFAULT 0,
    hints_requested INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    ended_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_session_student ON rm_active_session(student_id, is_active);
CREATE INDEX idx_rm_session_section ON rm_active_session(section_id) WHERE is_active;

-- READ MODEL: Teacher dashboard class summary
-- Projected from: MasteryUpdated, SessionStarted/Ended, AlertGenerated events
CREATE TABLE rm_class_dashboard (
    section_id UUID NOT NULL,
    knowledge_component_id UUID NOT NULL,
    total_students INT NOT NULL DEFAULT 0,
    mastered_count INT NOT NULL DEFAULT 0,
    learning_count INT NOT NULL DEFAULT 0,
    struggling_count INT NOT NULL DEFAULT 0,      -- p_mastery declining
    avg_mastery NUMERIC(5,4) NOT NULL DEFAULT 0,
    common_misconception_id UUID,
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (section_id, knowledge_component_id)
);

-- READ MODEL: Enrollment roster
-- Projected from: EnrollmentCreated, EnrollmentUpdated events
CREATE TABLE rm_enrollment (
    user_id UUID NOT NULL,
    section_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    role TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    enrolled_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, section_id)
);

CREATE INDEX idx_rm_enrollment_section ON rm_enrollment(section_id);

-- READ MODEL: User account (projected from identity events)
CREATE TABLE rm_user_account (
    id UUID PRIMARY KEY,
    institution_id UUID NOT NULL,
    display_name TEXT NOT NULL,
    email TEXT,
    locale TEXT NOT NULL DEFAULT 'en',
    is_minor BOOLEAN NOT NULL DEFAULT false,
    roles TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL,
    last_active_at TIMESTAMPTZ
);

-- READ MODEL: Misconception frequency analytics
-- Projected from: MisconceptionDetected events
CREATE TABLE rm_misconception_analytics (
    misconception_id UUID NOT NULL,
    knowledge_component_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    section_id UUID,
    time_bucket TIMESTAMPTZ NOT NULL,             -- hourly or daily bucket
    occurrence_count INT NOT NULL DEFAULT 0,
    unique_students INT NOT NULL DEFAULT 0,
    avg_confidence NUMERIC(5,4),
    PRIMARY KEY (misconception_id, institution_id, time_bucket, section_id)
);

-- READ MODEL: xAPI-compatible statement view for LRS export
CREATE TABLE rm_xapi_statement (
    statement_id UUID PRIMARY KEY,
    actor_id UUID NOT NULL,
    verb_id TEXT NOT NULL,
    verb_display TEXT NOT NULL,
    object_type TEXT NOT NULL,
    object_id TEXT NOT NULL,
    result_success BOOLEAN,
    result_score_scaled NUMERIC(5,4),
    context_registration UUID,
    full_statement JSONB NOT NULL,
    stored_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_xapi_actor ON rm_xapi_statement(actor_id, stored_at);
```

## Projection Workers (Pseudocode)

```sql
-- Checkpoint tracking for projection workers
CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example: how the knowledge state projector works (pseudocode)
--
-- FOR EACH new MasteryUpdated event:
--   UPSERT rm_student_knowledge SET
--     p_mastery = event.payload.new_p_mastery,
--     mastery_status = event.payload.new_status,
--     total_attempts = total_attempts + 1,
--     correct_attempts = correct_attempts + (1 if event related to correct attempt),
--     last_attempt_at = event.occurred_at,
--     last_updated_at = event.occurred_at
--   WHERE student_id = event.actor_id
--     AND knowledge_component_id = event.payload.knowledge_component_id
--
-- FOR EACH new MisconceptionDetected event:
--   UPDATE rm_misconception_analytics
--     INCREMENT occurrence_count
--     RECALCULATE unique_students, avg_confidence
--   WHERE misconception_id = event.payload.misconception_id
--     AND time_bucket = date_trunc('hour', event.occurred_at)
```

## Temporal Query Examples

```sql
-- "What was student X's mastery of KC Y on March 15, 2026?"
-- Replay MasteryUpdated events up to that date:
SELECT payload->>'new_p_mastery' AS p_mastery,
       payload->>'new_status' AS mastery_status
FROM event_store
WHERE stream_type = 'student_knowledge'
  AND actor_id = 'student-uuid'
  AND event_type = 'MasteryUpdated'
  AND (payload->>'knowledge_component_id') = 'kc-uuid'
  AND occurred_at <= '2026-03-15 23:59:59+00'
ORDER BY occurred_at DESC
LIMIT 1;

-- "Show me all misconceptions student X exhibited this semester"
SELECT payload->>'misconception_code' AS misconception,
       payload->>'confidence' AS confidence,
       payload->>'evidence' AS evidence,
       occurred_at
FROM event_store
WHERE actor_id = 'student-uuid'
  AND event_type = 'MisconceptionDetected'
  AND occurred_at BETWEEN '2026-01-15' AND '2026-05-15'
ORDER BY occurred_at;

-- "How many students struggled with linear equations in Section A last week?"
SELECT COUNT(DISTINCT actor_id)
FROM event_store
WHERE event_type = 'MasteryUpdated'
  AND (payload->>'knowledge_component_id') = 'kc-linear-eq-uuid'
  AND (payload->>'new_p_mastery')::NUMERIC < 0.4
  AND occurred_at >= now() - INTERVAL '7 days'
  AND stream_id IN (
      SELECT stream_id FROM event_store
      WHERE event_type = 'SessionStarted'
        AND (payload->>'section_id') = 'section-a-uuid'
  );
```

## GDPR Erasure via Crypto-Shredding

```sql
-- When a user exercises their right to erasure:
-- 1. Mark their encryption key as destroyed
UPDATE encryption_key
SET status = 'destroyed', destroyed_at = now()
WHERE user_id = 'user-to-delete-uuid';

-- 2. Events remain in the store but PII fields in payloads
--    are encrypted with the now-destroyed key -> effectively unreadable
-- 3. Insert a tombstone event
INSERT INTO event_store (stream_id, stream_type, event_type, event_version, actor_id, actor_type, institution_id, payload)
VALUES ('user-to-delete-uuid', 'system', 'DataErased', 1, 'system', 'system', 'inst-uuid',
        '{"reason": "gdpr_right_to_erasure", "fields_affected": ["display_name", "email", "content"]}'::jsonb);

-- 4. Clear read models
DELETE FROM rm_user_account WHERE id = 'user-to-delete-uuid';
DELETE FROM rm_student_knowledge WHERE student_id = 'user-to-delete-uuid';
DELETE FROM rm_active_session WHERE student_id = 'user-to-delete-uuid';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store (partitioned), event_type_registry |
| Reference Data | 6 | institution, knowledge_component, misconception, curriculum_standard, assessment_item, lti_registration |
| Crypto-Shredding | 1 | encryption_key |
| Projection Infrastructure | 1 | projection_checkpoint |
| Read Models | 7 | rm_student_knowledge, rm_active_session, rm_class_dashboard, rm_enrollment, rm_user_account, rm_misconception_analytics, rm_xapi_statement |
| **Total** | **17** | Read models are rebuildable from events; actual "source of truth" is only 9 tables |

---

## Key Design Decisions

1. **Single event store as source of truth** -- all mutable state derives from events. The event_store table plus reference tables (which are append-only configuration) are the only data that must be backed up and protected.

2. **Monthly partitioning** -- event_store is range-partitioned by `occurred_at` for query performance on time-range scans and to enable archival of old partitions to cold storage.

3. **Event type registry with JSON Schema validation** -- every event type has a registered schema, enabling both write-time validation and tooling that can discover what events exist in the system.

4. **Direct xAPI mapping** -- each event type has an optional `xapi_verb_iri` mapping, so events can be projected into xAPI statements without transformation logic. The event store effectively IS the Learning Record Store.

5. **Crypto-shredding for GDPR** -- PII in event payloads is encrypted with per-user AES keys. Deleting the key makes the PII unrecoverable without mutating the immutable event log. This resolves the tension between event sourcing immutability and GDPR right-to-erasure.

6. **Read models are disposable** -- every `rm_*` table can be dropped and rebuilt from events. This means new analytics requirements can be addressed by writing a new projector and replaying history, without any schema migration on the source data.

7. **Projection checkpoints** -- each projector tracks its position in the event stream, enabling restart after failure and parallel projection of different read models.

8. **Eventual consistency is acceptable** -- teacher dashboards and knowledge state views lag behind the event store by the projection delay (typically sub-second). For real-time feedback during a tutoring session, the session aggregate maintains in-memory state that is consistent within the session.

9. **Event payloads carry business intent** -- events like `MisconceptionDetected` capture the reasoning (confidence, evidence) not just the fact, enabling richer analytics and ML training without re-running the detection model.

10. **Low table count** -- only 17 tables total (9 source-of-truth + 7 read models + 1 projection infra), compared to 28+ in a normalized relational approach. Complexity lives in the projectors (application code) rather than the schema.
