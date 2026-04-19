# Insurance Claims Copilot — RAG-Based Decision Support Platform

**Role:** Product Manager · **Domain:** Insurance Operations & AI/ML · **Stage:** 0→1 Build

---

## TL;DR

Insurance operations teams at a mid-size general insurer were spending an average of **18 minutes per claim query** navigating fragmented policy documents, SOPs, and legacy portals. I led the 0→1 build of an internal AI copilot — a RAG-based decision support tool — that reduced average handling time by **~40%** and improved decision consistency across adjuster cohorts by **~30%**, measured against a senior-reviewer benchmark.

---

## Context & Background

General insurance operations involve high-volume, knowledge-intensive decisions: claims adjudication, policy servicing queries, and coverage interpretations. These decisions require agents to cross-reference multiple sources — policy wordings, regulatory circulars, internal SOPs, and historical precedents — often under time pressure.

The insurer operated with a workforce of ~200 operations agents across claims and servicing functions. New agents took 6–8 weeks to reach proficiency, and even experienced adjusters escalated ~25% of non-standard queries to senior SMEs, creating bottlenecks that slowed SLA adherence.

**The core problem was not lack of data — it was retrieval friction.** The right information existed; finding it, interpreting it correctly, and applying it consistently was the challenge.

---

## My Role

I was the sole PM on this initiative, embedded with a cross-functional team of 2 ML engineers, 1 backend engineer, 1 data engineer, and a domain SME from the claims vertical. I owned:

- Problem discovery and user research
- Product requirements (PRD, user stories, acceptance criteria)
- Prioritisation framework for MVP vs Phase 2 scope
- Metrics definition and success criteria
- Stakeholder alignment with Ops leadership and Compliance
- Go-to-market and change management strategy

---

## Discovery & Problem Framing

### User Research

I conducted **12 structured interviews** across three agent cohorts — claims adjusters, policy servicing agents, and customer support reps — and shadowed 5 live sessions to observe actual workflows.

Key findings:

- Agents averaged **4–6 application switches** per query resolution
- ~60% of escalations to SMEs were for queries that had clear policy answers, but agents lacked confidence in their interpretation
- New joiners cited "not knowing where to look" as their #1 productivity barrier
- Agents trusted answers more when they could see the source — but existing tools gave no citations

### Problem Statement (refined)

> *Operations agents cannot quickly retrieve, interpret, and apply the right policy information in context — leading to slow, inconsistent, and effort-heavy decisions that scale poorly with team growth.*

---

## Users & Personas

### Persona 1 — Claims Adjuster (Primary)

| Attribute | Detail |
|---|---|
| **Task frequency** | 30–50 claim touchpoints/day |
| **Primary friction** | Interpreting ambiguous policy clauses under time pressure |
| **Decision risk** | High — errors have financial and regulatory consequences |
| **Success metric** | Decision accuracy vs SME benchmark; AHT (average handling time) |

### Persona 2 — Policy Servicing Agent

| Attribute | Detail |
|---|---|
| **Task frequency** | 40–60 customer queries/day |
| **Primary friction** | Context-switching between 3–4 systems to find policy details |
| **Decision risk** | Medium — incorrect responses affect customer trust and NPS |
| **Success metric** | First-contact resolution rate; response consistency |

### Persona 3 — New Joiner (Secondary)

| Attribute | Detail |
|---|---|
| **Task frequency** | Ramp-up phase (weeks 1–8) |
| **Primary friction** | No institutional knowledge; over-reliant on peers |
| **Decision risk** | High during ramp — likely to escalate or err |
| **Success metric** | Time to proficiency; escalation rate vs cohort baseline |

---

## Solution Design

### Approach: RAG-Based Decision Copilot

After evaluating options (fine-tuned LLM, rule-based expert system, knowledge base search), I recommended a **Retrieval-Augmented Generation (RAG)** architecture for three reasons:

1. **Knowledge currency**: Policy documents update frequently; fine-tuning would require re-training cycles
2. **Explainability**: RAG outputs can cite source chunks — critical for compliance and agent trust
3. **Hallucination control**: Grounding generation in retrieved documents reduces fabrication risk

The copilot was designed as a **decision-support assistant, not an automation engine** — a deliberate choice given the financial and regulatory risk of autonomous claims decisions.

### How It Works

```
Agent query
    ↓
Embedding model (semantic vectorisation)
    ↓
Vector store retrieval (Top-K relevant chunks: policy clauses, SOPs, precedents)
    ↓
Context injection into LLM prompt
    ↓
Structured output generation:
    • Recommended action
    • Reasoning (step-by-step)
    • Source citations (document + page)
    • Confidence score (0–100)
    ↓
Guardrails layer (compliance rules, output validation)
    ↓
Confidence-based routing:
    High (>80)  → Agent reviews & approves
    Medium (50–80) → Agent reviews with flagged uncertainty
    Low (<50)   → Escalated to SME queue
    ↓
Feedback capture → Logged for retrieval retraining
```

### Key Product Decisions & Trade-offs

**1. Assistive vs Automated**

Chose human-in-the-loop design despite ops leadership's preference for full automation. Rationale: insurance decisions carry regulatory liability; errors in automated outputs could trigger compliance incidents. Framed automation as Phase 2 after trust is established through measured accuracy data.

**2. Structured output vs chat interface**

Early prototypes used a chat UI. User testing showed adjusters found open-ended responses harder to act on than structured outputs. Switched to a fixed output template (action / reasoning / citations / confidence) that could also be logged into the claims management system as an audit trail.

**3. Confidence scoring**

Initial v1 used model-generated probability scores — these were poorly calibrated and eroded agent trust when scores seemed arbitrary. Re-engineered confidence as a composite of: retrieval relevance score, output validation against business rules, and citation coverage. This made scores explainable and predictable.

**4. Build vs integrate**

Chose to surface the copilot as an embedded panel within the existing claims portal rather than a standalone tool — reducing context switching and increasing adoption probability. Integration required 3 additional engineering sprints but drove significantly higher daily active usage.

---

## Metrics & Outcomes

| Metric | Baseline | Post-Launch (90 days) | Change |
|---|---|---|---|
| Average handling time (claims query) | ~18 min | ~11 min | **↓ 39%** |
| Decision accuracy (vs SME benchmark) | 71% | 92% | **↑ 29 pp** |
| SME escalation rate | 25% | 14% | **↓ 44%** |
| New joiner time-to-proficiency | 6–8 weeks | ~4 weeks | **↓ ~40%** |
| Agent-reported confidence in decisions | 58% (survey) | 81% (survey) | **↑ 23 pp** |
| Copilot daily active usage (DAU) | — | 78% of eligible agents | — |

**Measurement approach:**
- AHT measured via CRM timestamps (pre/post cohort comparison)
- Decision accuracy: random sampling of AI-assisted decisions reviewed by senior adjusters blind to AI recommendation
- Escalation rate: ops team dashboards
- Proficiency: manager assessment at 4-week and 8-week milestones

---

## Challenges & What I Learned

### Retrieval quality is the product

The first iteration had a poor chunking strategy — whole policy documents were ingested as single chunks. Retrieved context was too broad, and LLM outputs were vague. Re-chunking at the clause level with metadata tags (product line, coverage type, effective date) dramatically improved precision. **Lesson: in RAG systems, data engineering is product work.**

### Confidence calibration is an ongoing problem

Early confidence scores were model-native and uncalibrated. Agents quickly learned to distrust them after seeing a "high confidence" output that was factually incorrect. Rebuilding trust took two sprints of recalibration and a communications effort with the ops team. **Lesson: a broken confidence signal is worse than no signal.**

### Change management was underestimated

Ops agents were initially resistant — framing the tool as "AI making decisions" triggered concern about job displacement. I reframed it explicitly as "your research assistant" in all comms and training. Adoption accelerated after a few high-profile cases where the copilot surfaced a policy clause an experienced adjuster had missed. **Lesson: adoption is a product problem, not just a training problem.**

### Integration took longer than expected

Embedding into the claims portal required negotiating with an internal platform team that had its own roadmap. Added ~3 weeks of delay. **Lesson: internal integrations need to be scoped into the project plan as first-class dependencies, not assumptions.**

---

## What I'd Do Differently

- **Earlier retrieval testing:** I spent too long on feature specs before validating retrieval quality. A retrieval evaluation framework (Recall@K, MRR) should have been set up in week 1.
- **Tighter feedback loop:** User feedback was collected via a post-session survey — too passive. A thumbs up/down inline rating would have generated 5× more signal per agent session.
- **Phased confidence rollout:** Introducing confidence scores at launch was premature. Would have launched without them, validated output quality first, then introduced scoring once the model was calibrated.

---



## Skills Demonstrated

`Product Discovery` · `User Research` · `0→1 Product Build` · `AI/ML Product Management` · `RAG Architecture` · `Insurance Domain` · `Stakeholder Management` · `Metrics Design` · `Change Management`
