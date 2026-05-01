# AI Tutoring System

> Candidate #114 · Researched: 2026-05-01

## Existing Products and Software Packages

| Name | Description | Model | Pricing |
|------|-------------|-------|---------|
| Khanmigo (Khan Academy) | Socratic AI tutor built on GPT-4; guides K-12 students through math, science, and humanities via questioning rather than answer-giving; 1.4M users, 380+ district partners; $42M federal grant to deploy to 2.1M Title I students | Freemium + institutional | Free for individual learners; district partnerships |
| Carnegie Learning MATHia | AI-driven cognitive tutor for grades 6–12 math; based on ACT-R cognitive architecture research from CMU; step-by-step adaptive hints; $75M Series D (Sep 2025) | SaaS | Per-student/yr (~$20–30); district contracts |
| Duolingo (Max tier) | Language-learning platform with GPT-4-powered AI conversations, video calls, and personalised lessons; 113M MAU; ~$748M 2025 revenue | Freemium + subscription | Free; $168/yr (Super); ~$360/yr (Max) |
| Chegg (Chegg Study / CheggMate) | Homework help and AI tutoring platform for higher education; subject-matter Q&A with AI step-by-step explanations; facing headwinds from ChatGPT competition | SaaS subscription | $19.95–$29.95/month |
| Socratic (Google) | AI-powered homework help app for K-12; learners photograph a problem and receive step-by-step explanations with linked resources | Freemium | Free |
| Photomath (Google) | Math problem-solving app with step-by-step solutions via camera; acquired by Google 2023; 300M+ downloads | Freemium | Free; Plus subscription ~$9.99/month |
| Tutor.com / The Princeton Review | On-demand human tutoring with AI-assisted matching and session tools; blended human-AI model | Subscription + pay-per-session | From $39.99/month; per-session pricing |
| Wolfram Alpha / Wolfram Problem Generator | Computational knowledge engine with step-by-step STEM solutions; increasingly integrated with LLM-based tutoring interfaces | Freemium | Free basic; Pro $7.25/month |
| Synthesis (formerly SpaceX internal) | Collaborative problem-solving and critical thinking platform for ages 7–14; uses game-based adaptive challenges | Subscription | ~$99/month |
| Numerade | AI-powered STEM tutoring video platform with step-by-step problem solutions; targets college students | Subscription | $9.99/month |
| Cognii | Conversational AI tutoring using natural language assessment; targets higher education and corporate training | Enterprise SaaS | Enterprise contract |

## Relevant Industry Standards or Protocols

| Standard | Relevance |
|----------|-----------|
| IMS QTI 3.0 (Question and Test Interoperability) | Standard for portable assessment items; enables AI tutoring systems to import and export questions across platforms |
| xAPI (Experience API) | Captures fine-grained learner interaction data (hints requested, attempts, time-on-task) essential for tutoring analytics and adaptive sequencing |
| IMS LTI 1.3 | Enables AI tutoring tools to be embedded within LMS environments with SSO and grade passback |
| IEEE P2247 (Adaptive Instructional Systems) | Emerging standard defining reference architectures for adaptive and intelligent instructional systems |
| FERPA | Governs student data collected during tutoring sessions in K-12 and higher education contexts |
| COPPA | Applies to AI tutoring platforms serving children under 13; requires parental consent for data collection |
| WCAG 2.2 | Accessibility requirements for tutoring interfaces; particularly important for assistive technology compatibility |
| GDPR / CCPA | Data privacy regulations governing learner PII in EU and California; apply to session transcripts and behavioural data |
| AI Ethics Guidelines for Education (UNESCO 2023) | UNESCO framework on AI in education addressing transparency, fairness, student agency, and teacher involvement |
| Section 508 (US Rehabilitation Act) | Federal accessibility standard for US educational technology; required for federally funded school deployments |

## Available Research Materials

| Citation | Type |
|----------|------|
| Bloom, B. S. (1984). "The 2 Sigma Problem: The Search for Methods of Group Instruction as Effective as One-to-One Tutoring." *Educational Researcher*, 13(6), 4–16. | Foundational empirical study |
| VanLehn, K. (2011). "The Relative Effectiveness of Human Tutoring, Intelligent Tutoring Systems, and Other Tutoring Systems." *Educational Psychologist*, 46(4), 197–221. | Meta-analysis |
| Anderson, J. R., Corbett, A. T., Koedinger, K. R., & Pelletier, R. (1995). "Cognitive Tutors: Lessons Learned." *Journal of the Learning Sciences*, 4(2), 167–207. | Foundational peer-reviewed study |
| Corbett, A. T., & Anderson, J. R. (1994). "Knowledge tracing: Modeling the acquisition of procedural knowledge." *User Modeling and User-Adapted Interaction*, 4(4), 253–278. | Peer-reviewed journal article |
| Kulik, J. A., & Fletcher, J. D. (2016). "Effectiveness of Intelligent Tutoring Systems: A Meta-Analytic Review." *Review of Educational Research*, 86(1), 42–78. | Meta-analysis |
| Research Square. (2025). "Socratic AI in K–12 Science Classrooms: Effects on Critical Thinking, Motivation, and Self-Regulation in a Randomized Controlled Trial." rs-8118546. | Pre-print / RCT |
| Frontiers in Medicine. (2025). "Evaluation of the impact of AI-driven personalized learning platform on medical students' learning performance." *Frontiers in Medicine*, 12, 1610012. | Randomised controlled study |
| Tutor CoPilot. (2024). *A Human-AI Approach for Scaling Real-Time Tutoring*. EdWorkingPapers.com, 24-1054. | Working paper |
| Harvard University. (2024). *AI-powered tutor study: Students learned more than twice as much vs. active learning classrooms in 49 minutes vs. 60 minutes*. (Published in peer-reviewed venue 2024.) | Experimental study |
| Regional Lens. (2025). "From Tool to Tutor: Socratic AI Tutoring, Metacognitive Engagement, and Prior Knowledge as Determinants of Learning Gains in Gateway STEM Courses." *Regional Lens*, 3(1), 184. | Peer-reviewed journal article |

## Market Research

**Market Size:** The global AI tutoring market is valued at approximately $6.8 billion in 2025 and is forecast to reach $37.4 billion by 2034 at a CAGR of 19.5% (Mordor Intelligence / Future Market Insights). Alternative estimates place the 2025 figure at $3.7 billion, reflecting definitional differences in scope.

**Pricing Landscape:**

| Tier | Profile | Typical Cost |
|------|---------|-------------|
| Free AI tutoring | Khanmigo (Title I), Socratic, Photomath basic | Free |
| Consumer app subscription | Duolingo Max, Chegg, Photomath Plus | $10–$30/month |
| Premium consumer / family | Synthesis, Khanmigo family | $50–$100/month |
| Higher-ed / corporate SaaS | Cognii, Numerade, Chegg Study | $10–$30/month per learner |
| District/institutional | MATHia, Khanmigo district, Tutor.com | $20–$50/student/yr; bulk district contracts |
| On-demand human-AI blend | Tutor.com, Wyzant | $40–$80/hour (human tutor); monthly plans |

**Key Buyer Personas:**
- K-12 parents seeking affordable, on-demand tutoring alternatives to $50–$100/hr human tutors
- School districts using Title I funding to provide equitable access to AI tutoring at scale
- College students needing on-demand help with STEM coursework outside office hours
- Higher-education institutions supplementing teaching assistant capacity with AI tools
- Corporate L&D teams deploying skill-building tutors for technical upskilling and compliance training

**Notable Funding / Acquisitions:**
- Carnegie Learning raised $75 million Series D (September 2025, led by Owl Ventures) for MATHia international expansion
- Khan Academy partnered with the US Department of Education to deploy Khanmigo to 2.1 million Title I students funded by a $42 million federal grant
- Google acquired Photomath in 2023 (terms undisclosed) and Socratic previously, consolidating two major free AI homework help tools
- Chegg's stock declined ~90% from its 2021 peak by 2024 due to ChatGPT cannibalisation of its homework-help business, illustrating AI disruption risk even for incumbents
- Duolingo raised its 2025 annual forecast following ~30% stock surge driven by Duolingo Max (GPT-4) adoption

## AI-Native Opportunity

- **Socratic dialogue at scale:** Human tutors who employ Socratic questioning are the most effective but the scarcest resource; AI can deliver genuine Socratic dialogue — posing targeted questions, detecting reasoning gaps, and withholding answers — to every student simultaneously, directly addressing Bloom's 2 Sigma problem at a fraction of the cost of human tutoring.
- **Persistent student model across subjects and years:** Current tutoring tools maintain session-level or course-level student models; an AI-native system could maintain a longitudinal knowledge model that persists across grade levels, subjects, and even institutions — enabling truly personalised scaffolding based on a student's full learning history.
- **Misconception detection and targeted remediation:** LLMs fine-tuned on domain knowledge can identify the specific misconception underlying a wrong answer (not just that it is wrong), then generate targeted counter-examples or analogies that directly address that misconception — capability that requires significant expertise from human tutors.
- **Multilingual and culturally responsive tutoring:** AI can deliver high-quality tutoring in any language without the cost premium of bilingual human tutors, enabling equitable access for English Language Learners and underserved communities globally — a segment largely ignored by premium tutoring services.
- **Real-time teacher dashboards and early warning:** AI tutoring systems generate dense interaction data; an AI-native platform could synthesise this into actionable alerts for classroom teachers — identifying which concepts need re-teaching, which students are struggling silently, and which are ready to accelerate — turning the tutoring system into a feedback loop that improves whole-class instruction.
