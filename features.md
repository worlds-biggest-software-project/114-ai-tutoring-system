# AI Tutoring System — Feature & Functionality Survey

> Candidate #114 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Khanmigo (Khan Academy) | Socratic AI tutor (K-12) | Freemium / institutional (proprietary) | khanacademy.org/khan-labs |
| Carnegie Learning MATHia | Cognitive AI math tutor (grades 6–12) | Commercial SaaS (proprietary) | carnegielearning.com/products/software-platform/math-software |
| Duolingo Max | AI language-learning tutor | Freemium + subscription (proprietary) | duolingo.com/max |
| Chegg Study / CheggMate | Homework help + AI tutoring (higher-ed) | SaaS subscription (proprietary) | chegg.com |
| Socratic (Google) | AI homework help app (K-12) | Freemium (proprietary) | socratic.org |
| Photomath (Google) | AI math problem-solver (camera input) | Freemium + subscription (proprietary) | photomath.com |
| Wolfram Alpha / Wolfram Problem Generator | Computational knowledge + step-by-step STEM | Freemium + subscription (proprietary) | wolframalpha.com |
| Synthesis Tutor | Game-based critical thinking (ages 7–14) | Subscription (proprietary) | synthesis.com |
| Numerade | AI STEM tutoring video platform (higher-ed) | Subscription (proprietary) | numerade.com |
| Cognii | Conversational AI natural-language assessment | Enterprise SaaS (proprietary) | cognii.com |

## Feature Analysis by Solution

### Khanmigo (Khan Academy)

**Core features**
- Socratic AI tutor built on GPT-4 that guides learners through problems by asking questions rather than providing direct answers
- K-12 coverage across math, science, humanities, SAT/ACT prep, and computer science
- Tutor mode: step-by-step guided problem solving with Socratic questioning scaffolds
- Essay feedback: detailed writing suggestions on structure, evidence, and argumentation
- Teacher tools: Khanmigo for Teachers generates lesson plans, class discussion questions, and student feedback drafts
- Safety guardrails: content filtering and session monitoring appropriate for K-12 audiences

**Differentiating features**
- Only at-scale Socratic AI tutor available free to all learners; 1.4M users and 380+ district partners
- $42M federal grant deploying to 2.1M Title I students — validated equity-at-scale use case
- Deliberately withholds direct answers — pedagogically distinct from Photomath and Chegg which show solutions immediately
- Multi-subject coverage within a single trusted educational brand

**UX patterns**
- Chat interface embedded alongside Khan Academy exercises; Khanmigo appears as a helper panel during practice
- Conversational question-posing style; responses are warm and encouraging in tone
- Teacher-facing mode with separate workflows for lesson planning and feedback generation

**Integration points**
- LTI integration for district LMS embedding
- IMS OneRoster for school roster import
- Google Classroom assignment sync

**Known gaps**
- Safety filters can restrict depth of response on nuanced or sensitive topics relevant to older learners
- GPT-4 base model means hallucination risk on niche or highly technical subject matter
- No persistent long-term student knowledge model across sessions and years
- Limited coverage of university-level or advanced professional content

**Licence / IP notes**
- Khanmigo is a proprietary AI product; Khan Academy platform content is CC BY-NC-SA 4.0 (non-commercial use)

---

### Carnegie Learning MATHia

**Core features**
- Step-by-step adaptive math tutoring for grades 6–12 using the ACT-R cognitive architecture
- Contextual hint system: multiple hint levels that scaffold without revealing the final answer
- Mastery-based progression: students must demonstrate consistent proficiency before advancing
- LiveLab: real-time teacher dashboard showing each student's current skill, time-on-task, and stuck state
- Fine-grained student knowledge model tracking 400+ individual math skills
- Automatic identification of stuck students and recommendation for teacher intervention

**Differentiating features**
- Deepest research foundation of any reviewed AI tutor: 30+ years of CMU cognitive tutor research and published efficacy data
- 400-skill cognitive model is the most granular student knowledge representation in K-12 math
- LiveLab real-time class visibility is unique — teachers see students' current struggle in real time
- Blended design: MATHia works alongside Carnegie Learning's printed curriculum, not as a replacement

**UX patterns**
- Text-based problem interface with scaffolded hint button; step-by-step worked examples
- Progress indicators: mastery bars per skill and section completion
- Teacher-facing LiveLab: colour-coded grid showing each student's current activity and mastery state

**Integration points**
- LTI 1.3 for LMS grade passback
- IMS OneRoster for roster management
- SIS integration for district demographic data

**Known gaps**
- Mathematics only — no cross-subject tutoring
- Text-heavy interface is less engaging than game-based or conversational alternatives for younger learners
- Requires significant teacher professional development for effective blended implementation
- High per-student cost ($20–$30/student/year) relative to free alternatives

**Licence / IP notes**
- Proprietary commercial SaaS; cognitive model and content are not available under open licence

---

### Duolingo Max

**Core features**
- AI language learning with GPT-4-powered conversation practice (Roleplay feature)
- AI-powered video call simulation with virtual characters for speaking practice
- Personalised lesson sequencing based on spaced repetition and learner error patterns
- Explain My Answer: AI-generated explanations for why an answer is right or wrong
- Leaderboards, streaks, and XP gamification driving daily engagement
- 40+ languages with adaptive difficulty calibration

**Differentiating features**
- Largest active user base of any AI tutoring platform (113M MAU) — unique at-scale behavioural data for model training
- Roleplay conversations with AI characters simulate real-world language scenarios
- Gamification and habit-forming engagement mechanics more sophisticated than any other reviewed AI tutor
- Daily learning streak is a core retention driver with measurable completion rate impact

**UX patterns**
- Mobile-first design with daily lesson sessions of 5–20 minutes
- Bite-sized exercise variety: translation, listening, speaking, reading, matching
- Max tier surfaces AI explanations and roleplay in a clearly differentiated UX layer above standard lessons

**Integration points**
- Duolingo for Schools: teacher dashboard for classroom assignment and progress tracking
- No LTI or LMS integration; primarily consumer-direct

**Known gaps**
- Language learning only — not applicable to STEM, writing, or other academic subjects
- Roleplays are engaging but not deeply Socratic; AI provides conversation practice, not pedagogical scaffolding
- Gamification can incentivise streak maintenance over deep learning
- No persistent long-term language proficiency model exportable to external systems

**Licence / IP notes**
- Proprietary commercial platform; user data retained and used for model improvement (governed by Duolingo privacy policy)

---

### Chegg Study / CheggMate

**Core features**
- Subject Q&A library: expert-written step-by-step solutions to textbook problems
- CheggMate: AI tutor (GPT-4 powered) providing contextual explanations and guided problem-solving
- Textbook solution manuals for 9,000+ textbooks across STEM and business subjects
- Writing assistance: grammar checking, citation formatting, and essay review
- 24/7 on-demand access without human tutor scheduling constraints

**Differentiating features**
- Textbook solution manual depth — millions of worked solutions mapped to specific textbook editions
- CheggMate AI contextualises explanations relative to the specific textbook problem a student is working on
- Established college student market presence; brand recognition among US undergraduates

**UX patterns**
- Search-first interface: students search or photograph a problem to find matching solutions
- CheggMate chat interface layered over existing Q&A database
- Mobile app with camera-based problem search (similar to Photomath)

**Integration points**
- No LMS or institutional integration; consumer-direct subscription only

**Known gaps**
- Business model under severe pressure from ChatGPT; stock declined ~90% from 2021 peak by 2024
- Core Q&A product can facilitate academic dishonesty — significant institutional risk
- CheggMate answers can be too direct rather than pedagogically scaffolded
- Limited to higher-education subject matter; not suitable for K-12

**Licence / IP notes**
- Proprietary SaaS; content licensing from textbook publishers creates ongoing IP exposure

---

### Socratic (Google)

**Core features**
- Camera-based problem recognition: photograph a question and receive step-by-step explanations
- Multi-subject coverage: math, science, social studies, English, and history for K-12
- Visual explanations: diagrams, charts, and illustrated step-by-step solutions
- AI-generated explanations supplemented by curated web resources (Wikipedia, Khan Academy links)
- Free with no advertising — funded by Google

**Differentiating features**
- Camera-input-first UX — most frictionless problem entry of any reviewed tool
- Broad K-12 subject coverage (not limited to STEM) including history and English
- Entirely free with no subscription required; Google backing ensures continued investment
- Google Lens integration for seamless camera problem capture on Android

**UX patterns**
- Single-screen camera UI with instant visual result display
- Results page: visual explanation at top, curated supplementary resources below
- No login required for basic use; reduces friction to zero for first-time users

**Integration points**
- Google ecosystem integration (Lens, Search) on Android
- No LMS, LTI, or institutional integration

**Known gaps**
- No Socratic scaffolding — provides answers and explanations, not guided questioning
- No student progress tracking, persistence, or teacher visibility
- Explanations can be surface-level for advanced topics
- No community, assignment, or institutional features

**Licence / IP notes**
- Proprietary Google product; free to use; user data governed by Google Privacy Policy

---

### Photomath (Google)

**Core features**
- Camera-based math problem recognition and instant step-by-step solution display
- Animated step-by-step explanations showing each algebraic transformation
- Multiple solution methods shown for the same problem (e.g., factoring vs. quadratic formula)
- Word problem solver using AI to extract mathematical structure from text
- Photomath Plus: additional explanation depth, hints, and animated walkthroughs

**Differentiating features**
- 300M+ downloads — largest consumer install base of any math AI tool reviewed
- Multiple solution methods per problem — pedagogically useful for showing different approaches
- Animated explanations of individual algebraic steps are visually clearer than text-only competitors
- Google acquisition (2023) ensures sustained investment and integration with Google's education ecosystem

**UX patterns**
- Camera scan → instant result on a single screen; lowest friction math help experience reviewed
- Step-by-step animated expansion of each solution step
- Handwritten equation recognition alongside printed text

**Integration points**
- Google Classroom integration roadmap (post-Google acquisition)
- No LTI or institutional integration currently

**Known gaps**
- Math only — no coverage of science, writing, history, or other subjects
- Instant solution delivery without Socratic scaffolding encourages answer-copying rather than learning
- No student knowledge model or progress tracking
- Academic integrity concerns in institutional settings

**Licence / IP notes**
- Proprietary Google product (acquired 2023); free basic tier with Plus subscription for advanced features

---

### Wolfram Alpha / Wolfram Problem Generator

**Core features**
- Computational knowledge engine with step-by-step solutions for math, science, engineering, and data analysis
- Wolfram Problem Generator: generates randomised practice problems across mathematical topics
- Symbolic computation: calculus, linear algebra, statistics, differential equations with exact solutions
- Natural-language query interpretation — type a problem in plain English for computation
- Wolfram Language integration for programmable knowledge and computation

**Differentiating features**
- Most mathematically rigorous step-by-step solutions of any reviewed tool — exact symbolic computation, not numerical approximation
- Wolfram Problem Generator enables unlimited randomised practice on specific mathematical topics
- Coverage of advanced undergraduate and graduate STEM content — unique in the reviewed set
- LLM integration layer (WolframGPT plugin, ChatGPT plugin) enables AI assistants to call Wolfram for computation

**UX patterns**
- Text-based query interface; accepts natural language and formal mathematical notation
- Results page: computed answer + step-by-step derivation + related topics
- Pro users get additional steps, alternate forms, and downloadable outputs

**Integration points**
- Wolfram API for programmatic computation in third-party apps and AI assistants
- ChatGPT and other LLM plugin integrations for computational grounding
- Mathematica / Wolfram Language for programmatic access

**Known gaps**
- Not a tutoring system in the pedagogical sense — provides solutions, not guided learning
- No student model, progress tracking, or teacher dashboard
- UI is optimised for mathematically sophisticated users; not accessible for younger or struggling learners
- No gamification, engagement mechanics, or adaptive sequencing

**Licence / IP notes**
- Proprietary Wolfram commercial product; Pro subscription required for advanced step-by-step features

---

### Synthesis Tutor

**Core features**
- Game-based critical thinking and collaborative problem-solving for ages 7–14
- Adaptive challenge selection: difficulty calibrates in real-time based on performance
- Multi-player collaborative challenges: teams of students solve problems together
- Cognitive skills focus: systems thinking, optimisation, and logical reasoning — not curriculum-aligned drill
- Parent dashboard: session summaries, challenge completion, and skill development snapshots

**Differentiating features**
- Only reviewed AI tutoring platform focused exclusively on transferable cognitive skills rather than curriculum content mastery
- Multiplayer collaboration mechanic is unique — most AI tutors are individual, asynchronous experiences
- Originated as an internal programme for SpaceX employees' children — unconventional research backing
- High engagement reported: game-based format drives voluntary practice beyond assigned time

**UX patterns**
- Game-like challenge environment; learners experience it as play rather than study
- No traditional lesson or quiz format; all learning is embedded in game scenarios
- Post-session summary highlights cognitive skills exercised during the session

**Integration points**
- No LTI, LMS, or SIS integration — consumer-direct subscription
- Parent email reporting; no institutional integration

**Known gaps**
- No curriculum alignment (CCSS, state standards) — difficult to integrate into formal school assessment frameworks
- High price ($99/month) for a consumer product limits accessibility
- Limited independent research validating cognitive skill transfer to academic performance
- Age-limited to 7–14; not applicable to secondary or post-secondary audiences

**Licence / IP notes**
- Proprietary subscription product

---

### Cognii

**Core features**
- Conversational AI assessment using natural language: students type free-text answers and receive immediate formative feedback
- Virtual Learning Assistant (VLA): guides students through open-ended questions with Socratic dialogue
- Automated essay and short-answer grading with detailed feedback on reasoning quality
- Course-embedded assessment: deployed within existing LMS as an assignment activity
- Analytics: mastery estimates per learning objective based on open-ended response analysis

**Differentiating features**
- Natural-language open-ended response assessment — uniquely addresses the limitation of multiple-choice in measuring deep understanding
- Socratic dialogue in free-text conversation rather than structured hint sequences
- Automated rubric-based grading of short answers at scale — replaces or supplements human grading
- Higher-education and corporate training focus; complements rather than competes with K-12 platforms

**UX patterns**
- LMS-embedded activity: appears as a standard assignment in the existing course environment
- Student experience: text conversation with the VLA, receiving immediate feedback on each response
- Instructor dashboard: class-wide mastery estimates per objective with individual response review

**Integration points**
- LTI 1.3 for LMS embedding (Canvas, Blackboard, D2L, Moodle)
- xAPI for detailed interaction data export
- REST API for custom integrations

**Known gaps**
- Limited to text-based disciplines; not well-suited for mathematics or STEM problem-solving
- Smaller brand recognition than Khanmigo or Chegg in the AI tutoring market
- No consumer or K-12 offering; enterprise-only pricing limits accessibility
- No gamification or engagement mechanics for less motivated learners

**Licence / IP notes**
- Proprietary enterprise SaaS; no public pricing

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Step-by-step worked solution presentation for target subject matter
- Immediate formative feedback on each learner response
- Multi-step hint scaffolding that guides without revealing the final answer directly
- Mobile-accessible interface (app or responsive web) for on-demand use
- FERPA / COPPA compliance for K-12 data handling
- GDPR / CCPA compliance for learner PII management
- Learner session history and progress summary

### Differentiating Features
- Socratic questioning mode that withholds answers and guides reasoning (Khanmigo leads; Cognii addresses open-ended text)
- Fine-grained cognitive student knowledge model (400+ skills in MATHia — most granular reviewed)
- Real-time teacher visibility into individual student struggles during active sessions (MATHia LiveLab is unique)
- Camera-based problem capture for zero-friction question entry (Photomath and Socratic)
- Gamification and habit-forming engagement mechanics driving daily active use (Duolingo leads decisively)
- Free or near-free access model removing financial barrier for underserved learners (Khanmigo, Socratic, Photomath)
- Multiplayer collaborative problem-solving (Synthesis Tutor — unique in reviewed set)
- Natural-language open-ended response assessment (Cognii — unique in reviewed set)

### Underserved Areas / Opportunities
- Persistent longitudinal student knowledge model that spans grade levels, subjects, and institutions — no reviewed tool maintains this
- Multilingual Socratic tutoring without cost premium — free high-quality tutoring available only in English at scale
- Misconception-targeted remediation: identifying the specific conceptual error behind a wrong answer and generating a tailored counter-example
- Real-time teacher dashboard synthesising AI tutoring session data into actionable whole-class re-teaching signals
- Equitable access for English Language Learners — most platforms are English-only or English-primary
- Competency-mapped credential issuance based on AI tutoring session mastery data (not just completion)
- Honest academic integrity design: pedagogically scaffolded AI help that cannot be trivially misused for answer submission

### AI-Augmentation Candidates
- Socratic dialogue at scale: LLM-powered guided questioning that detects reasoning gaps and withholds answers for every student simultaneously
- Misconception detection: fine-tuned models identifying the specific conceptual error underlying each wrong answer and generating targeted remediation
- Persistent cross-session and cross-subject student model that accumulates knowledge state across years
- Multilingual tutoring: delivering equivalent-quality Socratic tutoring in any language without cost premium
- Real-time teacher alerts: synthesising dense tutoring session interaction data into actionable classroom re-teaching recommendations

---

## Legal & IP Summary

- **All reviewed platforms are proprietary:** No open-source AI tutoring system with comparable capability exists in the reviewed set; open-source LLMs (LLaMA, Mistral) provide a foundation but require substantial fine-tuning and safety infrastructure to reach production quality for K-12 use
- **FERPA:** AI tutoring session transcripts, interaction logs, and student knowledge model data are education records subject to FERPA when associated with an identifiable student in an institutional context; platforms must operate under school official agreements with districts
- **COPPA (2025 amendments):** Platforms serving users under 13 must implement verifiable parental consent for data collection; session transcript data is particularly sensitive; data retention must be minimised
- **GDPR / CCPA:** Session interaction data and student knowledge models are personal data requiring lawful basis for processing, subject access rights, and deletion capability on request
- **AI Ethics (UNESCO 2023 framework):** Transparency about AI involvement, student agency to opt out of AI responses, and teacher involvement in AI-informed decisions are emerging procurement requirements — particularly for publicly funded school deployments
- **Academic integrity:** Platforms that deliver answers rather than scaffolding (Photomath, Chegg, Socratic) face institutional bans and reputational risk; new platforms should design explicit academic integrity safeguards into the product
- **IP in LLM-generated tutoring content:** Regulatory clarity on copyright in AI-generated explanations and worked examples is still evolving; platforms should document their content generation processes

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Socratic dialogue engine: LLM-powered tutoring that poses guiding questions and withholds direct answers, configurable per institution for age-appropriateness
- Step-by-step worked solution mode: student-controlled reveal of solution steps with explanation at each step
- Subject coverage for initial target domain (e.g., K-12 mathematics) with standards-aligned content tagging
- Student session progress tracking: problems attempted, hints used, mastery signals, and time-on-task
- FERPA / COPPA compliant data model with parental consent workflow for under-13 users and audit-ready session logging
- Teacher dashboard: class-wide session activity, stuck-student alerts, and mastery-by-objective summary

**Should-have (v1.1)**:
- Misconception detection: classification of error type underlying wrong answers with targeted counter-example generation
- Persistent student knowledge model accumulating mastery signals across sessions and topics
- Multilingual tutoring interface: localised prompting and response generation for at least the top 5 learner languages in target markets
- Camera / image problem input for zero-friction question entry on mobile devices
- LTI 1.3 integration for LMS embedding with grade passback and IMS OneRoster roster import

**Nice-to-have (backlog)**:
- Real-time teacher alert system synthesising session data into actionable whole-class re-teaching recommendations
- Longitudinal cross-subject knowledge model tracking mastery across curriculum areas and academic years
- Gamification layer: streaks, challenges, and mastery milestones calibrated to support intrinsic motivation without answer-seeking shortcuts
- Competency-mapped credential issuance based on demonstrated AI tutoring session mastery rather than simple completion
- Multiplayer collaborative problem-solving mode for classroom group work facilitated by the AI tutor
