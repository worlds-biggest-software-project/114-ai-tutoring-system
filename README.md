# AI Tutoring System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native Socratic tutoring platform for K-12 and higher education that guides learners through reasoning rather than handing them answers.

The AI Tutoring System is a candidate open-source platform that delivers scaffolded, dialogue-based tutoring to learners across K-12 and higher education. It is aimed at students, parents, teachers, and institutions who need affordable, equitable alternatives to scarce human tutors and to answer-vending homework apps. The core problem it addresses is Bloom's "2 Sigma" gap: one-to-one Socratic tutoring is the most effective form of instruction yet remains the least accessible.

---

## Why AI Tutoring System?

- Every reviewed competitor is proprietary; no open-source AI tutoring system with comparable capability exists, leaving institutions dependent on closed platforms such as Khanmigo, MATHia, Duolingo Max, and Chegg.
- Incumbents that deliver direct answers (Photomath, Chegg, Socratic) facilitate academic dishonesty and face institutional bans, while truly Socratic alternatives (Khanmigo) are limited to a single vendor's curriculum and brand.
- Pricing creates an equity gap: MATHia at $20–$30/student/yr, Synthesis at $99/month, and human tutors at $50–$100/hour are out of reach for many learners, while free tools like Socratic and Photomath provide no scaffolding or progress tracking.
- No reviewed platform maintains a longitudinal student knowledge model that persists across grade levels, subjects, or institutions — student history is fragmented across vendors.
- Multilingual, culturally responsive tutoring at parity with English remains underserved, leaving English Language Learners and underserved global communities without equivalent quality.

---

## Key Features

### Socratic Dialogue & Scaffolding

- LLM-powered Socratic engine that poses guiding questions and withholds direct answers
- Multi-step hint scaffolding configurable for age-appropriateness per institution
- Step-by-step worked solution mode with student-controlled reveal of each step
- Immediate formative feedback on each learner response
- Natural-language open-ended response assessment for free-text disciplines

### Student Modelling & Misconception Detection

- Fine-grained per-skill mastery tracking informed by cognitive tutor research
- Misconception detection that classifies the conceptual error behind a wrong answer and generates targeted counter-examples
- Persistent longitudinal student knowledge model accumulating mastery signals across sessions, topics, subjects, and academic years
- Session progress tracking covering problems attempted, hints used, mastery signals, and time-on-task

### Teacher Tools & Classroom Visibility

- Teacher dashboard surfacing class-wide session activity, stuck-student alerts, and mastery-by-objective summaries
- Real-time alerting that synthesises tutoring session data into actionable whole-class re-teaching recommendations
- Audit-ready session logging and assignment-level views suitable for blended classroom use

### Access, Inclusion & Engagement

- Multilingual tutoring interface with localised prompting and response generation for major learner languages
- Camera and image problem input for zero-friction question entry on mobile devices
- Gamification calibrated to support intrinsic motivation without encouraging answer-seeking shortcuts
- Multiplayer collaborative problem-solving mode for classroom group work facilitated by the AI tutor

### Compliance & Integrity

- FERPA and COPPA compliant data model with parental consent workflow for under-13 users
- GDPR / CCPA aligned handling of session interaction data and student knowledge models
- Academic integrity safeguards designed into the product to prevent trivial misuse for answer submission
- Transparency, opt-out, and teacher-in-the-loop controls aligned with the UNESCO 2023 AI in Education framework

---

## AI-Native Advantage

LLMs make it possible to deliver genuine Socratic dialogue — targeted questioning, reasoning-gap detection, and answer withholding — to every student simultaneously, addressing Bloom's 2 Sigma problem at a fraction of human-tutor cost. Fine-tuned models can identify the specific misconception behind a wrong answer and generate tailored counter-examples, a capability that previously required expert human tutors. AI further enables high-quality multilingual tutoring without the bilingual-tutor cost premium, and synthesises dense interaction data into real-time teacher alerts that turn the tutoring system into a feedback loop for whole-class instruction.

---

## Tech Stack & Deployment

The platform is intended to support institutional deployment alongside consumer access, with LTI 1.3 for LMS embedding and grade passback, IMS OneRoster for roster import, and IMS QTI 3.0 for portable assessment items. xAPI is used for fine-grained learner interaction data feeding tutoring analytics and adaptive sequencing. The architecture aligns with the emerging IEEE P2247 reference for adaptive instructional systems and targets WCAG 2.2 and Section 508 accessibility. Open-source LLMs (such as LLaMA or Mistral) provide a foundation for the Socratic engine, with fine-tuning and safety infrastructure layered on top for K-12 use.

---

## Market Context

The global AI tutoring market is valued at approximately $6.8 billion in 2025 and is forecast to reach $37.4 billion by 2034 at a CAGR of 19.5% (Mordor Intelligence / Future Market Insights), with alternative estimates placing the 2025 figure at $3.7 billion. Pricing spans free institutional tools (Khanmigo on a $42M federal grant to 2.1M Title I students), consumer subscriptions ($10–$30/month for Duolingo Max, Chegg, Photomath Plus), district contracts ($20–$50/student/yr for MATHia and similar), and human-AI blended services ($40–$80/hour). Primary buyers include K-12 parents, school districts deploying Title I funding, college students, higher-education institutions supplementing TA capacity, and corporate L&D teams.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
