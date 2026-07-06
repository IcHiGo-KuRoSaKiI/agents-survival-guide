# 15. Domain Playbooks: BFSI, Healthcare, Education, General Enterprise

The same architecture keeps winning independently in every regulated domain: **the LLM handles language and proposal; a deterministic, human-authored artifact (dosing protocol, policy engine, semantic model, escalation rule, licensed human) makes the consequential decision.** The FDA reached it, DeepMind reached it, banks reached it, an insurance claims bot reached it, a math tutor reached it. What differs per domain is which actions are adverse, who must sign, and which regulator is watching.

## Cross-domain rules

1. **Asymmetric autonomy is law, not preference:** agents may autonomously take positive/reversible/bounded actions (pay a claim, deflect a chat, draft a note, scaffold a hint); adverse actions (deny, diagnose, grade, restrict, trade) are human-gated, mandated in US healthcare UM (CMS + CA SB 1120), US credit (ECOA/Reg B), EU high-risk (AI Act Art. 14).
2. **Internal-facing first.** Nearly all successful BFSI GenAI is employee-facing copilots; customer-facing autonomy is rare and earned. Ship the copilot, collect eval data, then graduate scoped autonomy.
3. **Evals are your regulatory evidence.** Expert-graded eval suites gating each release (the Morgan Stanley pattern) double as SR 11-7 "ongoing monitoring," EU AI Act conformity evidence, and FDA lifecycle documentation. Build once, comply thrice.
4. **Expect vendor-claim vs reality gaps of ~10x** (scribes: claimed 1-3% error, independently measured 26%; support bots: claimed 67-76% resolution, case-studies 42-50%). Measure with ground truth in *your* deployment before scaling.
5. **Test with the assistance removed.** The PNAS education result generalizes: raw ChatGPT access boosted practice +48% and *cut* subsequent unassisted exam scores −17%; the scaffolded variant eliminated the harm. If your agent's users perform worse without it than they would have unaided, you built dependence, not capability.
6. **Named accountability + explainability wherever decisions touch people:** reason-code extraction for adverse decisions (checklist reasons that don't reflect the model's actual drivers are illegal in US credit), a named human owner, a model/agent inventory mapped to NIST AI RMF / ISO 42001 / local regimes.

## BFSI

**What works at scale:** internal copilots (JPMorgan LLM Suite: 200K employees in ~8 months; Goldman: ~46.5K; Morgan Stanley: 98% advisor-team adoption on RAG over research + meeting summarization *reviewed before CRM entry*). Autonomous-but-asymmetric claims: Lemonade's AI Jim settles ~40% of claims zero-touch, **AI approves and pays; denials route to humans.** Adjudicator-guidance (EvolutionIQ at 70% of top disability carriers) over autonomous adjudication.

**Cautionary datapoints:** Klarna's arc: "does the work of 700 agents" (Feb 2024) -> CEO admits it "went too far," quality dropped, humans rehired (May 2025); deflection metrics missed the quality regression. Bank of America's Erica, 3B+ interactions, is *deliberately not generative*: deterministic NLU + pre-approved responses. The most successful customer-facing banking "AI" is not an LLM; let that calibrate you.

**Regulatory map:** SR 11-7 (LLMs are in-scope models: inventory, tiered validation, effective challenge; compensating controls where full validation is impossible) · EU AI Act Annex III high-risk: creditworthiness (5b), life/health insurance pricing (5c), high-risk obligations from Aug 2026 (postponement proposed, not law) · FINRA 24-09 · India: RBI FREE-AI (board-approved AI policy, accountable executive, grievance->human review) + SEBI: you are *solely responsible* for third-party AI outputs · Insurance: NAIC model bulletin (24 states), Colorado SB 21-169 (annual fairness testing mandate).

**Patterns:** maker-checker (agent=maker, human=checker for material actions) · four-eyes on anything customer-impacting · per-agent identity + audit trail (Singapore IMDA framework) · reason codes for every adverse recommendation.

## Healthcare

**The scaled success is ambient scribes**: 92% of US health systems deploying/piloting; Kaiser's 7,260-physician study: >15,700 documentation hours saved, burnout measurably down. Why it works: **the clinician signs the note**, human-in-loop by design, liability intact. The counter-evidence to respect: independent testing found 26.3% note error rates (worst: documented exams that never happened); Whisper-based pipelines hallucinated medications. The signature is load-bearing.

**The FDA-endorsed architecture:** the only cleared LLM-enabled device (UpDoc, Dec 2025) was cleared *on condition* that the LLM only converses while a deterministic, provider-configured engine decides insulin dosing. Non-device CDS lane: show sources, no time-critical directives, clinician can independently review the basis.

**Patterns:** clinician-in-the-loop before chart entry · **abstain-and-escalate** (out-of-scope/urgent -> immediate human handoff; Hippocratic's voice agents hard-ban diagnosis/treatment advice and escalate chest pain mid-call) · supervisor-stack (Polaris: one conversational model + 21 specialist verifier models for meds/labs/escalation) · **"AI may approve, never deny"**: CMS MA rules + CA SB 1120 make this law for utilization management.

**Plumbing:** BAAs are configuration-dependent (zero-retention modes are approvals/flags, not defaults, verify per provider); de-identification is impractical for clinical free text, so production = BAA + zero-retention; Section 1557 nondiscrimination duties now cover AI decision-support tools.

## Education

**The convergent design:** answer-withholding Socratic scaffolding (Khanmigo; OpenAI Study Mode; Anthropic Learning Mode; Google LearnLM), and it's evidence-based, not ideology: the PNAS Turkey RCT (rule 5) shows raw answers damage learning while hints-not-answers preserves it.

**The Khanmigo math-error saga is the domain's tool-design lesson:** the fix for arithmetic errors was (a) a **calculator tool** so computation never rides on token prediction, (b) retrieval of human-authored exercises before responding, (c) an error-rate dashboard. Tool delegation + grounding + measurement, not better prompting.

**Where efficacy is proven:** well-engineered tutors beat active lecture (Harvard physics RCT, 2x learning); the World Bank Nigeria RCT: +0.31 SD in 6 weeks; **Tutor CoPilot** (+9pp mastery for students of the *weakest* tutors at ~$20/tutor/yr): AI-as-copilot compresses the quality distribution from below. Adoption runs far ahead of evidence (no Khanmigo RCT exists); be the vendor with the RCT.

**Safety for minors:** FERPA "school official" DPAs (no training on student data, deletion clauses); COPPA 2025 amendments (opt-in defaults); the teacher is the human-in-the-loop (every student interaction visible + high-risk alerts); companion-bot legislation (CA SB 243: disclosure to minors, break reminders) shapes the constraint set; exam scoring/proctoring/admissions are EU AI Act high-risk.

## General enterprise

The compliance-grade checklist (works everywhere, mandatory in the domains above):

1. Per-agent identity, scoped short-lived credentials, end-user entitlements (Ch 13).
2. Deterministic action gate at the tool boundary; default-deny (Ch 13).
3. Lethal-trifecta review at design time; human/architectural break when all three coexist.
4. Curated interfaces over raw power (semantic layers, verified queries, workflow tools: Ch 14, 03).
5. Layered classifiers as defense-in-depth (Ch 13 rule 6).
6. Asymmetric HITL with durable approvals; maker-checker for material actions.
7. Dual audit artifacts: decision provenance (OTel) + immutable audit logs.
8. Expert-graded eval gates on every model/prompt/tool change; incident-replay regression packs; staged rollout (offline -> sandbox -> shadow -> canary) with a kill switch.
9. Ground-truth quality dashboards (domain error metrics), not self-grades; assume vendor claims overstate.
10. Named owner; reason codes for adverse decisions; inventory mapped to NIST AI RMF / ISO 42001 / EU AI Act evidence.

## Sources
- https://arxiv.org/pdf/2503.15668 (SR 11-7 for GenAI) · https://artificialintelligenceact.eu/annex/3/ · https://rbidocs.rbi.org.in/rdocs/PublicationReport/Pdfs/FREEAIR130820250A24FF2D4578453F824C72ED9F5D5851.PDF
- https://openai.com/index/morgan-stanley/ · https://www.lemonade.com/blog/lemonade-sets-new-world-record/ · https://www.bloomberg.com/news/articles/2025-05-08/klarna-turns-from-ai-to-real-person-customer-service
- https://innolitics.com/articles/updoc-fda-cleared-ai-agent/ · https://catalyst.nejm.org/doi/abs/10.1056/CAT.25.0040 · https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2025.1691499/full (scribe error rates) · https://hippocraticai.com/polaris-3/
- https://www.fenwick.com/insights/publications/californias-sb-1120-regulates-ai-in-health-plan-utilization-review-and-management-activities-starting-in-january · https://files.consumerfinance.gov/f/documents/cfpb_adverse_action_notice_circular_2023-09.pdf
- https://www.pnas.org/doi/10.1073/pnas.2422633122 (PNAS RCT) · https://blog.khanacademy.org/khanmigo-math-computation-and-tutoring-updates/ · https://arxiv.org/abs/2410.03017 (Tutor CoPilot) · https://blogs.worldbank.org/en/education/From-chalkboards-to-chatbots-Transforming-learning-in-Nigeria
- https://openai.com/index/practices-for-governing-agentic-ai-systems/ · https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents · https://saif.google/focus-on-agents
