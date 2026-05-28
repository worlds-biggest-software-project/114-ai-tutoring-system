# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: AI Tutoring System · Created: 2026-05-19

## Philosophy

This model combines PostgreSQL relational tables for transactional CRUD operations with a property graph layer for the relationship-heavy aspects of an AI tutoring system: knowledge prerequisite chains, misconception networks, learning pathways, student-to-concept mastery graphs, and curriculum standard hierarchies. The graph layer uses PostgreSQL's `ltree` extension for hierarchical data and dedicated `graph_node` / `graph_edge` tables for arbitrary relationships, avoiding a separate graph database while enabling graph traversal queries.

The core insight is that an AI tutoring system is fundamentally a graph problem: knowledge components form prerequisite chains, misconceptions relate to concepts in non-trivial ways, curriculum standards form deep hierarchies, students have mastery relationships with hundreds of concepts, and adaptive sequencing requires traversing these graphs in real time. A purely relational model forces these graph queries into recursive CTEs that become unwieldy; a graph-native layer makes them natural.

This approach draws from the ACT-R cognitive architecture behind Carnegie Learning's MATHia (which models knowledge as a network of production rules), knowledge graph-enhanced RAG systems used in modern AI tutoring (connecting LLM retrieval to structured domain knowledge), and the W3C RDF/OWL ontology patterns used in educational knowledge representation research.

**Best for:** Platforms with complex prerequisite chains, misconception networks, adaptive learning path generation, and curriculum standard hierarchies that require efficient graph traversal.

**Trade-offs:**
- (+) Natural representation of prerequisite chains, learning pathways, and knowledge hierarchies
- (+) Efficient graph traversal queries for adaptive sequencing ("what should the student learn next?")
- (+) `ltree` enables fast ancestor/descendant queries on curriculum hierarchies without recursive CTEs
- (+) Misconception-concept-remediation networks enable sophisticated diagnostic reasoning
- (+) Can power knowledge graph-enhanced RAG for the Socratic dialogue engine
- (-) More complex schema with dual relational + graph layers
- (-) Graph consistency must be maintained across both layers
- (-) `ltree` and graph patterns are PostgreSQL-specific; harder to port to other databases
- (-) Developers need to understand both relational and graph query patterns
- (-) Graph edge tables can grow very large with many students x many knowledge components

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C RDF / OWL 2 | Graph node/edge model is compatible with RDF triple patterns; enables future linked data export |
| IEEE P2247 (AIS) | Domain Model represented as a knowledge graph; Learner Model as student-node-to-KC-node edges |
| Ed-Fi Data Standard v5 | Enrollment and assessment entities in relational tables compatible with Ed-Fi |
| IMS QTI 3.0 | Assessment items as relational entities; KC mappings as graph edges |
| xAPI | Graph edges encode xAPI-compatible relationships (student-attempted-item, student-mastered-KC) |
| IMS LTI 1.3 | LTI configuration in relational tables |
| FERPA / COPPA | Audit log in relational tables; graph queries enable impact analysis for data deletion |

---

## Graph Layer

```sql
-- Enable ltree extension for hierarchical data
CREATE EXTENSION IF NOT EXISTS ltree;

-- Graph node types (reference table)
CREATE TABLE graph_node_type (
    name TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    properties_schema JSONB                       -- JSON Schema for node properties
);

INSERT INTO graph_node_type (name, description) VALUES
    ('knowledge_component', 'A fine-grained skill or concept tracked by the tutor'),
    ('misconception', 'A known incorrect mental model or reasoning error'),
    ('curriculum_standard', 'An education standard (CCSS, NGSS, state standard)'),
    ('assessment_item', 'A question or problem used in tutoring'),
    ('remediation_strategy', 'An approach for correcting a misconception'),
    ('learning_objective', 'A high-level goal composed of multiple KCs'),
    ('subject_area', 'A top-level subject domain'),
    ('student', 'A learner node (linked to user_account)');

-- Graph edge types (reference table)
CREATE TABLE graph_edge_type (
    name TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    source_type TEXT NOT NULL REFERENCES graph_node_type(name),
    target_type TEXT NOT NULL REFERENCES graph_node_type(name),
    properties_schema JSONB
);

INSERT INTO graph_edge_type (name, description, source_type, target_type) VALUES
    ('requires', 'KC A requires mastery of KC B before attempting', 'knowledge_component', 'knowledge_component'),
    ('recommends', 'KC A benefits from prior study of KC B', 'knowledge_component', 'knowledge_component'),
    ('aligned_to', 'KC is aligned to a curriculum standard', 'knowledge_component', 'curriculum_standard'),
    ('tests', 'Assessment item tests a knowledge component', 'assessment_item', 'knowledge_component'),
    ('indicates', 'Wrong answer pattern indicates a misconception', 'assessment_item', 'misconception'),
    ('targets', 'Misconception targets a specific KC', 'misconception', 'knowledge_component'),
    ('remediates', 'Strategy addresses a misconception', 'remediation_strategy', 'misconception'),
    ('composed_of', 'Learning objective composed of KCs', 'learning_objective', 'knowledge_component'),
    ('belongs_to', 'KC belongs to a subject area', 'knowledge_component', 'subject_area'),
    ('parent_standard', 'Standard is a child of another standard', 'curriculum_standard', 'curriculum_standard'),
    ('has_mastery', 'Student has mastery relationship with KC', 'student', 'knowledge_component'),
    ('exhibited', 'Student exhibited a misconception', 'student', 'misconception');

-- The graph nodes
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type TEXT NOT NULL REFERENCES graph_node_type(name),
    external_id UUID,                             -- links to relational entity (user_account.id, assessment_item.id, etc.)
    label TEXT NOT NULL,                          -- human-readable label
    properties JSONB NOT NULL DEFAULT '{}',       -- node-specific properties
    -- ltree path for hierarchical nodes (curriculum standards, subject areas)
    hierarchy_path LTREE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_external ON graph_node(external_id);
CREATE INDEX idx_gnode_hierarchy ON graph_node USING GIST (hierarchy_path) WHERE hierarchy_path IS NOT NULL;
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- The graph edges
CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_type TEXT NOT NULL REFERENCES graph_edge_type(name),
    source_node_id UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    weight NUMERIC(5,4) DEFAULT 1.0,              -- edge weight (mastery probability, confidence, strength)
    properties JSONB NOT NULL DEFAULT '{}',       -- edge-specific properties
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(), -- temporal validity
    valid_to TIMESTAMPTZ,                         -- null = currently valid
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (source_node_id != target_node_id)
);

CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_gedge_temporal ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties jsonb_path_ops);

-- Unique constraint: prevent duplicate active edges of the same type
CREATE UNIQUE INDEX idx_gedge_unique_active
    ON graph_edge(edge_type, source_node_id, target_node_id)
    WHERE valid_to IS NULL;
```

## Relational Layer: Identity & Enrollment

```sql
CREATE TABLE institution (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    institution_type TEXT NOT NULL,
    jurisdiction_country CHAR(2) NOT NULL,
    privacy_framework TEXT NOT NULL DEFAULT 'ferpa',
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    graph_node_id UUID REFERENCES graph_node(id), -- link to graph layer (for students)
    email TEXT,
    display_name TEXT NOT NULL,
    roles TEXT[] NOT NULL DEFAULT '{"student"}',
    locale TEXT NOT NULL DEFAULT 'en',
    is_minor BOOLEAN NOT NULL DEFAULT false,
    profile JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_institution ON user_account(institution_id);
CREATE INDEX idx_user_graph_node ON user_account(graph_node_id) WHERE graph_node_id IS NOT NULL;

CREATE TABLE consent_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    minor_user_id UUID NOT NULL REFERENCES user_account(id),
    parent_user_id UUID REFERENCES user_account(id),
    consent_type TEXT NOT NULL,
    status TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    granted_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    title TEXT NOT NULL,
    subject TEXT NOT NULL,
    grade_level TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE section (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES course(id),
    title TEXT NOT NULL,
    term TEXT,
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
    UNIQUE (user_id, section_id)
);

CREATE INDEX idx_enrollment_section ON enrollment(section_id);
```

## Relational Layer: Assessment Items

```sql
CREATE TABLE assessment_item (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id UUID REFERENCES graph_node(id), -- link to graph layer
    subject TEXT NOT NULL,
    item_type TEXT NOT NULL,
    content JSONB NOT NULL,                       -- QTI-compatible item content
    max_score NUMERIC(6,2) NOT NULL DEFAULT 1.0,
    language CHAR(5) NOT NULL DEFAULT 'en',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Relational Layer: Tutoring Sessions

```sql
CREATE TABLE tutoring_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES user_account(id),
    section_id UUID REFERENCES section(id),
    session_mode TEXT NOT NULL,
    language CHAR(5) NOT NULL DEFAULT 'en',
    llm_model TEXT NOT NULL,
    metrics JSONB NOT NULL DEFAULT '{}',
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_student ON tutoring_session(student_id);
CREATE INDEX idx_session_section ON tutoring_session(section_id);

CREATE TABLE conversation_turn (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES tutoring_session(id) ON DELETE CASCADE,
    sequence_number INT NOT NULL,
    role TEXT NOT NULL,
    turn_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_turn_session_seq ON conversation_turn(session_id, sequence_number);

CREATE TABLE student_attempt (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES tutoring_session(id),
    assessment_item_id UUID NOT NULL REFERENCES assessment_item(id),
    student_id UUID NOT NULL REFERENCES user_account(id),
    attempt_number SMALLINT NOT NULL DEFAULT 1,
    student_response JSONB NOT NULL,
    is_correct BOOLEAN NOT NULL,
    partial_credit NUMERIC(4,3),
    hints_requested SMALLINT NOT NULL DEFAULT 0,
    time_spent_seconds INT,
    -- Misconception detection stored here AND as graph edge
    detected_misconception_node_id UUID REFERENCES graph_node(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attempt_student ON student_attempt(student_id);
CREATE INDEX idx_attempt_session ON student_attempt(session_id);
```

## Relational Layer: Integration & Audit

```sql
CREATE TABLE lti_registration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institution(id),
    client_id TEXT NOT NULL,
    issuer TEXT NOT NULL,
    platform_config JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE xapi_statement (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id UUID NOT NULL UNIQUE,
    actor_id UUID NOT NULL REFERENCES user_account(id),
    verb_id TEXT NOT NULL,
    object_id TEXT NOT NULL,
    statement JSONB NOT NULL,
    stored_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_actor ON xapi_statement(actor_id, stored_at);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES user_account(id),
    institution_id UUID REFERENCES institution(id),
    action TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

## Graph Query Examples

```sql
-- QUERY 1: Find all prerequisites for a knowledge component (transitive closure)
-- "What must a student know before attempting 'quadratic_formula'?"
WITH RECURSIVE prereqs AS (
    -- Base case: direct prerequisites
    SELECT ge.target_node_id AS kc_node_id, gn.label, 1 AS depth
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.target_node_id
    WHERE ge.source_node_id = (SELECT id FROM graph_node WHERE label = 'quadratic_formula' AND node_type = 'knowledge_component')
      AND ge.edge_type = 'requires'
      AND ge.valid_to IS NULL

    UNION ALL

    -- Recursive case: prerequisites of prerequisites
    SELECT ge.target_node_id, gn.label, p.depth + 1
    FROM prereqs p
    JOIN graph_edge ge ON ge.source_node_id = p.kc_node_id
    JOIN graph_node gn ON gn.id = ge.target_node_id
    WHERE ge.edge_type = 'requires'
      AND ge.valid_to IS NULL
      AND p.depth < 10  -- prevent infinite loops
)
SELECT DISTINCT kc_node_id, label, MIN(depth) AS min_depth
FROM prereqs
GROUP BY kc_node_id, label
ORDER BY min_depth;

-- QUERY 2: Recommend next KC for a student based on mastery and prerequisites
-- "What should student X learn next?"
WITH student_mastered AS (
    -- KCs the student has mastered
    SELECT ge.target_node_id AS kc_node_id
    FROM graph_edge ge
    WHERE ge.source_node_id = (SELECT graph_node_id FROM user_account WHERE id = 'student-uuid')
      AND ge.edge_type = 'has_mastery'
      AND ge.valid_to IS NULL
      AND ge.weight >= 0.85  -- mastery threshold
),
ready_to_learn AS (
    -- KCs where ALL prerequisites are mastered
    SELECT gn.id AS kc_node_id, gn.label, gn.properties->>'difficulty_level' AS difficulty
    FROM graph_node gn
    WHERE gn.node_type = 'knowledge_component'
      AND gn.id NOT IN (SELECT kc_node_id FROM student_mastered)
      AND NOT EXISTS (
          -- No unmastered prerequisites
          SELECT 1 FROM graph_edge ge
          WHERE ge.source_node_id = gn.id
            AND ge.edge_type = 'requires'
            AND ge.valid_to IS NULL
            AND ge.target_node_id NOT IN (SELECT kc_node_id FROM student_mastered)
      )
)
SELECT * FROM ready_to_learn
ORDER BY difficulty::INT ASC
LIMIT 5;

-- QUERY 3: Find all misconceptions related to a KC and their remediation strategies
SELECT misc.label AS misconception,
       misc.properties->>'description' AS description,
       strat.label AS remediation_strategy,
       strat.properties->>'content' AS strategy_content
FROM graph_node kc
JOIN graph_edge e1 ON e1.target_node_id = kc.id AND e1.edge_type = 'targets'
JOIN graph_node misc ON misc.id = e1.source_node_id AND misc.node_type = 'misconception'
LEFT JOIN graph_edge e2 ON e2.target_node_id = misc.id AND e2.edge_type = 'remediates'
LEFT JOIN graph_node strat ON strat.id = e2.source_node_id AND strat.node_type = 'remediation_strategy'
WHERE kc.label = 'solving_linear_equations'
  AND e1.valid_to IS NULL;

-- QUERY 4: Curriculum standard hierarchy using ltree
-- "Find all standards under the 8th grade math branch"
SELECT label, hierarchy_path, properties->>'code' AS standard_code
FROM graph_node
WHERE node_type = 'curriculum_standard'
  AND hierarchy_path <@ 'ccss.math.grade8'::ltree
ORDER BY hierarchy_path;

-- "Find the full path from a standard to the root"
SELECT label, hierarchy_path
FROM graph_node
WHERE node_type = 'curriculum_standard'
  AND hierarchy_path @> (
      SELECT hierarchy_path FROM graph_node
      WHERE properties->>'code' = 'CCSS.MATH.CONTENT.8.EE.A.1'
  )
ORDER BY hierarchy_path;

-- QUERY 5: Student misconception network
-- "What misconceptions has student X exhibited, and what KCs do they affect?"
SELECT misc.label AS misconception,
       ge_exhibited.properties->>'confidence' AS detection_confidence,
       ge_exhibited.properties->>'last_seen' AS last_seen,
       kc.label AS affected_kc,
       mastery_edge.weight AS current_mastery
FROM graph_edge ge_exhibited
JOIN graph_node misc ON misc.id = ge_exhibited.target_node_id AND misc.node_type = 'misconception'
JOIN graph_edge ge_targets ON ge_targets.source_node_id = misc.id AND ge_targets.edge_type = 'targets'
JOIN graph_node kc ON kc.id = ge_targets.target_node_id AND kc.node_type = 'knowledge_component'
LEFT JOIN graph_edge mastery_edge ON mastery_edge.source_node_id = ge_exhibited.source_node_id
    AND mastery_edge.target_node_id = kc.id
    AND mastery_edge.edge_type = 'has_mastery'
    AND mastery_edge.valid_to IS NULL
WHERE ge_exhibited.source_node_id = (SELECT graph_node_id FROM user_account WHERE id = 'student-uuid')
  AND ge_exhibited.edge_type = 'exhibited'
  AND ge_exhibited.valid_to IS NULL
ORDER BY ge_exhibited.properties->>'last_seen' DESC;
```

## Populating the Knowledge Graph

```sql
-- Example: building the math prerequisite graph
-- Insert KC nodes
INSERT INTO graph_node (id, node_type, label, properties) VALUES
    ('uuid-kc-add', 'knowledge_component', 'addition_integers', '{"subject": "math", "difficulty_level": 2, "grade_band": "3-5"}'),
    ('uuid-kc-sub', 'knowledge_component', 'subtraction_integers', '{"subject": "math", "difficulty_level": 2, "grade_band": "3-5"}'),
    ('uuid-kc-mult', 'knowledge_component', 'multiplication_integers', '{"subject": "math", "difficulty_level": 3, "grade_band": "3-5"}'),
    ('uuid-kc-div', 'knowledge_component', 'division_integers', '{"subject": "math", "difficulty_level": 3, "grade_band": "3-5"}'),
    ('uuid-kc-lineq', 'knowledge_component', 'solving_linear_equations', '{"subject": "math", "difficulty_level": 5, "grade_band": "6-8"}'),
    ('uuid-kc-quad', 'knowledge_component', 'quadratic_formula', '{"subject": "math", "difficulty_level": 7, "grade_band": "9-12"}');

-- Insert prerequisite edges
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id) VALUES
    ('requires', 'uuid-kc-mult', 'uuid-kc-add'),     -- multiplication requires addition
    ('requires', 'uuid-kc-div', 'uuid-kc-mult'),      -- division requires multiplication
    ('requires', 'uuid-kc-lineq', 'uuid-kc-add'),     -- linear equations require addition
    ('requires', 'uuid-kc-lineq', 'uuid-kc-sub'),     -- linear equations require subtraction
    ('requires', 'uuid-kc-lineq', 'uuid-kc-mult'),    -- linear equations require multiplication
    ('requires', 'uuid-kc-lineq', 'uuid-kc-div'),     -- linear equations require division
    ('requires', 'uuid-kc-quad', 'uuid-kc-lineq');    -- quadratic formula requires linear equations

-- Insert misconception nodes and edges
INSERT INTO graph_node (id, node_type, label, properties) VALUES
    ('uuid-misc-neg', 'misconception', 'distribute_negative_sign',
     '{"description": "Failure to distribute negative sign across terms", "severity": "critical"}');

INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, properties) VALUES
    ('targets', 'uuid-misc-neg', 'uuid-kc-lineq', '{"frequency": "high"}');

-- Insert remediation strategy
INSERT INTO graph_node (id, node_type, label, properties) VALUES
    ('uuid-rem-neg', 'remediation_strategy', 'visual_distribution_model',
     '{"content": "Show expanded form with parentheses before simplifying", "modality": "visual+worked_example"}');

INSERT INTO graph_edge (edge_type, source_node_id, target_node_id) VALUES
    ('remediates', 'uuid-rem-neg', 'uuid-misc-neg');

-- Insert curriculum standard hierarchy using ltree
INSERT INTO graph_node (id, node_type, label, hierarchy_path, properties) VALUES
    ('uuid-std-root', 'curriculum_standard', 'Common Core State Standards', 'ccss', '{"framework": "CCSS"}'),
    ('uuid-std-math', 'curriculum_standard', 'Mathematics', 'ccss.math', '{"framework": "CCSS"}'),
    ('uuid-std-g8', 'curriculum_standard', 'Grade 8', 'ccss.math.grade8', '{"framework": "CCSS"}'),
    ('uuid-std-ee', 'curriculum_standard', 'Expressions & Equations', 'ccss.math.grade8.ee', '{"framework": "CCSS"}'),
    ('uuid-std-eea1', 'curriculum_standard', 'Apply exponent properties', 'ccss.math.grade8.ee.a1',
     '{"framework": "CCSS", "code": "CCSS.MATH.CONTENT.8.EE.A.1"}');

-- Link KC to standard via graph edge
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, properties) VALUES
    ('aligned_to', 'uuid-kc-lineq', 'uuid-std-ee', '{"alignment_strength": "primary"}');

-- Record student mastery as a graph edge
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, properties) VALUES
    ('has_mastery', 'student-graph-node-uuid', 'uuid-kc-add', 0.95,
     '{"status": "mastered", "total_attempts": 20, "correct_attempts": 18, "bkt": {"p_transit": 0.09, "p_slip": 0.05, "p_guess": 0.20}}');
```

## Updating Mastery (Temporal Edges)

```sql
-- When mastery changes, close the old edge and create a new one
-- This provides a complete temporal history of mastery progression

-- Close the old mastery edge
UPDATE graph_edge
SET valid_to = now()
WHERE source_node_id = 'student-graph-node-uuid'
  AND target_node_id = 'uuid-kc-lineq'
  AND edge_type = 'has_mastery'
  AND valid_to IS NULL;

-- Create new mastery edge with updated weight
INSERT INTO graph_edge (edge_type, source_node_id, target_node_id, weight, properties) VALUES
    ('has_mastery', 'student-graph-node-uuid', 'uuid-kc-lineq', 0.62,
     '{"status": "learning", "total_attempts": 8, "correct_attempts": 5, "bkt": {"p_transit": 0.09, "p_slip": 0.10, "p_guess": 0.20}}');

-- Query mastery at a specific point in time
SELECT target_node_id, weight AS p_mastery, properties->>'status' AS status
FROM graph_edge
WHERE source_node_id = 'student-graph-node-uuid'
  AND edge_type = 'has_mastery'
  AND valid_from <= '2026-03-15'
  AND (valid_to IS NULL OR valid_to > '2026-03-15');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 4 | graph_node_type, graph_edge_type, graph_node, graph_edge |
| Identity & Multi-Tenancy | 3 | institution, user_account, consent_record |
| Courses & Enrollment | 3 | course, section, enrollment |
| Assessment Items | 1 | assessment_item (linked to graph via graph_node_id) |
| Tutoring Sessions | 3 | tutoring_session, conversation_turn, student_attempt |
| Integration | 2 | lti_registration, xapi_statement |
| Audit | 1 | audit_log |
| **Total** | **17** | Knowledge components, misconceptions, standards, and mastery all live in the graph layer |

---

## Key Design Decisions

1. **Graph layer in PostgreSQL, not a separate database** -- using `graph_node` / `graph_edge` tables with PostgreSQL's recursive CTEs and `ltree` avoids the operational complexity of running Neo4j or similar alongside PostgreSQL. For the expected scale (thousands of KCs, millions of mastery edges), PostgreSQL handles this well.

2. **Typed nodes and edges with schema registry** -- `graph_node_type` and `graph_edge_type` tables enforce which edge types can connect which node types, providing structural integrity in the graph without a graph schema language.

3. **Temporal edges for mastery tracking** -- mastery is modeled as `has_mastery` edges with `valid_from` / `valid_to` timestamps. Old edges are closed (not deleted) when mastery changes, providing full temporal history without a separate snapshot table.

4. **`ltree` for curriculum hierarchies** -- curriculum standards form deep trees (CCSS > Math > Grade 8 > Expressions & Equations > specific standard). `ltree` enables instant ancestor/descendant queries using `@>` and `<@` operators, vastly more efficient than recursive CTEs on adjacency lists.

5. **Student nodes in the graph** -- each student has a `graph_node` linked to their `user_account`. This enables graph queries like "find all misconceptions exhibited by students in section X" using pure graph traversal.

6. **Misconception-KC-remediation as a subgraph** -- misconceptions, knowledge components, and remediation strategies form a rich subgraph that the Socratic dialogue engine can traverse at inference time: detect misconception -> find targeted KC -> select remediation strategy.

7. **Knowledge graph as RAG context** -- the graph layer can feed the LLM's Socratic engine via knowledge graph-enhanced RAG. When a student answers incorrectly, the system traverses the graph to find the misconception, its related KC, prerequisite gaps, and available remediation strategies, then passes this structured context to the LLM.

8. **Relational for CRUD, graph for traversal** -- sessions, conversation turns, and enrollment are relational because they are primarily accessed by direct lookup or simple filtering. Knowledge relationships and mastery state are in the graph because they are primarily accessed by traversal.

9. **Edge weights as mastery probabilities** -- the `weight` field on `has_mastery` edges stores the BKT P(mastery), enabling graph queries that filter by mastery threshold (e.g., "find KCs with mastery below 0.5 that have prerequisites above 0.85").

10. **Dual-indexed node properties** -- graph nodes have both PostgreSQL GIN-indexed JSONB `properties` for flexible metadata and `ltree`-indexed `hierarchy_path` for hierarchical data. This supports both containment queries and path queries without compromise.
