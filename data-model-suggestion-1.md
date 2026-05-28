# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI Tutoring System · Created: 2026-05-19

## Philosophy

This model follows classical relational database design with full normalization (3NF/BCNF) and explicit foreign key relationships between every entity. Every concept in the domain -- students, institutions, knowledge components, tutoring sessions, conversation turns, assessment items, misconceptions -- gets its own table with well-defined constraints. Junction tables handle all many-to-many relationships explicitly.

This approach draws directly from the Carnegie Learning MATHia cognitive tutor architecture (400+ individually tracked skills with explicit mastery thresholds) and the Ed-Fi Unifying Data Model (separate entities for students, assessments, learning standards, and their relationships). It prioritises data integrity, query flexibility, and alignment with education data standards (xAPI, QTI 3.0, Ed-Fi, OneRoster).

The normalized structure makes it straightforward to enforce referential integrity, generate compliance reports (FERPA audit logs, COPPA consent records), and integrate with institutional systems via standard APIs. Every relationship is explicit and queryable without parsing embedded JSON or replaying event streams.

**Best for:** Institutions requiring strict data integrity, complex cross-entity reporting, and alignment with established education data standards (Ed-Fi, xAPI, QTI).

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Standard SQL queries for any cross-entity analysis
- (+) Clean mapping to Ed-Fi, xAPI, and QTI data models
- (+) Straightforward FERPA/GDPR compliance with explicit audit tables
- (-) High table count (~55-65 tables) increases migration and maintenance complexity
- (-) Many JOIN operations for common read paths (e.g., loading a student dashboard)
- (-) Schema changes require migrations; adding jurisdiction-specific fields means new columns or tables
- (-) Knowledge state queries spanning thousands of skills per student can be slow without careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Ed-Fi Data Standard v5 | Student, course, section, and assessment entities mirror Ed-Fi UDM naming and structure for district reporting |
| xAPI (Experience API) | `xapi_statement` table stores Actor-Verb-Object triples; LRS-compatible export via REST |
| IMS QTI 3.0 | `assessment_item` and `item_interaction` tables map to QTI assessment-item and response-declaration structures |
| IMS LTI 1.3 | `lti_registration`, `lti_deployment`, and `lti_context` tables model tool registration and launch context |
| IMS OneRoster 1.1 | `institution`, `course`, `section`, `enrollment` tables align with OneRoster org/class/enrollment entities |
| IEEE P2247 (AIS) | Separate Learner Model (`knowledge_state`), Domain Model (`knowledge_component`, `curriculum_standard`), and Pedagogical Model (`pedagogical_strategy`) tables |
| ISO 3166 | `jurisdiction` table uses ISO 3166-1/3166-2 codes for country and subdivision |
| FERPA / COPPA | `audit_log`, `consent_record`, and `data_retention_policy` tables with explicit consent tracking |
| WCAG 2.2 | `accessibility_preference` table stores per-user accessibility settings |

---

## Multi-Tenancy & Identity

```sql
-- Tenant hierarchy: platform > institution > department
CREATE TABLE institution (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_institution_id UUID REFERENCES institution(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    institution_type TEXT NOT NULL CHECK (institution_type IN ('district', 'school', 'university', 'department', 'corporate')),
    jurisdiction_country CHAR(2) NOT NULL,       -- ISO 3166-1 alpha-2
    jurisdiction_subdivision VARCHAR(6),          -- ISO 3166-2
    timezone TEXT NOT NULL DEFAULT 'UTC',
    privacy_framework TEXT NOT NULL DEFAULT 'ferpa' CHECK (privacy_framework IN ('ferpa', 'gdpr', 'coppa', 'ccpa', 'custom')),
    data_residency_region TEXT NOT NULL DEFAULT 'us-east-1',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_institution_parent ON institution(parent_institution_id);
CREATE INDEX idx_institution_slug ON institution(slug);

-- Users: students, teachers, parents, admins
CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    external_id TEXT,                             -- SIS / OneRoster sourcedId
    email TEXT,
    display_name TEXT NOT NULL,
    locale TEXT NOT NULL DEFAULT 'en',
    date_of_birth DATE,                           -- needed for COPPA age-gating
    is_minor BOOLEAN NOT NULL DEFAULT false,
    accessibility_preferences JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_institution ON user_account(institution_id);
CREATE INDEX idx_user_external ON user_account(institution_id, external_id);

-- RBAC roles
CREATE TABLE role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,                    -- 'student', 'teacher', 'parent', 'admin', 'institution_admin'
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES role(id),
    institution_id UUID NOT NULL REFERENCES institution(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by UUID REFERENCES user_account(id),
    PRIMARY KEY (user_id, role_id, institution_id)
);

-- COPPA parental consent
CREATE TABLE consent_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    minor_user_id UUID NOT NULL REFERENCES user_account(id),
    parent_user_id UUID REFERENCES user_account(id),
    consent_type TEXT NOT NULL CHECK (consent_type IN ('coppa_parental', 'gdpr_data_processing', 'ferpa_directory', 'marketing')),
    status TEXT NOT NULL CHECK (status IN ('pending', 'granted', 'denied', 'revoked')),
    consent_method TEXT NOT NULL,                 -- 'email_verification', 'signed_form', 'in_app'
    ip_address INET,
    granted_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_minor ON consent_record(minor_user_id);
```

## Curriculum & Knowledge Domain

```sql
-- Curriculum standards (Common Core, state standards, etc.)
CREATE TABLE curriculum_standard (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework TEXT NOT NULL,                      -- 'CCSS', 'NGSS', 'state_TX', 'IB'
    code TEXT NOT NULL,                           -- e.g. 'CCSS.MATH.CONTENT.8.EE.A.1'
    title TEXT NOT NULL,
    description TEXT,
    grade_band TEXT,                              -- 'K-2', '3-5', '6-8', '9-12', 'higher-ed'
    subject TEXT NOT NULL,
    parent_standard_id UUID REFERENCES curriculum_standard(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_curriculum_framework_code ON curriculum_standard(framework, code);
CREATE INDEX idx_curriculum_parent ON curriculum_standard(parent_standard_id);

-- Knowledge components (fine-grained skills tracked by the tutor)
CREATE TABLE knowledge_component (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject TEXT NOT NULL,
    name TEXT NOT NULL,                           -- e.g. 'solving_linear_equations_one_variable'
    display_name TEXT NOT NULL,                   -- e.g. 'Solving Linear Equations (One Variable)'
    description TEXT,
    difficulty_level SMALLINT CHECK (difficulty_level BETWEEN 1 AND 10),
    prerequisite_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kc_subject ON knowledge_component(subject);

-- KC prerequisite graph
CREATE TABLE kc_prerequisite (
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    prerequisite_id UUID NOT NULL REFERENCES knowledge_component(id),
    strength TEXT NOT NULL DEFAULT 'required' CHECK (strength IN ('required', 'recommended')),
    PRIMARY KEY (knowledge_component_id, prerequisite_id),
    CHECK (knowledge_component_id != prerequisite_id)
);

-- Map KCs to curriculum standards (many-to-many)
CREATE TABLE kc_standard_alignment (
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    curriculum_standard_id UUID NOT NULL REFERENCES curriculum_standard(id),
    alignment_strength TEXT NOT NULL DEFAULT 'primary' CHECK (alignment_strength IN ('primary', 'secondary')),
    PRIMARY KEY (knowledge_component_id, curriculum_standard_id)
);

-- Common misconceptions per KC
CREATE TABLE misconception (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    code TEXT NOT NULL,                           -- e.g. 'DISTRIBUTE_NEGATIVE_SIGN'
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'moderate' CHECK (severity IN ('minor', 'moderate', 'critical')),
    remediation_strategy TEXT,
    counter_example_template TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_misconception_kc ON misconception(knowledge_component_id);
```

## Assessment Items (QTI-Aligned)

```sql
-- Assessment items (QTI 3.0 aligned)
CREATE TABLE assessment_item (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id TEXT,                             -- QTI identifier
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    item_type TEXT NOT NULL CHECK (item_type IN (
        'choice', 'multiple_choice', 'order', 'text_entry',
        'extended_text', 'match', 'gap_match', 'hotspot', 'custom'
    )),                                           -- QTI interaction types
    difficulty NUMERIC(4,3) CHECK (difficulty BETWEEN 0 AND 1),
    discrimination NUMERIC(4,3),                  -- IRT discrimination parameter
    content_body TEXT NOT NULL,                    -- the question text (may include LaTeX/MathML)
    correct_response JSONB NOT NULL,              -- QTI response-declaration mapping
    scoring_rules JSONB,                          -- QTI response-processing rules
    max_score NUMERIC(6,2) NOT NULL DEFAULT 1.0,
    language CHAR(5) NOT NULL DEFAULT 'en',       -- BCP 47
    created_by UUID REFERENCES user_account(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_subject ON assessment_item(subject);
CREATE INDEX idx_item_type ON assessment_item(item_type);

-- Link items to KCs (many-to-many)
CREATE TABLE item_kc_mapping (
    assessment_item_id UUID NOT NULL REFERENCES assessment_item(id),
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    PRIMARY KEY (assessment_item_id, knowledge_component_id)
);

-- Hint sequences per item
CREATE TABLE item_hint (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_item_id UUID NOT NULL REFERENCES assessment_item(id),
    sequence_number SMALLINT NOT NULL,
    hint_type TEXT NOT NULL CHECK (hint_type IN ('socratic_question', 'worked_step', 'counter_example', 'direct_hint')),
    content TEXT NOT NULL,
    reveals_answer BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_hint_item_seq ON item_hint(assessment_item_id, sequence_number);
```

## Courses & Enrollment (OneRoster-Aligned)

```sql
CREATE TABLE course (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    grade_level TEXT,
    external_id TEXT,                             -- OneRoster sourcedId
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_institution ON course(institution_id);

CREATE TABLE section (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES course(id),
    title TEXT NOT NULL,
    term TEXT,
    external_id TEXT,                             -- OneRoster sourcedId
    start_date DATE,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_section_course ON section(course_id);

CREATE TABLE enrollment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID NOT NULL REFERENCES section(id),
    role TEXT NOT NULL CHECK (role IN ('student', 'teacher', 'aide')),
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'completed')),
    external_id TEXT,                             -- OneRoster sourcedId
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    UNIQUE (user_id, section_id)
);

CREATE INDEX idx_enrollment_section ON enrollment(section_id);
CREATE INDEX idx_enrollment_user ON enrollment(user_id);
```

## Tutoring Sessions & Conversations

```sql
-- A tutoring session: one student working on one or more problems
CREATE TABLE tutoring_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID REFERENCES section(id),       -- null for self-study
    session_mode TEXT NOT NULL CHECK (session_mode IN ('socratic', 'worked_example', 'practice', 'assessment')),
    language CHAR(5) NOT NULL DEFAULT 'en',
    llm_model TEXT NOT NULL,                      -- e.g. 'llama-3.1-70b-ft-tutor-v2'
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    total_turns INT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_student ON tutoring_session(student_id);
CREATE INDEX idx_session_section ON tutoring_session(section_id);
CREATE INDEX idx_session_started ON tutoring_session(started_at);

-- Conversation turns within a session
CREATE TABLE conversation_turn (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES tutoring_session(id) ON DELETE CASCADE,
    sequence_number INT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('student', 'tutor', 'system')),
    content TEXT NOT NULL,
    content_format TEXT NOT NULL DEFAULT 'text' CHECK (content_format IN ('text', 'latex', 'markdown', 'image_url')),
    assessment_item_id UUID REFERENCES assessment_item(id),
    hint_level SMALLINT,                          -- which hint was shown (0 = none)
    is_correct BOOLEAN,                           -- null if not an answer turn
    detected_misconception_id UUID REFERENCES misconception(id),
    response_latency_ms INT,                      -- time student took to respond
    llm_latency_ms INT,                           -- time LLM took to generate
    token_count INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_turn_session_seq ON conversation_turn(session_id, sequence_number);
CREATE INDEX idx_turn_misconception ON conversation_turn(detected_misconception_id) WHERE detected_misconception_id IS NOT NULL;

-- Student attempts at assessment items within a session
CREATE TABLE student_attempt (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES tutoring_session(id),
    assessment_item_id UUID NOT NULL REFERENCES assessment_item(id),
    student_id UUID NOT NULL REFERENCES user_account(id),
    attempt_number SMALLINT NOT NULL DEFAULT 1,
    student_response JSONB NOT NULL,              -- the raw response
    is_correct BOOLEAN NOT NULL,
    partial_credit NUMERIC(4,3) CHECK (partial_credit BETWEEN 0 AND 1),
    score NUMERIC(6,2),
    hints_requested SMALLINT NOT NULL DEFAULT 0,
    time_spent_seconds INT,
    detected_misconception_id UUID REFERENCES misconception(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attempt_student ON student_attempt(student_id);
CREATE INDEX idx_attempt_session ON student_attempt(session_id);
CREATE INDEX idx_attempt_item ON student_attempt(assessment_item_id);
```

## Student Knowledge Model (BKT-Aligned)

```sql
-- Per-student, per-KC mastery state (Bayesian Knowledge Tracing parameters)
CREATE TABLE knowledge_state (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES user_account(id),
    knowledge_component_id UUID NOT NULL REFERENCES knowledge_component(id),
    p_mastery NUMERIC(5,4) NOT NULL DEFAULT 0.3,  -- P(L_n): probability of mastery
    p_transit NUMERIC(5,4) NOT NULL DEFAULT 0.09,  -- P(T): probability of learning per opportunity
    p_slip NUMERIC(5,4) NOT NULL DEFAULT 0.1,      -- P(S): probability of slip (knows but answers wrong)
    p_guess NUMERIC(5,4) NOT NULL DEFAULT 0.2,     -- P(G): probability of guess (doesn't know but answers right)
    total_attempts INT NOT NULL DEFAULT 0,
    correct_attempts INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    mastery_status TEXT NOT NULL DEFAULT 'not_started' CHECK (mastery_status IN ('not_started', 'learning', 'mastered', 'forgotten')),
    mastery_achieved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (student_id, knowledge_component_id)
);

CREATE INDEX idx_ks_student ON knowledge_state(student_id);
CREATE INDEX idx_ks_kc ON knowledge_state(knowledge_component_id);
CREATE INDEX idx_ks_mastery ON knowledge_state(mastery_status);

-- Historical snapshots of knowledge state for temporal queries
CREATE TABLE knowledge_state_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    knowledge_state_id UUID NOT NULL REFERENCES knowledge_state(id),
    p_mastery NUMERIC(5,4) NOT NULL,
    mastery_status TEXT NOT NULL,
    triggered_by_attempt_id UUID REFERENCES student_attempt(id),
    snapshot_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ks_snapshot_state ON knowledge_state_snapshot(knowledge_state_id, snapshot_at);
```

## Teacher Dashboard & Alerts

```sql
CREATE TABLE teacher_alert (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    teacher_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID NOT NULL REFERENCES section(id),
    alert_type TEXT NOT NULL CHECK (alert_type IN ('student_stuck', 'class_misconception', 'reteach_needed', 'mastery_achieved')),
    severity TEXT NOT NULL DEFAULT 'info' CHECK (severity IN ('info', 'warning', 'critical')),
    title TEXT NOT NULL,
    detail TEXT,
    related_student_id UUID REFERENCES user_account(id),
    related_kc_id UUID REFERENCES knowledge_component(id),
    is_read BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at TIMESTAMPTZ
);

CREATE INDEX idx_alert_teacher ON teacher_alert(teacher_id, is_read);
CREATE INDEX idx_alert_section ON teacher_alert(section_id);

-- Aggregated class mastery view (materialised for dashboard performance)
CREATE MATERIALIZED VIEW mv_class_mastery AS
SELECT
    e.section_id,
    ks.knowledge_component_id,
    COUNT(DISTINCT ks.student_id) AS total_students,
    COUNT(DISTINCT ks.student_id) FILTER (WHERE ks.mastery_status = 'mastered') AS mastered_count,
    COUNT(DISTINCT ks.student_id) FILTER (WHERE ks.mastery_status = 'learning') AS learning_count,
    AVG(ks.p_mastery) AS avg_mastery_probability
FROM enrollment e
JOIN knowledge_state ks ON ks.student_id = e.user_id
WHERE e.role = 'student' AND e.status = 'active'
GROUP BY e.section_id, ks.knowledge_component_id;

CREATE UNIQUE INDEX idx_mv_class_mastery ON mv_class_mastery(section_id, knowledge_component_id);
```

## LTI 1.3 Integration

```sql
CREATE TABLE lti_registration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    client_id TEXT NOT NULL,
    issuer TEXT NOT NULL,                          -- platform issuer URL
    auth_endpoint TEXT NOT NULL,
    token_endpoint TEXT NOT NULL,
    jwks_url TEXT NOT NULL,
    platform_name TEXT,
    public_key TEXT NOT NULL,                     -- our public key (PEM)
    private_key TEXT NOT NULL,                    -- our private key (PEM, encrypted at rest)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lti_deployment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registration_id UUID NOT NULL REFERENCES lti_registration(id),
    deployment_id TEXT NOT NULL,                   -- platform-assigned deployment ID
    context_type TEXT NOT NULL CHECK (context_type IN ('institution', 'course', 'section')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (registration_id, deployment_id)
);

CREATE TABLE lti_launch_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deployment_id UUID NOT NULL REFERENCES lti_deployment(id),
    user_id UUID REFERENCES user_account(id),
    lti_message_type TEXT NOT NULL,
    lti_roles TEXT[] NOT NULL,
    context_id TEXT,
    resource_link_id TEXT,
    launched_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lti_launch_deployment ON lti_launch_log(deployment_id, launched_at);
```

## xAPI Statement Store

```sql
CREATE TABLE xapi_statement (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id UUID NOT NULL UNIQUE,            -- xAPI statement UUID
    actor_id UUID NOT NULL REFERENCES user_account(id),
    verb_id TEXT NOT NULL,                         -- e.g. 'http://adlnet.gov/expapi/verbs/answered'
    verb_display TEXT NOT NULL,                    -- e.g. 'answered'
    object_type TEXT NOT NULL,                     -- 'Activity', 'Agent', 'StatementRef'
    object_id TEXT NOT NULL,                       -- activity IRI
    result_success BOOLEAN,
    result_score_raw NUMERIC(8,2),
    result_score_scaled NUMERIC(5,4),
    result_duration INTERVAL,
    context_registration UUID,
    context_extensions JSONB,
    full_statement JSONB NOT NULL,                 -- complete xAPI JSON for LRS compliance
    stored_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_xapi_actor ON xapi_statement(actor_id);
CREATE INDEX idx_xapi_verb ON xapi_statement(verb_id);
CREATE INDEX idx_xapi_stored ON xapi_statement(stored_at);
CREATE INDEX idx_xapi_object ON xapi_statement(object_id);
```

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES user_account(id),
    institution_id UUID REFERENCES institution(id),
    action TEXT NOT NULL,                          -- 'create', 'read', 'update', 'delete', 'export', 'consent_change'
    entity_type TEXT NOT NULL,                     -- 'user_account', 'knowledge_state', 'tutoring_session', etc.
    entity_id UUID NOT NULL,
    old_value JSONB,
    new_value JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_institution ON audit_log(institution_id, created_at);

-- Data retention policy per institution
CREATE TABLE data_retention_policy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    data_category TEXT NOT NULL,                   -- 'session_transcript', 'knowledge_state', 'audit_log', 'xapi_statement'
    retention_days INT NOT NULL,
    deletion_strategy TEXT NOT NULL DEFAULT 'hard_delete' CHECK (deletion_strategy IN ('hard_delete', 'anonymise', 'archive')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, data_category)
);
```

## Row-Level Security

```sql
-- Enable RLS on key tables
ALTER TABLE user_account ENABLE ROW LEVEL SECURITY;
ALTER TABLE tutoring_session ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation_turn ENABLE ROW LEVEL SECURITY;
ALTER TABLE knowledge_state ENABLE ROW LEVEL SECURITY;
ALTER TABLE student_attempt ENABLE ROW LEVEL SECURITY;

-- Example policy: students see only their own data
CREATE POLICY student_own_data ON tutoring_session
    FOR SELECT
    USING (student_id = current_setting('app.current_user_id')::UUID);

-- Example policy: teachers see students in their sections
CREATE POLICY teacher_section_data ON tutoring_session
    FOR SELECT
    USING (section_id IN (
        SELECT section_id FROM enrollment
        WHERE user_id = current_setting('app.current_user_id')::UUID
        AND role = 'teacher'
    ));
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | institution, user_account, role, user_role, consent_record |
| Curriculum & Knowledge Domain | 5 | curriculum_standard, knowledge_component, kc_prerequisite, kc_standard_alignment, misconception |
| Assessment Items | 3 | assessment_item, item_kc_mapping, item_hint |
| Courses & Enrollment | 3 | course, section, enrollment |
| Tutoring Sessions | 3 | tutoring_session, conversation_turn, student_attempt |
| Student Knowledge Model | 2 | knowledge_state, knowledge_state_snapshot |
| Teacher Dashboard | 1 + 1 MV | teacher_alert, mv_class_mastery (materialised view) |
| LTI Integration | 3 | lti_registration, lti_deployment, lti_launch_log |
| xAPI | 1 | xapi_statement (stores full JSON alongside indexed columns) |
| Audit & Compliance | 2 | audit_log, data_retention_policy |
| **Total** | **28 + 1 MV** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** -- enables distributed ID generation across microservices and aligns with xAPI statement IDs and LTI deployment IDs which are already UUIDs.

2. **Explicit BKT parameters in knowledge_state** -- stores P(mastery), P(transit), P(slip), P(guess) per student-KC pair rather than just a binary mastery flag. This enables the pedagogical engine to update mastery estimates incrementally without re-computing from raw attempts.

3. **knowledge_state_snapshot for temporal queries** -- rather than overwriting mastery estimates, each update creates a snapshot. This supports "what did the student know on date X?" queries required for progress reports and longitudinal research.

4. **Separate misconception table** -- misconceptions are first-class entities linked to knowledge components, not free-text annotations on attempts. This enables systematic tracking of which misconceptions are most common and which remediation strategies work best.

5. **xAPI dual storage** -- the `xapi_statement` table stores both indexed columns (actor, verb, object) for fast querying AND the full JSON statement for LRS compliance. This avoids needing a separate LRS while still enabling xAPI export.

6. **PostgreSQL Row-Level Security** -- multi-tenancy enforced at the database level via RLS policies using `current_setting('app.current_user_id')`. This prevents application-level bugs from leaking data across tenants.

7. **Materialised view for dashboard performance** -- `mv_class_mastery` pre-aggregates student mastery by section and KC, avoiding expensive JOINs on every teacher dashboard load. Refreshed periodically or on demand.

8. **LTI 1.3 registration/deployment separation** -- mirrors the LTI 1.3 spec's distinction between registration (credentials) and deployment (context scope), supporting the Canvas/Moodle model where one registration can have multiple deployments.

9. **Consent records as immutable log** -- consent is never updated in place; new records are inserted for grants, revocations, and re-grants. This provides a complete audit trail for COPPA/GDPR compliance.

10. **Ed-Fi alignment for enrollment entities** -- course, section, and enrollment tables use naming and structure compatible with Ed-Fi v5, enabling straightforward data exchange with US school district systems.
