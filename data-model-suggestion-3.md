# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: AI Tutoring System · Created: 2026-05-19

## Philosophy

This model uses PostgreSQL relational tables for core structural data (identity, enrollment, sessions) while leveraging JSONB columns extensively for domain-specific data that varies by subject, jurisdiction, institution, or pedagogical approach. The key insight is that an AI tutoring system must serve wildly different contexts -- a K-5 math class in Texas has different curriculum standards, interaction patterns, and compliance requirements than a university physics course in Germany -- and a rigid schema cannot anticipate every variation.

Core relationships (student belongs to institution, enrolled in section, has tutoring sessions) are relational with full foreign key integrity. But the details of each interaction -- how a student's response is structured, what metadata a particular LLM produces, what jurisdiction-specific consent fields are required, what a knowledge component looks like across subjects -- are stored in JSONB columns with GIN indexes for efficient querying.

This hybrid approach draws from platforms like Canvas LMS (which stores tool configuration as JSON), Duolingo (which uses flexible schemas to support 40+ languages with different exercise types), and modern SaaS platforms that need to ship features across diverse customer configurations without per-customer migrations. It optimises for rapid iteration and multi-domain extensibility.

**Best for:** Rapid MVP development, multi-subject/multi-jurisdiction deployments where schema varies by context, and teams that want relational integrity for core entities with flexibility for everything else.

**Trade-offs:**
- (+) Fewer tables than fully normalized (~20 core tables vs ~28+)
- (+) Adding new subjects, item types, or jurisdiction fields requires no schema migration
- (+) JSONB GIN indexes enable efficient containment and existence queries
- (+) Faster development velocity -- new features often only require application-level changes
- (+) Natural fit for storing LLM inputs/outputs which have variable structure
- (-) JSONB columns lack foreign key constraints -- referential integrity must be enforced in application code
- (-) Complex JSONB queries can be slower than indexed relational joins for deeply nested data
- (-) Schema validation must be done at the application layer (JSON Schema) rather than database constraints
- (-) Data quality risks: flexible fields can accumulate inconsistent structures over time without discipline
- (-) Reporting across JSONB fields requires more complex SQL (->>, @>, jsonb_path_query)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| xAPI (Experience API) | xAPI statements stored as full JSONB documents; indexed by actor/verb/object for LRS queries |
| IMS QTI 3.0 | Assessment items store QTI-compatible structure in JSONB `content` and `response_declaration` fields |
| IMS LTI 1.3 | LTI configuration stored as JSONB allowing flexible platform-specific settings |
| IMS OneRoster 1.1 | Core enrollment tables are relational and OneRoster-compatible; extended fields in JSONB |
| Ed-Fi Data Standard v5 | Relational structure for student/course/section aligns with Ed-Fi; extended demographics in JSONB |
| IEEE P2247 (AIS) | Learner model stored as JSONB per-KC knowledge state; domain model as JSONB item metadata |
| FERPA / COPPA / GDPR | Consent and audit data in relational tables; jurisdiction-specific fields in JSONB |
| ISO 3166 | Jurisdiction codes in relational columns; jurisdiction-specific rules in JSONB config |

---

## Multi-Tenancy & Identity

```sql
CREATE TABLE institution (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_institution_id UUID REFERENCES institution(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    institution_type TEXT NOT NULL,
    jurisdiction_country CHAR(2) NOT NULL,
    jurisdiction_subdivision VARCHAR(6),
    privacy_framework TEXT NOT NULL DEFAULT 'ferpa',
    -- Flexible institution-level configuration
    config JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "allowed_session_modes": ["socratic", "worked_example"],
    --   "max_hint_levels": 4,
    --   "coppa_age_threshold": 13,
    --   "data_retention_days": { "sessions": 730, "audit": 2555 },
    --   "llm_config": { "model": "llama-3.1-70b", "max_tokens": 2048 },
    --   "branding": { "logo_url": "...", "primary_color": "#1a73e8" },
    --   "enabled_subjects": ["math", "science", "english"],
    --   "gdpr_dpo_email": "dpo@example.de"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_institution_config ON institution USING GIN (config);

CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    email TEXT,
    display_name TEXT NOT NULL,
    roles TEXT[] NOT NULL DEFAULT '{"student"}',    -- PostgreSQL array for simple RBAC
    locale TEXT NOT NULL DEFAULT 'en',
    is_minor BOOLEAN NOT NULL DEFAULT false,
    -- Flexible user profile and preferences
    profile JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "external_ids": { "sis": "STU12345", "oneroster": "abc-def" },
    --   "date_of_birth": "2014-03-15",
    --   "accessibility": { "screen_reader": true, "high_contrast": false, "font_size": "large" },
    --   "learning_preferences": { "hint_style": "socratic", "language": "es" },
    --   "guardian_contacts": [{ "name": "Jane Doe", "email": "jane@example.com", "relationship": "parent" }]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_institution ON user_account(institution_id);
CREATE INDEX idx_user_roles ON user_account USING GIN (roles);
CREATE INDEX idx_user_profile ON user_account USING GIN (profile jsonb_path_ops);

-- Consent tracking (relational for auditability)
CREATE TABLE consent_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    minor_user_id UUID NOT NULL REFERENCES user_account(id),
    parent_user_id UUID REFERENCES user_account(id),
    consent_type TEXT NOT NULL,
    status TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',           -- jurisdiction-specific consent fields
    granted_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_minor ON consent_record(minor_user_id);
```

## Curriculum & Knowledge Domain

```sql
CREATE TABLE knowledge_component (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject TEXT NOT NULL,
    name TEXT NOT NULL,
    display_name TEXT NOT NULL,
    difficulty_level SMALLINT,
    -- Flexible metadata varying by subject
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example for math KC:
    -- {
    --   "standards_alignment": [
    --     { "framework": "CCSS", "code": "CCSS.MATH.CONTENT.8.EE.A.1" },
    --     { "framework": "state_TX", "code": "8.2A" }
    --   ],
    --   "prerequisites": ["uuid-1", "uuid-2"],
    --   "misconceptions": [
    --     { "code": "DISTRIBUTE_NEGATIVE", "name": "Distributing negative sign", "severity": "critical",
    --       "remediation": "Show expanded form with parentheses before simplifying" }
    --   ],
    --   "bloom_level": "application",
    --   "estimated_minutes": 15
    -- }
    -- Example for language KC:
    -- {
    --   "standards_alignment": [{ "framework": "CEFR", "level": "B1" }],
    --   "target_language": "es",
    --   "grammar_topic": "subjunctive_mood",
    --   "vocabulary_set": ["querer", "esperar", "dudar"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kc_subject ON knowledge_component(subject);
CREATE INDEX idx_kc_metadata ON knowledge_component USING GIN (metadata jsonb_path_ops);

-- Assessment items with flexible content structure
CREATE TABLE assessment_item (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject TEXT NOT NULL,
    item_type TEXT NOT NULL,                       -- 'choice', 'text_entry', 'extended_text', 'image_input', etc.
    language CHAR(5) NOT NULL DEFAULT 'en',
    max_score NUMERIC(6,2) NOT NULL DEFAULT 1.0,
    -- All item content and structure in JSONB (QTI-aligned)
    content JSONB NOT NULL,
    -- Example for multiple-choice math:
    -- {
    --   "stem": "Solve for x: 3x + 5 = 20",
    --   "stem_format": "latex",
    --   "choices": [
    --     { "id": "A", "value": "x = 5", "correct": true },
    --     { "id": "B", "value": "x = 7", "correct": false, "misconception": "IGNORE_CONSTANT" },
    --     { "id": "C", "value": "x = 15", "correct": false, "misconception": "ADD_INSTEAD_OF_SUBTRACT" }
    --   ],
    --   "hints": [
    --     { "level": 1, "type": "socratic", "text": "What operation undoes addition?" },
    --     { "level": 2, "type": "worked_step", "text": "First subtract 5 from both sides: 3x = 15" },
    --     { "level": 3, "type": "direct", "text": "Now divide both sides by 3" }
    --   ],
    --   "solution_methods": ["isolation", "guess_and_check"],
    --   "images": [],
    --   "knowledge_components": ["uuid-kc-1"]
    -- }
    -- Example for free-text English essay:
    -- {
    --   "prompt": "Write a persuasive paragraph about renewable energy",
    --   "rubric": {
    --     "thesis": { "max_points": 2, "description": "Clear thesis statement" },
    --     "evidence": { "max_points": 3, "description": "Supporting evidence with citations" },
    --     "mechanics": { "max_points": 2, "description": "Grammar and spelling" }
    --   },
    --   "word_count_range": [100, 300],
    --   "knowledge_components": ["uuid-kc-persuasive-writing"]
    -- }
    difficulty NUMERIC(4,3),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_subject ON assessment_item(subject);
CREATE INDEX idx_item_type ON assessment_item(item_type);
CREATE INDEX idx_item_content ON assessment_item USING GIN (content jsonb_path_ops);
```

## Courses & Enrollment

```sql
CREATE TABLE course (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    grade_level TEXT,
    external_id TEXT,
    config JSONB NOT NULL DEFAULT '{}',            -- course-level tutoring configuration
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_institution ON course(institution_id);

CREATE TABLE section (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES course(id),
    title TEXT NOT NULL,
    term TEXT,
    external_id TEXT,
    start_date DATE,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE enrollment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID NOT NULL REFERENCES section(id),
    role TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    UNIQUE (user_id, section_id)
);

CREATE INDEX idx_enrollment_section ON enrollment(section_id);
CREATE INDEX idx_enrollment_user ON enrollment(user_id);
```

## Tutoring Sessions & Interactions

```sql
CREATE TABLE tutoring_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID REFERENCES section(id),
    session_mode TEXT NOT NULL,
    language CHAR(5) NOT NULL DEFAULT 'en',
    llm_model TEXT NOT NULL,
    -- Session-level metrics (updated as session progresses)
    metrics JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "total_turns": 24,
    --   "items_attempted": 5,
    --   "items_correct": 3,
    --   "hints_requested": 7,
    --   "misconceptions_detected": ["uuid-1", "uuid-2"],
    --   "kcs_practiced": ["uuid-kc-1", "uuid-kc-2"],
    --   "engagement_score": 0.82,
    --   "socratic_turns": 12,
    --   "total_tokens": 4830
    -- }
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_student ON tutoring_session(student_id);
CREATE INDEX idx_session_section ON tutoring_session(section_id);
CREATE INDEX idx_session_started ON tutoring_session(started_at);
CREATE INDEX idx_session_metrics ON tutoring_session USING GIN (metrics jsonb_path_ops);

-- Conversation turns: relational for ordering, JSONB for variable content
CREATE TABLE conversation_turn (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES tutoring_session(id) ON DELETE CASCADE,
    sequence_number INT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('student', 'tutor', 'system')),
    -- Turn content and metadata in JSONB (varies by turn type)
    turn_data JSONB NOT NULL,
    -- Example student answer turn:
    -- {
    --   "content": "x = 7",
    --   "content_format": "text",
    --   "item_id": "uuid-item-1",
    --   "attempt_number": 1,
    --   "is_correct": false,
    --   "response_latency_ms": 34200,
    --   "detected_misconception": {
    --     "id": "uuid-misc-1",
    --     "code": "DISTRIBUTE_NEGATIVE",
    --     "confidence": 0.87,
    --     "evidence": "Student failed to distribute the negative sign"
    --   }
    -- }
    -- Example tutor response turn:
    -- {
    --   "content": "Good try! Let's look at that negative sign more carefully...",
    --   "content_format": "markdown",
    --   "hint_level": 1,
    --   "hint_type": "socratic_question",
    --   "llm_latency_ms": 1200,
    --   "token_count": 89,
    --   "pedagogical_strategy": "counter_example"
    -- }
    -- Example image input turn:
    -- {
    --   "content_format": "image",
    --   "image_url": "s3://bucket/uploads/uuid.jpg",
    --   "ocr_result": "3x + 5 = 20",
    --   "ocr_confidence": 0.95
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_turn_session_seq ON conversation_turn(session_id, sequence_number);
CREATE INDEX idx_turn_data ON conversation_turn USING GIN (turn_data jsonb_path_ops);
```

## Student Knowledge State

```sql
-- Per-student knowledge state with BKT parameters in JSONB
CREATE TABLE student_knowledge (
    student_id UUID NOT NULL REFERENCES user_account(id),
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    mastery_status TEXT NOT NULL DEFAULT 'not_started',
    p_mastery NUMERIC(5,4) NOT NULL DEFAULT 0.3,
    -- Full BKT and extended parameters in JSONB
    model_params JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "bkt": { "p_transit": 0.09, "p_slip": 0.10, "p_guess": 0.20 },
    --   "total_attempts": 12,
    --   "correct_attempts": 7,
    --   "streak_current": 3,
    --   "streak_best": 5,
    --   "time_spent_total_seconds": 1820,
    --   "misconceptions_exhibited": {
    --     "DISTRIBUTE_NEGATIVE": { "count": 3, "last_seen": "2026-05-15T10:30:00Z", "remediated": true },
    --     "SIGN_ERROR": { "count": 1, "last_seen": "2026-05-10T14:00:00Z", "remediated": false }
    --   },
    --   "spaced_repetition": { "interval_days": 7, "next_review": "2026-05-25" },
    --   "mastery_history": [
    --     { "date": "2026-05-01", "p_mastery": 0.30 },
    --     { "date": "2026-05-08", "p_mastery": 0.55 },
    --     { "date": "2026-05-15", "p_mastery": 0.78 }
    --   ]
    -- }
    last_attempt_at TIMESTAMPTZ,
    mastery_achieved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (student_id, knowledge_component_id)
);

CREATE INDEX idx_sk_student ON student_knowledge(student_id);
CREATE INDEX idx_sk_mastery ON student_knowledge(mastery_status);
CREATE INDEX idx_sk_params ON student_knowledge USING GIN (model_params jsonb_path_ops);
```

## Teacher Alerts & Dashboard

```sql
CREATE TABLE teacher_alert (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    teacher_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID NOT NULL REFERENCES section(id),
    alert_type TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'info',
    title TEXT NOT NULL,
    -- Alert details in JSONB (varies by alert type)
    details JSONB NOT NULL DEFAULT '{}',
    -- Example for 'class_misconception':
    -- {
    --   "misconception_code": "DISTRIBUTE_NEGATIVE",
    --   "affected_students": 8,
    --   "total_students": 24,
    --   "knowledge_component": "solving_linear_equations",
    --   "suggested_reteach": "Review distribution of negative signs with visual models",
    --   "sample_errors": ["x = 7 (should be -7)", "x = 3 (should be -3)"]
    -- }
    is_read BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at TIMESTAMPTZ
);

CREATE INDEX idx_alert_teacher ON teacher_alert(teacher_id, is_read);
CREATE INDEX idx_alert_details ON teacher_alert USING GIN (details jsonb_path_ops);
```

## LTI & xAPI Integration

```sql
CREATE TABLE lti_registration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    -- Core LTI fields relational
    client_id TEXT NOT NULL,
    issuer TEXT NOT NULL,
    -- All platform-specific config in JSONB
    platform_config JSONB NOT NULL,
    -- Example:
    -- {
    --   "auth_endpoint": "https://canvas.example.com/api/lti/authorize_redirect",
    --   "token_endpoint": "https://canvas.example.com/login/oauth2/token",
    --   "jwks_url": "https://canvas.example.com/api/lti/security/jwks",
    --   "platform_name": "Canvas",
    --   "deployments": [
    --     { "deployment_id": "dep-123", "context_type": "institution" },
    --     { "deployment_id": "dep-456", "context_type": "course" }
    --   ],
    --   "supported_messages": ["LtiResourceLinkRequest", "LtiDeepLinkingRequest"],
    --   "scopes": ["lineitem", "score", "result"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- xAPI statements as full JSONB documents
CREATE TABLE xapi_statement (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id UUID NOT NULL UNIQUE,
    actor_id UUID NOT NULL REFERENCES user_account(id),
    verb_id TEXT NOT NULL,
    object_id TEXT NOT NULL,
    -- Full xAPI statement as JSONB for LRS compliance
    statement JSONB NOT NULL,
    stored_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_xapi_actor ON xapi_statement(actor_id, stored_at);
CREATE INDEX idx_xapi_verb ON xapi_statement(verb_id, stored_at);
CREATE INDEX idx_xapi_statement ON xapi_statement USING GIN (statement jsonb_path_ops);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES user_account(id),
    institution_id UUID REFERENCES institution(id),
    action TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,                                -- { "field": { "old": ..., "new": ... } }
    context JSONB NOT NULL DEFAULT '{}',          -- IP, user agent, request ID
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

## JSONB Query Examples

```sql
-- Find all students who have exhibited a specific misconception
SELECT sk.student_id, ua.display_name,
       sk.model_params->'misconceptions_exhibited'->'DISTRIBUTE_NEGATIVE' AS misconception_data
FROM student_knowledge sk
JOIN user_account ua ON ua.id = sk.student_id
WHERE sk.model_params @> '{"misconceptions_exhibited": {"DISTRIBUTE_NEGATIVE": {}}}'
  AND (sk.model_params->'misconceptions_exhibited'->'DISTRIBUTE_NEGATIVE'->>'remediated')::boolean = false;

-- Find assessment items targeting a specific KC
SELECT id, content->>'stem' AS question,
       content->'choices' AS choices
FROM assessment_item
WHERE content @> '{"knowledge_components": ["uuid-kc-1"]}'
  AND item_type = 'choice';

-- Get session metrics for a student this week
SELECT id, started_at,
       metrics->>'items_attempted' AS items,
       metrics->>'items_correct' AS correct,
       metrics->>'engagement_score' AS engagement
FROM tutoring_session
WHERE student_id = 'student-uuid'
  AND started_at >= now() - INTERVAL '7 days'
ORDER BY started_at DESC;

-- Find institutions with GDPR privacy framework
SELECT id, name, config->>'gdpr_dpo_email' AS dpo_email
FROM institution
WHERE config @> '{"privacy_framework": "gdpr"}'
   OR privacy_framework = 'gdpr';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | institution, user_account, consent_record |
| Curriculum & Knowledge Domain | 2 | knowledge_component, assessment_item |
| Courses & Enrollment | 3 | course, section, enrollment |
| Tutoring Sessions | 2 | tutoring_session, conversation_turn |
| Student Knowledge Model | 1 | student_knowledge (JSONB holds BKT + history + misconceptions) |
| Teacher Dashboard | 1 | teacher_alert |
| LTI & xAPI Integration | 2 | lti_registration, xapi_statement |
| Audit & Compliance | 1 | audit_log |
| **Total** | **15** | Significantly fewer tables than normalized; complexity in JSONB structure |

---

## Key Design Decisions

1. **JSONB for variable-structure data, relational for fixed structure** -- identity, enrollment, and session existence are relational with foreign keys. Everything that varies by subject, jurisdiction, or pedagogical approach lives in JSONB.

2. **GIN indexes with jsonb_path_ops** -- `jsonb_path_ops` operator class is used for containment queries (`@>`) which are the primary access pattern for JSONB data. This provides efficient indexing without indexing every key.

3. **Knowledge state with embedded mastery history** -- instead of a separate snapshot table, `model_params` contains a `mastery_history` array. This trades off temporal query efficiency for simpler schema and faster reads of a single student's progress.

4. **Misconceptions embedded in knowledge state** -- rather than separate misconception tracking tables, misconceptions exhibited by each student are embedded in the `model_params` JSONB. This keeps the student's full learning profile in one row per KC.

5. **Assessment item content is entirely JSONB** -- a multiple-choice math item and a free-text essay prompt have fundamentally different structures. JSONB allows both to live in the same table with subject-appropriate structure validated at the application layer via JSON Schema.

6. **Institution config as JSONB** -- per-institution settings (allowed session modes, hint levels, LLM configuration, branding, subject enablement) are JSONB rather than separate config tables. This eliminates the need for migrations when adding new institution-level features.

7. **Conversation turn_data as JSONB** -- student text input, tutor markdown responses, image uploads, and OCR results have different fields. A single JSONB column accommodates all turn types without null columns.

8. **LTI deployments embedded in registration JSONB** -- rather than a separate `lti_deployment` table, deployments are an array within the registration's `platform_config`. This is simpler for typical read patterns (load all config for a registration at once).

9. **15 tables total** -- roughly half the table count of the normalized model, with equivalent functional coverage. The trade-off is that application code must enforce JSONB schema consistency that the database handles automatically in a normalized model.

10. **JSON Schema validation at application boundary** -- every JSONB column has a documented example structure and should have a corresponding JSON Schema validated on write. This compensates for the lack of database-level constraints on JSONB content.
