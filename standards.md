# Standards & API Reference

> Project: AI Tutoring System · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC JTC 1/SC 36 — Information Technology for Learning, Education and Training**
- URL: https://www.iso.org/committee/45392.html
- The primary ISO subcommittee responsible for standards supporting learning, education, and training. Covers metadata for learning resources, competency exchange, and learner data interoperability. Its standards inform data model design for AI tutoring systems.

**ISO/IEC 19788 — Metadata for Learning Resources (MLR)**
- URL: https://www.iso.org/standard/61399.html
- Defines metadata elements for describing learning resources in a technology-neutral manner. Relevant for cataloguing tutoring content, adaptive exercises, and domain knowledge assets so they are discoverable across platforms.

### IEEE Standards

**IEEE 1484.12.1-2020 — Standard for Learning Object Metadata (LOM)**
- URL: https://standards.ieee.org/standard/1484_12_1-2020.html
- Specifies a conceptual data schema for describing learning objects. Defines nine categories of metadata (General, Lifecycle, Technical, Educational, Rights, Relation, Annotation, Classification). An AI tutoring system should tag exercises, explanations, and assessments with LOM-compliant metadata to enable content portability and discovery.

**IEEE P2247 — Adaptive Instructional Systems (AIS)**
- URL: https://sagroups.ieee.org/ltsc/
- Emerging standard from the IEEE Learning Technology Standards Committee defining reference architectures and APIs for adaptive and intelligent instructional systems. Directly applicable to AI tutoring: specifies the Learner Model, Domain Model, Pedagogical Model, and Platform components that form the core architecture of an intelligent tutor.

**IEEE 1484 Series — eLearning Technology Bundle**
- URL: https://webstore.ansi.org/standards/ieee/IEEE1484Series2016
- The broader family of IEEE standards covering computer-managed instruction, runtime environments, and content packaging. Includes IEEE 1484.11.1 (Data Model for Content to LMS communication) and IEEE 1484.12.3 (XML Binding for LOM).

### 1EdTech (IMS Global) Standards

**IMS LTI 1.3 — Learning Tools Interoperability**
- URL: https://www.imsglobal.org/spec/lti/v1p3
- The primary standard for integrating AI tutoring tools with Learning Management Systems (Canvas, Moodle, Blackboard). Uses OpenID Connect for authentication and OAuth 2.0 Client Credentials for service calls. Enables deep linking, grade passback (AGS), roster sync (NRPS), and single sign-on. Essential for institutional deployments.

**1EdTech Security Framework v1.1**
- URL: https://www.imsglobal.org/spec/security/v1p1
- Defines the OAuth 2.0 and OpenID Connect security model underpinning LTI 1.3 and Caliper. Specifies signed JWT tokens, asymmetric key pairs, and token endpoint flows. Required for any compliant LTI tool implementation.

**IMS QTI 3.0 — Question and Test Interoperability**
- URL: https://www.imsglobal.org/spec/qti/v3p0/oview
- Standard XML/JSON data model for assessment items and tests. Supports adaptive questioning (Computer Adaptive Testing), portable custom interactions (HTML5), and result reporting. An AI tutoring system should import/export practice questions in QTI format to enable content exchange with publishers, item banks, and LMS assessment engines.

**IMS Caliper Analytics 1.2**
- URL: https://www.imsglobal.org/spec/caliper/v1p2
- Provides a structured vocabulary and Sensor API for emitting fine-grained learning events (AssessmentEvent, FeedbackEvent, NavigationEvent). Uses JSON-LD and UUID-based event identifiers. Complementary to xAPI for institutional analytics pipelines; enables learning activity data to flow from the AI tutor into campus data warehouses.

**IMS OneRoster 1.1**
- URL: https://www.imsglobal.org/activity/oneroster
- Defines a standard CSV/REST API for syncing rostering data (students, teachers, classes, enrolments) between Student Information Systems and educational tools. An AI tutoring platform needs OneRoster integration to receive class lists and push completion records back to districts.

### W3C & IETF Standards

**Experience API (xAPI) — ADL Initiative**
- URL: https://github.com/adlnet/xAPI-Spec
- Not an IETF standard but a widely adopted open specification maintained by the Advanced Distributed Learning Initiative. Defines a RESTful Learning Record Store (LRS) API and a JSON statement format (Actor-Verb-Object) for capturing any learning experience. Critical for an AI tutoring system: records hint requests, attempt counts, partial credit, mastery transitions, and off-platform practice. Authentication via OAuth 1.0 or HTTP Basic Auth.

**cmi5 — AICC/ADL xAPI Profile**
- URL: https://aicc.github.io/CMI-5_Spec_Current/
- An xAPI profile that adds SCORM-like launch, packaging, and sequencing semantics on top of xAPI. Defines mandatory xAPI statements (launched, initialized, completed, passed, failed, abandoned, waived) and a course structure XML. Useful for distributing AI tutoring modules within enterprise LMS environments that require launch tracking.

**SCORM 2004 4th Edition**
- URL: https://www.adlnet.gov/scorm/
- The legacy eLearning packaging and runtime standard from the ADL Network. Still required by many corporate and government LMS environments. Defines a JavaScript API (SCORM RTE) for content-to-LMS communication. An AI tutoring system should support SCORM export for backwards-compatible deployment, though xAPI/cmi5 is preferred for new integrations.

**W3C Web Accessibility Initiative — WCAG 2.1 (Level AA)**
- URL: https://www.w3.org/TR/WCAG21/
- The accessibility standard referenced by ADA Title II, Section 508, and the European Accessibility Act (EAA, in force June 2025). AI tutoring interfaces must meet WCAG 2.1 AA: keyboard navigation, screen reader compatibility, sufficient colour contrast, and captions for audio/video content. Non-compliance creates legal exposure for institutional customers.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines the JWT format used by LTI 1.3, OAuth 2.0, and OpenID Connect for secure claims transmission. AI tutoring systems receive user identity and context (course, role, locale) in signed JWTs at launch time.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The authorisation framework underlying LTI 1.3 service calls, Google Classroom API, and most modern EdTech integrations. The Client Credentials flow (RFC 6749 §4.4) is used by LTI services; Authorization Code flow is used for user-facing OAuth consent (e.g. Google Classroom).

### Data Model & API Specifications

**Ed-Fi Data Standard v5**
- URL: https://docs.ed-fi.org/reference/data-exchange/data-standard/
- Open REST API and data model standard for K-12 student data. Widely adopted by US state education agencies. Provides canonical representations of students, courses, grades, assessments, and interventions. An AI tutoring system integrated into US public schools should align its data model with Ed-Fi to enable reporting into district data platforms.

**OpenAPI 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The specification for describing REST APIs used by the AI tutoring system's own API surface. Any LTI services, LRS endpoints, or public developer APIs should be documented as OpenAPI 3.1 specs to enable SDK generation and integration testing.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- Used for validating xAPI statements, QTI item packages, and Caliper event payloads. Should be used to define and validate the AI tutoring system's internal data models.

### Security & Privacy Standards

**FERPA — Family Educational Rights and Privacy Act**
- URL: https://studentprivacy.ed.gov/ferpa
- US federal law governing student education records. Vendors must be designated as "school officials" under FERPA, limiting data use to the contracted educational purpose. AI tutoring systems must not use student interaction data for model training without explicit district authorisation. Requires data processing agreements (DPAs) with institutional customers.

**COPPA — Children's Online Privacy Protection Act**
- URL: https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa
- Applies to AI tutoring platforms serving children under 13 in the US. Requires verifiable parental consent before collecting personal data. Platforms operating in school contexts may rely on the "school official" exception, but must limit data collection to educational purposes only.

**GDPR — General Data Protection Regulation**
- URL: https://gdpr.eu/
- EU regulation governing personal data processing. Requires lawful basis for processing student data, data minimisation, right to erasure, and Data Protection Impact Assessments (DPIAs) for high-risk AI processing. Applies to any AI tutoring system with EU users.

**NIST AI Risk Management Framework (AI RMF 1.0)**
- URL: https://airc.nist.gov/RMF
- NIST framework for managing risks specific to AI systems, covering Govern, Map, Measure, and Manage functions. Increasingly referenced by US EdTech procurement as a baseline for AI safety and trustworthiness. Relevant for documenting bias testing, accuracy monitoring, and explainability in the AI tutor's feedback.

---

## Similar Products — Developer Documentation & APIs

### Canvas LMS (Instructure)

- **Description:** The most widely deployed LMS in US higher education; full LTI 1.3 platform with REST API covering courses, assignments, grades, users, and discussions.
- **API Documentation:** https://developerdocs.instructure.com/services/canvas
- **REST API Reference:** https://canvas.instructure.com/doc/api/
- **SDKs/Libraries:** Official Ruby gem; community Python/JS wrappers; LTI 1.3 libraries available for multiple languages.
- **Developer Guide:** https://developerdocs.instructure.com
- **Standards:** REST/JSON, LTI 1.3, OAuth 2.0, xAPI (via plugins), OneRoster
- **Authentication:** OAuth 2.0 (Authorization Code for user tokens; Client Credentials for LTI services)

### Moodle LMS

- **Description:** Open-source LMS with the largest global install base; extensible plugin architecture; full LTI 1.3 support; built-in xAPI/Caliper subsystem.
- **API Documentation:** https://moodledev.io/docs/5.1/apis
- **Web Services Reference:** https://docs.moodle.org/dev/Web_service_API_functions
- **SDKs/Libraries:** PHP-native plugin API; RESTful webservice plugin (community); Python/JS client libraries available.
- **Developer Guide:** https://moodledev.io
- **Standards:** REST/JSON (via plugin), XML-RPC, LTI 1.3, xAPI, QTI
- **Authentication:** Token-based (admin-issued); OAuth 2.0 for external service connections

### Google Classroom API

- **Description:** Google's K-12 classroom management API; manages courses, coursework, student submissions, and announcements within Google Workspace for Education.
- **API Documentation:** https://developers.google.com/classroom
- **REST Reference:** https://developers.google.com/workspace/classroom/reference/rest
- **SDKs/Libraries:** Official client libraries for Python, JavaScript/Node, Java, Go, Ruby, PHP, .NET.
- **Developer Guide:** https://developers.google.com/classroom/quickstart/python
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 (Authorization Code); service accounts for domain-wide delegation

### Anthropic Claude API

- **Description:** LLM API providing access to Claude models for conversational AI, structured outputs, tool use, and long-context reasoning. Well-suited for Socratic dialogue, hint generation, and explanation synthesis in AI tutoring.
- **API Documentation:** https://docs.anthropic.com/en/api/getting-started
- **SDK Links:** Python SDK (anthropic-sdk-python on GitHub); TypeScript/JavaScript SDK; also available via AWS Bedrock and Google Cloud Vertex AI.
- **Developer Guide:** https://docs.anthropic.com/en/home
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** API Key (Bearer token in `x-api-key` header); supports prompt caching for cost reduction on repetitive tutoring contexts.

### OpenAI API

- **Description:** LLM and multi-modal API providing GPT-4o, o3, and other models. Powers Khanmigo and CheggMate. Supports function calling, structured outputs, Assistants API (thread/message management), and fine-tuning.
- **API Documentation:** https://developers.openai.com/api/docs
- **SDKs/Libraries:** Official Python and TypeScript/JavaScript SDKs; community libraries for Java, Go, Ruby, .NET.
- **Developer Guide:** https://developers.openai.com/api/docs/quickstart
- **Standards:** REST/JSON
- **Authentication:** API Key (Bearer token); project-level and organisation-level keys

### Coursera for Campus / Developer API

- **Description:** Coursera's API for enterprise and campus deployments; provides access to course catalogue, enrolment data, learner progress, and grades for integration with institutional systems.
- **API Documentation:** https://dev.coursera.com/get-started
- **Standards:** REST/JSON, OAuth 2.0, LTI 1.1/1.3
- **Authentication:** OAuth 2.0

### Khan Academy API

- **Description:** Public API for accessing Khan Academy's content catalogue, user data, and exercise information. The primary integration path for Khanmigo is via the Canvas LTI plugin rather than a standalone public API.
- **API Documentation:** https://github.com/Khan/khan-api
- **Standards:** REST/JSON
- **Authentication:** OAuth 1.0 (legacy public API)

### ADL xAPI / SCORM Engine (Rustici Software)

- **Description:** Commercial SCORM and xAPI runtime engine used by many LMS vendors to add standards-compliant content playback. Provides reference implementation of xAPI LRS and SCORM RTE.
- **API Documentation:** https://docs.rusticisoftware.com/engine/23.x/ExperienceApi/Overview.html
- **Standards:** xAPI 1.0.3, SCORM 1.2, SCORM 2004, cmi5
- **Authentication:** Basic Auth / API Key

---

## Notes

**Emerging standard — IEEE P2247**: This standard for Adaptive Instructional Systems is still in development as of 2026. Projects should monitor progress at https://sagroups.ieee.org/ltsc/ and design internal architectures (Learner Model, Domain Model, Pedagogical Model) to align with the anticipated API contracts, but should not depend on it for initial interoperability.

**MCP (Model Context Protocol)**: Anthropic's open protocol for connecting AI models to external tools and data sources is increasingly relevant for AI tutoring systems that need to give the LLM access to student performance records, curriculum databases, and exercise repositories at inference time. The MCP specification is maintained at https://modelcontextprotocol.io and enables structured tool calls without custom prompt engineering.

**Knowledge Graph & Ontology tooling**: No single ISO or IEEE standard governs educational ontologies, but OWL 2 (W3C, https://www.w3.org/TR/owl2-overview/) and RDF Schema (https://www.w3.org/TR/rdf-schema/) are the de facto standards for defining domain and learner model ontologies in research implementations. Projects should track emerging work in knowledge graph-enhanced RAG (KG-RAG) for tutoring systems.

**Privacy regulation divergence**: FERPA (US), GDPR (EU), and COPPA (US, under-13) have meaningfully different requirements for consent, data minimisation, and deletion rights. An AI tutoring system targeting multiple geographies must implement a configurable privacy layer; a single global policy will not satisfy all jurisdictions.
